## trace

<!-- vim-markdown-toc GitLab -->

## 基本理念

在 https://github.com/brendangregg/perf-tools 可以看到 bash 脚本 execsnoop，其中就是使用 ftrace 在 /sys 下提供的接口来使用的。
而 https://github.com/iovisor/bcc/ 中，有如下的两个文件:
- libbpf-tools/execsnoop.c : 基于 libbpf [^13]，overhead 更加小
- tools/execsnoop.py : 编译 BCC 程序来实现的

- https://www.scylladb.com/2021/09/28/hunting-a-numa-performance-bug/ : 一个具体的案例

## 问题
- [ ] 所以 lttng 相对于 bpf 有什么优势吗?
- [ ] perf 工具比我想想的要强大，应该好好的重新分析一下
  - https://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html
- [ ] bpf 相对于 perf 有什么优势吗?
- [ ] QEMU 的编译选项中:

## [valgrind](http://valgrind.org/)
使用上很简单:
https://stackoverflow.com/questions/5134891/how-do-i-use-valgrind-to-find-memory-leaks

## [uftrace](https://github.com/namhyung/uftrace)
仅仅支持 c / c++，可以非常精准将函数的调用图生成出来。

## [lttng](https://lttng.org/docs/)
其分析居然还要使用一个 : https://babeltrace.org/

## usdt
```sh
sudo apt install systemtap-sdt-dev
```

## [ ] QEMU 中的 trace
- [ ] 测试一下, 添加新的 Trace 的方法
- [ ]  为什么会支持各种 backend, 甚至有的 dtrace 的内容?
- [ ] 这里显示的 trace 都是做什么的 ?
```plain
➜  vn git:(master) ✗ qemu-system-x86_64 -trace help
```
- [ ] 例如这些文件:
/home/maritns3/core/qemu/hw/i386/trace-events

### log
util/log.c 定义所有的 log, 其实整个机制是很容易的

- `asm_in` : `accel/tcg/translator.c::translator_loop`
- `asm_out` : `tcg/translate-all.c::tb_gen_code`

## 小众的 profile 工具
- https://oprofile.sourceforge.io/about/

## 更加好用的前端分析工具

### [ ] [hotspot](https://github.com/KDAB/hotspot)
这个工具是不是需要特殊的配置啊，搞出来的火焰图明显不对

### [pcm](https://github.com/opcm/pcm)
```sh
git clone https://github.com/opcm/pcm
cd pcm
mkdir build
cd build
cmake ..
cmake --build . --parallel

./build/pcm
```

![](https://raw.githubusercontent.com/wiki/opcm/pcm/pcm.x.jpg)

- [ ] https://github.com/cyring/CoreFreq
- [ ] https://github.com/intel/msr-tools
- [ ] https://github.com/tycho/cpuid

### [flamescope](https://github.com/Netflix/flamescope)
可以用于查看一个范围空间中的 flamegraph

### [ ] kernelshark

### [ ] [pprof](https://github.com/google/pprof)

和这个东西是什么关系?
- https://github.com/gperftools/gperftools
- [ ] 如果 pprof 似乎是可以 C 语言工作的，但是 gperf 据说已经很好用了

- [ ] https://github.com/jrfonseca/gprof2dot
  - 这个工具是被我们使用上了，但是本身是一个将各种 perf 结果生成新的结果的工具，可以看看原来的结果的位置

### [ ] [tracecompass](https://www.eclipse.org/tracecompass/index.html)


## 其他
### ltrace
library call trace

### [ ] [sysdig](https://github.com/draios/sysdig)

使用 csysdig 可以得到一些类似 htop 的界面
```sh
csysdig -l
```

- sysdig 可以监控整个系统中发生的所有正在发生那些
  - 系统调用 : 我们发现很多都是 futex
  - 所有的 IO : pipe unix netlink
    - 并且检查所有在使用 pipe 的进程
      - 而且可以将这些进程的 backtrace 打印出来

### [prometheus](https://prometheus.io)
集群的管理

也许应该新增加一个栏目，叫做 monitor 的
- https://github.com/winsiderss/systeminformer

### [ ] [memray](https://github.com/bloomberg/memray)

### [ ] https://github.com/google/schedviz

### [ ] https://github.com/javamelody/javamelody

### [ ] https://github.com/intel/msr-tools
### [ ] https://github.com/corpaul/flamegraphdiff
### [ ] https://github.com/andikleen/pmu-tools
### [ ] https://github.com/aquasecurity/tracee
### [ ] https://github.com/cloudflare/ebpf_exporter
### [ ] https://github.com/bytedance/trace-irqoff

### [ ] iperf
https://load-balancer.inlab.net/manual/performance/measuring-internal-bandwidth-with-iperf/

## [ ] https://github.com/sysstat/sysstat

## [ ] https://man7.org/linux/man-pages/man1/ipcs.1.html
分析 ipc 的性能的

## [ ] irqtop(1)

# 传统的统计工具
- lsof

## dperf
dpdk 测试工具

- https://github.com/baidu/dperf

## 参考
- https://github.com/adriannovegil/awesome-observability

## rusage
- time 的源码: https://savannah.gnu.org/git/?group=time
  - [ ] nixos 上不知道为什么，我不能正确的编译
- 原来是有个系统调用的: https://man7.org/linux/man-pages/man2/getrusage.2.html
- fs/proc/stat.c 中的信息才是 top 如何统计的
- 通过 `vtime_guest_enter` 去理解为什么 qemu 在运行起来的时候，发现 user 是占据大多数的，因为统计将 non-root 中的运行统计到 user 中了。

## [ ] cflow
- https://graphviz.org/
- https://graphviz.org/pdf/gvpr.1.pdf
- https://www.gnu.org/software/cflow/manual/cflow.html : 可以绘制整个图形的

## sudo cat /proc/self/stack
检查一个进程在内核中的 stack

## rtla
- https://lwn.net/Articles/869563/
- https://bristot.me/and-now-linux-has-a-real-time-linux-analysis-rtla-tool/


## lshw
- lshw -c disk : 可以查看一个 disk 所在的 CPU

## lscpu
- 检查 numa CPU

## numactl
- numactl -H

## fgprof
- https://github.com/felixge/fgprof

其实一直没有搞懂，为什么会存在语言相关的 profiler 的

## depmod -a / -A

## iostat

## pidstat

## mpstat

- https://wiki.ubuntu.com/Kernel/Reference/stress-ng
- https://gitee.com/openeuler/release-management/pulls/417/

## iftop

## tiptop

## dtrace 真的还有人用吗？

## https://github.com/benfred/py-spy
- [ ] 为什么每一个语言都是需要存在一个对应的观测工具呀？

## 针对于特定语言的
- python : py-spy

## do_user_addr_fault 中的这个 perf_sw_event
```c
	perf_sw_event(PERF_COUNT_SW_PAGE_FAULTS, 1, regs, address);
```

## 为什么这个函数无法 trace 啊 trace_kprobe
```txt
[  323.274009] trace_kprobe: Could not probe notrace function sg_miter_stop
[  323.274155] trace_kprobe: Could not probe notrace function sg_miter_stop
[  361.337420] trace_kprobe: Could not probe notrace function sg_miter_stop
[  361.337512] trace_kprobe: Could not probe notrace function sg_miter_stop
[  379.112269] trace_kprobe: Could not probe notrace function sg_miter_skip
[  379.112361] trace_kprobe: Could not probe notrace function sg_miter_skip
```
[^4]: [An introduction to KProbes](https://lwn.net/Articles/132196/)
[^5]: [Using user-space tracepoints with BPF](https://lwn.net/Articles/753601/)
[^7]: [kernelshark](https://www.cnblogs.com/arnoldlu/p/9014365.html)
[^8]: [Linux Performance](http://www.brendangregg.com/linuxperf.html)
[^9]: [Linux tracing systems & how they fit together](https://jvns.ca/blog/2017/07/05/linux-tracing-systems/)
[^11]: [perf tutorial](https://perf.wiki.kernel.org/index.php/Tutorial)
[^12]: https://github.com/NanXiao/perf-little-book
[^13]: https://pingcap.com/blog/why-we-switched-from-bcc-to-libbpf-for-linux-bpf-performance-analysis
