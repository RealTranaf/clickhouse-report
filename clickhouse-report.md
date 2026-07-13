<h1 align="center">ClickHouse</h1>

ClickHouse là một hệ quản trị cơ sở dữ liệu OLAP (Online Analytical Processing) dạng cột, được thiết kế để xử lý tối ưu hàng tỷ bản ghi, truy vấn cực nhanh, phân tích dữ liệu trong thời gian thực và nén dữ liệu tốt. 

Khác với các cơ sở dữ liệu OLTP, ClickHouse được tối ưu cho ứng dụng đọc và phân tích dữ liệu. Với việc sử dụng mô hình lưu trữ cột, ClickHouse có khả năng tối ưu hóa hiệu suất cho các truy vấn chỉ yêu cầu một số lượng nhỏ cột, tiết kiệm băng thông và thời gian xử lý.

**OLTP và OLAP**

![alt text](images/1-1.png)

OLTP (Online Transactional Processing) là hệ thống cơ sở dữ liệu xử lý các giao dịch phát sinh liên tục. VD: website bán hàng, internet banking, POS, đăng nhập người dùng... Mỗi thao tác thường tác động tới ít bản ghi.

Đặc điểm:

- Nhiều người dùng truy cập cùng lúc.

- Dữ liệu luôn được cập nhật.

- Yêu cầu tính ACID (Atomicity: tính nguyên tử, Consistency: tính nhất quán, Isolation: tính cô lập, Durability: tính bền vững)

- Tập trung vào transaction. Khi một hệ thống OLTP hoạt động, mọi bước phải thành công hoặc có thể rollback.

OLAP (Online Analytical Processing) là hệ thống CSDL được thiết kế để làm việc với lượng dữ liệu rất lớn, phục vụ việc phân tích dữ liệu. VD: thống kê doanh thu, top sản phẩm bán chạy hay nhật ký hoạt động của server, ứng dụng.

Đặc điểm:

- Được tối ưu cho việc đọc dữ liệu rất lớn, nén dữ liệu, tính toán song song, truy vấn tổng hợp và scan hàng tỉ bản ghi.

- Không để tâm tới đảm bảo tính chính xác và toàn vẹn của giao dịch dữ liệu hay cập nhật dữ liệu cũ.

Một ví dụ thực tế: Trong một nền tảng mua hàng trực tuyến, những hoạt động liên quan tới mua bán như tạo đơn hàng, cập nhật kho hàng, thực hiện giao dịch... sẽ sử dụng OLTP. Những hoạt động thống kê như doanh thu, xu hướng mua hàng, lịch sử mua hàng... sẽ sử dụng OLAP.

ClickHouse là một trong những CSDL OLAP phổ biến nhất, được sử dụng rộng rãi cho những nhu cầu lưu trữ log, big data...

**Đặc điểm**

- Kiến trúc lưu trữ dạng cột: Một CSDL OLTP như MySQL sẽ lưu dữ liệu thành hàng kiểu như sau

```
Row1
1 frontend 200 10

Row2
2 backend 500 25

Row3
3 frontend 200 15
```

Nếu muốn query kiểu SELECT avg(duration) thì MySQL sẽ phải đọc toàn bộ hàng. Còn Clickhouse sẽ lưu theo cột kiểu:

```
id
-------
1
2
3

service
-------
frontend
backend

duration
-------
20
35
12

status
-------
200
500
200
```

Khi cần tính avg(duration) thì ClickHouse chỉ cần đọc cột duration chứ ko cần đọc các cột khác, thời gian query sẽ nhanh hơn rất nhiều. Kiến trúc dạng cột cho phép khi query, ClickHouse phải lọc qua ít dữ liệu hơn; compress dữ liệu tốt hơn.

**MergeTree engine**

MergeTree engine là thành phần quan trọng nhất trong ClickHouse, là engine vận hành bảng. Những đặc điểm chính của MergeTree table engine:

- Dữ liệu được lưu theo part. MergeTree ko ghi trực tiếp vào 1 file dữ liệu duy nhất. Mỗi lần insert, engine tạo ra 1 data part mới. VD:

```
INSERT 100,000 rows -> Part A
INSERT 50,000 rows -> Part B
INSERT 80,000 rows -> Part C
```

Sau đó, engine gộp các part nhỏ thành part lớn hơn. Giúp ghi dữ liệu nhanh, nhiều tiến trình có thể ghi đồng thời.

- Background merge: MergeTree liên tục chạy các tiến trình nền để hợp nhất các part, giúp giảm số lượng file, giảm lượng index cần đọc, tăng tốc độ truy vấn.

- Lưu trữ theo cột: engine lưu từng cột của bảng thành file riêng biệt. VD: bảng có 4 cột Timestamp, Service, Status, Duration thì sẽ được lưu thành 4 file Timestamp.bin, Service.bin, Status.bin, Duration.bin. Nếu truy vấn SELECT avg(Duration) FROM logs; thì ClickHouse chỉ cần đọc Duration.bin thay vì toàn bộ dữ liệu như lưu trữ dạng hàng.

- MergeTree ko tạo index cho từng dòng, primary index chỉ lưu giá trị đầu tiên của mỗi granule, một granule là mặc định 8192 dòng => index rất nhỏ, tiết kiệm RAM, tìm dữ liệu nhanh nhưng ko chính xác bằng các CSDL thông thường.

- Khóa chính không tham chiếu đến từng hàng riêng lẻ mà tham chiếu đến các khối gồm 8192 hàng được gọi là granule, giúp cho khóa chính của các tập dữ liệu khổng lồ đủ nhỏ để vẫn được lưu trữ trong bộ nhớ, đồng thời vẫn cung cấp khả năng truy cập nhanh vào dữ liệu trên đĩa. Khóa chính của bảng cũng xác định thứ tự sắp xếp trong từng phần của bảng (clustered index). 

- TTL (Time to live): dữ liệu sẽ bị xóa sau khi thời gian TTL của nó hết. Thời gian này có thể thay đổi được.

**Những tính năng quan trọng**

- Mã nguồn mở.

- Hiệu suất cao, có thể thực hiện query phức tạp trên khối dữ liệu lớn trong thời gian ngắn.

- Xử lý dữ liệu cột, cho phép lọc và truy vấn dữ liệu hiệu quả hơn so với phương pháp lưu trữ theo hàng. Khi cần truy vấn một vài cột dữ liệu, ClickHouse chỉ cần đọc dữ liệu từ những cột đó thay vì quét toàn bộ bảng.

**Khi nào nên sử dụng ClickHouse**

- Phân tích dữ liệu lớn (OLAP): ClickHouse được tối ưu cho việc xử lý các truy vấn phân tích phức tạp trên tập dữ liệu khổng lồ.

- Phân tích nhật ký hoạt động của server và ứng dụng: ClickHouse xử lý hiệu quả các tập tin nhật ký lớn, dễ dàng trích xuất thông tin, phát hiện sự cố, tối ưu hiệu năng và cải thiện trải nghiệm người dùng.

- Phân tích dữ liệu thời gian thực: Khả năng xử lý luồng dữ liệu thời gian thực của ClickHouse giúp bạn nắm bắt nhanh chóng các xu hướng và hành vi người dùng, hỗ trợ ra quyết định nhanh chóng.

- Tạo báo cáo và bảng điều khiển: ClickHouse tạo ra các báo cáo trực quan và bảng điều khiển thông tin hiệu quả, giúp theo dõi hiệu suất kinh doanh, hoạt động marketing và các chỉ số quan trọng khác.

**Khi nào không nên sử dụng ClickHouse**

- ClickHouse không được thiết kế để xử lý các truy vấn cập nhật dữ liệu thường xuyên (OLTP), Nếu cần thực hiện các giao dịch trực tuyến thường xuyên thì ClickHouse không phải là lựa chọn tốt.

- Hạn chế trong việc xử lý các transaction phức tạp: ClickHouse đơn giản hóa việc quản lý transaction, do đó không phù hợp với các ứng dụng cần xử lý các transaction phức tạp.

- Không hiệu quả khi cần thực hiện truy xuất dữ liệu theo từng hàng: ClickHouse không thể thực hiện tác vụ truy xuất và tìm kiếm nhanh các hàng riêng lẻ theo khóa.

**Kết luận**

ClickHouse lý tưởng cho các ứng dụng phân tích dữ liệu quy mô lớn. Với tốc độ xử lý truy vấn nhanh, hỗ trợ các truy vấn phức tạp và khả năng mở rộng, ClickHouse đang ngày càng được ưa chuộng trong nhiều lĩnh vực.