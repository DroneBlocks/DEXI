# Update repos

- sudo apt update
- sudo add-apt-repository universe
- sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(source /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
- sudo apt update

# Create the dexi package (first time setup)

ros2 pkg create dexi --dependencies rclcpp --build-type ament_cmake

# ROSBRIDGE

- sudo apt install ros-humble-rosbridge-server
- ros2 launch rosbridge_server rosbridge_websocket_launch.xml

# Camera

Make sure to enable legacy camera support on Pi 4 (Ubuntu 22.04 LTS) in /boot/firmware/config.txt:

```
start_x=1
```

and make sure to comment out the following line:

```
#camera_auto_detect=1
```

### Build

- cd /root/ros2_ws/src
- git clone https://github.com/Kapernikov/cv_camera
- cd /root/ros2_ws
- rosdep install --from-paths src -y --ignore-src
- colcon build

### Run

- source install/setup.bash
- ros2 run cv_camera cv_camera_node

# Web video server

### Build

- cd /root/ros2_ws/src
- git clone https://github.com/RobotWebTools/web_video_server/
- cd web_video_server
- git checkout ros2
- cd ~/ros2_ws
- rosdep install --from-paths src -y --ignore-src
- colcon build

### Run

- source install/setup.bash
- ros2 run web_video_server web_video_server

# Micro DDS Client

### Build

- cd ~/ros2_ws/src
- git clone https://github.com/eProsima/Micro-XRCE-DDS-Agent.git
- cd ~/ros2_ws
- colcon build

### Run

- source install/setup.bash
- MicroXRCEAgent serial --dev /dev/ttyAMA0 -b 921600

# MAVROS and SITL

Run the following from your host machine in the docker directory. It will spin up PX4-SITL and a ROS Humble container for DEXI development. You will need to make sure PX4 SITL is configured to send packets to the DEXI container IP

```
docker-compose up
```

Alternatively, if you have a DEXI container already running you can find its IP address and then start SITL. You'll need to make sure to specify the IP of your DEXI container. You can find this by running:

```
docker inspect bridge
```

Find your DEXI IP and then run:

```
docker run --rm -it jonasvautherin/px4-gazebo-headless:1.14.0 172.17.0.2 <- This is your DEXI container IP
```

In your DEXI container make sure MAVROS is installed:

```
sudo apt install ros-humble-mavros
```

and make sure to update your px4.launch file in:

```
/opt/ros/humble/shared/mavros/launch/px4.launch
```

with:

```
<arg name="fcu_url" default="udp://:14540@:14540" />
```

Install the geographic lib dependencies:

```
/opt/ros/humble/lib/mavros/install_geographiclib_datasets.sh
```

Finally, test the node:

```
source /opt/ros/humble/setup.bash
ros2 launch mavros px4.launch
```

# PX4 ROS Messages and ROS Com Example

### Build

- cd ~/ros2_ws/src
- git clone https://github.com/PX4/px4_msgs
- git clone https://github.com/PX4/px4_ros_com
- cd ~/ros2_ws
- colcon build

# LED

Notes: the led core files live in the ws281x package. To run the dexi launch file with LED support you need to give elevated privileges to the node:

```
ros2 pkg prefix ws281x
sudo chown -R root:root /home/droneblocks/ros2_ws/install/ws281x/lib/
sudo chmod -R +s /home/droneblocks/ros2_ws/install/ws281x/lib
```

Call the set_effect service:

```
ros2 service call set_effect dexi_msgs/srv/SetLedEffect "{effect: 'fill', r: 255, g: 0, b: 0}"
ros2 service call set_effect dexi_msgs/srv/SetLedEffect "{effect: 'fade', r: 255, g: 255, b: 255, brightness: 64, duration: 10, priority: 1}"
```

# GPIO

### Install

From here: https://abyz.me.uk/rpi/pigpio/download.html do the following:

```
cd
git clone https://github.com/joan2937/pigpio
cd pigpio
make
sudo make install
sudo pigpiod
```

### Run

In the dexi/droneblocks/python-examples folder run the test connected to GPIO 18:

```
python3 laser.py
```

# Docker Dev

### New Container

Mapping port 9090 for rosbridge websocket:

- docker run -it -p 9090:9090 --name dexi -v ${PWD}:/root/ros2_ws/src osrf/ros:humble-desktop

### Existing Container

- docker start 12b3e9d32467
- docker exec -it 12b3e9d32467 /bin/bash

# Calling DroneBlocks Services

- ros2 service call /droneblocks/run dexi_msgs/srv/Run "{code: 'mission code goes in here'}"
- ros2 service call /droneblocks/stop std_srvs/srv/Trigger
- ros2 service call /droneblocks/load dexi_msgs/srv/Load

# nginx for DroneBlocks

- docker run -it --rm -d -p 7777:80 --name droneblocks -v ${PWD}/droneblocks/www:/usr/share/nginx/html nginx

# Thanks

A special thanks to the following projects for inspiration:

- https://github.com/CopterExpress/clover
- https://github.com/StarlingUAS/clover_ros2_pkgs
