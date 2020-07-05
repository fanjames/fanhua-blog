---
title: GPU服务器配置的问题记录
date: 2018-06-30 12:47:20
categories: 硬件
tags:
	  - 机器学习
	  - 硬件
	  - Docker
---


这几点一直在折腾GPU服务器，大问题倒也没有，就是小麻烦不断出现。为避免以后遇到类似问题又要重新搜索，干脆就记录在这里，以备不时之需。

<!-- more -->

## 显卡独立电源接线
手上有两块公版1080ti和两台i7 6700、8G主机，但是因为电源的问题，闲置了一段时间。1080ti额定功率250W，峰值功率会达到300W，所以，考虑同时给两块显卡供电，电源功率要不小于600W。前天从某东上入手了海韵的650FX，比618的时候贵了不少，只能说不赶趟了。

电源到手后遇到的第一个问题就是，启动电源要有触发信号。一般情况下，电源装在机箱里，与主板连接，这时就不需要考虑触发信号的问题，直接开机就能使用。但是由于两台主机是HP的品牌机，主板是定制化的，与电源的接口不兼容。上网一查，原来短接电源M/B接口的两个针脚就可以无主机启动电源了。

<font color=red>**还有一点必须特别注意的是：**
在用独立电源给显卡供电的时候，必须先打开电源，再开机，否则将启动主板的集显，独立显卡将不会工作。
</font>

两台GPU服务器能正常工作后就是安装显卡驱动、运行环境这些工作了。

## 安装显卡驱动
1、安装编译环境：gcc、kernel-devel、kernel-headers
```
yum -y install gcc kernel-devel "kernel-devel-uname-r == $(uname -r)" dkms
```
2、 修改`/etc/modprobe.d/blacklist.conf`文件，以阻止 nouveau 模块的加载。添加`blacklist nouveau`，注释掉`blacklist nvidiafb`（如果存在）, `blacklist.conf`不存在时，执行下面的脚本
```
echo -e "blacklist nouveau\noptions nouveau modeset=0" > /etc/modprobe.d/blacklist.conf
```
3、重新建立initramfs image文件
```
mv /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak
dracut /boot/initramfs-$(uname -r).img $(uname -r)
```

4、添加 ELRepo 源:
```bash
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
sudo rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
```
直接安装最新版的驱动即可:
```bash
sudo yum install nvidia-x11-drv nvidia-x11-drv-32bit
```

5、安装cuda和cudnn
从 https://developer.nvidia.com/cuda-toolkit-archive 下载所需版本的cuda，直接安装
```bash
sh cuda_9.0.176_384.81_linux-run
# 添加环境变量
# 在 ~/.bashrc的最后面添加下面两行
export PATH=/usr/local/cuda-9.0/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-9.0/lib64:/usr/local/cuda-9.0/extras/CUPTI/lib64:$LD_LIBRARY_PATH
# 使生效
source ~/.bashrc
```
相应地，从 https://developer.nvidia.com/rdp/cudnn-archive 下载所需版本的cuDNN，解压，将解压后的文件移到对应的目录下即可。
```bash
tar -xvzf cudnn-9.0-linux-x64-v7.0.tgz
cp -P cuda/include/cudnn.h /usr/local/cuda-9.0/include
cp -P cuda/lib64/libcudnn* /usr/local/cuda-9.0/lib64
chmod a+r /usr/local/cuda-9.0/include/cudnn.h /usr/local/cuda-9.0/lib64/libcudnn*
```



## 显卡风扇转速动态调节

### 安装X server
首先安装X server环境，这是系统中才有`xinit`命令。
```bash
yum groupinstall 'X Window System' -y
```
在系统中使用以下命令即可关闭：
```bash
systemctl set-default multi-user.target
```
如果需要启用X-Windows在命令行中运行以下命令即可：
```bash
startx
```
启动gui
```bash
systemctl start graphical.target
```

### GPU风扇动态调节脚本
下载GitHub上的这个库(https://github.com/boris-dimitrov/set_gpu_fans_public)
修改目录名
```bash
sudo mv set_gpu_fans_public set-gpu-fans
```
创建一个符号链接，让系统知道这个代码在哪里：
```bash
ln -sf ~/set-gpu-fans /home/set-gpu-fans
```
不知道是什么原因，原文章用的是`tcsh`执行，进入`set-gpu-fans`目录，执行
```bash
sh ./cool_gpu >& controller.log & tail -f controller.log
```

参考：https://www.leiphone.com/news/201707/z88LWb5adH1MzTAR.html





## Docker全局透明代理