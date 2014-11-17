# Bộ nhớ flash

## Cấu trúc của SSD

SSD là một thiết bị nhớ sử dụng bộ nhớ NAND flash. Trong SSD, nhiều *flash
memory package* được kết nối song song với bộ điều khiển SSD (*SSD Controller*)
qua các *channel*, dữ liệu có thể cùng lúc đi qua các channel đến các package,
cho phép truy cập song song.

![ssd-architecture](figures/ssd-architecture.jpg)

Bộ điều khiển SSD là trung gian giữa các module bộ nhớ flash và host (thiết bị
sử dụng SSD). Nó cung cấp các chức năng cơ bản của SSD để HDH sử dụng như đọc,
ghi, xóa, ... 

### Lớp chuyển đổi flash - FTL
Các hệ điểu hành ngày nay sử dụng khái niệm *Logical Block Address* (LBA) để
đánh địa chỉ cho từng khối của thiết bị nhớ. Với HDD thì do các sector luôn có
thể ghi đè lên giống như ghi mới, nên mỗi LBA luôn trỏ tới một vị trí cố định
trên đĩa. Tuy nhiên với SSD thì điều này không đúng: do SSD không thể ghi đè,
đồng thời cần phải phân bố việc ghi một cách đều đặn đối với các flash cell nên
có thể lúc đầu một file ở vị trí (vật lý) này, nhưng sau khi ghi đè thì nó bị
chuyển sang vị trí (vật lý) khác, tức là một LBA có thể cần trỏ tới nhiều vị trí
khác nhau trong SSD. Do vậy cần phải có một cơ chế để ánh xạ giữa vị trí vật
lý và LBA sao cho khi HDH yêu cầu thao tác tới một LBA thì SSD phải chỉ để đúng
vị trí vật lý tương ứng.

Thành phần này gọi là *lớp chuyển đổi flash* (*flash translation layer* - FTL).
FTL có hai nhiệm vụ chính: ánh xạ khối logic và thu dọn rác (garbage
collection).

#### Ánh xạ khối logic

FTL chuyển đổi từ LBA ở HDH sang PBA (physical block address) trên các flash
package tương ứng. Ánh xạ này được lưu vào một bảng ở trong bộ nhớ RAM của SSD
để tăng tốc độ truy cập, và lưu cả vào trong bộ nhớ flash để đọc lại mỗi khi
khởi động.

#### Thu dọn rác (garbage collection)

