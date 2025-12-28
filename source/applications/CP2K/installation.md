## 从源代码配置CP2K

***Last Updated: 2025-12-28***

**看思想家公社（sobereva）的文章[《CP2K第一性原理程序在Linux中的安装方法》](http://sobereva.com/586)即可，toolchain一步可以根据自己的实际需求作修改。**

下面给一点补充说明：

* toolchain会自动检查系统是否存在MKL配置，如果没有检测到MKL（包括新的oneMKL，下同），默认会把OpenBLAS、ScaLAPACK和FFTW一起安装下来，此时如果你已经事先有安装它们（包括其中任意一个）且环境变量配置正确，最好写上例如 <code style="font-size: 14px;">\--with-openblas=system</code> 这样的选项。**注意：无论OpenBLAS是否需要作为数学库安装，也无论其是否被标为“system”，其源码都会被下载并解压用以执行另外一个必要的检查步骤，而关于OpenBLAS的一切直接或间接设定只能影响其会不会被编译和安装。**

* **不要使用Intel oneAPI做并行化编译器，因为CP2K（截至v2025.2）还没有做好对ifx的支持（而新的oneAPI已经不再支持较旧的ifort），** 虽然toolchain一步可能会成功但后续编译步骤会出现内存爆浆问题而中断（亲身实践教训）。与oneAPI的并行编译器不同，**新的Intel oneMKL是受支持的**；建议安装好oneMKL后进入fftw3xf目录（例如/opt/intel/oneapi/mkl/2025.1/share/mkl/interfaces/fftw3xf）手动编译产生fftw3库文件（在该目录运行 <code style="font-size: 14px;">make libintel64</code>；根据Makefile的设定，编译过程使用icx和gcc都可以，然而实际启动编译时如果没有检测到icx就跳过gcc检查直接报错，所以如果想用gcc必须显式指定“CC=gcc”）。

* 根据个人测试经验，OpenMPI并行结合oneMKL数学库选项配置的CP2K在运行时会出现内存配置错误，而MPICH不会，原因不明。鉴于这一情况，如果坚持使用Intel oneMKL作为数学库，那我建议你用MPICH作为并行化工具来配置和运行CP2K；而在使用开源的OpenBLAS、ScaLAPACK和FFTW组合当数学库时，则放心用社区流行度更高的OpenMPI。

* <code style="font-size: 14px;">\--with-tblite</code> 代表安装Grimme的tblite程序，要加这个必须同时加上 <code style="font-size: 14px;">\--with-ninja</code>。按照CP2K的说明，tblite同时包含DFT-D4，因此这时也就不用刻意加 <code style="font-size: 14px;">\--with-dftd4</code> 了（即使加上了也会被自动跳过；但是 <code style="font-size: 14px;">\--with-ninja</code> 还得有）。注意从CP2K的toolchain下载的tblite可能不完整，缺胳膊少腿的，碰上这种情况应当自行去tblite官网下载tar.xz源码包并重新包装成tar.gz来替代原来的包。

* 如果你使用自行事先编译的HDF5并加了 <code style="font-size: 14px;">\--with-hdf5=system</code> 选项，toolchain脚本默认会自动把“-lsz”加到arch设置里，此时应当在正式编译CP2K前先运行 <code style="font-size: 14px;">sudo dnf install libaec-devel</code> 命令装上libsz，否则会出现“找不到-lsz”的错误提示。

* **从版本2026.1开始，CP2K的编译将全面转为cmake，彻底放弃GNU Makefile和相应的arch文件集。** 我自己根据目前 *（2025-12-28 21:30）* 的CP2K开发版安装包尝试从cmake编译，有一个小问题，即toolchain尚未实现针对自定义的配置设计合适的cmake指令，因此只能自己根据CMakeLists.txt里面的选项逐个添加与既有toolchain配置相对应的到命令行中，比较麻烦。另外，凭说明信息建议的这一编译选项目前无法同时编译ssmp和psmp（检测出MPI就只编译psmp，否则只编译ssmp），且编译成的程序没有相应的软链接sopt和popt；不过这不算什么大问题，毕竟psmp同时支持MPI和OpenMP并行，只要恰当设置OMP_NUM_THREADS环境变量且不用mpirun指令就相当于运行ssmp，只要<code style="font-size: 14px;">export OMP_NUM_THREADS=1</code> 并用<code style="font-size: 14px;">mpirun -np N</code> （N为并行核数）运行就相当于运行popt了。

***补充：最推荐的从cmake正确编译CP2K可执行文件的步骤：***

1. 完成前述toolchain配置。

2. 切到cp2k源码目录，运行构建指令；由于前面说了的麻烦之处，这里需要手动敲入构建选项。比如我的toolchain选项为：

<code style="font-size: 14px;">--with-sirius=no --with-openblas=system --with-fftw=system --with-scalapack=system --with-hdf5=system --with-ninja=system --with-cmake=system --with-tblite</code> 

那么我的cmake预配置选项就是：

<code style="font-size: 14px;">cmake -S . -B build -DCMAKE_INSTALL_PREFIX=/root/CP2K/src/cp2k-dev/exe/ -DCP2K_USE_TBLITE=ON -DCP2K_USE_FFTW3=ON-DCP2K_USE_LIBINT2=ON -DCP2K_USE_LIBXC=ON -DCP2K_USE_MPI=ON -DCP2K_USE_SPGLIB=ON -DCP2K_USE_VORI=ON -DCP2K_USE_COSMA=ON -DCP2K_USE_ELPA=ON -DCP2K_USE_LIBXSMM=ON -DCP2K_USE_HDF5=ON</code> 

可见命令相当麻烦。这里<code style="font-size: 14px;">-DCMAKE_INSTALL_PREFIX</code> 之所以自己设置了一个，是因为不自己设置的话，后面执行安装的目标目录容易默认导到toolchain里面dbcsr的目录下；而且CP2K默认读取基组信息的data文件夹也会被认为是在相应目录下的share/cp2k/data，而不是原来源代码目录下的data文件夹（不过这个可以通过设置环境变量来解决，即<code style="font-size: 14px;">export CP2K_DATA_DIR={path}</code>）。另外，OpenBLAS是强制性的，Scalapack在有MPI的情况下是强制性的，因此无论如何都会检查，所以这里无需写出。

3. 构建完成后，运行<code style="font-size: 14px;">cmake --build build -j N</code>，N是并行核数。

4. 执行安装步骤：<code style="font-size: 14px;">cmake --install build</code>；这里无法并行运行，因此也没有加“-j N”。

5. 删除build文件夹以腾出一部分空间。注意不要随意删除源码包目录下别的东西。
