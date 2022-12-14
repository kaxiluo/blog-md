在日常工作中，我们可能会遇到根据数据存在/不存在，再决定是否插入数据的场景。例如不存在则插入数据，存在则更新数据（或不变）。然而在高并发情况下，可能会重复插入或因唯一约束导致的报错，下面给出写数据时的处理方案。

示例：每天生成一张背景图，创建背景图表，在`date`字段创建唯一索引。
```sql
CREATE TABLE `backgrounds` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `date` date NOT NULL COMMENT '日期',
  `url` varchar(255) NOT NULL DEFAULT '' COMMENT '地址',
  `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `udx_date` (`date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**对于存在的定义：主键或`UNIQUE KEY`在表中已存在**

## 不存在插入，存在不变

### INSERT IGNORE INTO
不存在时：影响1行； 存在时：影响0行；
```sql
INSERT IGNORE INTO backgrounds(`date`, `url`)
VALUES('2022-08-18', 'https://1.png');
```

### INSERT INTO NOT EXISTS
不存在时：影响1行； 存在时：影响0行；
```sql
INSERT INTO backgrounds(`date`, `url`)
SELECT '2022-08-18' as `date`, 'https://1.png' as `url` FROM backgrounds 
WHERE NOT EXISTS (SELECT 1 FROM backgrounds WHERE `date` = '2022-08-18');
```

## 不存在插入，存在则更新

### ON DUPLICATE KEY UPDATE
不存在时：影响1行； 存在时：若新增数据与原数据有变化影响2行，mysql内部先执行了delete，然后再insert；
```sql
INSERT INTO backgrounds(`date`, `url`)
VALUES('2022-08-19', 'https://123.png') 
ON DUPLICATE KEY UPDATE `url` = 'https://123.png';
```

### REPLACE INTO
不存在时：影响1行； 存在时：影响2行，mysql内部先执行了delete，然后再insert；
```sql
REPLACE INTO backgrounds(`date`, `url`)
VALUES('2022-08-18', 'https://1.png');
```

## 其他方案

- 插入数据时，锁定表的读写`LOCK TABLES`，但是会降低并发性。

- 根据Redis集合元素的唯一性判断是否存在。

- 借助外部锁，例如Redis锁，插入时获取锁。
