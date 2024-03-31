# F1Cx00S设计与调试记录

作者：weirdo-xo

时间：2024-3-30

设计目标：使用全志F1C200s设计一款最简单的Linux开发板，并且引出板子的LCD部分接口。由于音频、视频部分对个人而言不是必须的，所以去掉音频+视频部分。



## 1. 关于板子的供电设计：分支

本次设计的板子中，提供了两种电源供电方案：PCB布局布线中使用的时TLV的供电方案，主要是方便调试。

+ EA3056供电方案：对应于master分支
+ TLV62519供电方案：对应于TLV分支





## 2. 初始版本存在的问题：【现在的版本中已修复】

在本次版本的电路中，由于SD卡的线序设计错误，导致使用故障：非常明显的可以看出原理图库中DAT2和DAT3没有按照顺序排列，导致接线出错。在F1C200S启动时，会读取SD卡中的启动标志位信息，由于这里的接线错误会导致识别步到SD卡中的启动标志信息。

![SD-Card](https://github.com/werido-xo/Mini-F1C200S/blob/TLV62569/Image/fault-design.png)



解决方案：使用飞线连接SD卡卡槽，纠正SD卡的线序

<img src="https://github.com/werido-xo/Mini-F1C200S/blob/TLV62569/Image/fly-sdcard.jpg" alt="sd-card-fly" style="zoom:50%;" />



重新启动，可以看到已经成功的进入了U-Boot:

![uboot-run](https://github.com/werido-xo/Mini-F1C200S/blob/TLV62569/Image/run-uboot.jpg)



关于U-Boot和内核移植工作：

+ U-Boot的移植：主要是修改串口输出，在本次的设计中使用的是UART2（使用PE7,8引脚），可以按照如下的参考链接进行修改：https://blog.csdn.net/qq_17833651/article/details/127707195

​	注意：CONS_INDEX是从1开始的，这里我们要设置串口3的化需要修改CONS_INDEX为3



+ Linux的移植：Linux的移植主要也是串口输出，使用Linux时使用的默认配置文件应该是：

  + 配置文件：linux-licheepi_nano_defconfig [我已经上传到本仓库]
  + 设备树文件：arch/arm/boot/dts/allwinner/suniv-f1c200s-lctech-pi.dts

  

我对设备树的修改如下：

```diff
ubuntu@VM-8-10-ubuntu:~/Linux/linux$ git diff
diff --git a/arch/arm/boot/dts/allwinner/suniv-f1c100s.dtsi b/arch/arm/boot/dts/allwinner/suniv-f1c100s.dtsi
index 3c61d59ab..6831f224c 100644
--- a/arch/arm/boot/dts/allwinner/suniv-f1c100s.dtsi
+++ b/arch/arm/boot/dts/allwinner/suniv-f1c100s.dtsi
@@ -213,6 +213,11 @@ uart1_pa_pins: uart1-pa-pins {
                                pins = "PA2", "PA3";
                                function = "uart1";
                        };
+
+                       uart2_pe_pins: uart2-pe-pins {
+                               pins = "PE7", "PE8";
+                               function = "uart2";
+                       };
                };

                i2c0: i2c@1c27000 {
diff --git a/arch/arm/boot/dts/allwinner/suniv-f1c200s-lctech-pi.dts b/arch/arm/boot/dts/allwinner/suniv-f1c200s-lctech-pi.dts
index 2d2a3f026..b60463ad5 100644
--- a/arch/arm/boot/dts/allwinner/suniv-f1c200s-lctech-pi.dts
+++ b/arch/arm/boot/dts/allwinner/suniv-f1c200s-lctech-pi.dts
@@ -16,7 +16,8 @@ / {
                     "allwinner,suniv-f1c100s";

        aliases {
-               serial0 = &uart1;
+               //serial0 = &uart1;
+               serial0 = &uart2;
        };

        chosen {
@@ -55,12 +56,20 @@ flash@0 {
        };
 };

-&uart1 {
+//&uart1 {
+//     pinctrl-names = "default";
+//     pinctrl-0 = <&uart1_pa_pins>;
+//     status = "okay";
+//};
+
+&uart2 {
        pinctrl-names = "default";
-       pinctrl-0 = <&uart1_pa_pins>;
+       pinctrl-0 = <&uart2_pe_pins>;
        status = "okay";
 };

+
+
 /*
  * This is a Type-C socket, but CC1/2 are not connected, and VBUS is connected
  * to Vin, which supplies the board. Host mode works (if the board is powered
ubuntu@VM-8-10-ubuntu:~/Linux/linux$
```



​	可以看到我将`serial0`指向了`uart2`， 在内核的驱动中会直接将`uart2`在设备中生成`ttyS0`， 所以u-boot指定的内核启动参数应该是：console=ttyS0,115200。传递给内核的还有一个参数比较有意思：rootwait, 如果没有加这个参数，那么会挂载文件系统失败；加了这个参数，挂载文件系统才会成功【我的文件系统在SD卡中是做好的，EXT4格式】



内核的启动参数的打印：Kernel Hacking->arm Debugging

![kernel-haking](https://github.com/werido-xo/Mini-F1C200S/blob/TLV62569/Image/kernel-hacking.png)

这里填写UART的基地址后，就可以打印Linux内核启动的详细信息了，对于调试这一点还是非常有用的。关于这里为什么可以，其实和硬件寄存器有一点关系：

![uart](https://github.com/werido-xo/Mini-F1C200S/blob/TLV62569/Image/uart2.png)

也就是说我们可以直接将要发送的数据写入0x00偏移的地址就可以通过UART发送调试信息了。可以参考如下的博客：https://whycan.com/p_94368.html



到这里第一版的调试基本就完成！
