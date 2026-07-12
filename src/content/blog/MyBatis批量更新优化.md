---
title: 'MyBatis / MyBatis-Plus 批量更新优化'
description: 'MyBatis / MyBatis-Plus 在 MySQL 场景下的批量更新优化方案'
pubDate: '2026-06-09'
---

# 1. 批量更新为什么比批量插入更复杂

上一篇文章已经聊过批量插入的优化思路，核心是尽量减少数据库交互次数，让多条 `INSERT` 尽可能合并执行。

但是到了批量更新场景，事情就没那么简单了。

原因很直接：
- 批量插入时，很多记录的 SQL 结构天然一致，只是 `values` 不同
- 批量更新时，不同行可能更新不同字段，也可能更新成不同值
- 所以“批量更新”并不是一个固定操作，而是一组不同形态的更新方案

也就是说，批量更新的关键不是先问“用哪个 API”，而是先问下面这两个问题：

1. 这些记录是不是更新成同一个值
2. 这些记录更新的字段集合是不是一致

这两个问题基本决定了你后面该选哪种方案。

# 2. 常用的批量更新方案

在 `Java + MyBatis / MyBatis-Plus + MySQL` 里，常见的批量更新方案主要有 5 类：

|方案|适用场景|核心特点|
|---|---|---|
|`UPDATE ... WHERE id IN (...)`|多条记录更新成相同值|最简单，通常优先考虑|
|`CASE WHEN` 批量更新|多条记录更新成不同值|单条 SQL 完成多行不同值更新|
|`updateBatchById` / `ExecutorType.BATCH`|想快速接入批处理，改造成本低|本质是多条 `UPDATE` 批量发送|
|`INSERT ... ON DUPLICATE KEY UPDATE`|存在则更新，不存在则插入|适合导入、同步、回写|
|临时表 + `JOIN UPDATE`|超大批量更新|数据库侧方案，稳定且可扩展|

如果只记一句话，那就是：

**同值更新优先 `WHERE IN`，不同值更新优先 `CASE WHEN`，数据量特别大优先“临时表 + JOIN UPDATE”。**

# 3. 第一类：多条记录更新成同一个值

这是最常见、也最容易被忽视的批量更新场景。

比如：
- 批量把订单状态改成“已取消”
- 批量把用户账号标记为“冻结”
- 批量把一批数据的 `deleted` 字段改成 `1`

这类场景里，虽然更新的是很多行，但**每一行改成的值都一样**。这时候最合适的方案，通常就是一条普通的 `UPDATE` 配合 `IN` 条件。

```sql
UPDATE user
SET status = 1
WHERE id IN (101, 102, 103, 104);
```

## 3.1 在 MyBatis-Plus 中怎么写

```java
LambdaUpdateWrapper<User> wrapper = Wrappers.lambdaUpdate(User.class)
        .set(User::getStatus, 1)
        .in(User::getId, ids);

userService.update(wrapper);
```

## 3.2 这种方式为什么值得优先考虑

- SQL 最简单，可读性最高
- 只有一条更新语句，网络往返少
- 数据库只需要解析一次 SQL
- 不需要循环调用 `updateById`

## 3.3 它的边界是什么

- 只适合“同一批数据更新成相同值”
- `IN` 列表不能无限大，数据量太大时 SQL 会很长，锁持有时间也会变长
- 实际项目里通常要按批次拆分，比如每批 `500`、`1000` 或 `2000`

如果你的场景满足这一类，不要急着上复杂方案，这通常就是性价比最高的写法。

# 4. 第二类：多条记录更新成不同值

这是批量更新里最典型、也最容易让人纠结的场景。

比如：
- 给不同用户设置不同积分
- 给不同订单回写不同金额
- 给不同商品设置不同价格

这时候已经不能用简单的 `WHERE IN` 了，因为每一行要更新成的值都不一样。

比较经典的做法是使用 `CASE WHEN`。

```sql
UPDATE user_score
SET score = CASE id
    WHEN 1 THEN 100
    WHEN 2 THEN 80
    WHEN 3 THEN 95
END
WHERE id IN (1, 2, 3);
```

如果要同时更新多个字段，也可以继续写多个 `CASE`：

```sql
UPDATE user_score
SET score = CASE id
    WHEN 1 THEN 100
    WHEN 2 THEN 80
    WHEN 3 THEN 95
END,
level = CASE id
    WHEN 1 THEN 'A'
    WHEN 2 THEN 'B'
    WHEN 3 THEN 'A'
END
WHERE id IN (1, 2, 3);
```

## 4.1 在 MyBatis XML 中怎么组织

```xml
<update id="batchUpdateUserScore">
    UPDATE user_score
    SET
    score = CASE id
        <foreach collection="list" item="item">
            WHEN #{item.id} THEN #{item.score}
        </foreach>
    END,
    level = CASE id
        <foreach collection="list" item="item">
            WHEN #{item.id} THEN #{item.level}
        </foreach>
    END
    WHERE id IN
    <foreach collection="list" item="item" open="(" separator="," close=")">
        #{item.id}
    </foreach>
</update>
```

## 4.2 这种方式的优点

- 真正只执行一条 `UPDATE`
- 网络交互少
- 很适合“多行不同值”的更新

## 4.3 什么时候不适合

- 每条记录更新的字段集合差异很大
- 一次更新的数据量太大，SQL 会非常长
- 业务字段很多时，XML 会比较难维护

所以 `CASE WHEN` 最适合的场景通常是：

- 更新字段相对固定
- 每行值不同
- 数据量中等

比如几百条到几千条，往往是比较舒服的区间。再往上走，就要开始考虑临时表方案了。

# 5. 第三类：Java 层批处理

很多项目里，第一反应会直接用：

- `MyBatis-Plus` 的 `updateBatchById`
- MyBatis 的 `ExecutorType.BATCH`

这两个方案当然可以用，但一定要先理解一件事：

**它们不等于“自动帮你拼成一条大 UPDATE SQL”。**

它们的本质更接近：

- Java 层循环调用多次 `update`
- JDBC / MyBatis 以批处理方式统一 `flush`

也就是说，这类方案的优化重点是“减少多次单独执行的开销”，但它本质上还是**多条更新语句**，并不是 `CASE WHEN` 那种单条 SQL。

## 5.1 MyBatis-Plus 的 `updateBatchById`

```java
userService.updateBatchById(userList, 500);
```

这一类写法的优点是接入成本很低，代码也简单。

但是它更适合下面这些场景：
- 业务上已经是按实体逐条更新
- 不想自己写复杂 XML
- 更新量有一些，但还没有大到必须手写 SQL

### 为什么说它不是单条大 SQL

MyBatis-Plus 官方文档给出的 `updateBatchById` 示例，生成的 SQL 本身就是多条 `UPDATE`。另外，官方批量操作文档还专门提到，批量结果是按 `MappedStatement + SQL` 分组的。

这意味着：
- 如果 10 条数据生成的 SQL 完全一样，只是参数不同，批处理效果更好
- 如果 10 条数据里，有些更新 1 个字段，有些更新 2 个字段，那么 SQL 形状就不一致，批处理会被拆成多组

这也是为什么在实际项目里，**实体字段更新得越“整齐”，批处理效果通常越好。**

## 5.2 原生 MyBatis 的 `ExecutorType.BATCH`

如果你不用 MyBatis-Plus，也可以直接用 MyBatis 的批处理执行器。

```java
try (SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH, false)) {
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    int batchSize = 500;

    for (int i = 0; i < userList.size(); i++) {
        userMapper.updateById(userList.get(i));

        if ((i + 1) % batchSize == 0 || i == userList.size() - 1) {
            sqlSession.flushStatements();
            sqlSession.commit();
            sqlSession.clearCache();
        }
    }
}
```

## 5.3 把 `updateBatchById` 的执行链路拆开看

这个过程如果只用一句话概括，很容易越讲越混。

所以更稳妥的方式，是把它拆成 4 层看：

1. `MyBatis-Plus` 层：遍历实体，反复调用 `updateById`
2. `MyBatis` 层：使用 `ExecutorType.BATCH`，先缓存这批更新
3. `JDBC` 层：在合适的时机调用 `executeBatch()`
4. `MySQL 驱动` 层：决定这批 `UPDATE` 最后怎么发给 MySQL

可以把 `updateBatchById` 理解成下面这个近似过程：

```text
for (T entity : entityList) {
    sqlSession.update("xxx.updateById", entity);
    if (达到 batchSize) {
        sqlSession.flushStatements();
    }
}
最后再 flushStatements();
```

这个近似过程说明了两个关键点：

- `updateBatchById` 的“批量”，首先来自 **MyBatis / JDBC 的 batch 机制**
- 它的 SQL 语义仍然是 **多条 `UPDATE`**，不是单条批量更Dddd新 SQL

所以你不能把它理解成：

```sql
UPDATE user
SET email = CASE id
    WHEN 1 THEN 'a@example.com'
    WHEN 2 THEN 'b@example.com'
END
WHERE id IN (1, 2);
```

它更接近的是：

```sql
UPDATE user SET email = ? WHERE id = ?;
UPDATE user SET email = ? WHERE id = ?;
UPDATE user SET email = ? WHERE id = ?;
```

只是这些 `UPDATE` 并不是每条都立刻执行，而是先进入 batch，再统一提交。

## 5.4 它和自己写 `for updateById` 的区别到底在哪

这个点非常重要，因为很多人会觉得：

- 反正底层也是循环
- 那我自己 `for` 一下 `updateById` 不就一样了

表面上看，两者都在循环；但差别在于：**是不是每次循环都立刻执行数据库操作。**

### 自己写循环

```java
for (User user : userList) {
    userService.updateById(user);
}
```

在默认情况下，这更接近：

- 调一次 `updateById`
- 立刻执行一次 `UPDATE`
- 循环多少次，就执行多少次

也就是说，它通常是：

- N 次业务调用
- N 次数据库执行

### 使用 `updateBatchById`

```java
userService.updateBatchById(userList, 500);
```

它更接近：

- Java 层仍然遍历 N 条数据
- 但不是每次都立刻发数据库
- 而是先攒成 batch
- 达到 `batchSize` 或最终 `flush` 时再统一执行

所以更准确的理解应该是：

- `for updateById`：**N 次循环，通常也是 N 次立即执行**
- `updateBatchById`：**N 次逻辑 update，但只有较少次数的 batch 提交**

这也是为什么两者虽然最终都是多条 `UPDATE`，但执行效率通常差很多。

## 5.5 这一类方案适合什么场景

- 你已经有成熟的 `updateById` / `mapper.update(...)` 逻辑
- 改 SQL 成本高，不想手写复杂动态更新语句
- 更看重开发效率，而不是极致 SQL 形态

## 5.6 这一类方案的局限

- 本质不是单条 SQL
- 对网络和数据库解析次数的优化，不如 `CASE WHEN` 或 `JOIN UPDATE` 直接
- 如果更新字段集合不一致，批处理效果会继续打折

所以它适合“快速落地”，但不一定适合“性能极致优化”。

# 6. 第四类：存在则更新，不存在则插入

有些业务嘴上说的是“批量更新”，但实际需求并不是纯更新，而是：

- 有就更新
- 没有就新增

这时候更合适的往往不是 `UPDATE`，而是 `INSERT ... ON DUPLICATE KEY UPDATE`。

比如同步用户标签、导入商品价格、回写第三方数据时，经常就是这种模式。

```sql
INSERT INTO user_score (id, score, level)
VALUES
    (1, 100, 'A'),
    (2, 80, 'B'),
    (3, 95, 'A') AS new
ON DUPLICATE KEY UPDATE
    score = new.score,
    level = new.level;
```

## 6.1 它适合什么场景

- 主键或唯一键已确定
- 希望一条语句同时处理“新增 + 更新”
- 数据同步、导入、幂等写入

## 6.2 它的优点

- 对业务很直接
- 省掉“先查再决定插入还是更新”
- 对批量导入类场景很实用

## 6.3 它的注意点

- 它本质上不是纯 `UPDATE`，而是 `UPSERT`
- 依赖主键或唯一索引
- 如果表上有多个唯一索引，行为边界要特别小心
- MySQL 新版本里，`ON DUPLICATE KEY UPDATE` 更推荐使用行别名 / 列别名，而不是旧的 `VALUES()` 写法

如果你的业务其实是“同步数据”，这一类方案往往比单纯的批量更新更顺手。

# 7. 第五类：超大批量更新

当更新量再往上走，比如几万、几十万，甚至更多时，前面的方案都会逐渐暴露问题：

- `IN` 太长
- `CASE WHEN` 太长
- Java 层批处理提交次数多
- 锁持有时间长

这时候更稳妥的方案通常是：

**先把待更新数据写入临时表，再通过 `JOIN UPDATE` 回写目标表。**

```sql
CREATE TEMPORARY TABLE tmp_user_score (
    id BIGINT PRIMARY KEY,
    score INT NOT NULL,
    level VARCHAR(16) NOT NULL
);

INSERT INTO tmp_user_score (id, score, level)
VALUES
    (1, 100, 'A'),
    (2, 80, 'B'),
    (3, 95, 'A');

UPDATE user_score u
JOIN tmp_user_score t ON u.id = t.id
SET
    u.score = t.score,
    u.level = t.level;
```

## 7.1 这种方案为什么适合大数据量

- 更新值先落到一张结构化临时表里，SQL 不会无限膨胀
- `UPDATE` 语句本身保持简洁
- 更容易分批导入和分批回写
- 对几十万级数据更友好

## 7.2 它的工程形态通常是什么

一般流程是：

1. Java 侧把待更新数据整理成 `id + 目标值`
2. 批量写入临时表或中间表
3. 执行 `JOIN UPDATE`
4. 清理临时表

如果是离线任务、数据修复、导入回写，这类方案通常是最稳的。

## 7.3 它的代价是什么

- 实现比前几种方案复杂
- 需要额外的临时表或中间表管理
- 更适合离线任务、后台任务，不一定适合所有在线接口

# 8. `rewriteBatchedStatements` 在批量更新里怎么理解

这个点非常容易误解，因为很多人会把“批量插入”的经验直接套到“批量更新”里。

上一篇讲批量插入时，`rewriteBatchedStatements=true` 非常重要，这没有问题。

但如果到了批量更新场景，一定要先记住一件事：

**`rewriteBatchedStatements` 是 MySQL JDBC 驱动参数，不是 MyBatis-Plus 参数。**

它影响的是：

- **JDBC batch 最后怎么发给 MySQL**

它不影响的是：

- **MyBatis-Plus 怎样生成 `updateById` 这批 SQL**

所以它解决的不是“SQL 生成形态”问题，而是“驱动发送策略”问题。

## 8.1 不开启时，发生了什么

假设这一批数据生成的 SQL 形状一致，例如都是：

```sql
UPDATE user SET email = ? WHERE id = ?
```

如果 `rewriteBatchedStatements=false`，你可以把它理解成：

- `updateBatchById` 先把这批更新交给 MyBatis Batch
- MyBatis 再把这批更新交给 JDBC `executeBatch()`
- 驱动按 batch 方式执行这批 `UPDATE`
- **但不会把它们改写成一条大 SQL 文本**

也就是说，这时候更接近：

```text
SQL模板:
UPDATE user SET email = ? WHERE id = ?

参数批:
('a@example.com', 1)
('b@example.com', 2)
('c@example.com', 3)
...
```

注意，这里仍然不是“单条 SQL 更新多行”，而是：

**同一条 SQL 模板 + 多组参数 + 一次 batch 执行。**

## 8.2 开启时，发生了什么

如果 `rewriteBatchedStatements=true`，对批量更新来说，更容易让人误解。

一定不要把它理解成：

```sql
UPDATE user
SET email = CASE id
    WHEN 1 THEN 'a@example.com'
    WHEN 2 THEN 'b@example.com'
END
WHERE id IN (1, 2);
```

这不是驱动帮你做的事情。

根据 MySQL Connector/J 官方文档，`rewriteBatchedStatements` 对 `PreparedStatement` 的重写重点，是：

- `INSERT`
- `REPLACE`

也就是说，它对批量插入、批量 upsert 的帮助更直接。

而对 `UPDATE` 来说，更稳妥的理解是：

- 它不会把多条 `UPDATE` 自动改写成单条批量更新 SQL
- 它可能影响驱动如何执行和发送这一批语句
- 但具体发送形态更接近驱动实现细节，不建议把它当作稳定行为去依赖

所以这里最重要的结论不是“最终一定会变成什么文本”，而是：

- `rewriteBatchedStatements=true` **不会**把 `updateBatchById` 变成单条 `CASE WHEN` 风格更新
- `updateBatchById` 的本质仍然是 **多条 `UPDATE` 的批处理**

## 8.3 开启与不开启，到底差在哪

可以直接压缩成这张表：

|开关|MyBatis-Plus 生成的 SQL 语义|驱动层处理方式|会不会变成单条批量 `UPDATE`|
|---|---|---|---|
|`rewriteBatchedStatements=false`|多条 `UPDATE`|按 JDBC batch 执行|不会|
|`rewriteBatchedStatements=true`|多条 `UPDATE`|驱动可能进一步重写发送方式|不会|

所以这件事最容易讲乱的地方就在这里：

- **开不开，都不会改变 `updateBatchById` 的本质**
- 它本质上始终都是 **多条 `UPDATE` 的批处理**
- 区别主要在 **驱动层的发送方式**，以及可能带来的性能差异

## 8.4 最后记一句话

对 `updateBatchById` 来说，`rewriteBatchedStatements` 的影响可以概括成：

- 不开启：已经有 batch 效果
- 开启：驱动可能进一步优化发送方式
- 但对批量更新来说，它没有批量插入那么“决定性”

所以在批量更新场景里，它不是没用，而是：

- **不能作为核心选型依据**
- 真正更关键的，还是你选的是哪一类更新方案

# 9. 批量更新结果到底该怎么看

很多人做批量更新时，只关心 SQL 有没有执行成功，但真正到了工程里，你往往还要继续回答下面几个问题：

- 这次批量更新到底改了多少条
- 这 500 条里，哪些是没命中
- 哪些是命中了，但值本来就一样，所以其实没有变更
- 如果批次中途失败了，前面已经执行过的那些算不算成功

所以这里一定要先区分两个概念：

1. **执行阶段的影响行数**
2. **事务提交后的最终成功结果**

这两个概念，经常不是一回事。

## 9.1 为什么“500 条返回 480”不一定等于“20 条失败”

假设你更新 500 条数据，方法返回 `480`，这至少有 4 种可能：

- 20 条记录根本不存在
- 20 条记录命中了，但新值和旧值一样，MySQL 默认不会把它算作真正“变更”
- 你使用了某些允许跳过错误的写法，比如 `UPDATE IGNORE`
- 批处理执行里，驱动只告诉你“执行成功”，但不给出精确影响行数

MySQL 官方文档对 `UPDATE` 的说明很明确：**`UPDATE` 返回的是实际发生变化的行数，不是你这次想更新的目标记录数。**

所以在工程里，`480` 更准确的含义通常是：

**有 480 行真的被改掉了。**

但剩下那 20 行，到底是“不存在”、还是“值没变”、还是“被跳过”，单靠这个返回值本身是看不出来的。

## 9.2 还要再区分“执行成功”与“最终成功”

这个点非常关键。

比如你有 1000 条数据，按每批 500 条执行：

- 第一批 `flush` 后，返回更新了 `480`
- 第二批执行时报错
- 整个事务最终回滚

那从数据库最终结果看，这次成功条数不是 `480`，而是 **`0`**

因为前面那 480 条虽然执行过，但没有提交成功。

所以在业务语义里，真正应该关注的是：

- 如果你是**一个大事务包住全部批次**，那就只有两种结果：
  - 全部提交成功
  - 全部回滚，最终成功数为 `0`
- 如果你是**每批独立提交**
  - 那么第一批成功多少、第二批成功多少，才有单独统计意义

也就是说，**不先说清楚事务边界，讨论“成功了多少条”本身就是不完整的。**

## 9.3 不同批量更新方案，结果可观测性差别很大

|方案|通常能直接拿到什么|能不能直接知道“哪些记录成功了”|
|---|---|---|
|`UPDATE ... WHERE IN (...)`|总影响行数|不能|
|`CASE WHEN` 批量更新|总影响行数|不能|
|临时表 + `JOIN UPDATE`|总影响行数|不能直接拿到，但最好扩展|
|`updateBatchById`|通常只有粗粒度结果，例如是否成功|不能直接定位到每条|
|`ExecutorType.BATCH` + `flushStatements()`|可以拿到 `BatchResult` 分组结果|相对更容易|
|`INSERT ... ON DUPLICATE KEY UPDATE`|总影响行数|通常也不能直接定位到每条|

这张表很重要，因为它说明了一件事：

**单条大 SQL 很适合做性能优化，但不擅长做“逐条结果归因”。**

如果你的业务强依赖“知道是哪几条成功、哪几条失败”，那你就不能只从性能角度选方案。

## 9.4 如果你只想知道“总共成功了多少条”

这种场景通常出现在：

- 在线接口
- 强事务场景
- 不允许部分成功

这类场景里，通常不一定要拿“逐条成功列表”，但至少要记录下面几类信息：

- 本次请求原始记录数
- 分批大小
- 每批执行耗时
- 每批影响行数
- 是否有异常
- 最终事务是否提交

这里要注意一个现实问题：

单看 MyBatis 方法返回值，很多时候你拿到的是“影响行数”，不是“命中行数”。

如果你需要区分：

- 目标有 500 条
- 实际命中了多少条
- 实际变更了多少条

那通常要额外做一次校验，比如：

1. 更新前先查目标 ID 是否存在
2. 更新后再查目标值是否已落库

否则仅靠一个返回值，很难把这几个概念拆开。

## 9.5 如果你想知道“具体哪些记录成功了”

这时候就不能只靠一个整型返回值了。

### 方案一：Java 层逐条批处理，读取 `BatchResult`

如果你走的是 MyBatis 原生批处理，那么 `flushStatements()` 会返回 `List<BatchResult>`。  
`BatchResult` 里可以拿到：

- `getUpdateCounts()`
- `getParameterObjects()`

这就意味着，你可以把“第几个更新结果”映射回“第几条参数对象”。

```java
try (SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH, false)) {
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);

    for (User user : userList) {
        userMapper.updateById(user);
    }

    List<BatchResult> results = sqlSession.flushStatements();

    for (BatchResult result : results) {
        List<Object> params = result.getParameterObjects();
        int[] counts = result.getUpdateCounts();

        for (int i = 0; i < counts.length; i++) {
            Object param = params.get(i);
            int count = counts[i];

            // count >= 0: 有明确影响行数
            // Statement.SUCCESS_NO_INFO: 执行成功，但影响行数未知
            // Statement.EXECUTE_FAILED: 执行失败（仅在驱动继续处理后续语句时可能出现）
        }
    }

    sqlSession.commit();
}
```

这个方案最大的优点是：**它最适合做“逐条结果归因”**。

但它也有边界：

- 如果整个事务最后回滚了，这些“成功计数”不能直接当作最终业务成功数
- 如果不同记录生成的 SQL 形状不同，MyBatis 会分组返回多个 `BatchResult`
- `SUCCESS_NO_INFO` 这种情况说明“成功了”，但具体影响多少行，驱动并没有告诉你

### 方案二：单条大 SQL 后做结果复查

如果你走的是：

- `WHERE IN`
- `CASE WHEN`
- `JOIN UPDATE`

这种单条大 SQL 方案，那么数据库通常只会返回一个总数，不会直接告诉你是哪几个 ID 成功了。

这时候常见做法是补一轮“前后校验”：

1. 先拿到本次目标 ID 列表
2. 执行批量更新
3. 再按 ID 列表查询一次目标值、版本号或 `updated_at`
4. 根据结果判断哪些真正更新成功

这类方式本质上是在用“状态复查”代替“逐条返回值”。

### 方案三：临时表 / 中间表里直接记录处理状态

如果你的场景是：

- 导入任务
- 数据同步
- 离线修复
- 需要补偿重试

那最推荐的做法通常不是死盯 `UPDATE` 返回值，而是引入一张任务明细表或临时表，额外保存：

- 业务主键
- 目标更新值
- 批次号
- 处理状态
- 错误原因
- 重试次数

这样你最后拿到的就不只是“这批改了多少条”，而是：

- 哪些成功
- 哪些失败
- 为什么失败
- 哪些可以重试

这是最适合工程化落地的方式。

## 9.6 我的建议

如果你的业务只是普通在线更新接口：

- 优先保证事务一致性
- 记录总输入条数、影响行数、异常信息
- 不一定要拿逐条成功列表

如果你的业务是导入、同步、补偿、修复任务：

- **必须考虑结果归因**
- 不能只靠 `boolean` 或单个影响行数
- 最好有批次维度和记录维度两层结果

也就是说：

**批量更新除了要关注“怎么更快”，还要关注“怎么证明它真的按预期更新了”。**

# 10. 最终对比总表

前面几种方案如果逐段看，容易记混。

所以这里再用一张总表，把最常见的几种写法压缩到一起：

|方案|SQL 形态|是否依赖 batch 机制|数据库交互特征|适合场景|
|---|---|---|---|---|
|自己 `for` 循环 `updateById`|多条 `UPDATE`|不依赖|通常是多次立即执行|最简单，但性能通常最差|
|`updateBatchById`|多条 `UPDATE`|依赖 MyBatis / JDBC batch|按批次提交，不是每条都立刻执行|想快速接入，优先开发效率|
|`updateBatchById` + `rewriteBatchedStatements=false`|多条 `UPDATE`|依赖 MyBatis / JDBC batch|按 JDBC batch 执行|默认就可以接受的批处理方案|
|`updateBatchById` + `rewriteBatchedStatements=true`|多条 `UPDATE`|依赖 MyBatis / JDBC batch|驱动可能进一步优化发送方式|想继续榨一点驱动层优化空间|
|`UPDATE ... WHERE IN (...)`|单条 `UPDATE`|不依赖|一次执行一条 SQL|同值批量更新|
|`CASE WHEN` 批量更新|单条 `UPDATE`|不依赖|一次执行一条 SQL|不同值批量更新，且字段集合固定|
|临时表 + `JOIN UPDATE`|单条或少量 SQL|通常不依赖 Java 层 batch|偏数据库侧处理|超大批量、离线任务、后台任务|

这张表里最容易看错的点只有一个：

**`updateBatchById` 的批量，主要来自 batch 执行机制，不来自 SQL 合并。**

## 10.1 五句话总结

1. `updateBatchById` 的本质仍然是多条 `UPDATE`，不是单条大 SQL。  
2. `updateBatchById` 比自己写 `for updateById` 更高效，核心原因是它会走 MyBatis / JDBC batch。  
3. `rewriteBatchedStatements` 影响的是 MySQL 驱动发送策略，不改变 `updateBatchById` 的 SQL 语义。  
4. 同值更新优先 `WHERE IN`，不同值更新且字段集合固定优先 `CASE WHEN`。  
5. 数据量再大时，应该优先考虑临时表 + `JOIN UPDATE` 这类数据库侧方案。  

# 11. 批量更新时几个最容易踩坑的点

## 11.1 更新字段不统一，批处理效果会下降

比如一批数据里：
- 5 条只更新 `score`
- 5 条同时更新 `score` 和 `level`

那么最终生成的 SQL 形状不同，批处理就会被拆组，效果会打折。

所以做 Java 层批处理时，尽量让同一批数据的更新字段集合保持一致。

## 11.2 批量越大不一定越快

很多人会本能地觉得“一次更大更快”，但实际上批量太大时会带来新的问题：

- SQL 太长
- 锁持有时间更久
- 回滚成本更高
- 连接占用时间更长

所以批量更新一般都需要压测，而不是拍脑袋。

经验上可以从这些批次开始试：
- `200`
- `500`
- `1000`
- `2000`

然后结合你的表结构、索引、连接池和数据库负载来调。

## 11.3 真正的瓶颈常常不在 Java 代码

当数据量比较大时，真正影响性能的往往是：

- 索引维护成本
- 行锁竞争
- undo / redo 日志压力
- binlog 写入成本

所以“把循环改成批量”只是第一步，不代表性能问题就彻底解决了。

## 11.4 大事务要特别小心

一次事务里更新太多数据，会带来：
- 回滚时间长
- 锁范围大
- 失败重试成本高

因此大批量更新通常更推荐“分批提交”，而不是一个超级大事务一把梭。

# 12. 最后怎么选

如果你在项目里要做批量更新，可以按下面这个顺序判断：

|场景|优先方案|
|---|---|
|多条记录更新成同一个值|`UPDATE ... WHERE IN (...)`|
|多条记录更新成不同值，且字段集合固定|`CASE WHEN`|
|想快速接入，优先开发效率|`updateBatchById` / `ExecutorType.BATCH`|
|本质是“有则更新，无则插入”|`INSERT ... ON DUPLICATE KEY UPDATE`|
|数据量很大，偏离线任务或后台任务|临时表 + `JOIN UPDATE`|

可以把它理解成：

- **简单场景，优先简单 SQL**
- **中等复杂场景，优先 `CASE WHEN`**
- **大规模场景，优先数据库侧方案**

# 13. 总结

批量更新这件事，最容易犯的错就是把它理解成“找一个批量 API 调一下”。

但实际上，批量更新真正重要的是先分清场景：

- 是不是同值更新
- 是不是不同值更新
- 是不是新增更新混合
- 数据量到底有多大

只有先把这个问题分清楚，后面的 `MyBatis`、`MyBatis-Plus`、`MySQL` 优化手段才有意义。

所以这篇文章最后再收束成一句话：

**批量更新不是一个技巧，而是一套选型。**

## 参考资料

- MyBatis Java API: https://mybatis.org/mybatis-3/zh_CN/java-api.html
- MyBatis-Plus 持久层接口: https://baomidou.com/en/guides/data-interface/
- MyBatis-Plus 批量操作: https://baomidou.com/guides/batch-operation/
- MySQL Connector/J `rewriteBatchedStatements`: https://dev.mysql.com/doc/connector-j/en/connector-j-connp-props-performance-extensions.html
- MySQL `UPDATE` 语句: https://dev.mysql.com/doc/refman/8.4/en/update.html
- MySQL `INSERT ... ON DUPLICATE KEY UPDATE`: https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html
