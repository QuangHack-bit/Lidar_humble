# Hướng dẫn chi tiết đọc LiDAR URG-04LX-UG01 bằng `urg_node` trên ROS 2 Jazzy

Hướng dẫn này đi theo **một con đường duy nhất**: cài `urg_node` qua `apt`, chạy node, và hiển thị trên RViz2. Bạn hãy mở **5 terminal** và làm đúng từng bước theo thứ tự.

Giả định:
- Hệ điều hành: **Ubuntu 24.04** với **ROS 2 Jazzy** đã cài sẵn.
- LiDAR: **Hokuyo URG-04LX-UG01** (kết nối USB).
- Khi cắm LiDAR, thiết bị xuất hiện ở `/dev/ttyACM0`.

---

## TERMINAL 1: Chuẩn bị môi trường và cài đặt `urg_node`

Mở **Terminal 1** và thực hiện từng lệnh bên dưới.

### Bước 1.1: Cập nhật danh sách package

```bash
sudo apt update
```

**Kết quả mẫu:**

```text
Hit:1 http://archive.ubuntu.com/ubuntu noble InRelease
Hit:2 http://packages.ros.org/ros2/ubuntu noble InRelease
...
Reading package lists... Done
Building dependency tree... Done
```

---

### Bước 1.2: Cài đặt các công cụ cần thiết

```bash
sudo apt install -y ros-dev-tools python3-rosdep git build-essential
```

**Kết quả mẫu:**

```text
Reading package lists... Done
...
Setting up ros-dev-tools (2.0.3-1) ...
Setting up python3-rosdep (0.22.0-1) ...
Processing triggers for man-db ...
```

---

### Bước 1.3: Khởi tạo và cập nhật `rosdep`

```bash
sudo rosdep init
rosdep update
```

**Kết quả mẫu:**

```text
ERROR: default sources list file already exists: /etc/ros/rosdep/sources.list.d/20-default.list
    Please delete if you wish to re-initialize
reading in sources list data from /etc/ros/rosdep/sources.list.d
Hit https://raw.githubusercontent.com/ros/rosdistro/master/rosdep/osrf.list
...
Query rosdistro index https://raw.githubusercontent.com/ros/rosdistro/master/index-v4.yaml
update complete
```

> Nếu `rosdep init` báo "already exists" như trên thì **không sao**, bỏ qua và tiếp tục.

---

### Bước 1.4: Cài đặt `urg_node` qua apt

```bash
sudo apt install -y ros-jazzy-urg-node
```

**Kết quả mẫu:**

```text
Reading package lists... Done
Building dependency tree... Done
...
Setting up ros-jazzy-urg-c (1.0.405-1noble.20240922.071500) ...
Setting up ros-jazzy-urg-node (1.1.1-2noble.20240922.080000) ...
Processing triggers for libc-bin ...
```

---

### Bước 1.5: Kiểm tra package đã cài

```bash
source /opt/ros/jazzy/setup.bash
ros2 pkg list | grep urg_node
```

**Kết quả mẫu:**

```text
urg_node
```

---

### Bước 1.6: Kiểm tra executable trong package

```bash
ros2 pkg executables urg_node
```

**Kết quả mẫu:**

```text
urg_node urg_node_driver
```

> Tên executable có thể là `urg_node_driver` hoặc `urg_node`. Hãy ghi nhớ tên này để dùng ở **Terminal 3**.

---

## TERMINAL 2: Cấp quyền USB và kiểm tra LiDAR

Mở **Terminal 2** và thực hiện các bước sau.

### Bước 2.1: Thêm user vào nhóm `dialout`

```bash
sudo usermod -aG dialout $USER
```

**Kết quả mẫu:**

```text
(không có output, trở về dấu nhắc lệnh)
```

> **Quan trọng:** Bạn **phải đăng xuất (Log Out) rồi đăng nhập lại** để quyền có hiệu lực. Hoặc đơn giản nhất là **reboot máy** rồi mở lại 5 terminal.

---

### Bước 2.2: Kiểm tra user có trong nhóm `dialout`

Sau khi reboot / đăng nhập lại:

```bash
groups
```

**Kết quả mẫu:**

```text
your_user adm cdrom sudo dip plugdev lpadmin sambashare dialout
```

> Phải có chữ `dialout` trong danh sách. Nếu không thấy, bạn chưa đăng xuất/đăng nhập lại.

---

### Bước 2.3: Cắm LiDAR URG-04LX-UG01 vào máy tính

Cắm cáp USB của LiDAR vào cổng USB của máy tính.

---

### Bước 2.4: Kiểm tra kernel có nhận LiDAR không

```bash
dmesg | tail -n 15
```

**Kết quả mẫu:**

```text
[  123.456789] usb 1-2: new full-speed USB device number 5 using xhci_hcd
[  123.678901] usb 1-2: New USB device found, idVendor=15d1, idProduct=0000
[  123.678905] usb 1-2: Product: URG-04LX
[  123.678908] usb 1-2: Manufacturer: HOKUYO AUTOMATIC CO., LTD.
[  123.789012] cdc_acm 1-2:1.0: ttyACM0: USB ACM device
```

> Dòng quan trọng nhất là `ttyACM0: USB ACM device`. Nếu thấy dòng này, Linux đã nhận LiDAR.

---

### Bước 2.5: Kiểm tra thiết bị xuất hiện

```bash
ls -l /dev/ttyACM*
```

**Kết quả mẫu:**

```text
crw-rw---- 1 root dialout 166, 0 Sep 29 10:15 /dev/ttyACM0
```

> Phải thấy `/dev/ttyACM0` với owner là `root` và group là `dialout`.

---

### Bước 2.6: Kiểm tra quyền đọc/ghi

```bash
test -r /dev/ttyACM0 && test -w /dev/ttyACM0 && echo "OK: có quyền đọc/ghi" || echo "LỖI: không có quyền"
```

**Kết quả mẫu:**

```text
OK: có quyền đọc/ghi
```

> Nếu báo `LỖI`, bạn quay lại Bước 2.1 và 2.2 để xử lý quyền.

---

## TERMINAL 3: Chạy `urg_node`

Mở **Terminal 3** (terminal này sẽ chạy node và giữ cho đến khi bạn tắt).

### Bước 3.1: Source ROS 2

```bash
source /opt/ros/jazzy/setup.bash
```

**Kết quả mẫu:**

```text
(không có output)
```

---

### Bước 3.2: Chạy `urg_node`

> Thay `urg_node_driver` bằng tên executable bạn thấy ở Bước 1.6.

```bash
ros2 run urg_node urg_node_driver --ros-args \
  -p serial_port:=/dev/ttyACM0 \
  -p frame_id:=laser
```

**Kết quả mẫu:**

```text
[INFO] [1727590234.123456789] [urg_node]: Connecting to device on port /dev/ttyACM0
[INFO] [1727590234.234567890] [urg_node]: Device initialized successfully
[INFO] [1727590234.345678901] [urg_node]: Starting scan publishing
[INFO] [1727590234.456789012] [urg_node]: Publishing LaserScan on topic /scan
```

> Terminal này sẽ **không trả về dấu nhắc lệnh**. Nó đang chạy node `urg_node`. **Giữ nguyên terminal này**, không đóng.

Nếu bạn thấy lỗi như `Failed to open port`, quay lại Terminal 2 để kiểm tra quyền và thiết bị.

---

## TERMINAL 4: Kiểm tra dữ liệu LiDAR

Mở **Terminal 4** (để kiểm tra dữ liệu, giữ chạy song song).

### Bước 4.1: Source ROS 2

```bash
source /opt/ros/jazzy/setup.bash
```

**Kết quả mẫu:**

```text
(không có output)
```

---

### Bước 4.2: Kiểm tra danh sách node đang chạy

```bash
ros2 node list
```

**Kết quả mẫu:**

```text
/urg_node
```

---

### Bước 4.3: Kiểm tra danh sách topic

```bash
ros2 topic list
```

**Kết quả mẫu:**

```text
/parameter_events
/rosout
/scan
```

> Topic `/scan` phải xuất hiện. Đây là topic chứa dữ liệu LiDAR.

---

### Bước 4.4: Kiểm tra tần số publish của `/scan`

```bash
ros2 topic hz /scan
```

**Kết quả mẫu (sau vài giây):**

```text
average rate: 10.005
    min: 0.099s max: 0.101s std dev: 0.00045s window: 11
average rate: 10.003
    min: 0.099s max: 0.101s std dev: 0.00042s window: 21
```

> Tần số khoảng **10 Hz** là bình thường với URG-04LX. Dùng `Ctrl+C` để thoát lệnh này khi đã kiểm tra xong.

---

### Bước 4.5: Xem thông tin topic `/scan`

```bash
ros2 topic info /scan --verbose
```

**Kết quả mẫu:**

```text
Type: sensor_msgs/msg/LaserScan
Publisher count: 1
Node name: urg_node
Node namespace: /
Topic type: sensor_msgs/msg/LaserScan
Endpoint type: PUBLISHER
GID: ...
QoS profile:
  Reliability: RELIABLE
  History (Depth): KEEP_LAST (10)
  Durability: VOLATILE
  Lifespan: Infinite
  Deadline: Infinite
  Liveliness: AUTOMATIC
  Liveliness lease duration: Infinite
Subscription count: 0
```

---

### Bước 4.6: Xem nội dung một message `/scan`

```bash
ros2 topic echo /scan --once
```

**Kết quả mẫu:**

```text
header:
  stamp:
    sec: 1727590500
    nanosec: 123456789
  frame_id: laser
angle_min: -2.094395160675049
angle_max: 2.094395160675049
angle_increment: 0.006135923322290182
time_increment: 9.999999747378752e-05
scan_time: 0.10000000149011612
range_min: 0.019999999552965164
range_max: 5.599999904632568
ranges:
- 1.234
- 1.240
- 1.238
- ... (hơn 600 giá trị)
intensities: []
---
```

> - `frame_id: laser` phải khớp với tham số `-p frame_id:=laser` bạn truyền ở Terminal 3.
> - `ranges` phải chứa các giá trị số thực (mét), không phải toàn `inf` hay `nan`.

---

## TERMINAL 5: Hiển thị dữ liệu LiDAR trên RViz2

Mở **Terminal 5** để chạy RViz2.

### Bước 5.1: Source ROS 2

```bash
source /opt/ros/jazzy/setup.bash
```

**Kết quả mẫu:**

```text
(không có output)
```

---

### Bước 5.2: Khởi chạy RViz2

```bash
rviz2
```

**Kết quả mẫu:**

```text
[INFO] [1727590600.123456789] [rviz2]: rviz version 14.x.x
[INFO] [1727590600.234567890] [rviz2]: Compiled against Qt version 5.15.x
[INFO] [1727590600.345678901] [rviz2]: Compiled against OGRE version 1.x.x
```

Cửa sổ **RViz2** sẽ mở ra trên màn hình.

---

### Bước 5.3: Cấu hình RViz2 để hiển thị LiDAR

Trong cửa sổ RViz2, thực hiện lần lượt các thao tác sau:

#### 5.3.1: Đặt Fixed Frame

- Ở panel bên trái, tìm ô **Global Options → Fixed Frame**.
- Mặc định đang là `map`.
- **Click vào ô đó**, xóa `map`, gõ `laser`, rồi nhấn **Enter**.

**Kết quả:** Ô Fixed Frame hiển thị `laser`. Không còn cảnh báo màu đỏ ở status bar.

---

#### 5.3.2: Thêm display LaserScan

- Ở panel **Displays** bên trái, click nút **Add** (phía dưới).
- Cửa sổ "Add a new display" hiện ra.
- Chọn tab **By topic**.
- Tìm dòng `/scan` → chọn **LaserScan** (có biểu tượng dấu cộng).
- Nhấn **OK**.

**Kết quả:** Trong panel Displays xuất hiện mục mới tên `LaserScan`.

---

#### 5.3.3: Cấu hình display LaserScan

- Click vào tam giác nhỏ bên cạnh `LaserScan` để mở rộng.
- Kiểm tra các trường:
  - **Topic**: `/scan`
  - **Reliability Policy**: `Best Effort` hoặc `Reliable` (thử cả hai nếu không thấy điểm).
  - **Style**: `Points` hoặc `Flat Squares`.
  - **Size (m)**: `0.05` (độ lớn mỗi điểm).
  - **Color Transformer**: `FlatColor` hoặc `Intensity`.
  - **Color**: chọn màu đỏ hoặc xanh lá cho dễ nhìn.

**Kết quả:** Bạn sẽ thấy các **chấm màu** hiển thị trong khung nhìn 3D ở giữa, tạo thành hình vòng cung 240° đại diện cho dữ liệu quét của URG-04LX.

---

### Bước 5.4: Xác nhận hiển thị thành công

Ở góc dưới bên trái cửa sổ RViz2, kiểm tra thanh **Status**:

- Nếu tất cả đều màu **xanh lá (OK)** → thành công.
- Nếu có mục **màu vàng hoặc đỏ**, kiểm tra lại:
  - `Fixed Frame` phải là `laser`.
  - Topic phải là `/scan`.
  - Thử đổi **Reliability Policy** sang `Best Effort` nếu đang là `Reliable` (hoặc ngược lại).

---

## TỔNG KẾT: Trạng thái 5 terminal sau khi hoàn thành

| Terminal | Nội dung đang chạy |
|---|---|
| **Terminal 1** | Đã cài xong `urg_node`, trả về dấu nhắc lệnh |
| **Terminal 2** | Đã kiểm tra xong thiết bị `/dev/ttyACM0`, trả về dấu nhắc lệnh |
| **Terminal 3** | Đang chạy `urg_node`, in log liên tục, **không đóng** |
| **Terminal 4** | Đã kiểm tra `/scan` xong, trả về dấu nhắc lệnh |
| **Terminal 5** | Đang chạy `rviz2`, cửa sổ RViz hiển thị các điểm LiDAR |

---

## Nếu gặp sự cố ở bước nào

### Trường hợp 1: Terminal 3 báo `Failed to open port /dev/ttyACM0`

Quay lại **Terminal 2** và chạy:

```bash
ls -l /dev/ttyACM0
groups
```

Nếu `groups` không có `dialout` → đăng xuất/đăng nhập lại hoặc reboot.

### Trường hợp 2: Terminal 4 không thấy topic `/scan`

Quay lại **Terminal 3** xem log có lỗi gì không. Nếu node đã thoát → chạy lại Bước 3.2.

### Trường hợp 3: RViz không hiển thị điểm dù có topic `/scan`

Trong RViz, đổi **Reliability Policy** của display `LaserScan`:
- Từ `Reliable` → `Best Effort`, hoặc
- Từ `Best Effort` → `Reliable`.

Kiểm tra lại **Fixed Frame** phải là `laser`.

### Trường hợp 4: `ranges` toàn `inf` hoặc `nan`

- Đảm bảo LiDAR không bị che sát mặt kính (vật thể quá gần dưới `range_min = 0.02 m`).
- Đảm bảo nguồn USB đủ (thử cổng USB khác, hoặc dùng hub USB có nguồn).
- Thử reboot LiDAR bằng cách rút cáp USB và cắm lại.

---

Như vậy bạn đã hoàn thành **toàn bộ quy trình từ đầu đến khi hiển thị được LiDAR URG-04LX-UG01 trên RViz2** thông qua `urg_node` trên ROS 2 Jazzy.
