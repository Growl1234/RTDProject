## Rocky Linux安装NVIDIA显卡驱动的基本步骤和注意事项

***Last Updated: 2025-11-15***

**【写在前面：本文只是一些对官网教程的补充和强调，部分步骤官网给了很详细的教程我就直接贴链接了，不再赘述；此外，本文不适用于VMware虚拟机。】**

### 1. 安装驱动

严格按照[NVIDIA官方文档相应页面](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide)操作（正式安装部分Rocky Linux属于“Red Hat Enterprise Linux”一节），至重新启动之前（完成下面的步骤2和3前不要重启）。

**特别注意以下几点：**

1. 文档中向系统添加软件仓库的指令里的“$distro”和“$arch”应当根据教程章节开头的对应关系自行修改成相应值（本例中$distro是rhel10或rhel9，$arch是x86_64），否则系统无法识别相应部分，导致添加软件仓库失败。

2. NVIDIA显卡的Linux版驱动区分专有模块（即“cuda-drivers”）和开源模块（“nvidia-open”），两者没有任何功能上的区别，但适用的架构不尽相同。文档中“Kernel Modules”一节给了大致说明，并建议对较新架构的GPU选用开源模块（尤其注意今年最新的GeForce RTX 50系列，该系列已不再支持专有模块）。如果不确定自己的显卡应该选择哪种模块，请在添加CUDA仓库后先安装并运行“nvidia-driver-assistant”，输出结果会直接告诉你答案。

3. **<font color=red>不要自以为是或者听信一些低质量教程到NVIDIA驱动下载页面直接下载和运行“.run”后缀的程序来安装驱动！</font>这种安装方式早已因兼容性差和乱改系统文件而闻名遐迩。**


### 2. 禁用系统自带的开源驱动Nouveau，避免可能的冲突

运行以下指令：
```
sudo grubby --args="nouveau.modeset=0 rd.driver.blacklist=nouveau" --update-kernel=ALL
```

### 3. 关于安全启动（Secure Boot）的配置

大多数个人电脑通常都默认开启安全启动（Secure Boot），但这会阻碍NVIDIA显卡驱动在Linux下的运行；又鉴于安全启动能起到的实际保护作用微乎其微，建议直接在第四步重启时进入BIOS将其关掉。

如果不想关安全启动，则需要导入dkms的密钥让设备进入Linux时允许运行该驱动程序，操作步骤如下：

（1）运行指令：
```
sudo mokutil --import /var/lib/dkms/mok.pub
```

（2）按照提示设置密码，建议设简单些（一个0都行），反正这个密码只会用一次。

（3）重启系统时，会弹出MOK蓝色对话框。按任意键进入MOK管理界面，依次选Enroll MOK → Continue → Yes，然后输入上面你自己设置的密码，输入正确后选Reboot即可。


### 4. 重启系统

### 5. 运行指令“nvidia-smi”检查驱动是否被正确加载

如果正确加载，会输出显卡信息和使用显卡的进程信息。之后你也可以通过 <code>pip install nvitop</code> 安装nvitop工具来实时监测显卡占用情况。

经个人实测，无论在RHEL、CentOS Stream上还是Rocky、AlmaLinux等下游重构版上，严格按上述步骤安装驱动程序100%不会出问题。尽管如此，总会有人抱有侥幸心理，“我行我素”地盲目操作，结果就是安装不成功，<code>nvidia-smi</code> 也无法正常执行。关于常见的两种运行 <code>nvidia-smi</code> 时可能的报错，我还是提一下：

1. <code>NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver. Make sure that the latest NVIDIA driver is installed and running.</code>，即该指令压根就没有跟驱动程序正确对接上。出现这个问题，很有可能是没有注意关于安全启动的问题，把涉及安全启动那一步完整弄了就行；还可能是你的系统和软件环境跟驱动版本不兼容（基本是系统内核或软件仓库太老或者太新，例如Fedora常常因为更新策略太激进而导致在最新发布版上出现这种不兼容问题，此时再黏着官方仓库不放也解决不了问题了，只能找找其他能够提供驱动程序的可靠仓库，比如ELRepo和RPM Fusion）。

2. <code>No devices were found.</code>，即该指令与驱动程序正确对接了，但却没有正确识别显卡设备。对于这个问题，应当认真查看相关的日志文件（一个有用的日志查看指令为 <code>journalctl</code>，建议了解下其用法），寻找病因并对症下药。选错安装模块是导致这一报错最常见的原因之一（例如，如果你的显卡是GeForce RTX 50系列但是选择安装了专有模块，运行 <code>nvidia-smi</code> 出现了这一报错，进行日志排查将发现其中明确写了“The NVIDIA GPU installed in this system requires use of the NVIDIA open kernel modules.”）。另外，记住这个指令：<code>nvidia-modprobe && nvidia-modprobe -u</code>，它往往在错误排查和问题解决中起到不小的辅助作用。

如果你非要一根筋不信邪地通过“.run”驱动安装程序装驱动，那么上面两种报错都有几率触发，或者即使 <code>nvidia-smi</code> 能正常显示却实际上还有其他各种各样的问题。解决办法就是把该驱动程序卸载掉（执行相应.run程序时后面加 <code>--uninstall</code> ），然后老老实实按第一部分中的官方文档来。

对于一些电脑，如果BIOS中设置为独显直连（关闭集显或核显），当Linux系统没有正确加载驱动时，图形界面会直接以800*600等极低分辨率显示，且无法调节亮度。此时虽然可以确定显卡驱动加载出了问题，但具体问题仍然需要根据 <code>nvidia-smi</code> 的输出结果进行判断。
