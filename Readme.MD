# AirSim ROS Noetic & PX4 SITL Docker Integration



[![Docker](https://img.shields.io/badge/docker-ready-blue.svg)](https://www.docker.com/)
[![ROS Noetic](https://img.shields.io/badge/ROS-Noetic-green)](https://wiki.ros.org/noetic)
[![AirSim](https://img.shields.io/badge/AirSim-v1.8.1-orange)](https://github.com/microsoft/AirSim)
[![PX4](https://img.shields.io/badge/PX4-v1.14.3-purple)](https://px4.io/)



---



## 🇬🇧 English



### Overview



This repository provides production-ready Dockerfiles to integrate **AirSim** (running on Windows) with **ROS Noetic** and **PX4 SITL** in a Linux container environment. It enables seamless development and testing of autonomous aerial vehicles using high-fidelity simulation, with all dependencies pre‑configured.



Two main components are provided:



1. **AirSim ROS Noetic Wrapper** – A Docker image with ROS Noetic and the `airsim_ros_pkgs` package. It connects to the AirSim simulator on a Windows host and publishes sensor data (cameras, LiDAR, IMU, GPS, etc.) as ROS topics.

2. **PX4 SITL + MAVSDK Server** – A Docker image containing the PX4 Autopilot SITL and MAVSDK server, configured to communicate with AirSim via TCP. This allows you to run a full flight controller in simulation and interact with it using MAVSDK or MAVLink.



Both images are optimized for **WSL2** (Windows Subsystem for Linux) and use `--net=host` for direct network access to the Windows host.



### Features



- **Multi‑stage Dockerfiles** – Lightweight, production‑ready images with minimal runtime dependencies.

- **ROS Noetic Integration** – Full support for `airsim_ros_pkgs`, publishing:

  - Odometry (`/odom_local_ned`)

  - GPS (`/global_gps`)

  - IMU, magnetometer, altimeter

  - Cameras (RGB, depth, segmentation)

  - LiDAR point clouds

- **PX4 SITL** – Pre‑built PX4 firmware (v1.14.3) configured for TCP connection to AirSim.

- **MAVSDK Server** – Ready‑to‑run gRPC server for programmatic drone control (Python/C++ SDKs).

- **Easy Windows Integration** – Works out‑of‑the‑box with AirSim on Windows via WSL2.

- **Docker Compose Support** – Launch the entire stack (ROS + PX4) with a single command.



### Requirements



- **Windows 10/11** with:

  - [AirSim](https://github.com/microsoft/AirSim) installed and running (Unreal Engine environment)

  - [Docker Desktop](https://www.docker.com/products/docker-desktop/) with WSL2 backend enabled

  - [VcXsrv](https://sourceforge.net/projects/vcxsrv/) or any X11 server (for GUI apps like RViz)

- **Linux/WSL2** environment (Ubuntu 20.04 recommended)



### Quick Start
#### 1. Clone the Repository

```bash
git clone https://github.com/onaqua/airsim-ros-px4-docker.git
cd airsim-ros-px4-docker
```

#### 2. Build the Docker Images

```bash
# Base image with AirLib
docker build -f Dockerfile.airsim-base -t airsim-client-base .

# ROS Noetic + airsim_ros_pkgs
docker build -f Dockerfile.airsim-ros -t airsim-ros-noetic .

# PX4 SITL + MAVSDK Server
docker build -f Dockerfile.px4 -t px4-sitl-mavsdk .
```

#### 3. Configure AirSim on Windows

Edit `AirSim\\settings.json`:

```json
{
  "SettingsVersion": 1.2,
  "SimMode": "Multirotor",
  "Vehicles": {
    "PX4": {
      "VehicleType": "PX4Multirotor",
      "UseSerial": false,
      "UseTcp": true,
      "QgcHostIp": "",
      "TcpPort": 4560,
      "ControlIp": "127.0.0.1",
      "ControlPort": 14580,
      "LocalHostIp": "0.0.0.0",
      "Cameras": {
        "depth_camera": {
          "CameraType": "Depth",
          "Enabled": true,
          "CaptureSettings": [
            {
              "ImageType": 2,
              "Width": 640,
              "Height": 480,
              "FOV_Degrees": 90,
              "AutoExposureSpeed": 100,
              "MotionBlurAmount": 0
            }
          ],
          "X": 0.0, "Y": 0.0, "Z": 0.0,
          "Pitch": 0.0, "Roll": 0.0, "Yaw": 0.0
        }
      },
      "Sensors": {
        "LidarSensor1": {
          "SensorType": 6,
          "Enabled": true,
          "NumberOfChannels": 16,
          "RotationsPerSecond": 10,
          "PointsPerSecond": 100000,
          "X": 0,
          "Y": 0,
          "Z": -1,
          "Roll": 0,
          "Pitch": 0,
          "Yaw": 0,
          "VerticalFOVUpper": -15,
          "VerticalFOVLower": -25,
          "HorizontalFOVStart": -20,
          "HorizontalFOVEnd": 20,
          "DrawDebugPoints": true,
          "DrawMode": 1,
          "DataFrame": "SensorLocalFrame"
        }
      }
    }
  }
}
```

Start AirSim and load your Unreal environment.

#### 4. Launch Containers (in WSL2)

**Run ROS wrapper only:**

```bash
docker run -it --rm --net=host \\
  -e WSL_HOST_IP=host.docker.internal \\
  -e DISPLAY=$DISPLAY \\
  -v /tmp/.X11-unix:/tmp/.X11-unix \\
  airsim-ros-noetic:latest /start_airsim_ros.sh
```

**Run PX4 SITL + MAVSDK only:**

```bash
docker run -it --rm --net=host --privileged \\
  -e WSL_HOST_IP=host.docker.internal \\
  px4-sitl-mavsdk:latest /start_all.sh
```

**Full stack with Docker Compose:**

```bash
docker-compose up
```



### Repository Structure

```
.
├── Dockerfile.airsim-base          # Base image with AirLib
├── Dockerfile.airsim-ros           # ROS Noetic + airsim_ros_pkgs
├── Dockerfile.px4                  # PX4 SITL + MAVSDK Server
├── docker-compose.yml              # Compose file for both services
├── README.md                       # This file
└── scripts/                        # Helper scripts (optional)
```



### Usage Examples



**Subscribe to ROS topics from AirSim** (inside ROS container):

```bash
rostopic list
rostopic echo /airsim_node/PX4/odom_local_ned
rosrun image_view image_view image:=/airsim_node/PX4/front_center/Scene
```

**Control drone with MAVSDK Python** (on host):

```bash
pip3 install mavsdk
```

```python
# takeoff.py
import asyncio
from mavsdk import System

async def run():
    drone = System()
    await drone.connect(system_address="127.0.0.1:50051")
    async for state in drone.core.connection_state():
        if state.is_connected:
            print("Connected!")
            break

    await drone.action.arm()
    await drone.action.takeoff()
    await asyncio.sleep(10)
    await drone.action.land()

asyncio.run(run())
```

```bash
python3 takeoff.py
```

### License
This project is licensed under the MIT License – see the [LICENSE](LICENSE) file for details.


---



## 🇷🇺 Русский



### Обзор



Репозиторий содержит готовые к использованию Dockerfile для интеграции **AirSim** (запущенного на Windows) с **ROS Noetic** и **PX4 SITL** в среде Linux-контейнеров. Это позволяет разрабатывать и тестировать автономные летательные аппараты в высококачественной симуляции со всеми предустановленными зависимостями.



Два основных компонента:



1. **Обёртка AirSim для ROS Noetic** – Docker-образ с ROS Noetic и пакетом `airsim_ros_pkgs`, который подключается к симулятору AirSim на Windows и публикует данные с сенсоров (камеры, лидар, IMU, GPS и т.д.) в топики ROS.

2. **PX4 SITL + MAVSDK Server** – Docker-образ, содержащий PX4 Autopilot SITL и MAVSDK-сервер, настроенный на взаимодействие с AirSim по TCP. Это позволяет запускать полноценный полётный контроллер в симуляции и управлять им через MAVSDK или MAVLink.



Оба образа оптимизированы для **WSL2** и используют `--net=host` для прямого сетевого доступа к хосту Windows.



### Возможности



- **Многоступенчатые Dockerfile** – лёгкие, готовые к продакшену образы с минимальными зависимостями.

- **Интеграция с ROS Noetic** – полная поддержка `airsim_ros_pkgs`, публикация:

  - Одометрии (`/odom_local_ned`)

  - GPS (`/global_gps`)

  - IMU, магнитометра, альтиметра

  - Камер (RGB, глубина, сегментация)

  - Облаков точек с лидара

- **PX4 SITL** – предварительно собранная прошивка PX4 (v1.14.3), настроенная на TCP-подключение к AirSim.

- **MAVSDK Server** – готовый к запуску gRPC-сервер для программного управления дроном (Python/C++ SDK).

- **Простая интеграция с Windows** – работает «из коробки» с AirSim на Windows через WSL2.

- **Поддержка Docker Compose** – запуск всего стека (ROS + PX4) одной командой.



### Требования



- **Windows 10/11** с:

  - Установленным и запущенным [AirSim](https://github.com/microsoft/AirSim) (окружение Unreal Engine)

  - [Docker Desktop](https://www.docker.com/products/docker-desktop/) с включённым бэкендом WSL2

  - [VcXsrv](https://sourceforge.net/projects/vcxsrv/) или любой X11-сервер (для GUI-приложений, например RViz)

- **Окружение Linux/WSL2** (рекомендуется Ubuntu 20.04)



### Быстрый старт



#### 1. Клонируйте репозиторий

```bash
git clone https://github.com/onaqua/airsim-ros-px4-docker.git
cd airsim-ros-px4-docker
```



#### 2. Соберите Docker-образы

```bash
docker build -f Dockerfile.airsim-base -t airsim-client-base .
docker build -f Dockerfile.airsim-ros -t airsim-ros-noetic .
docker build -f Dockerfile.px4 -t px4-sitl-mavsdk .
```


#### 3. Настройте AirSim на Windows

Отредактируйте `AirSim\\settings.json`:

```json
{
  "SettingsVersion": 1.2,
  "SimMode": "Multirotor",
  "Vehicles": {
    "PX4": {
      "VehicleType": "PX4Multirotor",
      "UseSerial": false,
      "UseTcp": true,
      "QgcHostIp": "",
      "TcpPort": 4560,
      "ControlIp": "127.0.0.1",
      "ControlPort": 14580,
      "LocalHostIp": "0.0.0.0",
      "Cameras": {
        "depth_camera": {
          "CameraType": "Depth",
          "Enabled": true,
          "CaptureSettings": [
            {
              "ImageType": 2,
              "Width": 640,
              "Height": 480,
              "FOV_Degrees": 90,
              "AutoExposureSpeed": 100,
              "MotionBlurAmount": 0
            }
          ],
          "X": 0.0, "Y": 0.0, "Z": 0.0,
          "Pitch": 0.0, "Roll": 0.0, "Yaw": 0.0
        }
      },
      "Sensors": {
        "LidarSensor1": {
          "SensorType": 6,
          "Enabled": true,
          "NumberOfChannels": 16,
          "RotationsPerSecond": 10,
          "PointsPerSecond": 100000,
          "X": 0,
          "Y": 0,
          "Z": -1,
          "Roll": 0,
          "Pitch": 0,
          "Yaw": 0,
          "VerticalFOVUpper": -15,
          "VerticalFOVLower": -25,
          "HorizontalFOVStart": -20,
          "HorizontalFOVEnd": 20,
          "DrawDebugPoints": true,
          "DrawMode": 1,
          "DataFrame": "SensorLocalFrame"
        }
      }
    }
  }
}
```

Запустите AirSim и загрузите ваше Unreal-окружение.



#### 4. Запустите контейнеры (в WSL2)

**Только ROS-обёртка:**

```bash
docker run -it --rm --net=host \\
  -e WSL_HOST_IP=host.docker.internal \\
  -e DISPLAY=$DISPLAY \\
  -v /tmp/.X11-unix:/tmp/.X11-unix \\
  airsim-ros-noetic:latest /start_airsim_ros.sh
```



**Только PX4 SITL + MAVSDK:**

```bash
docker run -it --rm --net=host --privileged \\
  -e WSL_HOST_IP=host.docker.internal \\
  px4-sitl-mavsdk:latest /start_all.sh
```



**Полный стек через Docker Compose:**

```bash
docker-compose up
```



### Структура репозитория

```
.
├── Dockerfile.airsim-base          # Базовый образ с AirLib
├── Dockerfile.airsim-ros           # ROS Noetic + airsim_ros_pkgs
├── Dockerfile.px4                  # PX4 SITL + MAVSDK Server
├── docker-compose.yml              # Compose-файл для обоих сервисов
├── README.md                       # Этот файл
└── scripts/                        # Вспомогательные скрипты (опционально)
```



### Примеры использования



**Подписка на топики ROS от AirSim** (внутри контейнера с ROS):

```bash
rostopic list
rostopic echo /airsim_node/PX4/odom_local_ned
rosrun image_view image_view image:=/airsim_node/PX4/front_center/Scene
```



**Управление дроном через MAVSDK Python** (на хосте):

```bash
pip3 install mavsdk
```

```python
# takeoff.py
import asyncio
from mavsdk import System

async def run():
    drone = System()
    await drone.connect(system_address="127.0.0.1:50051")
    async for state in drone.core.connection_state():
        if state.is_connected:
            print("Подключён!")
            break

    await drone.action.arm()
    await drone.action.takeoff()
    await asyncio.sleep(10)
    await drone.action.land()

asyncio.run(run())
```

```bash
python3 takeoff.py
```

### Лицензия

Проект распространяется под лицензией MIT. Подробнее см. в файле [LICENSE](LICENSE).
                            
