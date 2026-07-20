**Backup ClickHouse**

Khi backup một table hay database trong ClickHouse, CSDL sẽ lưu một copy của table hay database đó vào trong một thư mục nhất định. Những địa chỉ thư mục đặc biệt trong ClickHouse được gọi là Disk, và Disk có khả năng nhận backup sẽ do người dùng chỉ định trong config của ClickHouse. Backup có thể lưu trực tiếp dưới dạng thư mục hoặc lưu vào một file zip. Khi restore thì có thể restore trực tiếp từ các thư mục và file zip này.

Command để kiểm tra xem đã có disk để backup hay chưa:

```
SELECT * FROM system.disks;
```

Nếu ko có disk để backup mà chỉ có disk default thì cần config một disk mới.

Trong thư mục config của ClickHouse (thường là /root/signoz/deploy/common/clickhouse), cần tạo một file config riêng cho disk mới. VD backup_disk.xml

```
<clickhouse>
    <storage_configuration>
        <disks>
            <backups>
                <type>local</type>
                <path>/var/lib/clickhouse/backups/</path>
            </backups>
        </disks>
    </storage_configuration>

    <backups>
        <allowed_disk>backups</allowed_disk>
        <allowed_path>/var/lib/clickhouse/backups/</allowed_path>
    </backups>
</clickhouse>
```

- `<backups><type></type><path></path></backups>` định nghĩa địa chỉ của thư mục backup. Thư mục sẽ là một thư mục local, trỏ tới một thư mục nhất định. Địa chỉ sử dụng ở đây là một thư mục trong docker container đang chạy ClickHouse.

Sau đó, cần mount path của config này trong docker compose và mount thư mục backup trong container vào một thư mục trong máy thực để có thể lấy backup ra ngoài:

```
volumes:
    - ../common/clickhouse/backup_disk.xml:/etc/clickhouse-server/config.d/backup_disk.xml
    - ./backups:/var/lib/clickhouse/backups
```

Sau khi hoàn thành config và file compose, restart ClickHouse và truy cập clickhouse-client để check xem đã nhận diện disk mới chưa. Nếu disk backups xuất hiện thì đã config thành công.

Để backup một DB:

```
BACKUP DATABASE signoz_traces TO Disk('backups', 'signoz_traces.zip');
```

Backup nhiều DB:

```
BACKUP DATABASE signoz_logs,
DATABASE signoz_metrics,
DATABASE signoz_traces
TO Disk('backups', 'backup-2026-07-07.zip');
```

Backup toàn bộ:

```
BACKUP ALL TO Disk('backups', 'full-backup.zip');
```

Để restore DB:

```
RESTORE DATABASE signoz_traces FROM Disk('backups', 'signoz_traces.zip');
```

Lưu ý: Không thể restore một DB đã tồn tại trong ClickHouse. Để restore thì phải xóa DB cũ đi đã trước khi restore.

Lưu ý 2: Trong SigNoz, một số DB ko thể bị xóa như DB system.