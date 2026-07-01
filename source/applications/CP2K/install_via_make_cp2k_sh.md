## 利用全自动化的“make_cp2k.sh”从Spack构建依赖并编译安装CP2K

***Last Updated: 2026-07-01***

自CP2K v2026.1发布以来，开发者正推动继迁移至CMake后的另一重大改变：利用几乎专为科学计算服务的第三方包管理器Spack为CP2K提供和安装依赖。目前的开发分支在CP2K程序包根目录下新增了一个脚本`make_cp2k.sh`，旨在很好地实现这一目的；用户只需按照提示运行这个脚本，就可以构建依赖+编译CP2K一条龙全自动化实现，相当方便，将于2026.2版本可用。该脚本要求Bash版本>=4、Python版本>=3.10。

用户可以随时通过`./make_cp2k.sh -h`命令来获得该脚本的使用帮助和提示。以下是我使用`make_cp2k.sh`时的构建指令：

```bash
./make_cp2k.sh \
 -cv psmp \
 -df ace \
 -df deepmd \
 -df dlaf \
 -df gauxc \
 -df greenx \
 -df libsmeagol \
 -df libtorch \
 -df mimic \
 -df openpmd \
 -df pexsi \
 -df plumed \
 -df sirius \
 -ue \
 -j 24 -np 2
```

以下做一点解释：

* 如果在脚本运行过程中因出错等任何原因终止，重新运行时脚本若检测到有`spack/`目录存在但依赖构建未完成，不会继续在已有Spack环境中续跑，而是直接报错不干；因此必须带`-bd`（`--build_deps`）flag或者在运行前手动删除spack目录，且Spack本体下载和依赖安装流程都会从头开始。如果希望在万一遇到这种情况时节省时间，请务必避免使用`-uc none`（见下面关于`-uc`的说明），否则无法利用已有缓存，所有依赖都会从头下载和重新编译。

* `-cv`即`--cp2k_version`，指定你想安装的CP2K类型，目前该脚本支持五种：`psmp`、`ssmp`、`ssmp-static`、`pdbg`和`sdbg`，默认为`psmp`，即允许MPI并行，这也是最常用的版本。如果用了`-cv psmp`或`-cv pdbg`，则`-mpi`即`--mpi_mode`将会起作用，默认为`mpich`，还可以用`openmpi`；如果用了`-cv ssmp`或`-cv ssmp-static`或`-cv sdbg`，则`-mpi`将被设为`no`，你在实际运行命令时最好不传这个参数。如果你手动指定了与所想构建的CP2K版本不符的MPI选项，脚本将报错退出。

* `-df`即`--disable_feature`，这个选项可以用来取消你不需要的依赖支持，这样Spack就不会下载和安装你不需要的依赖，其余的都安装；相应地，还有个`-ef`即`--enable_feature`，意思与上面恰好相反，若手动指定`-ef`所跟的安装包，Spack就下载和安装相应的包。相应地，下面自动执行的CMake指令也会实施上述更改。默认配置相当于`-ef all -df dlaf`，对于GCC>=15或CUDA构建还禁用PEXSI。注意：1. 这两个选项不能同时跟多个包，必须为你想实施操作的每个依赖单独使用`-df`或`-ef`；2.`-ef`选项 **不会覆盖`-ef all`** ，因此如果你想 **只安装** 某些依赖（相当于只启用对应的特定功能），应当先加`-df all`来禁用所有可选依赖，然后再用`-ef`来加你想装的可选依赖（也可以像我一样直接使用`-df`选项去掉你不想要的其他依赖）；3. libvdwxc、spfft、spla、sirius这四个包有相互捆绑和依赖的关系，因此启用或禁用其中任何一个都会导致其中剩余的几个包一并被启用或禁用。

* `-uc`（即 `--use_cache`）用来选择 Spack 构建 CP2K 依赖时的缓存方式：默认的 `-uc folder` 会将已构建的软件包保存在本地目录缓存中，使后续重新创建 Spack 环境时能够复用这些包，避免重复下载和编译；`-uc minio` 则使用由本机 Podman 提供的 MinIO 对象存储作为缓存；`-uc no` 禁用缓存，无需为缓存创建额外的 Python 虚拟环境及相关依赖，也不会占用缓存空间，代价就是万一中间某一步失败了，重新运行的时候所有依赖都要从头下载和安装。

* `-ue`即`--use_externals`，用了这个就会通过`spack external find`尝试把一些系统已有且可被Spack识别的软件包比如python添加进Spack中用于后续安装流程。注意一些非系统基础包，比如MPI和数学库，需要你在配置文件中额外指定路径才可以使得`-ue`对其生效。目前在使用`-ue`时，如果你的系统安装了meson，那么Spack会尝试使用自带的Python作为meson构建的依赖，这可能与系统已有的Python发生冲突，此时应当去掉`-ue`或者同时加上`-uc no`禁用本地缓存功能来解决该问题。

* `-j M`和`-np N`均为并行相关设置。“M”就是总的并行核数，如果不设置，脚本将通过`nproc`检测你的电脑CPU的可用线程数并将其设为默认值（然而这一命令受到额外的cgroup限制）。N则为Spack安装依赖过程中并行安装的包的数目，默认为2；由于部分包的个别编译步骤时间较长（例如libxc），因此可以通过恰当设置这一选项来加快整体构建速度。

* 若依赖安装已经完成，只想重新运行CP2K编译，请使用`-rc`（即`--rebuild_cp2k`），该选项将删除build和install目录并重新编译CP2K，但不动spack目录。

* 若想编译完成后还跑regtest，可以加`-t`（即`--test`），然后跟上你想要的设置，比如`-t "--mpiranks 2 --ompthreads 1 --maxtasks 24 --flagslow"`。
