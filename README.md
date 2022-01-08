# demo

## 准备数据

- tiup playground
- ./setup.sh 初始化数据
- ./update.sh 长时间运行，创建足够多版本

## Demo

### 简单查看数据

```sql
; 示例使用一个不断更新的表
SELECT * FROM dodo_say;
```

### save point 配置查看

```sql
; 系统表中新增的存档点配置，默认每小时自动创建一个存档点，24小时后过期
SELECT variable_name, variable_value FROM mysql.tidb WHERE variable_name LIKE '%save_point%';
; 当前存档点列表
SELECT * FROM mysql.gc_save_point;
```

### 基本的 mvcc 历史信息查询

```sql
; mvcc 历史信息查询，_tidb_mvcc_ts / _tidb_mvcc_op 是增加的虚拟列
SELECT id, name, quote, _tidb_mvcc_ts, _tidb_mvcc_op FROM dodo_say WHERE name = "老实的渡渡鸟";

; 可以使用 TiDB 内置函数来让时间戳更好看
SELECT id, name, quote, TIDB_PARSE_TSO(_tidb_mvcc_ts) FROM dodo_say WHERE name = "老实的渡渡鸟";

; 可以观察到 SafePoint 之前的数据，每小时保留一个最新版本
```

### 条件过滤和各种SQL

```sql
; 虚拟列可以进行条件过滤
; 例如按版本选择最近5分钟的更新
SELECT id, name, quote, TIDB_PARSE_TSO(_tidb_mvcc_ts) FROM dodo_say WHERE TIDB_PARSE_TSO(_tidb_mvcc_ts) > SUBTIME(NOW(), '00:05:00');

; 也可以找到曾经说过某句话的鸟（即历史版本出现对应的值）
SELECT id, name, quote, TIDB_PARSE_TSO(_tidb_mvcc_ts) FROM dodo_say WHERE quote = '不在沉默中爆发，就在沉默中灭亡。';

; 查找整个表最近20条MVCC记录
SELECT id, name, quote, TIDB_PARSE_TSO(_tidb_mvcc_ts) FROM dodo_say ORDER BY _tidb_mvcc_ts DESC LIMIT 20;
```

### 更改旧 MVCC 版本

```sql
; 可以使用带 _tidb_mvcc_ts 虚拟列的 UPDATE 来篡改历史版本
SELECT id, name, quote, _tidb_mvcc_ts, _tidb_mvcc_op FROM dodo_say WHERE name = "老实的渡渡鸟" limit 5;
; 我们从上面的 ts 中选一条记录更新
UPDATE dodo_say set quote="hello" where name="老实的渡渡鸟" AND _tidb_mvcc_ts = ?;
; 重新查询
SELECT id, name, quote, _tidb_mvcc_ts, _tidb_mvcc_op FROM dodo_say WHERE name = "老实的渡渡鸟" limit 5;
```

### Flashback

```sql
; 删除一部分数据
DELETE FROM dodo_say WHERE id > 15;
; 确认数据已经删除
SELECT * FROM dodo_say;
; 可以通过扩展语法查看对应的删除记录
SELECT id, TIDB_PARSE_TSO(_tidb_mvcc_ts) FROM dodo_say WHERE _tidb_mvcc_op = 'Delete';
; 选择删除操作之前的时间戳进行回滚，这是一个 DDL 操作，不实际个修改
FLASHBACK TABLE dodo_say TO TIMESTAMP ?;
; 重新查询确定删除的数据已回滚
SELECT * FROM dodo_say;

; 支持多次 Flashback，我们来一次忘记带 WHERE 的 UPDATE
UPDATE dodo_say SET quote = 'hi';
; ops，全表都被改了
SELECT * FROM dodo_say;
; 别慌，回到 20 秒之前
FLASHBACK TABLE dodo_say TO TIMESTAMP SUBTIME(NOW(), '00:00:20');
; 回滚之后，数据恢复了
SELECT * FROM dodo_say;
```