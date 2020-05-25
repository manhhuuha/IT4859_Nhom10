# High Performance MySQL - Replication

- MySQL Replication là một quá trình sao chép dữ liệu từ 1 database tới 1 hoặc nhiều database khác 1 cách tự động.
- Là nền tảng cho việc xây dựng 1 hệ thống lớn, hiệu năng cao.
- Nó cũng được dùng để xây dựng hệ thống có khả năng sẵn sàng phục vụ cao, co dãn, phục hồi khi có sự cố và nhiều chức năng khác.

## Overview

- Vấn đề cơ bản của Replication: đảm bảo dữ liệu giữa các database được đồng bộ.
- MySQL Replication có khả năng tương thích ngược với các phiên bản trước.
- Replication thường không làm tăng quá nhiều khối lượng công việc cho master. Mặc dù việc sử dụng binary log sẽ làm tăng đáng kể khối lượng công việc, nhưng nó dùng để sao lưu và phục hồi 1 cách nhanh chóng.
- Trong 1 số trường hợp, Replication có thể khiến master phải chịu tải lớn.

```
Mỗi slave sẽ mở 1 I/O_Thread tới Master. Trong trường hợp slave tốn nhiều thời gian để xử lí 1 đoạn log, và không bắt kịp với sự kiện mới nhất trong log ở master, I/O_Thread sẽ tốn nhiều tài nguyên hơn để tìm và đọc các log cũ.
```

- Có 2 cách thức đồng bộ: đồng bộ theo câu lệnh và đồng bộ theo hàng dữ liệu. Cả 2 cách thức đều ghi dữ liệu vào binary log và chạy lại log trên các bản sao
- Đồng bộ theo câu lệnh: xuất hiện từ MySQL 3.23. master sẽ ghi các câu lệnh SQL lên binary log. 
Các bản sao sẽ lấy binary log và thực thi lại các lệnh.
- Đồng bộ theo hàng dữ liệu: xuất hiện từ MySQL 5.1. master sẽ ghi lại sự thay đổi của mỗi hàng dữ liệu trong bảng vào binary log. Các bản sao sẽ dựa vào sự thay đổi để cập nhật lại dữ liệu.
- Có thể sử dụng đồng thời cả 2 cách để đồng bộ dữ liệu. Mặc định, MySQL sẽ sử dụng đồng bộ theo câu lệnh. Nhưng có thể chuyển sang dùng đồng bộ theo hàng dữ liệu với các câu lệnh cụ thể, hoặc với các bộ lưu trữ cụ thể. (MyISAM, InnoDB, Memory, NDB)

## How Replication Works

- Gồm 3 giai đoạn chính:
    1. Master sẽ lưu lại những việc làm thay đổi dữ liệu vào binary log. Ngay trước khi dữ liệu được cập nhật, master sẽ ghi lại sự thay đổi vào binary log, sau đó, master mới hoàn thành việc cập nhật dữ liệu.
    2. Replica lấy dữ liệu từ binary log của master và lưu vào relay log của nó. Replica tạo I/O Thread để tạo kết nối tới master, và lấy dữ liệu từ binlog. I/O Thread sẽ chủ động đọc dữ liệu từ binlog của master mà không phải đợi master gọi đến. I/O Thread ghi dữ liệu nhận được vào relay log.
    > Trước MySQL 4.0, replication hoạt động theo các cách khác nhau. Ví dụ: MySQL ban đầu không có relay log, vì vậy replication chỉ dùng 2 thread thay vì 3.
    3. Replica cập nhật dữ liệu của nó theo relay log bằng SQL Thread. Các sự kiện SQL Thread xử lí có thể được ghi lại binlog của replica.

## Replication Under the Hood

### Statement-Based Replication

- Khi replica đọc dữ liệu từ relay log, nó sẽ thực thi lại các câu lệnh SQL.
- Ưu điểm:
    - Dễ cài đặt.
    - Tốn ít chi phí khi đọc, ghi binlog.
    - Có thể kiểm tra lại việc thực thi các lệnh SQL của replica.
    - Có thể thực thi khi thay đổi schema.
- Nhược điểm:
    - Có 1 số câu lệnh SQL mà replica không thể thực hiện đúng, ví dụ: `CURRENT_USER()`
    - Các bug liên quan tới stored procedures, trigger, AUTO_INCREMENT

### Row-Based Replication

- MySQL 5.1 bắt đầu hỗ trợ `row-based replication`. Các thay đổi về dữ liệu sẽ được ghi lại vào binlog.
- Ưu điểm:
    - Không tốn chi phí tính toán khi cập nhật dữ liệu.
    - Ghi lại dữ liệu thay đổi mà không ghi lại câu lệnh SQL làm thay đổi dữ liệu => Dễ phục hồi dữ liệu hơn trong 1 số trường hợp.
    - Row-based có thể giúp bạn tìm các vấn đề liên quan đến xung đột dữ liệu. Ví dụ: khi cập nhật 1 trường dữ liệu không tồn tại, row-based sẽ báo lỗi.
    - Giảm locking so với statement-based.
- Nhược điểm:
    - Tốn chi phí đọc, ghi trên binlog
    - Không kiểm tra được quá trình cập nhật dữ liệu.
    - Row-based không thực hiện được 1 số việc mà statement-based có thể làm được. Ví dụ như việc thay đổi schema trên replica, cập nhật dữ liệu không tồn tại sẽ báo lỗi và dừng việc cập nhật.
    - Nếu có nhiều lớp replica, các lớp được cấu hình với row-based, thì khi có câu lệnh yêu cầu đồng bộ với statement-based, nó sẽ chỉ được thực hiện với lớp replica đầu tiên. Các lớp replica sẽ đổi lại về dùng row-based.

### Replication Files

- `mysql-bin.index`: theo dõi các file binlog có trên disk, mỗi dòng chứa tên 1 file binlog.
- `mysql-relay-bin.index`: tương tự như `mysql-bin.index`, theo dõi các file relay log trên disk.
- `master.info`: chứa thông tin để replica kết nối tới master. Ngoài ra, file còn chứa password của user trên master dưới dạng text.
- `relay-log.info`: chứa vị trí của binlog và relay log
    - `Relay_log_pos`: vị trí hiện tại trong relay log, các sự kiện tính tới vị trí này thì đã được thực thi trên slave
    - `Master_log_pos`: vị trí trong file binlog ở master, các sự kiện tính tới vị trí này thì đã được thực thi trên slave

### Sending Replication Events to Other Replicas

- `log_slave_updates` cho phép bạn sử dụng replica này là master của replica khác. Nó sẽ ghi lại các lệnh SQL mà replica xử lí lên binlog của chính nó, cho phép các replica khác có thể lấy dữ liệu.
- Khi replica ghi lại sự kiện đã xử lí vào binlog, sự kiện đó gần như là sẽ có vị trí trong log khác với master. Điều này sẽ gây khó khăn cho việc thay đổi master của replica.
- Cần cấu hình cho mỗi replica một ID riêng, để tránh việc lặp vô hạn khi đồng bộ.

### Replication Filter

- Cho phép bạn lọc các dữ liệu để đồng bộ.
- Có 2 loại bộ lọc: lọc sự kiện từ binlog và lọc sự kiện từ relay log.
    - Với binlog: gồm 2 option: `binlog_do_db` và `binog_ignore_db`
    - Với relay log: gồm các option bắt đầu với `replication`
> Lưu ý: với các option kết thúc bằng `_do_db` và `_ignore_db`, nó sẽ thực hiện lọc trên bảng hiện tại, chứ không phải bảng được truy vấn. `binlog_do_db` và `binlog_ignore_db` cũng gây khó khăn khi recover.

> Không nên sử dụng replication filter. Thay vào đó, ta có thể sử dụng các bảng sử dụng Blackhole để lọc.

### Replication Topologies

- Các lưu ý cơ bản:
    - Mỗi replica chỉ có 1 master
    - Mỗi replica phải có 1 ID định danh
    - 1 master có thể có nhiều replica
    - 1 replica có thể làm master của replica khác, và lan truyền sự thay đổi của master.

#### Master and Multiple Replicas

- Đây là topo đơn giản nhất, hiệu quả khi cần đọc nhiều, ghi ít
- Đáp ứng nhiều nhu cầu khác nhau:
    - Sử dụng các replica với các vai trò khác nhau
    - Replica làm master dự phòng.
    - Replica làm kho dữ liệu để phục hồi.
    - Tạm dừng một vài replica để phục hồi dữ liệu.
    - Replica dùng để sao lưu, kiểm thử, phát triển hệ thống.
- Ưu điểm:
    - Dể cấu hình, quản lí.
- Nhược điểm:
    - Bottleneck tại master
    - Việc ghi dữ liệu chỉ thực hiện trên master.

#### Master-Master in Active-Active Mode

- Ưu điểm: Cho phép đọc và ghi trên cả 2 master.
- Nhược điểm: 
    - Xử lí xung đột dữ liệu thay đổi. Ví dụ: thêm dữ liệu kiểu AUTO_INCREMENT cùng lúc trên cả 2 master. Mặc dù, từ MySQL 5.0 đã thêm `auto_increment_increment` và `auto_increment_offset` để hỗ trợ replication, nhưng việc ghi dữ liệu trên cả 2 master vẫn có rủi ro cao.
        - `auto_increment_increment`: khoảng cách giữa các giá trị
        - `auto_increment_offset`: giá trị bắt đầu.
        - Mỗi master sẽ cấu hình 2 giá trị này khác nhau. Ví dụ: 1 master dùng các số lẻ, 1 master dùng các số chẵn.
    - Vấn đề khi replication có lỗi và dừng hoạt động, trong khi đó 2 master vẫn tiến hành ghi dữ liệu.

> MySQL không hỗ trợ replcation từ nhiều master.

#### Master-Master in Active-Passive Mode

- 1 master sẽ chỉ cho phép đọc dữ liệu.
- Ưu điểm:
    - Giải quyết vấn đề xung đột dữ liệu khi cùng ghi ở 2 master.
    - Dễ dàng chuyển đổi trạng thái của master giữa active và passive, cho phép bảo trì, nâng cấp, phục hồi mà không mất thời gian chờ đợi.
        - Ví dụ: để thực hiện ALTER TABLE trên passive, ta dừng việc nhận đồng bộ từ active trước, và khôi phục lại sau khi hoàn thành. Sau đó, chuyển active -> passive, passive -> active và thực hiện như trước đó.
- Nhược điểm:
    - Không làm tăng khả năng ghi dữ liệu của hệ thống.

#### Master-Master with Replicas

- Ưu điểm:
    - Cho phép replica thay thế master khi master có vấn đề.
    - Tăng khả năng đọc cho hệ thống.
- Nhược điểm:
    - Việc thay đổi master sẽ gây khó khăn nhất định.

#### Ring Replication

- 1 vòng gồm 3 master trở lên, mỗi nút là replica của nút trước và là master của nút sau.
- Ưu điểm:
    - Tăng khả năng ghi của hệ thống.
- Nhược điểm:
    - Phụ thuộc vào từng nút trong mạng, sẽ gây ra lỗi nếu có nút hỏng.
    - Cần định danh cho mỗi nút để tránh việc lặp vô hạn. Mặc dù vậy, nếu loại bỏ 1 nút ra khỏi mạng, các sự kiện khi lan truyền trong mạng sẽ rơi vào trạng thái lặp vô hạn, do không còn nút đó để lọc sự kiện.
    - Có thể thêm replica cho mỗi master để phục hồi khi có sự cố, nhưng không giải quyết được sự cố với vấn đề về đường truyền.

> Không nên dùng

#### Master, Distribution Master, and Replica

- Việc kết nối tới nhiều replica sẽ làm tăng tương đối khối lượng công việc lên master. Vì vậy, để giải quyết vấn đề, ta nên dùng 1 distribution master.
- Distribution master là 1 replica chỉ dùng để đọc và phân phối dữ liệu từ binlog của master. Để tránh việc distribution master thực thi lại các sự kiện khi đọc từ binlog của master, ta có thể sử dụng Blackhole storage engine.
- Khó để xác định số lượng replica mà master có thể xử lí trước khi nó cần đến distribution master.
    - Nếu master sử dụng gần hết công suất, ta sẽ không thể thêm replica trực tiếp vào master. Nhưng nếu master ghi ít hoặc chỉ thực hiện replicate 1 phần nhỏ dữ liệu (việc replication không tốn quá nhiều tài nguyên), master có thể xử lí thêm 1 vài replica nữa.
    - Có thể sử dụng nhiều distribution master.
- Ưu điểm:
    - Sử dụng Blackhole storage engine cho distribution master sẽ phục vụ được nhiều replica hơn các storage engine khác (Blackhole engine không lưu dữ liệu, vì vậy, khi distribution master thực thi lại query từ relay log, sẽ không có gì xảy ra).
- Nhược điểm:
    - Bảng sử dụng Blackhole engine vẫn còn bug, ví dụ: AUTO_INCREMENT. (thực hiện replication với bảng sử dụng Blackhole engine, có trường AUTO_INCREMENT là primary key. Vì Blackhole engine không chứa dữ liệu, nên không quản lí giá trị AUTO_INCREMENT. Vì vậy, replicate trên replica luôn lỗi vì trùng khoá chính)
    - Đảm bảo các bảng trong distribution master sử dụng Blackhole engine. Vấn đề xảy ra khi tạo bảng trên master mà xác định cụ thể storage engine sử dụng, vì việc đặt option `storage_engine = blackhole` sẽ chỉ có tác dụng với các lệnh `CREATE TABLE` mà không xác định cụ thể storage engine.
    - Khó khăn khi thay thế master bằng replica. (vị trí trong binlog distribution master luôn khác với vị trí trong binlog của master gốc)

#### Tree or Pyramid

- Khi master cần replicate với số lượng lớn replica (chia đều tải replication xuống cho các replica).
- Nếu có replica trung gian bị lỗi, các replica dưới nó sẽ bị ảnh hưởng.

#### Custom Replication Solutions

- Có thể tự thiết kế mô hình riêng để đáp ứng nhu cầu. Vấn đề lớn nhất đó là giám sát và quản lí tài nguyên hệ thống.

##### Selective replication

- Chia nhỏ dữ liệu cho các replica, tương tự như việc phân vùng
- Cách đơn giản nhất là phân vùng dữ liệu thành các bảng, rồi replicate các bảng với các replica khác nhau. Ví dụ: `replicate_wild_do_table = sales.%`
- Sử dụng distribution master để lọc và chia dữ liệu cho các replica.

##### Seperating functions

- Nhiều ứng dụng sử dụng xử lí giao dịch trực tuyến (OLTP) và xử lí phân tích trực tuyến (OLAP) (truy vấn OLTP thường nhỏ, truy vấn OLAP thường lớn và chậm). Sử dụng cả 2 loại truy vấn trên cùng 1 server sẽ làm giảm hiệu năng của server.
- Giải pháp: sử dụng replica để thực thi truy vấn OLAP.

##### Data archiving

- Có thể cất trữ dữ liệu bằng cách lưu dữ liệu trên replica và xoá nó trên master. Có 2 cách để thực hiện:
    - Thực hiện `SET SQL_LOG_BIN=0` trước quá trình xoá dữ liệu khỏi master. Ưu điểm: không cần cấu hình trên replica. Nhược điểm: không có khả năng phục hồi dữ liệu trên master.
    - Sử dụng `USE` tới 1 bảng được replica ignore (`replicate_ignore_db=purge`). Ưu điểm: khắc phục nhược điểm của cách 1. Nhược điểm: có khả năng có người thực hiện truy vấn trên bảng được ignore.
    - Sử dụng `binlog_ignore_db` (không dùng vì nguy hiểm không phục hồi được dữ liệu)

##### Use replicas for full-text searches

- Sử dụng replica với storage engine hỗ trợ full-text search. (MyISAM hỗ trợ full-text search nhưng không hỗ trợ transaction)

##### Read-only replicas

- Cấu hình read-only cho replica sẽ chặn những sự thay đổi không mong muốn lên replica.

##### Emulating multisource replication

- Cách 1: chia thành các khoảng thời gian để replica lấy dữ liệu từ các master, ví dụ: khoảng 1 lấy từ master 1, khoảng 2 lấy từ master 2. Khó khăn khi phân chia khoảng thời gian.
- Cách 2: sử dụng master-master (ring)

##### Creating a log server

- Tạo 1 server không có dữ liệu mà chỉ lưu trữ log, nhằm mục đích phục hồi dữ liệu nhanh chóng.
- Có 2 cách sao lưu binlog/relay log:
    - Cách 1: sử dụng `mysqlbinlog`
    - Cách 2: sử dụng replica
- Ưu điểm của replica với `mysqlbinlog`:
    - Replication nhanh hơn vì không phải trích xuất các câu lệnh từ log.
    - Có thể theo dõi quá trình replication
    - Replication bỏ qua các lệnh bị lỗi khi replicate
    - `mysqlbinlog` có thể không đọc được binlog và thay đổi cấu trúc file log

## Replication and Capacity Planning

- Việc ghi thường gây nghẽn cổ chai khi replication.
- Giả sử, khối lượng công việc là 20% ghi và 80% đọc, và có các điều kiện sau:
    - Truy vấn ghi và đọc có cùng khối lượng công việc
    - Mọi server đều phục vụ được 1000 truy vấn 1 giây
    - Replica và master có cùng hiệu năng
    - Có thể chuyển toàn bộ truy vấn đọc cho replica
- Nếu có 1 server xử lí 1000 truy vấn/giây, cần bao nhiêu replica để tăng khả năng xử lí lên gấp đôi.
    - Tổng 2000 truy vấn: 1600 đọc, 400 ghi
    - Với 2 replica, mỗi replica 800 đọc, 1 master 400 ghi. Nhưng mỗi replica cũng cần 400 ghi để đồng bộ với master
    - Với 3 replica, mỗi replica 550 đọc, 400 ghi
- Nếu tăng lên gấp 4: 3200 đọc, 800 ghi. Mỗi replica: 200 đọc, 800 ghi => cần 16 replica

### Why Replication Doesn't Help Scale Writes

- Phân vùng dữ liệu => đề cập ở phần sau
- Sử dụng Master-Master in Active-Active Mode => xung đột dữ liệu.

### When Will Replicas Begin to Lag?

- Khó để xác định replica bắt đầu không bắt kịp được master. Ta có thể dựa vào 1 số dấu hiệu để đánh giá khả năng của replica
    - Tăng vọt về độ trễ: bạn có thể thấy 1 vài điểm tăng vọt trên đồ thị biểu diễn độ trễ. Khi khối lượng công việc tăng tới gần khả năng tối đa của replica, các điểm tăng vọt này sẽ mở rộng ra => dấu hiệu cho thấy hệ thống đang gần đạt tới giới hạn.
    - Dừng replica trong 1 khoảng thời gian và đo thời gian replica bắt kịp được master. Nếu bạn dừng replica trong 1h và replica bắt kịp master sau 1h, replica đang chạy với một nửa công suất.
    - Percona Server và MariaDB cho phép đánh giá việc sử dụng replication. 

### Plan to Underutilize

- Việc sử dụng dưới công suất sẽ giúp hệ thống chịu được các đợt tăng đột ngột, xử lí các truy vấn lớn, bảo trì hệ thống, đảm bảo khả năng đồng bộ dữ liệu.
- Làm tăng hiệu năng replication bằng master-master là cách làm không hiệu quả. Master-master cần để ít hơn 50% read, tránh quá tải khi mất 1 master. Nếu cả 2 master có thể xử lí được, cũng không cần phải lo lắng và tải khi replication

## Replication Administration and Maintenance

### Monitoring Replication

- Replication làm tăng độ phức tạp khi giám sát 

### Measuring Replication Lag

### Determining Whether Replicas Are Consistent with the Master

### Resyncing a Replica from the Master

### Changing Masters

#### Planned Promotion

#### Unplanned Promotion

#### Locating the desired log positions

### Switching Roles in a Master-Master Configuration

## Replication Problems and Solutions

### Error Caused by Data Corruption or Loss

### Using Nontransactional Tables

### Mixing Transactional and Nontransactional Tables

### Nondeterministic Statements

### Different Storage Engine on the Master and Replica

### Data Changes on the Replica

### Nonunique Server IDs

### Undefined Server IDs

### Dependencies on Nonreplicated Data

### Missing Temporary Tables

### Not Replicating All Updates

### Lock Contention Caused by InnoDB Locking Selects

### Writing to Both Masters in Master-Master Replication

### Excessive Replication Lag

#### Don't duplicate the expensive part of writes

#### Do writes in parallel outside of replication

#### Prime the cache for the replication thread

### Oversized Packets from the Master

### Limited Replication Bandwidth

### No Disk Space

### Replication Limitations

## How Fast is Replication?



## Advanced Features in MySQL Replication



## Other Replication Technologies



## Summary
