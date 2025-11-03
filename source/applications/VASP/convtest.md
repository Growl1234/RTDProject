## VASP的收敛性测试及脚本文件

***Last Updated: 2025-11-04***

VASP中的平面波截断能（ENCUT）和k点数目是实际计算的两个非常重要的设置，这两个参数设得太小会让结果太不精确，设得太大对精度进一步改进不明显却耗时暴增，因此不能盲目设大。

[Vaspkit程序](https://vaspkit.com/)可以在生成输入文件时根据你期望的精度设置自动生成一个k点（多数情况偏大；如果要使用vaspkit生成k点，对于一般问题个人建议步骤为“102-->1-->0.04”）。ENCUT可以根据POTCAR文件来设置，原则上不得低于所有元素中ENMAX的最大值，经验上通常根据实际问题类型取ENMAX最大值的1~1.3倍。

严格来讲，ENCUT和k点都应大到随它们增大而能量基本收敛为止。为了选出一个合适的设置，最严谨的办法是对ENCUT和k点分别做收敛性测试。对于主要关注能量相关问题的一般任务，ENCUT或k点测试到能量收敛到1meV即可（对≥10原子体系，建议以1meV/atom为判断标准）；注意DOS计算需要较多k点，因此需要将收敛限卡得更严。

### 平面波截断能（ENCUT）的收敛性测试

下面给出一个做ENCUT随能量的收敛性测试的bash脚本，该脚本根据Sobereva为CP2K做的截断能随能量收敛性测试的脚本修改而来（原脚本出处是Sobereva举办的[北京科音CP2K第一性原理计算培训](http://www.keinsci.com/KFP)的“能量的计算及相关问题”部分），简单易懂。大家可以模仿这个脚本试着写一写做k点随能量的收敛性测试的bash脚本，针对CP2K做截断能（cutoff和rel_cutoff）和k点的收敛性测试的脚本也可以模仿着自己写出来（别问我要Sobereva的原始脚本，可能涉及版权问题）。

这个脚本中，“encut=”后面是你要测试的一串截断能数值，脚本中展示了两种定义方法，实际使用时任选一种即可。在使用这个脚本之前，先准备好一套单点能计算的输入文件，并在INCAR中设置ENCUT = TO_BE_TESTED，然后将该脚本放到与你准备的输入文件相同的目录中。该脚本的基本思路就是先创建对应你在脚本中设定的一串ENCUT下的多组文件夹（连带文件夹内的输入文件；视为多个计算任务），再分别执行各个计算任务，最后择取出能量并进行统计（下面的脚本取的是VASP自动做sigma->0外推后的能量）。通常情况下，只要有一行数据第一次出现与上一次的差值绝对值在1meV以下，就可以拿该行对应的ENCUT当实际计算要用的ENCUT值了。

显然，是否考虑k点、SCF是否收敛等影响能量及其精度的因素多多少少会对结果造成影响。由于VASP的算法原因，在做ENCUT收敛性测试时绝不能为节省时间而有意减小SCF步数上限甚至减到1（而CP2K可以），不过可以适当放宽SCF收敛限；至于是否可以为节约时间而选较粗的k点或者仅考虑gamma点，我还没有测试过，等测试了会更新这里的说明。

```
#Perform ENCUT convergence test
#Written by Growl1234, based on Sobereva's bash script for convergence test of CUTOFF for CP2K.
#!/bin/bash

template_file=INCAR   #Template file of present system
vasp_bin=/root/VASP/src/vasp.6.5.1/bin/vasp_std   #CP2K command
nproc_to_use=24   #Total number of CPU cores to use
recalc=0
#encuts= "400 425 450 475 500 525 550"
encuts=$(seq 400 25 550) #Considered encut range and step

if [ $recalc -eq 1 ] ; then
    echo "Running: rm -r encut_*"
    rm -r encut_*
fi
input_file=INCAR
output_file=vasp.out
output_file_2=OUTCAR
plot_file=ENCUT.txt

#Prepare input files
for ii in $encuts ; do
    work_dir=ENCUT_${ii}eV
    if [ ! -d $work_dir ] ; then
        mkdir $work_dir
    fi
    sed -e "s/TO_BE_TESTED/${ii}/g" $template_file > $work_dir/$input_file
    cp KPOINTS $work_dir/
    cp POSCAR $work_dir/
    cp POTCAR $work_dir/
done


#Run input files
for ii in $encuts ; do
   work_dir=ENCUT_${ii}eV
   cd $work_dir
   if [ ! -e $output_file_2 ] ; then
       echo "Running $work_dir/$input_file"
       mpirun -np $nproc_to_use $vasp_bin > $output_file
   else
       echo "$work_dir/$output_file_2 has existed, skip calculation"
   fi
   cd ..
done

#Print energies
echo "# ENCUT vs total energy" > $plot_file
echo "# Date: $(date)" >> $plot_file
echo "# PWD: $PWD" >> $plot_file
echo -n "#   ENCUT |  Energy (eV)  |   delte E  " >> $plot_file
printf "\n" >> $plot_file
itime=0
for ii in $encuts ; do
    work_dir=ENCUT_${ii}eV
    total_energy=$(grep -e 'sigma->0' $work_dir/$output_file_2 |tail -1 | awk '{print $7}')
    if (( $itime == 0 )); then
      printf "%8.0f  %15.8f               " $ii $total_energy >> $plot_file
    else
      E_var=$(echo "$total_energy - $E_last" | bc)
      printf "%8.0f  %15.8f  %12.8f" $ii $total_energy $E_var >> $plot_file
    fi
    printf "\n" >> $plot_file
    E_last=$total_energy
    itime=$(($itime+1))
done

echo "If finished normally, now check $plot_file"
```

### k点的收敛性测试（待完善）

做k点随能量的收敛性测试的bash脚本我还没有弄好（弄好了会贴到这里），不过基本思路跟上面一样，模仿着写即可。

需要强调的是，k点的收敛性测试应当让每一个分别任务的SCF都收敛。ENCUT的大小可能对结果造成影响，如果先做k点收敛性测试的话，先自己按经验做法选一个合适的ENCUT值。