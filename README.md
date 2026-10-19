# TurtleBot3 ROS2 Go-To-Goal (Yıldız Rover Destek Ekip - Ödev 4)

Gazebo'daki TurtleBot3'ü bir **Python ROS2 node'u** ile belirli bir (x, y) noktasına götüren ödev projesi.

Node, `/odom` topic'inden robotun konumunu ve yönünü okur (subscribe), `/cmd_vel` topic'ine hız komutu yayınlar (publish). Robot önce hedefe doğru döner, sonra ilerler. Hedefe varıldığı anlaşılınca hız **sıfırlanır** ve robot durur.

## İçindekiler

- [Kullanılan Teknolojiler](#kullanılan-teknolojiler)
- [Gereksinimler](#gereksinimler)
- [Kurulum ve Build](#kurulum-ve-build)
- [Çalıştırma](#çalıştırma)
- [Kullanılan ROS2 Topic'leri](#kullanılan-ros2-topicleri)
- [Hedefe Ulaşma Mantığı](#hedefe-ulaşma-mantığı)
- [Hedefe Ulaşınca Nasıl Durur](#hedefe-ulaşınca-nasıl-durur)
- [Ekran Görüntüleri](#ekran-görüntüleri)
- [Test Sonuçları](#test-sonuçları)
- [Proje Yapısı](#proje-yapısı)

## Kullanılan Teknolojiler

- Ubuntu 22.04, ROS2 **Humble**
- Python 3 (`rclpy`), `colcon`
- Gazebo (Gazebo Classic) + TurtleBot3 paketleri

## Gereksinimler

Ubuntu 22.04 üzerinde ROS2 Humble kurulu olmalı. Ardından:

```bash
sudo apt update
sudo apt install ros-humble-turtlebot3* ros-humble-gazebo-ros-pkgs python3-colcon-common-extensions
```

`~/.bashrc` dosyasına (bir kez):

```bash
source /opt/ros/humble/setup.bash
export TURTLEBOT3_MODEL=burger
```

Yeni bir terminal aç veya `source ~/.bashrc` çalıştır.

## Kurulum ve Build

```bash
git clone https://github.com/erdem365/turtlebot3-ros2-homework.git
cd turtlebot3-ros2-homework/ros2_ws
colcon build --packages-select turtlebot_controller
source install/setup.bash
```

## Çalıştırma

**Terminal 1: Gazebo (engelsiz ortam)**

```bash
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py
```

Bu dünyada duvar ve engeller bulunur. Ödev "engel olmayan ortam" istediği için Gazebo penceresinde her engelin üzerine **sağ tık → Delete** yapılır ve sahnede yalnızca robot bırakılır. (Gazebo kapatılıp açılırsa engeller geri gelir, silme işlemi her açılışta tekrarlanır.)

> Kurulumunda `empty_world.launch.py` mevcutsa, engel silmek yerine doğrudan boş dünya da açılabilir:
> `ros2 launch turtlebot3_gazebo empty_world.launch.py`

**Terminal 2: Node** (yeni açılan her terminalde önce `source ros2_ws/install/setup.bash` çalıştırılmalı)

```bash
ros2 run turtlebot_controller go_to_goal
```

Hedefi parametreyle değiştirmek için (koordinatlar `odom` çerçevesindedir; robotun doğduğu nokta (0, 0), baktığı yön +x):

```bash
ros2 run turtlebot_controller go_to_goal --ros-args -p goal_x:=1.5 -p goal_y:=1.0
```

Varsayılan hedef: **(1.5, 1.0) m**. Robot başlangıçta +x yönüne baktığı için hedef başlangıç doğrultusunda değildir (hedefe açı ≈ 33.7°), yani robot önce dönmek zorunda kalır. Node, hedef başlangıç doğrultusuyla ±10° içindeyse bunu hata olarak bildirir ve çalışmayı reddeder.

> Her denemeden önce Gazebo'yu yeniden başlatmak (veya robotu sıfırlamak, `Ctrl+R`) gerekir; çünkü `/odom` başlangıcı robotun doğduğu yerdir.

## Kullanılan ROS2 Topic'leri

| Topic      | Yön           | Mesaj tipi                | Amaç                        |
| ---------- | ------------- | ------------------------- | --------------------------- |
| `/cmd_vel` | node yayınlar | `geometry_msgs/msg/Twist` | Lineer ve açısal hız komutu |
| `/odom`    | node dinler   | `nav_msgs/msg/Odometry`   | Robotun konumu ve yönü      |

Mesaj tiplerini doğrulamak için: `ros2 topic info /cmd_vel` ve `ros2 topic info /odom`.

## Hedefe Ulaşma Mantığı

1. `/odom` geldikçe robotun x, y ve yaw (quaternion'dan hesaplanır) değeri güncellenir.
2. 10 Hz'lik zamanlayıcı ile hedefe uzaklık `d = hypot(dx, dy)` ve açı hatası `atan2(dy, dx) - yaw` hesaplanır.
3. Açı hatası 0.3 rad'dan büyükse robot yerinde döner; küçükse ilerler ve giderken yönünü düzeltir.
4. Hız sınırları: lineer 0.15 m/s, açısal 0.8 rad/s.

## Hedefe Ulaşınca Nasıl Durur

- `d < 0.05 m` olduğunda hedefe varılmış sayılır.
- Sıfır hızlı `Twist` mesajı 10 kez yayınlanır (komut kaybına karşı), sonra node kapanır.
- `Ctrl+C` ile kapatılsa bile çıkışta bir kez sıfır hız yayınlanır.

## Ekran Görüntüleri

Gazebo'da engelsiz ortamda robotun hareketi:

| Başlangıç | Hareket halinde | Hedefte (durmuş) |
| --------- | --------------- | ---------------- |
| ![Başlangıç](screenshots/start.png) | ![Hareket](screenshots/moving.png) | ![Hedef](screenshots/goal.png) |

- `screenshots/start.png`: Engelsiz ortamda robotun başlangıç konumu
- `screenshots/moving.png`: Robotun hedefe doğru hareketi
- `screenshots/goal.png`: Robotun hedefe ulaşıp durması

## Test Sonuçları

Node Gazebo'da (ROS2 Humble, TurtleBot3 burger) engelsiz ortamda test edilmiştir.

- Hedef: **(1.5, 1.0) m**, başlangıç doğrultusunda değil (≈ 33.7°)
- Robot önce hedefe doğru döndü, ardından ilerledi.
- Hedefe yaklaşınca hız komutu sıfırlandı ve robot hedefte durdu.
- Çalışma sırasında hata alınmadı.

## Proje Yapısı

```
turtlebot3-ros2-homework/
├── README.md
├── .gitignore
├── screenshots/
└── ros2_ws/
    └── src/
        └── turtlebot_controller/
            ├── package.xml
            ├── setup.py
            ├── setup.cfg
            ├── resource/turtlebot_controller
            └── turtlebot_controller/
                ├── __init__.py
                └── go_to_goal.py
```
