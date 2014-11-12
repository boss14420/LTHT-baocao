# Hệ thống file F2FS

f2fs (flash-friendly file system) là một hệ thống file mới dành cho Linux, dành
cho các thiết bị lưu trữ sử dụng bộ nhớ flash. Khác với các hệ thống file khác
như jffs2, logfs vốn dành cho “*raw flash*”, f2fs dành cho các phần cứng như
SSDs, eMMC, SD card và những thiết bị nhớ flash có `FTL` (Flash translation
layer). Trong quá trình hoạt động, f2fs sẽ để một phần công việc cho FTL thay
vì tự làm từ đầu.

Lý do mà f2fs được gọi là “*flash friendly*” là vì:

- Khi có nhiều block cần được ghi cùng một lúc, f2fs thu thập lại thành 
thao tác ghi lớn (large-scale), làm cho FTL xử lý dễ dàng hơn. Ngoài ra, thay
vì chỉ một thao tác ghi, f2fs tạo ra 6 thao tác ghi song song, giúp tận dụng
khả năng tính song song của các thiết bị nhớ flash.
- Các cấu trúc dữ liệu trên thiết bị nhớ được “*align*” theo FTL.

## Block, segment, section, zone

Đơn vị cơ bản nhất của f2fs là `block`, 1 block = 4KiB, bằng với kích thước một
trang bộ nhớ của hệ điều hành. Địa chỉ mỗi block được biểu diễn bởi số nguyên
32bit. Như vậy kích thước tối đa của một phân vùng là 2^(12+32)B = 16TiB.

Các block được hợp lại thành một `segment`, 1 segment = 512 block = 2MiB. Mỗi
segment có một block chứa thông tin về *owner* và *offset* của các block khác, gọi là
`segment summary block`. block này được dùng khi *dọn dẹp* để xác định xem
block nào cần phải *relocate* và cách cập nhật thống tin ngay sau đó.

Nhiều segment hợp thành một `section`, số lượng segment trên một `section` được
đặt khi tạo phân vùng, nhưng phải là lũy thừa của 2. Mặc định là 2^0 = 1
segment = 1 section. Một section tương ứng với một “*vùng (region)*” trong *log
structuring*.

f2fs cho phép 6 section được ghi cùng một lúc với các dữ liệu khác nhau trên
mỗi section. Việc phân chia thành các section cho phép dữ liệu 
