ClickHouse sử dụng ngôn ngữ SQL làm ngôn ngữ query nhưng có phần mở rộng riêng. Sau đây là một số lệnh quan trọng cần nhớ. Để tìm hiểu thêm: https://clickhouse.com/docs/sql-reference

**Query cơ bản**

ClickHouse hỗ trợ các lệnh query cơ bản như sau:

SELECT:

```
SELECT * FROM logs;
```

- Lấy một số cột:

```
SELECT
    timestamp,
    service_name,
    severity_text
FROM logs;
```

WHERE:

```
SELECT * FROM logs WHERE severity_text = 'ERROR';
```

- Thêm điều kiện:

```
SELECT *
FROM logs
WHERE service_name = 'frontend'
  AND severity_text = 'ERROR';
```

ORDER BY:

```
SELECT * FROM logs ORDER BY timestamp DESC;
```

LIMIT:

```
SELECT * FROM logs LIMIT 100;
```

DISTINCT:

```
SELECT DISTINCT service_name FROM logs;
```

HAVING:

```
SELECT
    service_name,
    count()
FROM logs
GROUP BY service_name
HAVING count() > 1000;
```

**Aggregate query**

ClickHouse có đủ các aggregate function thông thường của SQL (sum, avg, min, max, count)

count():

```
SELECT count() FROM logs;
SELECT service_name, count() FROM logs GROUP BY service_name;
```

avg():

```
SELECT avg(duration_ms) FROM traces;
```

sum():

```
SELECT sum(bytes) FROM logs;
```

**JOIN**

ClickHouse cũng có lệnh join của SQL.

INNER JOIN:

```
SELECT *
FROM orders o
JOIN customers c
ON o.customer_id = c.id;
```

LEFT JOIN:

```
SELECT *
FROM orders o
LEFT JOIN customers c
ON o.customer_id = c.id;
```

ClickHouse hỗ trợ đầy đủ INNER, LEFT, RIGHT, FULL, CROSS, ANY JOIN, ASOF JOIN.

**Subquery**

Cách để viết subquery trong ClickHouse:

```
SELECT * FROM
(
    SELECT * FROM logs WHERE severity_text='ERROR'
)
LIMIT 100;
```

**WITH**

Sử dụng WITH để đặt tên biến:

```
WITH now() AS current_time

SELECT current_time, count() FROM logs;
```

**UNION**: tương tự như các SQL khác

```
SELECT column_name(s) FROM table1
UNION ALL
SELECT column_name(s) FROM table2; 
```

**INSERT**: tương tự như các SQL khác:

```
INSERT INTO logs
VALUES
(
    now(),
    'frontend',
    'ERROR',
    'Something wrong'
);
```

hoặc

```
INSERT INTO logs
(
    timestamp,
    service_name,
    severity_text,
    body
)
VALUES
(...)
```

**CREATE, DROP**

```
CREATE DATABASE monitoring;
```

```
CREATE TABLE logs
(
    timestamp DateTime,
    service_name String,
    severity String,
    body String
)
ENGINE = MergeTree()
ORDER BY timestamp;
```

```
DROP TABLE logs;
DROP DATABASE monitoring;
```

**SHOW, DESCRIBE, USE**

DESCRIBE: mô tả table

```
DESCRIBE TABLE logs;
```

SHOW:

```
SHOW TABLES;
SHOW DATABASES;
```

USE:

```
USE signoz_logs;
```

**ALTER**

Thêm cột:

```
ALTER TABLE logs
ADD COLUMN host String;
```

Xóa cột:

```
ALTER TABLE logs
DROP COLUMN host;
```

Đổi tên cột:

```
ALTER TABLE logs
RENAME COLUMN host TO hostname;
```

**UPDATE và DELETE**

ClickHouse ko hỗ trợ update theo cách của MySQL mà sẽ thực hiện như sau:

```
ALTER TABLE users
UPDATE age = 20
WHERE id = 1;
```

```
ALTER TABLE logs
DELETE
WHERE timestamp < now() - INTERVAL 30 DAY;
```

2 hành động này đều là mutation, dữ liệu sẽ ko bị xóa hay thay đổi ngay lập tức.

**Backup và restore**

```
BACKUP DATABASE signoz_logs
TO Disk('backups', 'logs.zip');
```

```
RESTORE DATABASE signoz_logs
FROM Disk('backups', 'logs.zip');
```

**Các hàm thường dùng**

Datetime: now(), today(), yesterday(), toDate(), toDateTime(), toStartOfHour(), toStartOfDay(), toStartOfMonth()

```
SELECT
    toStartOfHour(timestamp),
    count()
FROM logs
GROUP BY toStartOfHour(timestamp);
```

String: length(), concat(), substring(), lower(), upper(), replaceAll(), match(), like()

```
SELECT * FROM logs
WHERE match(body, 'timeout');
```

Array: arrayJoin(), has(), length(), indexOf()

JSON: JSONExtractString(), JSONExtractInt(), JSONExtractBool()

```
SELECT JSONExtractString(body, 'service')
FROM logs;
```

**Map**

ClickHouse hỗ trợ kiểu data Map(K, V) để chứa các dữ liệu nhỏ hơn trong một cột. VD:

```
SELECT
    ResourceAttributes['service.name']
FROM traces;
```

sẽ lấy thuộc tính service.name trong cột ResourceAttribute. Đây là kiểu dữ liệu được sử dụng phổ biến trogn SigNoz để lưu các thuộc tính của OpenTelemetry.
