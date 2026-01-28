## Rocky Linux 10 中使用root用户的一些简要说明、注意事项及障碍解决

***Last Updated: 2025-11-27***

**【写在前面：root有风险，使用root用户时请谨慎操作，不要乱改系统文件！】**

### 我可以用root吗？

对于在PC上独自使用该系统的情况，如果对Linux有一定使用经验和自信，可以直接用root用户而非普通用户，这样没有权限限制，能省去一些不必要的麻烦，通常也不必过分担心因为用root而把系统弄坏（只要不乱搞；而做到这一点也很简单，遇到相关问题多动脑、多google即可）。但如果你对Linux系统熟悉和熟练程度不够，则不建议贸然使用root，以免不小心篡改了系统文件导致问题，毕竟使用root后你对系统文件的任何操作都几乎不受约束。如果是集体使用的计算机乃至计算集群，则肯定不能随随便便允许用root了，你一个不经意的越界操作就有可能对整个集群造成不良后果；通常只有指定的专门负责人才有使用root的权限和资格，且多数普通用户也没有管理员权限而不能运行sudo指令。

### 不想用root又嫌sudo操作要输入密码麻烦，怎么办？

**执行 `sudo visudo` 指令编辑相应管理文件，找到 `Same thing without a password` 一行，把下面的 `# %wheel	ALL=(ALL)	NOPASSWD: ALL` 一行去掉注释（即最前面的“#”）即可。** 注意这种方法只能免去在终端指令行中利用sudo执行指令的麻烦，在GNOME图形界面中若有涉及管理员权限的操作仍然需要频繁输入用户密码。

### root用户下系统无法输出声音，怎么办？

这种情况是因为**系统使用的pipewire组件配置屏蔽了root用户**。如果你使用root用户并且对声音输出有需要，只需在/usr/lib/systemd/user目录下找到pipewire.socket、pipewire.service、pipewire-pulse.socket、pipewire-pulse.service这四个文件，并将里面的"ConditionUser=!root"一行删掉即可，完后运行  `sudo systemctl daemon-reload && systemctl --user restart pipewire.service pipewire-pulse.service ` 指令或者重启电脑，音频输出就正常了。

### root无法正常运行一些程序，怎么办？

分为两种情况：

1. 对于**以沙盒方式运行**的程序，root用户无法运行之，必须在执行指令中加上  `--no-sandbox` 使其脱离沙盒环境运行。（也因为这个特性，我不建议在root用户下通过flatpak下载和安装这类软件，通过rpm安装的或者直接给了AppImage的要加 `--no-sandbox` 执行比flatpak软件包要方便不少。）

2. 不少程序包**本身**对root设置了额外的限制，这其中包括科学计算常用的OpenMPI、GNOME 46起的文件资源管理器Nautilus等。这种限制看似是对系统的一种保护，但在我来看颇有多管闲事之嫌，毕竟我自己就是用root用户，因为这样省去了许多不必要的麻烦，而且我从来没有也不可能因为root把系统弄坏（除非脑子坏了）。对于一些常用软件，可以通过修改源代码和设置环境变量等方法解决。

下面针对第二种情况给出三个示例，分别是GNOME文件资源管理器nautilus、MPI并行工具OpenMPI和功能强大的视频播放器VLC。

**<font size="4">(1) Nautilus（GNOME文件资源管理器）</font>**

<div style="margin-left: 20px; margin-right: 20px;">

在Rocky Linux 10使用的GNOME 47中，nautilus会在root用户打开时延迟、卡顿好几秒钟才能进入；从命令行运行nautilus指令时，会输出警告： ``This app cannot work correctly if run as root (not even with sudo). Consider running `nautilus admin:/` instead. ``。这种迷之操作挺令我感觉不适的。后来终于想到检查源代码了，检查发现原来是源代码目录下的src/nautilus-main.c文件中用if代码施加了这一限制操作，即用户启动nautilus时程序会自动检查当前用户身份，发现是root就执行限制操作（ `sleep (7)` 即延后七秒执行后续代码）。这种限制给我的感觉纯粹是多管闲事、败坏好感！

</div>

<div style="margin-left: 20px; margin-right: 20px;">

既然原因找出来了，解决办法也就显而易见了：<strong>直接修改源代码！</strong>完整步骤如下 <font color=blue>（如果懒得看，直接跳过步骤翻到后面，我给了已构建好的rpm包）</font>：

</div>

<div style="margin-left: 20px; margin-right: 20px;">

<strong>①</strong> 从镜像库中找到并下载软件对应的src.rpm文件（截至Rocky Linux 10.1，文件为<a href="https://mirrors.ustc.edu.cn/rocky/10/AppStream/source/tree/Packages/n/nautilus-47.1-1.el10.src.rpm">nautilus-47.1-1.el10.src.rpm</a>；超链接指向中科大镜像站的该文件，点击即可下载），执行 `rpmbuild -ivh nautilus-47.1-1.el10.src.rpm` 释放出真实的源代码架构和构建配置。释放后的文件架构在~/rpmbuild中，你应该在这里看到SOURCES和SPECS两个文件夹，其中SOURCES存放源代码压缩包（tar.xz），SPECS存放构建rpm的配置文件。

</div>

<div style="margin-left: 20px; margin-right: 20px;">

<strong>②</strong> 进入SOURCES文件夹，解压“nautilus-47.1.tar.xz”；随后进入文件夹，在src/nautilus-main.c文件里面找到“if (getuid () == 0)”，将对应段落（一直到这个if对应的}符号为止）删掉，保存。回到SOURCES目录，执行 `tar -cJf nautilus-47.1.tar.xz  nautilus-47.1` 将包含已修改代码的文件夹重新压缩成tar.xz以替代原来的压缩包。

</div>

<div style="margin-left: 20px; margin-right: 20px;">

<strong>③</strong> 执行 `export LD_LIBRARY_PATH= && export LD_RUN_PATH=` 以在当前终端临时清空这两个环境变量（否则这些里面包含的路径对构建过程造成干扰导致构建失败）；然后切到~/rpmbuild/SPECS目录下，运行 `rpmbuild -bb nautilus.spec` 重新构建二进制安装包。

</div>

<div style="margin-left: 40px; margin-right: 40px;">

<strong>注意这一步的构建需要很多额外的依赖程序包，</strong>如果缺失的话会在运行rpm -bb构建指令时一开始就报错并给出很清晰的提示；基本上每行的needed的前面都直接就是软件包名，如果是pkgconfig开头的则是后面括号里的是软件包名（有少数几个括号中gstream开头的则并非如此，它们是gstreamer1-plugins-base-devel的组件，对应需要安装的是gstreamer1-plugins-base-devel）；如果搞不懂的话，把那若干行贴出来给grok并问在Rocky Linux 10中分别需要什么包就可以得到答案。注意其中包括meson包，这个只有CRB仓库中有，因此需要先运行 `dnf config-manager --set-enabled crb` 启用CRB库才能装meson。

</div>

<div style="margin-left: 20px; margin-right: 20px;">

<strong>④</strong> 切到~/rpmbuild/RPMS/x86_64目录下，执行 `rpm -ivh --reinstall nautilus-47.1-1.el10.x86_64.rpm` 以覆盖安装nautilus（在覆盖安装前最好关闭文件资源管理器窗口）。

</div>

<div style="margin-left: 20px; margin-right: 20px;">

以上步骤完成后，你在root下打开nautilus就再也不会因为你是root而有那么长时间的延迟了。

</div>

<div style="margin-left: 20px; margin-right: 20px;">

<strong>与目前Rocky Linux 10使用的GNOME 47相对应的nautilus 47.1-1的rpm包我已重新构建好，我把压缩包直接放到下面供大家取用。里面有一个Note.txt文件，可以在操作覆盖安装之前看看。倘若日后系统仓库中的nautilus有小更新，我会及时将这里的安装包也同步到最新版本。</strong>

</div>

<div style="margin-left: 20px; margin-right: 20px; font-size: 17px;">

<strong><a href="../../_static/packages/nautilus.7z">📦nautilus.7z</a></strong>
（由于nautilus为自由软件，该重构包大家可以随意传播）

</div>

<div style="margin-left: 20px; margin-right: 20px;">

另注：Rocky Linux 9.x中的nautilus完全不存在这一问题。在Fedora 43使用的GNOME 49中，root用户下直接打不开nautilus窗口了，这个警告也变成了 `Running nautilus as root is not supported.`，原因和解决办法与上面所说相同。

</div>

**<font size="4">(2) OpenMPI</font>**

<div style="margin-left: 20px; margin-right: 20px;">

OpenMPI在root下运行mpirun时候，要求指令中额外加上 `--allow-run-as-root` 选项，然而对于一些将mpirun指令内置的程序（比如ORCA），想实现这种做法却没那么容易和直接（直接写指令或alias识别不进去，可能需要用到函数功能）。想要避免这种情况，同时让自己不再需要加 `--allow-run-as-root`，可以通过<strong>写入环境变量</strong>来解决，即往~/.bashrc文件中加入以下两行：

</div>

<div style="margin-left: 20px; margin-right: 20px;">

```bash
export OMPI_ALLOW_RUN_AS_ROOT=1
export OMPI_ALLOW_RUN_AS_ROOT_CONFIRM=1
```

</div>

<div style="margin-left: 20px; margin-right: 20px;">

这样，在 `source ~/.bashrc` 或者打开新终端后，就可以不加 `--allow-run-as-root` 直接运行mpirun任务了。

</div>

<div style="margin-left: 20px; margin-right: 20px;">

修改源代码的方法也可以实现上述目的，操作见<a href="http://sobereva.com/409">http://sobereva.com/409</a>，不过相比上面写入环境变量的做法麻烦多了（该博文也包含写入环境变量的做法）。

</div>

**<font size="4">(3) VLC</font>**

<div style="margin-left: 20px; margin-right: 20px;">

VLC是Linux下功能最强大的视频播放器之一，该程序默认在root用户中无法启动。解决方法很简单：运行指令 `sed -i 's/geteuid/getppid/' /usr/bin/vlc`（显然也是将getuid相应步骤去掉）。

</div>
