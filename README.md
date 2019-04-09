# project2
# CẤU HÌNH NFS CHIA SẺ FILE TRÊN UBUNTU 18.04
- Yêu cầu: 
  - Máy chủ ubuntuserver có địa chỉ ip `10.10.10.11/24`
  - Client1 có địa chỉ ip `10.10.10.4/24`
  - Client2 có địa chỉ ip `10.10.10.15/24`
 ## CẤU HÌNH NFS SERVER
 Bạn cần cài đặt gói nfs-kernel-server bằng cách gõ lệnh sau:
**Bước 1: Cài đặt NFS Server**
```sh
#sudo apt-get update
#sudo apt-get install nfs-kernel-server
```
**Bước 2: Tạo thư mục chia sẻ trên máy chủ**

– Trên server (10.10.10.11) tạo ra thư mục cho mục đích chia sẻ chung và gán quyền cho thư mục:
```sh
#sudo mkdir -p /share/testdata
 
#sudo chown -R nobody:nogroup /share/testdata
```
=> khi gán quyền sang nobody:nogroup thì superusers tại client sẽ không thể thực hiện hành động quản trị điển hình, như thay đổi chủ sở hữu của một tập tin hoặc tạo ra một thư mục mới cho một nhóm người dùng, trên dữ liệu chia sẻ NFS-mounted này.

– Exports (/etc/exports) thư mục đó để cho phép chia sẻ, chỉnh sửa file /etc/exports bằng cách gõ lệnh:
```sh
#sudo nano /etc/exports
```
Tạo ra một dòng cho mỗi thư mục mà chúng ta dự định chia sẻ.
```sh
share/testdata 10.10.10.4/24(rw,async,no_subtree_check,no_root_sqhash)
share/testdata 10.10.10.15/24(rw,async,no_subtree_check,no_root_sqhash)
```
-Trong đó, ý nghĩa của một số tùy chọn:
  - **rw**: Tùy chọn này cho phép máy tính client truy cập cả đọc và viết vào bộ đĩa (volume).
  - **sync**: Tùy chọn này bắt buộc NFS phải ghi các thay đổi vào đĩa trước khi trả lời. Điều này dẫn đến một môi trường ổn định và phù hợp hơn kể từ khi trả lời phản ánh tình trạng thực tế của bộ đĩa (volume) từ xa. Tuy nhiên, nó cũng làm giảm tốc độ của hoạt động tập tin.
  - **no_subtree_check**: tùy chọn này ngăn cản việc kiểm tra cây con, đó là một quá trình mà host phải kiểm tra xem các tập tin thực sự vẫn có sẵn trong cây xuất cho mỗi yêu cầu.
  - **no_root_squash** : Theo mặc định, NFS chuyển yêu cầu từ người dùng root từ xa vào một người dùng không có đặc quyền trên máy chủ. Điều này đã được dự định như là tính năng bảo mật để ngăn chặn một tài khoản root trên máy khách (client) sử dụng hệ thống tập tin của máy chủ như là root.
  
Sau đó khởi động lại dịch vụ nfs-kernel-server:
```sh
#sudo systemctl restart nfs-kernel-server
```

## CẤU HÌNH NFS CLIENT
**Bước 1: Cài đặt NFS Client**
cài đặt gói nfs-common sẽ bao gồm NFS Client, mà không bao gồm các thành phần máy chủ không cần thiết:
```sh
#sudo apt-get update
#sudo apt-get install nfs-common
```
**Bước 2: Tạo điểm kết nối trên Client** 
Tạo ra thư mục trên Client và kết nối (mount) thư mục này đến thư mục chia sẻ trên Server.
– Tạo ra thư mục rỗng (không chứa dữ liệu sẵn có):
```sh
#sudo mkdir -p /storage/nfs/testdata 
```
– Thực hiện mount đến thư mục chia sẻ trên Server:
```sh
#sudo mount 10.10.10.11:/share/testdata /storage/nfs/testdata/
```
Dùng lệnh kiểu tra việc gắn kết đã thành công chưa: 
```sh
#df -h
```
Nếu thấy có thông tin về thư mục mount thì bạn đã kết nối thành công. Ngoài ra, để xem có bao nhiêu không gian thật sự (dung lượng) đang được sử dụng theo từng điểm gắn kết (mount), sử dụng lệnh sau:
```sh
#du -sh /storage/nfs/testdata
```

## 3. Tháo gỡ các điểm gắn kết trên NFS Client
Nếu bạn không còn muốn các thư mục từ xa được gắn kết vào hệ thống trên NFS Server, bạn có thể gỡ bỏ nó bằng cách unmounting, thực hiện lệnh sau:
```sh
#sudo umount /storage/nfs/data
```
Bạn có thể kiểm tra kết quả bằng cách dùng lệnh: `df -h`
Nếu cấu hình này đã được lưu để kích hoạt khi Server khởi động (được mô tả ở bước 4), bạn cần chỉnh sửa file /etc/fstab và xóa các dòng cấu hình cho thư mục này.

## 4. Tự động mounted thư mục khi Server khởi động
Bạn phải cấu hình /etc/fstab để lưu lại các kết nối từ xa đến NFS Server mỗi khi bạn khởi động lại máy client.
– Thực hiện chỉnh sửa file /etc/fstab
```sh
#sudo nano /etc/fstab
```
– Thêm vào cấu hình như sau:
```sh
#10.10.10.11:/share/testdat /storage/nfs/testdata nfs auto,nofail,notime,nolock,intr,tcp,actimeo=1800 0 0
```
Khi đó, các máy client sẽ tự động gắn kết (mount) các phân vùng từ xa lúc khởi động.
## 5. Thử nghiệm
- Từ Client 1, tạo file Test.txt trong mục /storage/nfs/testdata bằng câu lệnh:
```sh
#sudo touch /storage/nfs/testdata/Test.txt
```
- Tại Client 2, mở thư mục /storage/nfs/testdata kiểm tra xem đã có file Test.txt được tạo chưa:
```sh
#cd /storage/nfs/testdata
#ls
```
## 6. Kết luận
- Dịch vụ NFS cho phép chia sẻ tập tin cho nhiều người dùng trên cùng mạng và người dùng có thể thao tác như với tập tin trên chính đĩa cứng của mình.
- Dịch vụ NFS cho phép các NFS client mount một phân vùng của NFS server như phân vùng cục bộ của nó.
- Tuy nhiên dịch vụ NFS không được security nhiều, vì vậy cần thiết phải tin tưởng các client được permit mount các phân vùng của NFS server.
 
