Dưới đây là hướng dẫn chi tiết đọc LiDAR **Hokuyo URG-04LX-UG01** bằng **`urg_node`** trên **ROS 2 Jazzy**.

LiDAR này thường giao tiếp qua USB, khi cắm vào Linux sẽ xuất hiện dưới dạng:

```bash
/dev/ttyACM0
```

Driver:

```text
urg_node
```

Topic dữ liệu:

```text
/scan
```

Kiểu dữ liệu:

```text
sensor_msgs/msg/LaserScan
```

---

# 1. Tóm tắt nhanh các bước

```text
1. Source ROS 2 Jazzy
2. Cấp quyền USB/serial
3. Cắm LiDAR, kiểm tra /dev/ttyACM0
4. Cài urg_node
5. Chạy urg_node
6. Kiểm tra /scan
7. Hiển thị bằng rviz2
```

---

# 2. Chuẩn bị môi trường ROS 2 Jazzy

Mở terminal và chạy:

```bash
source /opt/ros/jazzy/setup.bash
```

Nếu muốn tự động mỗi lần mở terminal:

```bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
```

Cài các công cụ cần thiết:

```bash
sudo apt update
sudo apt install -y ros-dev-tools python3-rosdep git build-essential
```

Khởi tạo `rosdep` nếu chưa có:

```bash
sudo rosdep init || true
rosdep update
```

---

# 3. Cấp quyền truy cập cổng USB cho LiDAR

LiDAR Hokuyo USB thường xuất hiện dưới dạng `/dev/ttyACM0`. Để người dùng bình thường đọc được thiết bị, bạn thêm user vào nhóm `dialout`.

```bash
sudo usermod -aG dialout $USER
```

Sau đó:

- Log out khỏi Ubuntu.
- Đăng nhập lại.
- Hoặc reboot máy.

Kiểm tra:

```bash
groups
```

Nếu thấy nhóm `dialout` là được.

---

# 4. Cắm LiDAR và kiểm tra thiết bị

Cắm URG-04LX-UG01 vào máy tính.

Kiểm tra log kernel:

```bash
dmesg | tail -n 20
```

Bạn nên thấy các dòng liên quan đến:

```text
cdc_acm
ttyACM0
Hokuyo
```

Kiểm tra thiết bị:

```bash
ls -l /dev/ttyACM* 2>/dev/null
```

Nếu thấy:

```bash
/dev/ttyACM0
```

thì hệ thống đã nhận LiDAR.

Kiểm tra USB:

```bash
lsusb
```

Bạn có thể thấy thiết bị tên `Hokuyo` hoặc vendor ID `15d1`.

---

## Nếu không thấy `/dev/ttyACM0`

Thử các lệnh sau:

```bash
dmesg | grep -E 'ttyACM|cdc_acm|Hokuyo'
lsusb
```

Nguyên nhân thường gặp:

- Dây USB chỉ có sạc, không có dữ liệu.
- Cổng USB bị lỗi.
- LiDAR không đủ nguồn.
- Đang dùng máy ảo nhưng chưa passthrough USB.
- Đang dùng WSL2 nhưng chưa attach USB.

Nếu dùng WSL2 trên Windows, bạn cần dùng `usbipd-win` để attach thiết bị USB vào WSL.

---

# 5. Cài đặt `urg_node`

## Cách 1: Cài bằng apt nếu có sẵn

Đây là cách đơn giản nhất:

```bash
sudo apt update
sudo apt install -y ros-jazzy-urg-node
```

Kiểm tra package:

```bash
ros2 pkg list | grep urg
```

Bạn có thể thấy:

```text
urg_node
```

Kiểm tra executable:

```bash
ros2 pkg executables urg_node
```

Kết quả thường là:

```text
urg_node urg_node
```

Nếu cài bằng apt thành công, bạn bỏ qua phần build từ source bên dưới.

---

## Cách 2: Build từ source nếu apt không có package

Nếu lệnh:

```bash
sudo apt install ros-jazzy-urg-node
```

báo không tìm thấy package, bạn build từ source.

Tạo workspace:

```bash
mkdir -p ~/urg_ws/src
cd ~/urg_ws
```

Clone `urg_node`:

```bash
git clone https://github.com/ros-drivers/urg_node.git src/urg_node
```

Chuyển sang nhánh ROS 2 nếu cần:

```bash
cd src/urg_node
git checkout ros2 || git checkout ros2-devel || true
cd ../..
```

Cài dependency:

```bash
rosdep install --from-paths src --ignore-src -r -y
```

Build:

```bash
colcon build --symlink-install
```

Nếu quá trình build báo thiếu `urg_c`, clone thêm:

```bash
git clone https://github.com/ros-drivers/urg_c.git src/urg_c
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
```

Sau khi build xong:

```bash
source install/setup.bash
```

Kiểm tra:

```bash
ros2 pkg list | grep urg
ros2 pkg executables urg_node
```

---

# 6. Chạy `urg_node` cho URG-04LX-UG01

## 6.1. Source môi trường

Nếu bạn cài bằng apt:

```bash
source /opt/ros/jazzy/setup.bash
```

Nếu bạn build từ source:

```bash
source ~/urg_ws/install/setup.bash
```

---

## 6.2. Chạy node trực tiếp

Nếu LiDAR đang là `/dev/ttyACM0`:

```bash
ros2 run urg_node urg_node --ros-args \
  -p serial_port:=/dev/ttyACM0 \
  -p frame_id:=laser
```

Trong đó:

```text
serial_port: cổng của LiDAR
frame_id: tên frame dùng trong RViz và TF
```

Nếu cần chỉ định baudrate:

```bash
ros2 run urg_node urg_node --ros-args \
  -p serial_port:=/dev/ttyACM0 \
  -p serial_baud:=115200 \
  -p frame_id:=laser
```

Với kết nối USB kiểu `/dev/ttyACM0`, nhiều trường hợp không cần quan tâm baudrate vì đây là USB CDC ACM. Tuy nhiên nếu không chạy, bạn có thể thử:

```bash
-p serial_baud:=19200
```

hoặc:

```bash
-p serial_baud:=115200
```

---

## 6.3. Chạy với debug log

Nếu chạy không được, hãy bật debug:

```bash
ros2 run urg_node urg_node --ros-args \
  -p serial_port:=/dev/ttyACM0 \
  -p frame_id:=laser \
  --log-level debug
```

Lệnh này sẽ in ra nhiều thông tin hơn, giúp biết lỗi nằm ở đâu.

---

# 7. Kiểm tra `urg_node` có đang chạy không

Mở terminal mới:

```bash
source /opt/ros/jazzy/setup.bash
```

Nếu build từ source:

```bash
source ~/urg_ws/install/setup.bash
```

Kiểm tra node:

```bash
ros2 node list
```

Bạn nên thấy:

```text
/urg_node
```

Kiểm tra topic:

```bash
ros2 topic list
```

Bạn cần thấy:

```text
/scan
```

Nếu có `/scan`, khả năng cao LiDAR đã được đọc thành công.

---

# 8. Kiểm tra dữ liệu `/scan`

Kiểm tra tần số:

```bash
ros2 topic hz /scan
```

Nếu có dữ liệu, bạn sẽ thấy tần số xuất hiện, ví dụ:

```text
average rate: 20.0
```

Tần số có thể thay đổi tùy cấu hình LiDAR.

Xem thông tin topic:

```bash
ros2 topic info /scan --verbose
```

Bạn nên thấy kiểu dữ liệu:

```text
sensor_msgs/msg/LaserScan
```

Xem frame của LiDAR:

```bash
ros2 topic echo /scan --field header.frame_id --once
```

Nếu bạn đặt `frame_id:=laser`, kết quả sẽ là:

```text
laser
```

Xem dữ liệu khoảng cách:

```bash
ros2 topic echo /scan --field ranges --no-arr --once
```

Xem khoảng cách nhỏ nhất và lớn nhất mà driver công bố:

```bash
ros2 topic echo /scan --field range_min --once
ros2 topic echo /scan --field range_max --once
```

---

# 9. Hiển thị LiDAR bằng RViz2

Chạy RViz:

```bash
rviz2
```

Trong RViz:

1. Đặt **Fixed Frame** thành:

   ```text
   laser
   ```

2. Chọn **Add**.

3. Chọn **LaserScan**.

4. Ở mục **Topic**, chọn:

   ```text
   /scan
   ```

5. Nhấn **OK**.

Bạn sẽ thấy dữ liệu quét 2D từ URG-04LX.

Nếu không thấy dữ liệu:

- Kiểm tra Fixed Frame phải đúng với `frame_id`.
- Kiểm tra topic:
  ```bash
  ros2 topic hz /scan
  ```
- Thử đổi **Reliability Policy** trong RViz:
  - `Best Effort`
  - hoặc `Reliable`

---

# 10. Publish TF nếu bạn muốn gắn LiDAR vào robot

Nếu bạn dùng frame robot là `base_link`, hãy publish TF tĩnh từ `base_link` sang `laser`:

```bash
ros2 run tf2_ros static_transform_publisher 0 0 0 0 0 0 base_link laser
```

Ý nghĩa:

```text
base_link: frame gốc của robot
laser: frame của LiDAR
```

Sau đó trong RViz, bạn có thể đặt Fixed Frame là:

```text
base_link
```

---

# 11. Dùng file tham số cho `urg_node`

Bạn có thể tạo file cấu hình để chạy cho gọn.

Tạo file:

```bash
nano hokuyo.yaml
```

Dán nội dung:

```yaml
/urg_node:
  ros__parameters:
    serial_port: /dev/ttyACM0
    frame_id: laser
    serial_baud: 115200
```

Lưu file.

Chạy:

```bash
ros2 run urg_node urg_node --ros-args --params-file hokuyo.yaml
```

Nếu bạn dùng udev rule tạo `/dev/hokuyo`, có thể sửa thành:

```yaml
/urg_node:
  ros__parameters:
    serial_port: /dev/hokuyo
    frame_id: laser
```

---

# 12. Các tham số quan trọng của `urg_node`

Dưới đây là các tham số thường dùng.

| Tham số | Ý nghĩa | Giá trị gợi ý |
|---|---|---|
| `serial_port` | Cổng USB/serial | `/dev/ttyACM0` |
| `serial_baud` | Baudrate nếu cần | `115200`, `19200` |
| `ip_address` | IP nếu dùng bản Ethernet | ví dụ `192.168.0.10` |
| `ip_port` | Port Ethernet | thường `10940` |
| `frame_id` | Frame của LiDAR | `laser` |
| `publish_intensity` | Publish cường độ | thường `false` với URG-04LX |
| `publish_multiecho` | Publish đa tia | thường `false` với URG-04LX |

Với URG-04LX-UG01 USB, lệnh cơ bản nhất là:

```bash
ros2 run urg_node urg_node --ros-args \
  -p serial_port:=/dev/ttyACM0 \
  -p frame_id:=laser
```

---

# 13. Đọc dữ liệu LiDAR bằng Python

Bạn có thể tạo script Python để đọc `/scan`.

Tạo file:

```bash
nano hokuyo_reader.py
```

Dán nội dung:

```python
import math
import rclpy
from rclpy.node import Node
from rclpy.qos import qos_profile_sensor_data
from sensor_msgs.msg import LaserScan


class HokuyoReader(Node):
    def __init__(self):
        super().__init__('hokuyo_reader')

        self.subscription = self.create_subscription(
            LaserScan,
            '/scan',
            self.scan_callback,
            qos_profile_sensor_data
        )

    def scan_callback(self, msg: LaserScan):
        valid = []

        for r in msg.ranges:
            if math.isfinite(r) and msg.range_min <= r <= msg.range_max:
                valid.append(r)

        if not valid:
            self.get_logger().warning('Không có dữ liệu khoảng cách hợp lệ')
            return

        min_range = min(valid)
        max_range = max(valid)
        mean_range = sum(valid) / len(valid)

        self.get_logger().info(
            f'Số điểm hợp lệ: {len(valid)}, '
            f'gần nhất: {min_range:.3f} m, '
            f'xa nhất: {max_range:.3f} m, '
            f'trung bình: {mean_range:.3f} m'
        )


def main(args=None):
    rclpy.init(args=args)
    node = HokuyoReader()

    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
```

Chạy script:

```bash
source /opt/ros/jazzy/setup.bash
python3 hokuyo_reader.py
```

Nếu script không nhận được dữ liệu, có thể do QoS. Bạn có thể đổi sang QoS reliable.

Sửa phần subscription thành:

```python
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy

qos = QoSProfile(
    depth=10,
    reliability=ReliabilityPolicy.RELIABLE,
    history=HistoryPolicy.KEEP_LAST
)

self.subscription = self.create_subscription(
    LaserScan,
    '/scan',
    self.scan_callback,
    qos
)
```

---

# 14. Hiểu dữ liệu LaserScan từ URG-04LX

Topic `/scan` có các trường quan trọng:

```text
header.frame_id
angle_min
angle_max
angle_increment
range_min
range_max
ranges
intensities
```

Trong đó:

| Trường | Ý nghĩa |
|---|---|
| `header.frame_id` | Frame của LiDAR |
| `angle_min` | Góc bắt đầu quét |
| `angle_max` | Góc kết thúc quét |
| `angle_increment` | Góc tăng giữa hai tia liên tiếp |
| `range_min` | Khoảng cách tối thiểu hợp lệ |
| `range_max` | Khoảng cách tối đa hợp lệ |
| `ranges` | Mảng khoảng cách theo mét |
| `intensities` | Cường độ phản hồi, có thể không có |

Góc của tia thứ `i`:

```python
angle = msg.angle_min + i * msg.angle_increment
```

Khoảng cách tương ứng:

```python
distance = msg.ranges[i]
```

Nếu giá trị là:

```text
inf
nan
```

thì điểm đó không hợp lệ.

---

# 15. Tạo udev rule để LiDAR luôn có tên cố định

Hiện tại LiDAR có thể nhận là:

```bash
/dev/ttyACM0
```

Nhưng lúc khác có thể là:

```bash
/dev/ttyACM1
```

Bạn có thể tạo symlink cố định:

```bash
/dev/hokuyo
```

Kiểm tra vendor/product ID:

```bash
udevadm info -a -n /dev/ttyACM0 | grep -E 'ATTRS{idVendor}|ATTRS{idProduct}' | head -n 10
```

Tạo file udev:

```bash
sudo nano /etc/udev/rules.d/99-hokuyo.rules
```

Ví dụ với Hokuyo, vendor ID thường là `15d1`:

```bash
SUBSYSTEM=="tty", ATTRS{idVendor}=="15d1", MODE="0666", SYMLINK+="hokuyo"
```

Nếu bạn biết chính xác product ID, có thể viết chặt chẽ hơn:

```bash
SUBSYSTEM=="tty", ATTRS{idVendor}=="15d1", ATTRS{idProduct}=="0000", MODE="0666", SYMLINK+="hokuyo"
```

Lưu file.

Reload udev:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Rút LiDAR ra cắm lại.

Kiểm tra:

```bash
ls -l /dev/hokuyo
```

Sau đó bạn có thể chạy:

```bash
ros2 run urg_node urg_node --ros-args \
  -p serial_port:=/dev/hokuyo \
  -p frame_id:=laser
```

---

# 16. Ghi dữ liệu LiDAR vào bag

Nếu bạn muốn ghi lại dữ liệu:

```bash
ros2 bag record hokuyo_bag -- /scan
```

Phát lại:

```bash
ros2 bag play hokuyo_bag
```

Khi phát lại, bạn có thể mở RViz hoặc chạy script Python để xem dữ liệu.

---

# 17. Lỗi thường gặp và cách xử lý

## Lỗi 1: Không thấy `/dev/ttyACM0`

Kiểm tra:

```bash
dmesg | tail -n 30
lsusb
ls -l /dev/ttyACM* 2>/dev/null
```

Nguyên nhân:

- Dây USB không có dữ liệu.
- Cổng USB lỗi.
- LiDAR không đủ nguồn.
- Máy ảo chưa nhận USB.
- WSL2 chưa attach USB.

Thử:

```bash
sudo modprobe cdc_acm
dmesg | grep cdc_acm
```

---

## Lỗi 2: Permission denied khi truy cập `/dev/ttyACM0`

Chạy:

```bash
sudo usermod -aG dialout $USER
```

Sau đó log out/login.

Kiểm tra:

```bash
ls -l /dev/ttyACM0
```

Tạm thời test nhanh:

```bash
sudo chmod 666 /dev/ttyACM0
```

Nhưng về lâu dài nên dùng udev rule.

---

## Lỗi 3: `urg_node` không mở được cổng serial

Kiểm tra cổng:

```bash
ls -l /dev/ttyACM*
```

Kiểm tra có tiến trình nào đang chiếm cổng không:

```bash
sudo lsof /dev/ttyACM0
```

Nếu có, tắt tiến trình đó.

Không chạy đồng thời hai node cùng đọc một LiDAR.

---

## Lỗi 4: Có node nhưng không có `/scan`

Kiểm tra:

```bash
ros2 node list
ros2 topic list
```

Nếu có node nhưng không có `/scan`:

- Sai cổng.
- Sai baudrate.
- LiDAR chưa sẵn sàng.
- Driver bị lỗi.
- Chưa source workspace.

Source lại:

```bash
source /opt/ros/jazzy/setup.bash
source ~/urg_ws/install/setup.bash
```

Chạy debug:

```bash
ros2 run urg_node urg_node --ros-args \
  -p serial_port:=/dev/ttyACM0 \
  -p frame_id:=laser \
  --log-level debug
```

---

## Lỗi 5: Có `/scan` nhưng RViz không hiển thị

Kiểm tra:

```bash
ros2 topic hz /scan
```

Nếu không có tần số, dữ liệu chưa được publish.

Nếu có tần số nhưng RViz không hiện:

- Fixed Frame phải là `laser`.
- Topic phải là `/scan`.
- Kiểm tra QoS trong RViz.
- Nếu dùng `base_link`, cần TF từ `base_link` tới `laser`.

Publish TF tĩnh:

```bash
ros2 run tf2_ros static_transform_publisher 0 0 0 0 0 0 base_link laser
```

---

## Lỗi 6: Dữ liệu toàn `inf` hoặc `nan`

Kiểm tra:

```bash
ros2 topic echo /scan --field range_min --once
ros2 topic echo /scan --field range_max --once
```

Nguyên nhân:

- Vật thể quá gần.
- Vật thể vượt tầm đo.
- LiDAR bị che.
- Bụi hoặc ánh sáng mạnh.
- LiDAR chưa quay ổn định.
- Nguồn USB yếu.

---

## Lỗi 7: Build từ source bị lỗi

Nếu bạn build từ source và gặp lỗi, kiểm tra:

```bash
cd ~/urg_ws
colcon build --symlink-install
```

Đọc kỹ lỗi cuối cùng.

Nếu thiếu `urg_c`:

```bash
git clone https://github.com/ros-drivers/urg_c.git src/urg_c
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
```

Nếu lỗi liên quan nhánh ROS1/catkin, hãy chắc rằng bạn đang dùng nhánh ROS 2:

```bash
cd src/urg_node
git branch -a
git checkout ros2 || git checkout ros2-devel
```

---

# 18. Checklist nhanh trước khi chạy

Trước khi chạy `urg_node`, kiểm tra:

```bash
source /opt/ros/jazzy/setup.bash
```

Nếu build từ source:

```bash
source ~/urg_ws/install/setup.bash
```

Kiểm tra thiết bị:

```bash
ls -l /dev/ttyACM*
```

Kiểm tra quyền:

```bash
groups
```

Phải có `dialout`.

Kiểm tra package:

```bash
ros2 pkg list | grep urg
```

Chạy:

```bash
ros2 run urg_node urg_node --ros-args \
  -p serial_port:=/dev/ttyACM0 \
  -p frame_id:=laser
```

Kiểm tra:

```bash
ros2 topic list
ros2 topic hz /scan
```

Hiển thị:

```bash
rviz2
```

---

# 19. Lệnh chạy chuẩn cho URG-04LX-UG01

Nếu mọi thứ đã sẵn sàng, bạn chỉ cần chạy:

```bash
source /opt/ros/jazzy/setup.bash
ros2 run urg_node urg_node --ros-args \
  -p serial_port:=/dev/ttyACM0 \
  -p frame_id:=laser
```

Terminal khác:

```bash
source /opt/ros/jazzy/setup.bash
ros2 topic hz /scan
rviz2
```

Trong RViz:

```text
Fixed Frame: laser
Add: LaserScan
Topic: /scan
```

---

# 20. Nếu bạn vẫn gặp lỗi

Hãy gửi output của các lệnh sau:

```bash
lsusb
dmesg | tail -n 30
ls -l /dev/ttyACM* 2>/dev/null
ros2 pkg list | grep urg
ros2 node list
ros2 topic list
```

Và gửi thêm log khi chạy debug:

```bash
ros2 run urg_node urg_node --ros-args \
  -p serial_port:=/dev/ttyACM0 \
  -p frame_id:=laser \
  --log-level debug
```
