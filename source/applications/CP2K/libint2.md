## 自行构建适合CP2K使用的libint2双电子积分库的方法

***Last Updated: 2025-02-15***

以下笔记以目前最新的libint v2.13.1为例（注意旧版的cmake指令可能有不同，请随机应变）。

Libint的源码可从[这里](https://github.com/evaleev/libint/releases)下载；注意应当下载的是发布信息中Assets列表里最底下的 “Source code (tar.gz)”。

关于libint2是如何自定义编译的，可以参考以下两个链接，一个是libint官方给的说明，一个是Spack中用Autotools配置libint（截至v2.11.2）的具体细节：

* [https://github.com/evaleev/libint/wiki](https://github.com/evaleev/libint/wiki)

* [https://github.com/spack/spack-packages/blob/develop/repos/spack_repo/builtin/packages/libint/package.py](https://github.com/spack/spack-packages/blob/develop/repos/spack_repo/builtin/packages/libint/package.py)

### 预构建可以直接用于编译的源码包

首先安装好此步骤需要的“boost-devel”和“eigen3-devel”两个包：

```
sudo dnf install boost-devel eigen3-devel -y
```

然后，打开下载好的libint-2.13.1.tar.gz，依次执行下列命令：（其中的`{lmax}`等地方记得替换为想要的数字，CP2K支持4～7这四个数字，toolchain中默认用5；lmax指所得到的程序支持的最大角动量，越大的数字意味着越大的程序包大小和越高的编译耗时；最后一步中的`N`替换为要并行的CPU核数）

```
mkdir build && cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_POSITION_INDEPENDENT_CODE=ON \
  -DLIBINT2_ENABLE_ERI=1 \
  -DLIBINT2_ENABLE_ERI2=1 \
  -DLIBINT2_ENABLE_ERI3=1 \
  -DLIBINT2_MAX_AM={lmax} \
  -DLIBINT2_ERI_MAX_AM="{lmax};{lmax-1}" \
  -DLIBINT2_ERI2_MAX_AM="{lmax+2};{lmax+1}" \
  -DLIBINT2_ERI3_MAX_AM="{lmax+2};{lmax+1}" \
  -DLIBINT2_OPT_AM=3 \
  -DLIBINT2_ENABLE_FORTRAN=ON \
  -DLIBINT2_ENABLE_UNROLLING=0
make export -j N
```

构建好后，当前的build目录会生成一个压缩包“libint-2.13.1-post999.tgz”和一个目录“libint-2.13.1-post999”。这两个地方所包含的东西是一样的。你可以将目录名字自行更改为与CP2K toolchain中的格式相匹配的形式并重新压缩。

这一步通常比较耗时（尤其`{lmax}`为6或7的情况下）；如果懒得自己动手构建，我自己已经根据这一步的做法构建出了一系列可以直接拿来编译的源代码包并托管到了[本RTD项目GitHub仓库的Release页面](https://github.com/Growl1234/RTDProject/releases/tag/libint-cp2k)，且实测编译可行且用于CP2K上可以通过与之相关的所有regtests，欢迎大家去下载和使用。

### 从上面构建好的源码包编译libint

假设你直接执行了上面步骤，没有经过重命名等操作，解压“libint-2.13.1-post999.tgz”或者直接复制“libint-2.13.1-post999”目录到你想放置的路径，运行如下命令：（参考toolchain脚本中的CMake指令；此处默认你安装到自己源码目录下的install文件夹里）

```
mkdir build && cd build
CXXFLAGS="-g1" cmake .. \
  -DCMAKE_INSTALL_PREFIX=../install \
  -DLIBINT2_INSTALL_LIBDIR=../install/lib \
  -DLIBINT2_ENABLE_FORTRAN=ON
make install -j N
```

### 关于toolchain中编译libint包的一个小tip

用来计算CP2K杂化泛函任务中的双电子积分的libint库是整个toolchain过程中编译最耗时间的包之一，笔者在自己的24核笔记本上跑toolchain，在libint这一步要花3～5分钟才能编译完成。如果你安装在个人电脑上（此时往往算不动杂化泛函）或者完全用不到杂化泛函计算，那么libint可以不安装，或者降低支持的角动量（toolchain中相应选项为`--libint-lmax`，默认为5，可以降低到4即最高支持到g角动量，编译耗时能降低到默认情况下的约3/4）。

