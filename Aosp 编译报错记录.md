# 1 启动emulator报错

报错信息：
qemu-system-x86_64: Could not open '/home/ferdinand/AospProject/android13_r16/out/target/product/generic_x86_64/userdata-qemu.img': No such file or directory

报错截图：

![](D:\GithubProjects\images\2024-11-30-14-18-25-image.png)

报错分析：

缺少userdata-qemu.img文件

解决方法：

修改AndroidProducts.mk文件，并重新lunch sdk_phone_x86_64-eng并编译。


![](D:\GithubProjects\images\2024-11-30-14-19-20-image.png)