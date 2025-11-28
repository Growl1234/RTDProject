计算化学应用与软件指南
======================


这里收录了一些关于如何应用计算化学和使用计算化学程序的资料，其中包含大量外部链接。

注意，关于VASP和CP2K的程序配置相关的说明都是完全基于Linux的（不同的外部链接基于的分发可能不同，但完全可以举一反三），一些相关的基本指令如果不会请自己在网上搜索和练习，这里不是Linux新手教程。

适用于Gaussian
~~~~~~~~~~~~~~

`🔗 Gaussian 16 Users Reference <https://gaussian.com/man/>`_

`📄 Exploring Chemistry with Electronic Structure Methods (2nd edition) <../_static/pdf/gaussian/ExpChem_2e.pdf>`_

`📄 Gaussian 量子化学计算手册（ExpChem第二版中文节译本） <../_static/pdf/gaussian/ExpChem_2e_Chinese_partly.pdf>`_

`向Slurm集群提交Gaussian任务的两个注意事项 <./Gaussian/slurm.html>`_

.. toctree::
   :maxdepth: 5
   :hidden:

   Gaussian/slurm

适用于ORCA
~~~~~~~~~~

`🔗 ORCA在线手册 <https://www.faccts.de/docs/orca/6.1/manual/index.html>`_

适用于VASP
~~~~~~~~~~

`🔗 Learn VASP the Hard Way <https://www.bigbrosci.com/>`_

`🔗 VASP Wiki <https://www.vasp.at/wiki/>`_

`关于配置VASP和一些重要工具（vaspkit、VTST）的简要说明 <./VASP/installation_with_extra_tools.html>`_

`VASP的收敛性测试及脚本文件 <./VASP/convtest.html>`_

.. toctree::
   :maxdepth: 5
   :hidden:

   VASP/installation_with_extra_tools
   VASP/convtest

适用于CP2K
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

关于CP2K的很多东西，下面推荐的思想家公社和计算化学公社中已经有不少博文和帖子讨论、说明得比较详细了，另外还有卢天老师（Sobereva）的 `北京科音CP2K第一性原理计算培训班 <http://www.keinsci.com/KFP>`_ 供你进一步系统学习CP2K（不是打广告，但该课程的确性价比高，感兴趣的话可以考虑考虑），因此未来大概率不会对这一模块做什么多余的扩充。

`从源代码配置CP2K <./CP2K/installation.html>`_

.. toctree::
   :maxdepth: 5
   :hidden:

   CP2K/installation

推荐博客及论坛
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**思想家公社和计算化学公社是国内最顶级、最专业的计算化学博客网站和论坛网站，没有之一！** 无论你是新手还是老手，都可以在里面找到非常有用、有价值的信息，以及参与该专业领域的讨论。网站涵盖范围也很广，不仅包括大量聚焦于应用型计算和比较典型的任务类型的内容（包括针对周期性体系的“第一性原理计算”），还有很多波函数分析（或称“电子结构分析”）方面非常丰富的介绍；博主兼社长卢天老师还开发了不少出色且很有实际用途的程序（尤其功能强大的波函数分析程序Multiwfn，现在已经非常流行）。不过博客和论坛也难以做到那么面面俱到，如果你的研究领域特别高深（比如量子动力学），那么这些论坛对你的实际帮助可能就不是特别明显了。

`🔗 思想家公社（sobereva） <http://sobereva.com/>`_

`🔗 计算化学公社 <http://bbs.keinsci.com/forum.php>`_

附：`📦 思想家公社中计算新手必看博文（涉及计算化学中最基础、最入门的内容） <../_static/pdf/Sobereva.zip>`_
