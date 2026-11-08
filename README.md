# camera_calibration_apriltag_rviz2
The repository is to provide a computer vision skill includes calibration, apriltag and rviz2.

# Introduction
ROS2 Humble/Jazzy 单目相机标定 + AprilTag 识别 + RViz2可视化，适配usb_cam

# Environment
OS:Ubuntu 22.04\
ROS2:Humble/Jazzy\
Hardware:Lubancat/Tashan-Pi

# Install Dependencies
```bash
sudo apt update
sudo apt install ros-$ROS_DISTRO-usb-cam
ros-$ROS_DISTRO-camera-calibration
ros-$ROS_DISTRO-image-proc
ros-$ROS_DISTRO-apriltag-ros
```

# Workflow
## 检查安装的包是否被正确安装
```bash
ros2 pkg list | grep -e usb_cam -e camera_calibration -e image_proc -e apriltag_ros
```
如果正确安装，则完整显示相应的包名

## 查找USB相机
```bash
v4l2-ctl --list-devices
```
找到USB Camera下的第一个/dev/videoXX

## 启动usb_cam
```bash
ros2 run usb_cam usb_cam_node_exe --ros-args -p video_device:=/dev/videoXX
```
## 开始相机标定
```bash
ros2 run camera_calibration cameracalibrator --size 7x10 --square 0.02 --ros-args -r image:=/image_raw
```
以上参数为示例，表示标定板内角点7x10，每个黑（白）格长度为0.02米，可根据实际标定板格数和单位格长度自行调整参数大小\
注：标定板的格数比如8x11，则其内角点为7x10\
1.进入图形化界面，保持图像中存在彩虹色的定位点，上下、左右、远近、俯仰、旋转移动标定板或相机，改变两者之间的位置，直到右侧X/Y/Size/Skew四项变为绿色，在此过程中，终端中同步记录图像记录信息\
2.待全部变为绿色后，点击下方CALIBRATE开始计算\
3.计算完成后，点击SAVE\
4.退出程序完成标定 

## 创建安全存放的目录并移动文件
```bash
mkdir -p ~/camera_calibration_data
mv /tmp/calibrationdata.tar.gz ~/camera_calibration_data/
```
## 进入该目录并解压
```bash
cd ~/camera_calibration_data
tar -xf calibrationdata.tar.gz
```
运行 ls 命令查看目录内容
```bash
ls
```
此时应该有保存的图片、ost.yaml和ost.txt等文件\
关闭usb_cam终端

## 重新启动usb_cam
重新启动usb_cam，此时添加标定的相机内参
```bash
ros2 run usb_cam usb_cam_node_exe --ros-args -p video_device:=/dev/videoXX
-p camera_info_url:=file:///home/用户名/camera_calibration_data/camera_info.yaml
```
验证内参是否生效
```bash
ros2 topic echo /camera_info --once
```
如果k,r,p矩阵不全为零，则内参生效
## 去畸变
在识别目标等物体时，我们需要平直的画面，但是相机往往存在畸变，因此我们需要去畸变\
打开一个新的终端，输入
```bash
ros2 run image_proc rectify_node --ros-args -r image:=/image_raw -r camera_info:=/camera_info
```
此时再打开一个新的终端，可通过查看话题或者rqt可视化工具观察原始图像（raw）和去畸变图像（rect）
```bash
ros2 topic list
```
或者
```bash
ros2 run rqt_image_view rqt_image_view
```
左侧栏中框选image_raw或者image_rect来查看去畸变前后的图像

## AprilTag检测
```bash
ros2 run apriltag_ros apriltag_node --ros-args 
  -r image_rect:=/image_rect 
  -r camera_info:=/camera_info 
  -p family:=36h11 
  -p size:=0.1335 
  -p max_hamming:=2
```
注：0.1335是AprilTag整体大小长度（米），汉明距离大小表示误差容错率

## 检测标签是否被识别
```bash
ros2 topic echo /detections
```
如果detections数组中出现变化的数字，则标签被成功识别

## 可视化验证
打开一个新的终端，打开rviz2
```bash
rviz2
```
在左侧Fixed Frame处修改为default_cam\
点击左下角Add，选择tf\
保证前面的三个终端运行（usb_cam;去畸变;AprilTag检测）\
此时能看到画面中央可视化中显示出相机本体坐标系和识别标签坐标系
