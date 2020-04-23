
---
title: NetCore应用配置Auto Core Dump
---

## 问题背景
前阵子线上用户标签服务频繁出现内存泄漏问题，早上服务运行的好好的，经常一到半夜服务就挂掉。由于事发半夜，很难加以人工干预，而早上dump出来的文件参考价值很低，迫切需要一种自动化的手段让服务在宕掉的时候能够保存完整的案发现场。事实上，.NET CORE可以支持这一需求，不过默认是不开启的，需要加以配置。

## Auto Core Dump配置方法

以下内容是我原封不动从dotnet/rumtime这个仓库拷贝过来的：

Environment variables supported:

- `COMPlus_DbgEnableMiniDump`: if set to "1", enables this core dump generation. The default is NOT to generate a dump.
- `COMPlus_DbgMiniDumpType`: See below. Default is "2" MiniDumpWithPrivateReadWriteMemory.
- `COMPlus_DbgMiniDumpName`: if set, use as the template to create the dump path and file name. The pid can be placed in the name with %d. The default is _/tmp/coredump.%d_.
- `COMPlus_CreateDumpDiagnostics`: if set to "1", enables the _createdump_ utilities diagnostic messages (TRACE macro).

COMPlus_DbgMiniDumpType values:


|Value|Minidump Enum|Description|
|-|:----------:|----------|
|1| MiniDumpNormal                               | Include just the information necessary to capture stack traces for all existing threads in a process. Limited GC heap memory and information. |
|2| MiniDumpWithPrivateReadWriteMemory (default) | Includes the GC heaps and information necessary to capture stack traces for all existing threads in a process. |
|3| MiniDumpFilterTriage                         | Include just the information necessary to capture stack traces for all existing threads in a process. Limited GC heap memory and information. |
|4| MiniDumpWithFullMemory                       | Include all accessible memory in the process. The raw memory data is included at the end, so that the initial structures can be mapped directly without the raw memory information. This option can result in a very large file. |


根据以上内容，可以在Dockerfile或者docker-compose文件加入以下环境变量

```
ENV COMPlus_DbgEnableMiniDump="1"  # enables core dump generation
ENV COMPlus_DbgMiniDumpName="/diagnostics/dumps/coredump_%d" # core dump文件位置，一般是一个挂在在宿主机的目录
ENV COMPlus_DbgMiniDumpType="4" # 建议选择4，也就是full dump，保存的“现场”更加完整，但文件也会非常大
```


## 实验

在一个可以被调用的接口加入以下代码：

```
Environment.FailFast()
```
这个方法可以让应用直接挂掉,挂掉后观察相应的挂载目录有无生成dump文件



## 参考

[xplat-minidump-generation](https://github.com/dotnet/runtime/blob/8497763bbfa70455e6f08ed7aa345d43db1d22d7/docs/design/coreclr/botr/xplat-minidump-generation.md)