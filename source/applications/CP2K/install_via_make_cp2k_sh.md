## 利用全自动化的“make_cp2k.sh”从Spack构建依赖并编译安装CP2K

***Last Updated: 2026-03-06***

<font color=red>

</font>

自CP2K v2026.1发布以来，开发者正推动继迁移至CMake后的另一重大改变：放弃原有的toolchain安装链，转为利用专为科学计算服务的第三方包管理器Spack为CP2K提供和安装依赖。目前的开发分支在CP2K程序包根目录下新增了一个脚本`make_cp2k.sh`，旨在很好地实现这一目的；用户只需按照提示运行这个脚本，就可以构建依赖+编译CP2K一条龙全自动化实现，相当方便。

目前这个脚本`make_cp2k.sh`已经相当完善，不过还是难免有一些漏洞之处；但照现在非常迅速的发展进度，我预计等到2026.2版本发布时，这一脚本应该已经能够几乎完全平替toolchain。

用户可以随时通过`./make_cp2k.sh -h`命令来获得该脚本的使用帮助和提示。以下是我使用`make_cp2k.sh`时的构建指令：

```bash
./make_cp2k.sh \
 -cv psmp \
 -mpi openmpi \
 -df ace \
 -df deepmd \
 -df dlaf \
 -df greenx \
 -df libsmeagol \
 -df libtorch \
 -df mimic \
 -df openpmd \
 -df pexsi \
 -df plumed \
 -df sirius \
 -df trexio \
 -dlc -ue \
 -j 24 -np 2
```

以下做一点解释：

* `-cv`即`--cp2k_version`，指定你想安装的CP2K版本，目前该脚本支持三种：`psmp`、`ssmp`和`ssmp-static`，默认为`psmp`，即允许MPI并行，这也是最常用的版本。如果用了`-cv psmp`，则`-mpi`即`--mpi_mode`将会起作用，默认为`mpich`，可以改为更流行的`openmpi`；如果用了`-cv ssmp`或`-cv ssmp-static`，则`-mpi`将被设为`no`。如果你手动指定了与所想构建的CP2K版本不符的MPI选项，脚本将报错退出。

* `-df`即`--disable_feature`，这个选项可以用来取消你不需要的依赖支持，这样Spack就不会下载和安装你所需要的依赖，其余的都安装；相应地，还有个`-ef`即`--enable_feature`，意思与上面恰好相反，若手动指定`-ef`所跟的安装包，Spack就**只**下载和安装相应的包，其余依赖都不安装。相应地，下面自动执行的CMake指令也会实施上述更改。默认为`-ef all`。注意：1. 这两个选项不能同时跟多个包，必须为你想实施操作的每个依赖单独使用`-df`或`-ef`；2. libvdwxc、spfft、spla、sirius这四个包有相互捆绑和依赖的关系，因此启用或禁用其中任何一个都会导致其中剩余的几个包一并被启用或禁用。

* `-dlc`即`--disable_local_cache`,这个的意思是不添加/保留本地缓存，个人认为主要作用是节省空间。

* `-ue`即`--use_externals`，用了这个就会把一些系统已有的Spack依赖比如python添加进Spack所使用的环境变量中。注意一些非系统基础包，比如MPI和数学库，需要你在配置文件中额外指定路径才可以使得`-ue`对其生效。

* `-j M`和`-np N`均为并行相关设置。“M”就是总的并行核数，如果不设置，脚本将通过`nproc`检测你的电脑CPU的可用线程数并将其设为默认值（然而这一命令受到额外的cgroup限制）。N则为Spack安装依赖过程中并行安装的包的数目，默认为4；由于部分包的个别编译步骤时间较长（例如libxc），因此可以通过恰当设置这一选项来加快整体构建速度。
