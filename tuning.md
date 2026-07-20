Để thực hiện tuning ClickHouse, có thể tối ưu hệ thống ở 4 lớp: schema, insert pipeline, query execution, server config.

**1. Schema tuning:**

**Chọn ORDER BY đúng**

ClickHouse ko có B-tree index như MySQL mà dựa vào Primary key (ORDER BY). VD:

```
ENGINE = MergeTree
PARTITION BY toYYYYMM(timestamp)
ORDER BY (
    service_name,
    timestamp
)
```

thì khi thực hiện query như:

```
WHERE service_name='frontend' AND timestamp > now()-1h
```

sẽ rất nhanh do ClickHouse chỉ cần đọc 1 phần rất nhỏ của table. Nếu chỉ order by (timestamp) thì sẽ phải đọc nhiều hơn -> thời gian query chậm lại.

