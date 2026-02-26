## 利用toolchain编译和配置CP2K

***Last Updated: 2026-02-26***

**看思想家公社（sobereva）的文章[《CP2K第一性原理程序在Linux中的安装方法》](http://sobereva.com/586)即可，toolchain一步可以根据自己的实际需求作修改。**

### 关于toolchain选项的一点补充说明

* toolchain会自动检查系统是否存在MKL配置，如果没有检测到MKL（包括新的oneMKL，下同），默认会把OpenBLAS、ScaLAPACK和FFTW一起安装下来，此时如果你已经事先有安装它们（包括其中任意一个）且环境变量配置正确，最好写上例如 `--with-openblas=system` 这样的选项。**注意：无论OpenBLAS是否需要作为数学库安装，也无论其是否被标为“system”，其源码都会被下载并解压用以执行另外一个必要的检查步骤，而关于OpenBLAS的一切直接或间接设定只能影响其会不会被编译和安装。**

* **不要使用Intel oneAPI做并行化编译器，因为CP2K（截至版本2026.1）还没有做好对ifx的支持（新的oneAPI已经不再支持较旧的ifort），** 虽然toolchain一步可能会成功但后续正式编译步骤会编译不过去（亲身实践教训：版本2025.2或2025.1中使用传统make构建时会内存溢出；2026.1中会在make install最后几步出错，但即使修复了问题可以编译成功，性能也依然不如GNU标准套餐编译的好，甚至还会有内存爬坡现象，我猜与CP2K的源代码跟oneAPI里面的ifort/ifx之间存在的底层不兼容问题有关）。不过，Intel oneMKL是可以支持的；根据笔者在Rocky Linux 10.1上的最新测试，GCC 14.3.1 + OpenMPI 5.0.9 + Intel oneMKL 2025.2 这个组合跑得不错，没有问题，尽管并不比用默认的OpenBLAS+FFTW+ScaLAPACK做数学库有什么优势。

* 注意使用OpenMPI作为并行工具时的CP2K在MPI+OpenMP混合并行时有问题，会强制绑定到前几个线程（包括开启了超线程的情况，此时运行会极慢），此时必须加上`--map-by node`才能正常并行。

* 如果你使用了`--with-openblas=system`或`--with-scalapack==system`，但没有把这些软件装到系统默认路径`/usr`内，请在安装ELPA前使用`export LIBRARY_PATH=`（不是`LD_LIBRARY_PATH`）命令使得这些库能够在ELPA编译时被搜索到，或者在`/usr/lib64`下创建这些库文件的软链接；如果用了`--with-fftw=system`且没有加`--with-sirius=no`，由于这意味着要同时安装libvdwxc，因此要把fftw的`CPATH`和`LIBRARY_PATH`都提前导入环境变量。

* `--with-tblite` 代表安装Grimme的tblite程序，如果加了这个那么ninja会被自动安装（相当于加了 `--with-ninja`）。按照CP2K的说明，tblite同时包含DFT-D4，因此这时也就不用刻意加 `--with-dftd4` 了（即使加上了也会被自动跳过）。

* 如果你使用自行事先编译的HDF5并加了 `--with-hdf5=system` 选项，toolchain脚本默认会自动把“-lsz”加到arch设置里，此时应当在正式编译CP2K前先运行 `sudo dnf install libaec-devel` 命令装上libsz，否则会出现“找不到-lsz”的错误提示。这个小问题会在未来的2026.2版本中解决。

### 安装完toolchain中的依赖后用CMake编译CP2K

**从版本2026.1开始，CP2K的编译已全面转为cmake，彻底放弃GNU makefile和相应的arch文件集。** 目前发行版本中toolchain尚未实现针对自定义的配置设计合适的cmake指令，因此只能自己根据CMakeLists.txt里面的选项逐个添加与既有toolchain配置相对应的到命令行中，比较麻烦；另外，通过cmake无法同时编译ssmp和psmp（有MPI就只编译psmp，否则只编译ssmp），且编译成的程序没有相应的符号链接sopt和popt，不过这不算什么大问题，毕竟psmp同时支持MPI和OpenMP并行，只要设置OMP_NUM_THREADS为物理核心数且不用mpirun指令就相当于运行ssmp，只要`export OMP_NUM_THREADS=1` 并用`mpirun -np N` （N为并行核数）运行就相当于运行popt了；此外，我发现CP2K对sopt/popt版本中相应设置的实现是通过[Fortran主代码中的几行](https://github.com/cp2k/cp2k/blob/master/src/start/cp2k.F#L155-L159)完成的，因此实际上你完全可以在二进制文件目录下自己创建这样的符号链接。***（补充：目前开发分支已经在cmake的install一步加回了popt和sopt符号链接的创建命令）***

以下是在toolchain配置完成后利用CMake编译CP2K的步骤（以2026.1为例）：

1. 按照控制台输出所说明的执行`source /root/CP2K/src/cp2k-2026.1/tools/toolchain/install/setup`。

2. 切到cp2k源码目录，执行`mkdir build && cd build`，进入构建和编译专用目录。

3. 运行构建指令。由于前面说过的原因，这里需要手动敲入构建选项，比如我的toolchain选项为：

```bash
./install_cp2k_toolchain.sh --with-cmake=system \
                            --with-ninja=system \
                            --with-sirius=no \
                            --with-openblas=system \
                            --with-fftw=system \
                            --with-scalapack=system \
                            --with-hdf5=system \
                            --with-tblite \
                            --libint-lmax=4 -j 24
```

这里包括了自己在系统单另已经安装好的cmake、ninja、MPI、OpenBLAS、ScaLAPACK、FFTW3和HDF5，toolchain默认安装的libint、libXC、libXSMM、Spglib、COSMA、ELPA、libvori，以及我选定安装的tblite。那么我的cmake预配置选项即如下所示，可见相当冗长、麻烦：

```bash
cmake .. -DCMAKE_INSTALL_PREFIX=../install \
         -DCP2K_DATA_DIR=/root/CP2K/src/cp2k-2026.1/data \
         -DCP2K_USE_MPI=ON \
         -DCP2K_USE_MPI_F08=ON \
         -DCP2K_USE_FFTW3=ON \
         -DCP2K_USE_LIBINT2=ON \
         -DCP2K_USE_LIBXC=ON \
         -DCP2K_USE_LIBXSMM=ON \
         -DCP2K_USE_COSMA=ON \
         -DCP2K_USE_ELPA=ON \
         -DCP2K_USE_SPGLIB=ON \
         -DCP2K_USE_HDF5=ON \
         -DCP2K_USE_VORI=ON \
         -DCP2K_USE_TBLITE=ON
```

其中`-DCMAKE_INSTALL_PREFIX` 设置到自己想安装到的路径（可以使用相对路径；为省事我直接设置在了父目录下一个新的文件夹；如果不设置，默认将为/usr/local）。由于OpenBLAS是强制性的、Scalapack在有MPI的情况下是强制性的，因此无论如何它们都会被检查，所以这里无需写出。之所以特别设置`-DCP2K_DATA_DIR=/root/CP2K/src/cp2k-2026.1/data`，是因为cmake构建系统安装好后默认读取基组的位置是`${CMAKE_INSTALL_PREFIX}/shared/cp2k/data`，会导致编译时生成与`/root/CP2K/src/cp2k-2026.1/data`内容完全重复的`/root/CP2K/src/cp2k-2026.1/install/shared/cp2k/data`目录，加上这一设置可以避免这一问题（但注意这里不能使用相对路径）。

**<font color=blue>2026-01-23补充：为解决需要自己手动添加cmake选项的麻烦，我在源代码包的toolchain模块中加入了能够让toolchain自动根据自己的安装选项生成相应cmake指令的脚本；相关变动我已在CP2K的GitHub仓库上提交Pull Request并被成功合并，未来的2026.2版本中会包含这一功能。大家可以直接去GitHub官方仓库下载现行开发分支来用，这样你就只需要复制toolchain安装成功后屏幕上提示的cmake指令就可以了。</font>**

4. 构建完成后，运行`make install -jN`，N是并行核数。

5. 写入以下三行至~/.bashrc中以添加环境变量：

```bash
source /root/CP2K/src/cp2k-2026.1/tools/toolchain/install/setup
export PATH=$PATH:/root/CP2K/src/cp2k-2026.1/install/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/root/CP2K/src/cp2k-2026.1/install/lib64
```

6. 删除该build文件夹（或者不删除但执行`make clean`）以腾出部分空间。

