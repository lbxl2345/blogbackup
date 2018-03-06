#### 关于UEFI，安全启动
#### UEFI是什么？
Unified Extensible Firmware Interface，统一可扩展固件接口。它是**操作系统**和**平台固件**之间的接口，是一个**标准**。和BIOS相比，它（1）不需要使用平台相关的汇编语言，而可以通过C/C++语言来进行编写（这样就能够进行上层的复用）;（2）支持模块化的驱动（动态加载），包含版本信息，升级更方便；（3）性能更好（不使用中断，而是使用event进行异步操作）；（4）**安全性更好**。UEFI在执行应用程序/驱动前，会先检测应用程序/驱动的证书，只有证书被信任时才会执行应用程序/驱动（格式为PE/COFF）。
#### meta-efi-secure-boot源码分析
UEFI firmware boot manager
#### shim
github上，shim的简介写道：shim is a trivial EFI application that, when run, attempts to open and execute another application. 它是一个用来“执行其他程序”的程序：它首先会对程序进行安全性检查，只有通过安全性检查的程序才会被真正执行。我把它理解成在程序运行前进行检查的工具。
#### yocto
`.bb`和`.bbappend`文件都是yocto的定制文件

http:version_tab