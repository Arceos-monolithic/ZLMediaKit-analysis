# ZLM 运行指引（三月）

## 目前情况

如何启动 zlm：看 [Starry 的 x86_ZLMediaKit 分支](https://github.com/Arceos-monolithic/Starry/tree/x86_ZLMediaKit) 的 README。

目前进展：[第一阶段](https://github.com/Arceos-monolithic/ZLMediaKit-analysis/blob/main/%E7%AC%AC%E4%B8%80%E9%98%B6%E6%AE%B5syscall%E6%94%AF%E6%8C%81.md)剩下三个任务：

1. 任务五(`clone3&vfork`)实现不完整，有bug

2. 任务九(`/tec/localtime`)还未合并

3. 任务十（`/sys/devices/system/cpu/online`）还没有实现

## 应该做什么？

首先，fork 一个 Starry 仓库然后完成剩余的任务。各位同学间可以相互借鉴合并，但先不要往 Starry 主仓库推 MR，因为最近大家都不太有空 review。

然后，在 x86_ZLMediakit 分支启动 zlm，得到和下面列出的相同的调试输出

接着可以开始完成实现：

1. 尝试修复任务五的 syscall（这个比较难，也可以等之后老师有空再来修）

2. 仿照[之前的文档](https://github.com/Arceos-monolithic/Starry/blob/x86_ZLMediaKit/doc/ZLMediaKit/1_6.md)和[分支](https://github.com/Arceos-monolithic/Starry/tree/zlm_1_6/doc/ZLMediaKit)实现第一阶段的任务十和第二阶段的所有任务。注意，**文档中有些目录和文件名在当前 Starry 中已经改变了**。也可以先仿照任务九的分支重新实现一遍，对比已有实现作为练手。

#### 怎么知道写得对不对

如果没有目前两个阶段文档的分析，我们只能遇到一个问题调一个问题，整个支持过程是串行的，效率比较低。现在有了分析文档之后，可以并行的支持需要的 syscall 了，但最终测试还是得串行地做。我们推荐**每个任务写一个单独的小测例**，先保证功能基本没问题，等前面的任务都完成时再在 ZLM 上真正测试。

到第一阶段剩余的任务完成时，执行 `./MediaServer -h` 命令可以获取正常的帮助输出。之后就可以在 ZLM 这个大应用上依次测第二阶段的测例了。

## 调试输出

1. 根据 README 生成 ZLM 对应的文件系统镜像

2. 把 `ulib/axstarry/src/syscall.rs` 里的所有 `info` 全局替换为 `error`，然后执行 `make run LOG=error`

3. 在 Starry 的终端中输入 `./MediaServer -h`
   
   > 每次输入字符会冒出 syscall 的输出，这是正常现象。如果害怕敲错，可以输入 `./M` 然后按 TAB 自动补全，然后再敲空格和 `-h`，最后再回车执行

4. 经过约半分钟后，得到最终输出如[Log文件](./log_stage_1_5.txt)所示。

#### 一些说明

输出的最后几行是

```
[ 17.978705 0:8 axmem:403] Page fault address VA:0x1ca383d not found in memory set 
[ 17.979084 0:8 axruntime::lang_items:5] panicked at modules/axmem/src/lib.rs:406:17:
FIXME: Page fault shouldn't cause a panic in kernel.
```

这里的 panic 是主动加的，如果不加的话内核会无限循环触发同一个 Page Fault 报错。修复完 bug 之后应该把这个 panic 删掉
