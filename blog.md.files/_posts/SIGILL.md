---
title: Debugging `Program terminated with signal 4, Illegal instruction`.
categories: program ##文章分類目錄 可以省略
tags: ##文章标签 可以省略
    - debug
    - 采坑
description: How I got out of this pitfall.
---
## 现象
程序莫名crash，报的错误是`Program terminated with signal 4, Illegal instruction.`。通过gdb看core的位置在一些绝不可能的位置（例如c++标准库的位置）。大部分情况都是同一个地方crash。偶尔会换一个位置crash。网上说这种情况多半变异有问题出现了`ud2a`指令，我通过`objdump -S`没有查看到`ud2a`指令。
## 原因
我这边发现有两个地方core了，通过gdb代开core文件，使用`x/i $pc`可以看到最后执行的汇编命令。我看到如下两个指令：
```
0x7f8d9b0c11c9 <std::mersenne_twister_engine<unsigned long, 32ul, 624ul, 397ul, 31ul, 2567483615ul, 11ul, 4294967295ul, 7ul, 2636928640ul, 15ul, 4022730752ul, 18ul, 1812433253ul>::operator()()+409>:	vpand  (%r8,%rax,1),%ymm3,%ymm1
```
```
0x7f66a2df710f <ray::ps::PSXxxxxXxxx::XxxXxxxx(ray::ps::pstat*)+2719>:	vpaddq -0x390(%rbp),%ymm0,%ymm0
```
看到两个指令`vpand`和`vpaddq`有共同的前缀，我有点警觉。查了[一些网站](http://osask.cn/front/ask/view/1696497)，明确这两个是AVX2的指令集指令。

然后，我通过`cat /proc/cpuinfo | grep flags`查询编译机器和运行机器的指令集：

flags: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave <font color=#FF0000 >avx</font> f16c rdrand lahf_lm abm epb invpcid_single tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 <font color=#FF0000 >avx2</font> smep bmi2 erms invpcid cqm xsaveopt cqm_llc cqm_occup_llc dtherm arat pln pts

flags: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic popcnt tsc_deadline_timer aes xsave <font color=#FF0000 >avx</font> f16c rdrand lahf_lm epb tpr_shadow vnmi flexpriority ept vpid fsgsbase smep erms xsaveopt dtherm arat pln pts

也就是编译机器包含了AVX2的指令集，但是运行机器不包含，因此出现了这个问题。以后出现类似问题都可以用类似的办法先进行排查。