# Log-structure file system (LFS)

Trong các hệ thống file ghi nhật ký (journaling file system), trước khi thực
hiện một thay đổi, một tóm tắt về thao tác đó luôn được ghi vào log, vốn
được lưu trong đĩa hoặc bộ nhớ NVRAM. Tóm tắt này (log record) chứa thông tin
để thực hiện lại toàn bộ các thao tác nếu như việc ghi sau đó bị hỏng giữa
chừng. Thao tác này gọi là `replay`. Như vậy, về cơ bản mọi thay đổi trong hệ
thống file ghi nhật kí đều được ghi 2 lần: một lần vào log, và một lần ghi
thật.

Johm K. Ousterhout và các cộng sự nhận thấy rằng có thể bỏ qua lần ghi thứ hai
nếu coi hệ thống file như là một log lơn. Như vậy thay vì ghi tóm tắt vào log
và ghi thực sự vào đĩa, ta chỉ cần phải ghi một lần vào log. Thao tác ghi (đè)
một file và inode là `copy-on-write`: phiên bản cũ được đánh dấu là vùng tự do,
và phiên bản mới được ghi vào cuối log. Như vậy, tìm trạng thái hiện tại của
hệ thống file tương đương với việc `replay` log từ đầu đến cuối. Trong thực tế,
LFS ghi `checkpoint` vào đĩa thường xuyên, checkpoint mô tả trạng thái của hệ
thống file tại thời điểm ghi mà không cần phải `replay` log. Mọi thay đổi của
hệ thống file kể sau đó có thể khôi phục bằng cách replay một số lượng
nhỏ log tính từ checkpoint đó.

Một trong những lợi ích của LFS là các thao tác ghi là tuần tự. Việc này rất
có ích cho bộ nhớ NAND flash. Với NAND flash, các thao tác ghi cần qua nhiều
bước: đầu tiên xóa toàn bộ block (mỗi block của NAND flash chiếm khoảng 2MiB)
tức là toàn bộ các bit được đưa về giá trị 1, sau đó mới ghi nội dung tương ứng
vào. Vì chi phi cho việc xóa là lớn (phải xóa theo từng block 2MiB) nên nếu hệ
thống file hoạt động theo kiểu ghi rải rác từng phần nhỏ kích thước vài KiB đến
vào chục KiB sẽ tốn rất nhiều lần xóa và ghi các block (chưa kể để chi phí để
di chuyển phần dữ liệu hợp lệ trong các block sẽ bị xóa). Nếu ghi tuần tự với
một lượng lớn thì số lần xóa sẽ giảm đi (giảm `write-amplification`).

# Hệ thống file F2FS

f2fs (flash-friendly file system) là một hệ thống file mới dành cho Linux, dành
cho các thiết bị lưu trữ sử dụng bộ nhớ flash. Khác với các hệ thống file khác
như jffs2, logfs vốn dành cho “*raw flash*”, f2fs dành cho các phần cứng như
SSDs, eMMC, SD card và những thiết bị nhớ flash có `FTL` (Flash translation
layer). Trong quá trình hoạt động, f2fs sẽ để một phần công việc cho FTL thay
vì tự làm từ đầu.

F2FS được thiết kế dựa trên LFS, vốn thích hợp cho bộ nhớ flash.

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
mỗi section. 6 section này bao gồm:

1. hot node
2. Warm mode
3. cold node
4. hot data
5. warm data
6. cold data

Các nội dung (dữ liệu) được phân vào 6 section này dựa trên một số đánh giá
heuristic. Sự phân chia thành các section cho phép dữ liệu (data) được tách rời so với
các thông tin chỉ mục (node). Ngoài ra việc phân chia thành “hot”, “warm”,
“cold” giúp cho các thao tác quản lý hoạt động hiệu quả hơn. Ví dụ: data được đánh giá là “cold” thường sẽ không thay
đổi nội dung trong một thời gian dài, do đó section với các cold block sẽ không
cần phải cleaning trong. Node được đánh giá là “hot” sẽ cần cập nhật thường
xuyên, do đó nếu đợi một thời gian thì một section với các hot node sẽ chỉ chứa
một số ít block là “live”, do đó thao tác clean sẽ đơn giản hơn.

Các section được hợp lại thành `zone`. Một zone có kích thước bằng một số
nguyên lần section, mặc định là 1. Mục đích của zone là phân 6 section “mở” đã
nói ở trên vào những vùng khác nhau trong thiết bị nhớ. Theo lý thuyết thì
thiết bị nhớ flash thường được tạo thành từ nhiều phần riêng biệt có thể hoạt
động độc lập với nhau (tức là có thể hoạt động song song).


## File, inode, đánh chỉ mục (indexing)

Nhiều hệ thống file hiện đại sử dụng B-trees hoặc các cấu trúc tương tự để quản
lý các chỉ mục của các block trong file, nhưng f2fs thì không. Nhiều hệ thống
file giảm kích thước của chỉ mục bằng cách sử dụng `extents` chứa thông tin về
điểm bắt đầu và kết thúc của một dãy các block liên tục thay vì chứa địa chỉ
của tất cả các block, nhưng f2fs cũng không sử dụng extents.

Cây chỉ mục của f2fs khác giống với các hệ thống file cũ trên Unix như *ext3*.
Mỗi *inode* chứa danh sách địa chỉ các block trong file, và một số địa chỉ gián
tiếp của của block (địa chỉ trỏ đến 1 node có chứa danh sách các địa chỉ), cũng
như các địa chỉ cấp 2, cấp 3. Mỗi inode của f2fs có dung lượng 4KiB, chứa 923
địa chỉ đến các data block, 2 địa chỉ đến `direct node`, 2 địa chỉ đến
`indirect node` và một địa chỉ đến `double indirect node`. Mỗi *direct node*
block chứa 1018 địa chỉ đến data block, và mỗi *indirect node block* chứa 1018
node block. Như vậy mỗi inode block có thể trỏ đến tối đa là:
    `4KiB * (923 + 2*1018 + 2*1018^2 + 1018^3) = 3.94TiB`

3.94TiB là dung lượng tối đa của một file trên hệ thống file f2fs.

![f2fsinode](figures/f2fsinode.png)

Tuy nhiều hệ thống file không có dùng nữa nhưng cách đánh chỉ mục file như trên
lại có ích lợi với 1 LFS. Vì f2fs không sử dụng extents, cây chỉ mục cho một
file ...

F2fs không sử dụng extend nhưng mỗi inode đều có một `extent` mô tả một số phần
trong cây chỉ mục, có chứa địa chỉ của một dãy các block liên tiếp nhau. F2fs
sẽ thử tìm extent lớn nhât có thể và sử dụng nó để tăng tốc quá trình tìm kiếm
địa chỉ. Trong trường hợp file được ghi tuần tự liên tục, toàn bộ file có thể
sẽ nằm trong extent đó, nhờ vậy mà không cần phải tìm kiếm trong cây chỉ mục.

Một điều bất tiện của của hệ thống file “copy-on-write” là khi một block bị ghi, địa
chỉ của nó thay đổi, do đó node cha trong cây chỉ mục cũng phải thay đổi và
“relocate”, cho đến gốc của cây.

Về vấn đề này, trong phần metadata của f2fs có một vùng gọi là `NAT` (*Node
Address Table*). *Node* ở đây nói đến các inode và *indirect indexing block*
và block lưu các thuộc tính mở rộng (xattr). Khi địa chỉ của một inode được lưu
vào một thư mục, hoặc một index block lưu trong một inode hoặc index block
khác, không phải là địa chỉ block được lưu, mà là offset của NAT. Địa chỉ thực
của block được lưu ở trong NAT tại offset đó. Có nghĩa là khi data block được
ghi, vẫn phải cập nhật các node trỏ tới nó, nhưng việc ghi node chỉ tương đương
với việc cập nhật NAT.

## Thư mục

Một tiêu chí quan trọng trong việc thiết kế cấu trúc thư mục của hệ thống file
là thời gian tìm kiếm theo tên nhanh, cung cấp địa chỉ ổn định (ít thay đổi)
cho mỗi tên (được trả về sử dụng hàm `telldir()`).

Nhiều hệ thống file hiện đại sử dụng B-trees để thể hiện cấu trúc thư mục. Một
trong những vấn đề của việc sử dụng B-trees là các *directory entry* nhiều lúc
cần di chuyển trong cây. Gây khó khăn cho việc `telldir()` trả về địa chỉ ổn
định.

F2fs sử dụng kết hợp giữa tìm kiếm tuần tự và băm (hashing), mô hình này cho
phép việc tìm kiếm đơn giản, hiệu quả và địa chỉ trả về bởi `telldir()` trở nên
ổn định.

### directory entry (dentry)
Một `dentry` chiếm 11 byte, bao gồm những thuộc tính sau:

1. `hash`: giá trị băm (hash) của tên file
2. `ino`: chỉ số inode
3. `len`: độ dài tên file
4. `type`: kiểu của file (directory, symlink, ...)

Một `dentry block` bao gồm 214 dentry và các tên file. Ngoài ra còn có một
bitmap (27 byte) để mô tả dentry tương ứng là hợp lệ hay không.

    dentry block (4KiB) = bitmap (27 byte) + reserved (3 byte) +
                          các dentry (11 * 214 byte) + tên file (8 * 214 bytes)

2 hoặc 4 dentry block liên tiếp nhau tạo thành một `bucket`.

### các bảng băm
F2fs sử dụng các `bảng băm nhiều mức` (*multi-level hash table*) cho cấu trúc
thư mục. Mỗi mức có một bảng băm với một số bucket xác định riêng.


| mức        | Các bucket                                   |
|------------|----------------------------------------------|
| mức #0     | A(2B)                                        |
| mức #1     | A(2B) - A(2B)                                |
| mức #2     | A(2B) - A(2B) - A(2B) - A(2B)                |
| mức #N/2   | A(2B) - A(2B) - A(2B) - A(2B) - ... - A(2B)  |
| mức #N     | A(4B) - A(4B) - A(4B) - A(4B) - ... - A(4B)  |

Trong đó:

- A : bucket
- B : block. A(2B) tức là một bucket chứa 2 block
- N : `MAX_DIR_HASH_DEPTH`

Số block và số bucket được xác định bởi:

- Số block trong một bucket ở mức #n = 2 nếu `n < MAX_DIR_HASH_DEPTH / 2`, còn
lại thì bằng 4,
- Số bucket ở mức #n = `2^(n + dir_level)` nếu `n + dir_level <
MAX_DIR_HASH_DEPTH / 2`, nếu không thì bằng `2^((MAX_DIR_HASH_DEPTH / 2) - 1`.
Trong đó `dir_level` là một hằng số được quy định tại lúc tạo phân vùng f2fs,
giá trị này lớn sẽ giúp tìm kiếm nhanh trong các thư mục lớn, nhưng sẽ tốn dung
lượng lưu trữ với các thư mục nhỏ, giá trị mặc định là 0.

Khi cần tìm một file theo tên, trước hết tính giá trị
hash của tên đó. Sau đó f2fs bắt đầu tìm kiếm từ mức #0 để tìm dentry với cùng
tên lấy inode tương ứng. Nếu không, tiếp tục tìm kiếm với các mức cao hơn. Ở
mỗi mức, f2fs chỉ tìm kiếm duy nhất một bucket tương ứng với giá trị hash đã
tính, theo phương trình sau:

    `vị trí bucket cần tìm tại mức #n = hash % số bucket tại mức #n`

Như vậy, độ phức tạp tính toán của việc tìm kiếm file là `O(log(số file))`.

Khi cần tạo file mới với tên cho trước, f2fs tìm *slot* trống trong các dentry
block mà có thể chứa đủ tên file.

Với cấu trúc thư mục như thế này, khi khi thêm hoặc xóa một file trong thư mục
thì chỉ có một block cần phải cập nhật. Ngoài ra vì các `dentry` không bao giờ
bị di chuyển, offset của nó không bao giờ thay đổi, do đó địa chỉ trả về bởi
`telldir()` là ổn định.

## Superblock, checkpoint và các siêu dữ liệu khác
![metadata](figures/f2fsmetadata.png)

F2fs cũng có `superblock` như nhiều hệ thống file khác, nhưng điểm khác biệt là
superblock của f2fs không thay đổi nội dung từ lúc phân vùng được tạo. Nó chứa
thông tin về kích thước phân vùng, kích thước mỗi segment, section, zone, không
gian bộ nhớ dành cho các siêu dữ liệu (metadata) khác.

Những thông tin như không gian còn trống, địa chỉ của segment tiếp theo sẽ được
ghi, và những thông tin có thay đổi được (volatile) khác được lưu trong
`checkpoint`. Vì f2fs là hệ thống file copy-on-write, có 2 segment liên tiếp
nhau trong hệ thống đều lưu checkpoint, trong đó chỉ có một là checkpoint hiện
tại hợp lệ. Các checkpoint được chứa số phiên bản, khi mount, checkpoint nào có số phiên
bản lớn hơn sẽ là checkpoint hiện tại.

Ngoài SSA và NAT đã đề cập đến ở các phần trước, trong metadata của f2fs còn
chứa một phần gọi là *SIT* (*Segment Info Table*).

SIT chứa thông tin về số lượng block hợp lệ (có thể ghi được) và bitmap về tính
hợp lệ cho tất cả các block của một segment.

Những thành phần như checkpoint, SIT, NAT, SSA đều được căn chỉnh (*align*)
theo kích thước của một segment.

Khi cần cập nhật NAT hoặc SIT, f2fs không cập nhật ngay, mà lưu vào bộ nhớ cho
đến khi checkpoint tiếp theo được ghi ... (còn tiếp)

## Cleaning (dọn dẹp)

Hệ thống file LFS sau một thời gian hoạt động sẽ bị “phân mảnh”: các segment
bị xóa xen lẫn với các segment chứa các dữ liệu hợp lệ do việc ghi trong LFS
là copy-on-write. Để có các không gian trống lớn để thích hợp cho việc ghi tuần
tự sau này thì cần phải thường xuyên dọn dẹp (cleaning).

F2fs thực hiện việc dọn dẹp khi có yêu cầu và chạy nền. Khi không có đủ segment
trống cho việc thực hiện các lời gọi VFS, sẽ có yêu cầu dọn dẹp. Việc chạy nền
được thực hiện bởi một kernel thread, khi hệ thống rỗi (idle).

F2fs sử dụng hai thuật toán: tham lam (gready) và cost-benefit. Trong thuật
toán tham lam, f2fs chọn segment có ít block hợp lệ nhất. Trong thuật toán
cost-benefit, f2fs chọn segment theo thời gian và cả số block hợp lệ.
