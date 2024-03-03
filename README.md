# Lab-A436-Guidebook
欢迎来到Lab A436！这个仓库中有你新手入门所需的文档。

## Ubuntu

### 安装Ubuntu
实验室使用`Ubuntu 20.04 LTS`，镜像文件`ubuntu-20.04.6-desktop-amd64.iso`可以从[清华镜像](https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/20.04/)下载，然后用任意一台安装了Ubuntu的电脑中的启动盘创建器或者`rufus`制作启动盘。如果使用Dell电脑，则在启动时按F2进入bios，调整boot sequence，将启动盘改为第一位并保存设置，然后按照提示安装Ubuntu。注意，最好选择清理整个磁盘来安装Ubuntu，这样可以防止驱动程序等相互干扰。

### 安装常用软件

#### QQ/Microsoft Edge/VS Code/Clash Verge
安装包下载地址：  
`QQ` https://im.qq.com/linuxqq/index.shtml   
`Microsoft Edge` https://www.microsoft.com/zh-cn/edge  
`VS Code` https://code.visualstudio.com/  
`Clash Verge` https://github.com/clash-verge-rev/clash-verge-rev/releases/tag/v1.5.4

.deb文件双击可以直接安装，如果不能直接安装，则尝试使用命令行安装：
```
sudo dpkg -i xxx.deb
```
Clash Verge是实验室使用的科学上网软件，安装好之后联系袁双杰导入配置文件才能使用。

#### Anaconda
从[清华镜像](https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/)下载`Anaconda3-2024.02-1-Linux-x86_64.sh`文件，进入所在文件夹，右键，选择在终端打开，然后用命令行安装：
```
bash Anaconda3-2024.02-1-Linux-x86_64.sh
```

## Franka Panda
关于Franka Panda机械臂的使用，可以首先查阅官方提供的[FCI文档](https://franka.cn/FCI/)。

### 配置环境

#### 安装实时内核
首先构建文件夹：
```
mkdir Franka_env
cd Franka_env
```
然后下载[linux-5.15.137.tar.gz](https://www.kernel.org/pub/linux/kernel/v5.x/linux-5.15.137.tar.gz)和[patch-5.15.137-rt71.patch.gz][https://www.kernel.org/pub/linux/kernel/projects/rt/5.15/patch-5.15.137-rt71.patch.gz](https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/5.15/older/)文件到`Franka_env`文件夹，并解压文件：
```
tar xvzf linux-5.15.137.tar.gz
gunzip patch-5.15.137-rt71.patch.gz
```
提取源代码并应用补丁：
```
cd linux-5.15.137
patch -p1 < ../patch-5.15.137-rt71.patch
```
安装必要的依赖项：
```
sudo apt-get install build-essential bc curl ca-certificates gnupg2 libssl-dev lsb-release libelf-dev bison flex dwarves zstd libncurses-dev
```
使用当前核的配置作为实时核的默认配置：
```
make olddefconfig
```
按照实时核的配置修改配置文件：
```
make menuconfig
```
这个命令会打开一个终端界面，你可以在其中配置抢占模型。  
使用箭头键导航到 `General Setup` > `Preemption Model` 并选择 `Fully Preemptible Kernel (Real-Time)` 。  
之后导航到 `Cryptographic API` > `Certificates for signature checking `（在列表的最底部）> `Provide system-wide ring of trusted keys` > `Additional X.509 keys for default system keyring`，从提示符中移除 `debian/canonical-certs.pem`，然后按OK。  
将此配置保存到 `.config` 并退出 TUI。我们建议将其他选项保留为默认值。
完成后编辑`.config`文档：
```
gedit .config
```
将下面的内容
```
CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem"
CONFIG_SYSTEM_REVOCATION_KEYS="debian/canonical-revoked-certs.pem"
CONFIG_DEBUG_INFO_BTF=y
```
改为
```
CONFIG_SYSTEM_TRUSTED_KEYS=""
CONFIG_SYSTEM_REVOCATION_KEYS=""
CONFIG_DEBUG_INFO_BTF=n
```
编译内核（-j后面的数字就是在终端中输入`nproc`，将输出的数字乘以2得到的数字）：
```
fakeroot make -j72 deb-pkg
```
等待内核编译完毕，就可以安装内核：
```
sudo dpkg -i ../linux-headers-5.15.137-rt71_5.15.137-rt71-1_amd64.deb ../linux-image-5.15.137-rt71*.deb
```
以上步骤均完成后需要重启系统：
```
sudo reboot
```
重启后查看并确认已经安装的内核版本：
```
uname -msr
```
应该出现`Linux 5.15.137-rt71 x86_64`  
接下来将用户添加到组中：
```
sudo addgroup realtime
sudo usermod -a -G realtime $(whoami)
```
然后修改`/etc/security/limits.conf`文件：
```
sudo gedit /etc/security/limits.conf
```
在末尾添加
```
@realtime soft rtprio 99
@realtime soft priority 99
@realtime soft memlock 102400
@realtime hard rtprio 99
@realtime hard priority 99
@realtime hard memlock 102400
```
再次重启即可完成实时内核的安装。

#### 构建libfranka
首先需要卸载已安装的`libfranka`和`libfranka_ros`避免冲突：
```
sudo apt remove "*libfranka*"
```
接下来安装依赖项：
```
sudo apt install build-essential cmake git libpoco-dev libeigen3-dev
```
然后新建`catkin_ws`文件夹，在此文件夹中下载`libfranka`的源代码：
```
git clone --recursive https://github.com/frankaemika/libfranka --branch 0.9.2
cd libfranka
```
然后输入以下命令运行CMAKE：
```
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=OFF ..
cmake --build .
```
使用以下命令构建`libfranka`Debian软件包：
```
cpack -G DEB
```
然后安装此软件包：
```
sudo dpkg -i libfranka*.deb
```

#### 安装ROS Noetic
推荐使用`鱼香ROS`的一键安装ROS的命令，根据提示进行安装：
```
wget http://fishros.com/install -O fishros && . fishros
```

#### 构建ROS包
首先选择一个目录中的Catkin工作区：
```
cd catkin_ws
source /opt/ros/noetic/setup.sh
catkin_init_workspace src
cd src
```
然后将`franka_ros`的源代码下载到`src`文件夹中：
```
git clone --recursive https://github.com/frankaemika/franka_ros src/franka_ros --branch 0.9.1
```
安装任何缺少的依赖项并构建包：（注意：以下命令中注意将 `/path/to/libfranka/build` 替换成自己的 `libfranka build` 构建路径。）
```
rosdep install --from-paths src --ignore-src --rosdistro noetic -y --skip-keys libfranka
catkin_make -DCMAKE_BUILD_TYPE=Release -DFranka_DIR:PATH=/path/to/libfranka/build
source devel/setup.sh
```
到这里，Franka Panda的环境就配置好了。
