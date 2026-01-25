## 从源代码配置CP2K

***Last Updated: 2025-01-23***

**看思想家公社（sobereva）的文章[《CP2K第一性原理程序在Linux中的安装方法》](http://sobereva.com/586)即可，toolchain一步可以根据自己的实际需求作修改。**

下面给一点补充说明：

* toolchain会自动检查系统是否存在MKL配置，如果没有检测到MKL（包括新的oneMKL，下同），默认会把OpenBLAS、ScaLAPACK和FFTW一起安装下来，此时如果你已经事先有安装它们（包括其中任意一个）且环境变量配置正确，最好写上例如 <code style="font-size: 14px;">\--with-openblas=system</code> 这样的选项。**注意：无论OpenBLAS是否需要作为数学库安装，也无论其是否被标为“system”，其源码都会被下载并解压用以执行另外一个必要的检查步骤，而关于OpenBLAS的一切直接或间接设定只能影响其会不会被编译和安装。**

* **不要使用Intel oneAPI做并行化编译器，因为CP2K（截至版本2026.1）还没有做好对oneAPI的支持（尤其是ifx；新的oneAPI已经不再支持较旧的icc/icpc/ifort），** 虽然toolchain一步可能会成功但后续正式编译步骤会编译不过去（亲身实践教训：版本2025.2或2025.1中使用传统make构建时会内存溢出，2026.1中会在make install最后几步出错）。与oneAPI的并行编译器不同，新的Intel oneMKL是受支持的（尽管相对于标准的OpenBLAS+FFTW+ScaLAPACK并没有什么比较明显的优势）；如果使用oneMKL，建议安装好oneMKL后进入fftw3xf目录（例如/opt/intel/oneapi/mkl/2025.1/share/mkl/interfaces/fftw3xf）手动编译产生fftw3库文件（在该目录运行 <code style="font-size: 14px;">make libintel64</code>；根据Makefile的设定，编译过程使用icx和gcc都可以，然而实际启动编译时如果没有检测到icx就跳过gcc检查直接报错，所以如果想用gcc必须显式指定“CC=gcc”）。

* 根据个人测试经验，OpenMPI并行结合oneMKL数学库选项配置的CP2K在运行时会出现内存配置错误，而MPICH不会，原因不明。鉴于这一情况，如果坚持使用Intel oneMKL作为数学库，那我建议你用MPICH作为并行化工具来配置和运行CP2K；而在使用开源的OpenBLAS、ScaLAPACK和FFTW组合当数学库时，则放心用社区流行度更高的OpenMPI。

* 用来计算CP2K杂化泛函任务中的双电子积分的libint库是整个toolchain过程中编译最耗时间的包之一；如果你安装在个人电脑上（此时往往算不动杂化泛函）或者完全用不到杂化泛函计算，那么libint可以不安装，或者降低支持的角动量（相应选项为`--libint-lmax`，支持4到7，越大的数字意味着越大的程序包大小和越高的编译耗时；默认为5，可以降低到4即最高支持到g角动量，编译耗时能降低到默认情况下的约3/4）。

* <code style="font-size: 14px;">\--with-tblite</code> 代表安装Grimme的tblite程序，要加这个必须同时加上 <code style="font-size: 14px;">\--with-ninja</code>。按照CP2K的说明，tblite同时包含DFT-D4，因此这时也就不用刻意加 <code style="font-size: 14px;">\--with-dftd4</code> 了（即使加上了也会被自动跳过；但是 <code style="font-size: 14px;">\--with-ninja</code> 还得有）。

* 如果你使用自行事先编译的HDF5并加了 <code style="font-size: 14px;">\--with-hdf5=system</code> 选项，toolchain脚本默认会自动把“-lsz”加到arch设置里，此时应当在正式编译CP2K前先运行 <code style="font-size: 14px;">sudo dnf install libaec-devel</code> 命令装上libsz，否则会出现“找不到-lsz”的错误提示。

* **从版本2026.1开始，CP2K的编译将全面转为cmake，彻底放弃GNU makefile和相应的arch文件集。** 我根据自己的编译体验，觉得cmake下编译比传统构建系统效率更高、报错概率更低。不过目前toolchain尚未实现针对自定义的配置设计合适的cmake指令，因此只能自己根据CMakeLists.txt里面的选项逐个添加与既有toolchain配置相对应的到命令行中，比较麻烦；另外，目前无法通过cmake同时编译ssmp和psmp（检测出MPI就只编译psmp，否则只编译ssmp），且编译成的程序没有相应的符号链接sopt和popt，不过这不算什么大问题，毕竟psmp同时支持MPI和OpenMP并行，只要设置OMP_NUM_THREADS为物理核心数且不用mpirun指令就相当于运行ssmp，只要<code style="font-size: 14px;">export OMP_NUM_THREADS=1</code> 并用<code style="font-size: 14px;">mpirun -np N</code> （N为并行核数）运行就相当于运行popt了。

**<font color=red>补充：最推荐的从cmake正确编译CP2K可执行文件的步骤（至少适用于2025.2及后续版本，以下以2026.1为例；假设使用root用户）：</font>**

*<font color=red>First Written: 2025-12-25; Last Updated: 2026-01-23</font>*

<ol>
<li> 完成前述toolchain配置后，按照控制台输出所说明的执行<code style="font-size: 14px;">source /root/CP2K/src/cp2k-2026.1/tools/toolchain/install/setup</code>。</li>

<li> 切到cp2k源码目录，执行<code style="font-size: 14px;">mkdir build && cd build</code>，进入构建和编译专用目录。</li>

<li> 运行构建指令。由于前面说过的原因，这里需要手动敲入构建选项，比如我的toolchain选项为：

```bash
./install_cp2k_toolchain.sh --with-sirius=no --with-openblas=system --with-fftw=system --with-scalapack=system --with-hdf5=system --with-ninja=system --with-cmake=system --with-tblite -j 24
```

这里包括了自己在系统单另已经安装好的cmake、ninja、MPI、OpenBLAS、ScaLAPACK、FFTW3和HDF5，toolchain默认安装的libint、libXC、libXSMM、Spglib、COSMA、ELPA、libvori，以及我选定安装的tblite。那么我的cmake预配置选项即如下所示，可见相当麻烦：

```bash
cmake -S .. -DCMAKE_INSTALL_PREFIX=../install -DCP2K_USE_TBLITE=ON -DCP2K_USE_FFTW3=ON -DCP2K_USE_LIBINT2=ON -DCP2K_USE_LIBXC=ON -DCP2K_USE_MPI=ON -DCP2K_USE_SPGLIB=ON -DCP2K_USE_VORI=ON -DCP2K_USE_COSMA=ON -DCP2K_USE_ELPA=ON -DCP2K_USE_LIBXSMM=ON -DCP2K_USE_HDF5=ON -DCP2K_DATA_DIR=/root/CP2K/src/cp2k-2026.1/data
```

其中<code style="font-size: 14px;">-DCMAKE_INSTALL_PREFIX</code> 设置到自己想安装到的路径（可以使用相对路径；为省事我直接设置在了父目录下一个新的文件夹；如果不设置，默认将为/usr/local）。由于OpenBLAS是强制性的、Scalapack在有MPI的情况下是强制性的，因此无论如何它们都会被检查，所以这里无需写出。之所以特别设置<code style="font-size: 14px;">-DCP2K_DATA_DIR=/root/CP2K/src/cp2k-2026.1/data</code>，是因为cmake构建系统安装好后默认读取基组的位置是<code style="font-size: 14px;">${CMAKE_INSTALL_PREFIX}/shared/cp2k/data</code>，会导致编译时生成与<code style="font-size: 14px;">/root/CP2K/src/cp2k-2026.1/data</code>内容完全重复的<code style="font-size: 14px;">/root/CP2K/src/cp2k-2026.1/install/shared/cp2k/data</code>目录，加上这一设置可以避免这一问题（但注意这里不能使用相对路径）。

**<font color=blue>2026-01-23补充：为解决需要自己手动添加cmake选项的麻烦，我在源代码包的toolchain模块中加入了能够让toolchain自动根据自己的安装选项生成相应cmake指令的脚本；相关变动我已在CP2K的GitHub仓库上提交Pull Request，若成功合并，未来的2026.2版本中会包含这一功能。这里我将集成了该脚本的CP2K 2026.1源代码包放到这里供大家取用，用了这个你就只需要复制toolchain安装成功后屏幕上提示的cmake指令就可以了。</font>**

<p style="margin-left: 20px; margin-right: 20px; font-size: 17px;">
<strong><a href="../../_static/packages/cp2k-2026.1.tar.xz">📦cp2k-2026.1.tar.xz</a></strong>
（由于CP2K为遵循GPL v2许可证的开源软件，该源码包允许传播）
</p>

<li> 构建完成后，运行<code style="font-size: 14px;">make install -jN</code>，N是并行核数。</li>

<li> 写入以下三行至~/.bashrc中以添加环境变量：

```bash
export PATH=$PATH:/root/CP2K/src/cp2k-2026.1/install/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/root/CP2K/src/cp2k-2026.1/install/lib64
source /root/CP2K/src/cp2k-2026.1/tools/toolchain/install/setup
```
</li>

<li> 删除该build文件夹（或者不删除但执行<code style="font-size: 14px;">make clean</code>）以腾出部分空间。</li>
</ol>
