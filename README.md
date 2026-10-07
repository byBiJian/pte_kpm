# pte_kpm —— ARM64 PTE 断点 KPM

**版本 v1.0**（首个稳定运行版本，非最终版，持续迭代）

部分实现参照了wxshadow的实现，感谢贡献

基于 KernelPatch / KPatch-Next 框架的 KPM 内核模块：**通过修改目标页 PTE 的 UXN 位（bit 54）制造指令权限异常，在内核态 `do_mem_abort` 路径拦截**，实现用户态完全不可见的指令级断点；命中时可直接修改目标进程寄存器（X0~X8 / W0~W8）。

全程**无 ptrace、无影子页/幽灵页、无代码段修改、无注入、不占用硬件断点槽（BRP/WRP）**，目标进程自身用 ptrace 占满硬件断点也不影响本模块。

仅用于网络安全研究。测试无 bug 后开源。

---

## 一、原理

```
正常执行:  PC -> 指令页(PTE.UXN=0) -> 正常取指

布防后:    PC -> 指令页(PTE.UXN=1) --取指--> 指令权限异常(IABT, FSC=PERM)
                                              |
                                              v
     do_mem_abort hook(内核态):
       1. 判定: IABT_LOW + 权限错 + 断点页 + 目标 tgid
       2. 精确命中断点地址 -> 修改寄存器(可选)
       3. 清除 UXN(恢复该页可执行) + TLB 刷新
       4. 启用单步(TIF_SINGLESTEP)
       5. skip_origin=1(吞掉信号, 用户态无感)

     user_step_hook(单步回调):
       6. 下一条指令仍在断点页内 -> 保持单步(重置 SPSR.SS)
       7. 离开断点页 -> 停单步 + 重新置 UXN(重新布防)
```

关键点:
- UXN 是**页级**属性(4KB 一页), 锁定后该页内**任何**指令取指都会触发异常;
- 异常是**同步精确异常**: 目标指令执行前被拦截, 寄存器修改必然先于该指令生效;
- 命中判定用精确地址 `far == bp_addr`, 页内其他地址的执行只是"页面锁的重放", 不记日志、不改寄存器;
- 单步扩展点(`register_user_step_hook`)是内核标准调试通道, 吞掉 SIGTRAP, 用户态零感知。

## 二、目录结构

```
pte_kpm/
├── pte_kpm.c       主实现(约 1000 行)
├── pte_kpm.h       常量/状态结构/KPM 元信息
├── pte_client.c    用户态交互客户端
├── pte_client      编译产物(已构建, arm64 静态)
├── pte_kpm.kpm     编译产物(KPM 模块)
├── Makefile        内核模块构建
├── build.sh        一键编译脚本
├── README.md       本文档
├── refs/           参考实现(不参与构建)
│   ├── wxshadow.c / wxshadow_scan.c / wx.h   wxshadow影子页断点方案(单步防死锁参照)
│   └── kpn_start.c / k_hook.h                KPatch-Next 内核源码摘录(pgtable 等)
└── tests/          测试程序(本机/5.15 通用)
    ├── demo.c        被调用库函数(demo_add/demo_mul)
    ├── host.c        单线程宿主(循环调用 demo_add)
    ├── host_game.c   游戏节奏仿真(5 线程 x 50 次随机间隔)
    ├── host_magic.c  机制实验(返回值当状态码, 验证改寄存器影响调用方)
    ├── host_h.c      并发压力(2 线程屏障同步 / 4 线程满速)
    ├── test_kpm.c    早期隔离测试模块源码
    └── kpm-*.kpm     早期测试模块产物
```

## 三、编译

环境要求:
- NDK r29(`~/android-ndk-r29`, 含 `linux-aarch64` 与 `linux-x86_64` 两个 host 目录, 同一份 LLVM 21);
- KernelPatch 源码(`~/KernelPatch`, 提供 kernel 头文件)。

一键编译(termux 内):

```sh
cd /data/user/0/com.termux/files/home/pte_kpm
sh build.sh
```

手动编译:

```sh
export KP_DIR=/data/user/0/com.termux/files/home/KernelPatch
export NDK_BIN=/data/user/0/com.termux/files/home/android-ndk-r29/toolchains/llvm/prebuilt/linux-aarch64/bin
make
```

完整命令(等价展开):

```sh
CC=$NDK_BIN/clang-21
LD=$NDK_BIN/ld.lld
$CC --target=aarch64-linux-android24 -std=gnu11 -fno-builtin -nostdinc -O2 -Wall \
    -Wno-unused-function -fno-stack-protector -mno-outline-atomics -fno-pic -fno-PIE \
    -fno-asynchronous-unwind-tables -fno-unwind-tables \
    -I$KP_DIR/kernel/. -I$KP_DIR/kernel/include -I$KP_DIR/kernel/patch/include \
    -I$KP_DIR/kernel/linux/include -I$KP_DIR/kernel/linux/arch/arm64/include \
    -I$KP_DIR/kernel/linux/tools/arch/arm64/include \
    -c -o pte_kpm.o pte_kpm.c
$LD -r -o pte_kpm.kpm pte_kpm.o
```

**必须保留的编译选项(否则加载失败)**:

| 选项 | 作用 | 缺失后果 |
|---|---|---|
| `-mno-outline-atomics` | 内联 CAS 原子指令, 不调 libatomic | `__aarch64_cas4/8_acq_rel` 未定义符号 -> `unknown symbol` 加载失败 |
| `-fno-pic -fno-PIE` | 外部内核符号直接寻址, 不走 GOT | 107 处 GOT 重定位(加载器不支持) -> 加载失败 |
| `-fno-asynchronous-unwind-tables -fno-unwind-tables` | 不生成 `.eh_frame` | PREL32 重定位溢出 -> 加载失败 |

原因: KernelPatch 模块加载器只支持 37 种非 GOT 重定位类型, 且符号表无 libatomic; 官方 demo 用 gcc(默认非 PIC)不踩坑, NDK/clang 必须显式关掉。

客户端编译:

```sh
$CC --target=aarch64-linux-android24 --sysroot=$(dirname $NDK_BIN)/sysroot -static -O2 \
    -o pte_client pte_client.c
```

## 四、快速开始(推荐: 客户端模式)

客户端模式 = 模块零参数加载, 运行时由用户态 `pte_client` 动态下发断点, **推荐所有场景使用**(地址解析在用户态, 绕开内核态 maps 读取限制)。

```sh
# 1. 加载模块(零参数)
/data/adb/modules/KPatch-Next/bin/kpatch kpm load /data/local/tmp/pte_kpm.kpm

# 2. 布防: 给游戏进程 libtersafe.so + 0x21F8F0 下执行断点
/data/local/tmp/pte_client -p com.tencent.mf.uam -b libtersafe.so -o 0x21F8F0

# 3. 操作游戏(走到目标场景)...

# 4. 查询命中次数
/data/local/tmp/pte_client -s

# 5. 撤防(进程存活时)
/data/local/tmp/pte_client -p com.tencent.mf.uam -d
#    进程已退出时用强制撤防(跳过 tgid 校验):
/data/local/tmp/pte_client -p 0 -d

# 6. 卸载模块
/data/adb/modules/KPatch-Next/bin/kpatch kpm unload kpm-pte-breakpoint
```

命中时修改寄存器(如把 W8 改为 2):

```sh
/data/local/tmp/pte_client -p com.tencent.mf.uam -b libtersafe.so -o 0x21F8F0 -r W8=2
```

> 注意: 纯观察计数不要带 `-r`。修改寄存器会改变目标程序看到的入参/返回值,
> 目标逻辑可能因此走不同分支(实测: 同一函数调用次数 185 -> 103)。
> 这是目标程序行为被真实改变, 不是漏拦。

## 五、pte_client 参数手册

```
用法: pte_client [选项]

-p <包名|pid>   目标进程(包名走 cmdline 匹配; 撤防时可传 0 跳过校验)
-b <so名>       目标 so(如 libtersafe.so, pgoff==0 段解析基址)
-o <偏移>       函数偏移(十六进制 0x... 或十进制)
-r <寄存器=值>  命中时修改寄存器: X0~X8 / W0~W8(可省略 = 只观察)
-q              查询布防状态(返回 1=已布防)
-s              查询本轮累计命中次数(布防时清零)
-d              撤防(清除页面锁 + 停止单步)
-v              探活(检测模块是否加载, 返回 0x10000)
-h              帮助

示例:
  pte_client -v                                  # 模块探活
  pte_client -p com.game.app -b libtersafe.so -o 0x21F8F0          # 布防(观察)
  pte_client -p com.game.app -b libtersafe.so -o 0x21F8F0 -r W8=2 # 布防(改 W8)
  pte_client -s                                  # 查命中数
  pte_client -p 0 -d                             # 强制撤防
```

通信协议(内核侧, `prctl` 专用码, 仅 root 可用):

| 命令码 | 名称 | 参数 | 返回 |
|---|---|---|---|
| `0x50540001` | SET_BP | a2=tgid a3=addr a4=寄存器包 a5=值 | 0=成功 |
| `0x50540002` | DEL_BP | a2=tgid(0=跳过校验) | 0=成功 |
| `0x50540003` | PING | - | `0x10000` |
| `0x50540004` | QUERY | a2=tgid | 1=已布防 |
| `0x50540005` | STATS | - | 累计命中数 |

寄存器打包: `REG_PACK(idx, w) = idx | (w ? 0x100 : 0)`; 不修改 = `(u64)-1`。

## 六、参数模式(备用)

加载时直接带 6 参数, 模块自行解析并布防(无需客户端):

```sh
kpatch kpm load /data/local/tmp/pte_kpm.kpm "com.tencent.mf.uam libtersafe.so 0x21F8F0 I W8 2"
#                         包名             so            偏移      O/I 寄存器 值
```

- `O` = 不修改寄存器, `I` = 修改;
- 已知限制: 内核态读 `/proc/<pid>/maps` 在部分内核返回 `-EINVAL`(日志 `maps read err=-22`),
  解析失败会安全降级(模块加载成功但未布防, 不影响系统)。**故推荐客户端模式。**

## 七、KPatch-Next WebUI 使用

- WebUI 的"临时加载" = `kpatch kpm load <路径>`(零参数) -> 客户端模式, 用 `pte_client` 操作;
- 勾"保存"会复制到 `/data/adb/kp-next/kpm/` 开机自动加载 —— **本项目严禁勾保存**(只临时加载, 避免任何开机残留);
- service.sh 对加载失败的模块会自动删文件(`kpatch kpm load "$kpm" || rm -f "$kpm"`)。

## 八、实测记录

### 8.1 全链路功能(6.1.118 本机 + 5.15.180 目标设备)

```
布防 libdemo.so + 0x5d8, -r X0=0x1234
命中 -> X0 被改为 0x1234(demo_add 返回值 0x1234+2)
目标输出 add=3 -> add=4662, 连续命中, 目标进程无感, 系统零崩溃
撤防 OK -> 卸载 OK
```

### 8.2 游戏实战对测(5.15.180, 与 stackplz 硬件断点计数对比)

同一会话、同时布防、同一时刻终止(划掉游戏后台):

| 方案 | 地址 | 命中数 |
|---|---|---|
| stackplz(HWBP 硬件断点) | libtersafe.so+0x21F8F4 | **231** |
| 本模块(PTE 断点) | libtersafe.so+0x21F8F0 | **231** |

**231 = 231, 零漏拦。**(地址相差 4 字节: 同一执行路径的相邻指令, 每次调用连跑, 计数 1:1 可比;
错开地址是因为硬件断点与软件断点同址冲突, 用户实操时有意分开。)

### 8.3 游戏节奏仿真(本机 6.1.118, 5 线程 x 50 次随机间隔 约 250 次调用)

| 轮次 | 模式 | 实际调用 | 命中 | 漏拦 |
|---|---|---|---|---|
| A | 只观察 | 250 | **250** | 0 |
| B | `-r W8=2` | 250 | **250** | 0 |

游戏密度(约 15 次/秒、多线程交错)下捕获率 100%。

### 8.4 机制实验(改寄存器影响调用方逻辑的受控证明)

`host_magic` 把函数返回值当"任务完成"状态码(期望值 4662):

| 轮次 | 模式 | 结果 |
|---|---|---|
| C | 只观察 | `first_r=3 calls_made=150 magic=0`(跑满 150 次) |
| D | `-r X0=0x1234` | `first_r=4662 calls_made=1 magic=1`(**第 1 次即终止**) |

-> 修改寄存器 -> 调用方看到的值变了 -> 逻辑分支变化 -> 调用次数变化。150 -> 1 是机制铁证。

### 8.5 并发压力(最坏情况, 2/4 线程死循环锤同一页)

| 场景 | 修复前 | 修复后 |
|---|---|---|
| 2 线程 x 20000 次 | 5 | 143 |
| 4 线程 x 20000 次 | 17 | 126 |

(修复后剩余差异为"页级锁固有的单指令窗口", 普通调用密度下不可见; 8.2/8.3 已实测零漏拦。)

## 九、内核稳定性修复史(7 个根因, 全部已修)

| # | 现象 | 根因 | 修复 |
|---|---|---|---|
| 1 | 加载后卸载即全系统渐冻 | KP 的 `unload_module` 持 `rcu_read_lock`, 官方 `unregister_user_step_hook` 内部调 `synchronize_rcu()` -> RCU 自死锁 | 弃官方接口, 自持 `debug_hook_lock` + 手动 `list_del_rcu`(`safe_step_unlink()`) |
| 2 | 带参数加载瞬间 NULL 空指针 panic | KPatch-Next 的 `task_struct_offset.pid_offset/tgid_offset/tasks_offset` 恒为 -1(从不赋值), 读 `task-1` 即崩 | 全弃坏偏移, 改用内核原生 `__task_pid_nr_ns` / `find_task_by_vpid` |
| 3 | 布防后约 1 分钟内核 BUG(slub) | 找 so 失败的清理路径 `mmput` 后忘清 `g_bp.mm` -> 悬空指针二次释放 -> rss 计数 BUG | 失败路径统一清指针 |
| 4 | 首次武装断点即 paging fault | `mm->pgd` 已是内核线性映射 VA, 又加了一次物理转虚拟偏移 -> 非法地址传给 `pgtable_entry` | 删掉多余加法, 直接传 `mm->pgd` |
| 5 | 多线程场景命中数骤降(熄火) | "离页重锁"门控写成"必须 0 线程在单步", 多线程互等 -> 页面永久解锁 | 重锁改为只判 `active && !disarming` |
| 6 | 同上(加剧) | 页内重锁被 `armed` 标志门控, 多线程下标志滞后 | 解除 `armed` 门控 |
| 7 | 槽位溢出降级 | `MAX_STEP=4`, 游戏页内 5 个线程并发单步 | 槽位 4 -> 8 |

## 十、已知限制

1. **热页开销(页级锁固有)**: 断点页内每条指令执行都走"异常->单步->重锁"(含 TLB 刷)。
   若断点页是高频热代码, 目标会被拖慢数倍。建议: 布防后尽快操作并撤防。
2. **多线程单指令窗口**: 命中线程解锁放行的那一瞬间, 同页其他线程可能"搭车"执行一次。
   普通调用密度下实测为零漏拦(8.2/8.3); 极限高频(万次/秒级死循环锤同页)有可见窗口。
3. **`-r` 会产生观测偏差**: 修改寄存器改变目标行为, 命中数会与纯观察不同(见 8.4)。
   纯计数请勿带 `-r`。
4. **参数模式 maps 受限**: 内核对目标 maps 读取可能返回 `-EINVAL`, 用客户端模式规避。
5. **与目标自身 ptrace 共存(已实现)**: step_hook 只处理本模块登记的 tid;
   命中时若该线程已在外部单步, 标记 ext-borrow 归还其 tracer。其余 tid 一律放行。
6. **进程退出前务必撤防**: 目标退出后用 `-p 0 -d` 强制撤防(清除页面锁)。
   忘记撤防直接卸载模块也可以, 模块 exit 会兜底恢复 PTE。

## 十一、日志与诊断

- `dmesg`(tag: `pte_kpm`): init/exit/符号解析等关键事件;
- `/data/local/tmp/pte_kpm.log`: 命中与诊断日志:
  - **只在布防 / 撤防 / 卸载 / 手动 flush 四个时刻落盘**(异常上下文不做文件 IO);
  - 环形缓冲 256 条, 每条带全局序号 `[N]`(判断是否发生覆盖);
  - 运行中实时查看: `kpatch kpm ctl0 kpm-pte-breakpoint flush` 然后读文件;
  - 撤防时追加 **DIAG 诊断块**:

```
DIAG pg=... bp=... tgid=... faults=<页内取指异常总数> sins=<单步次数> hits=<精确命中>
fh00..fh56: 16 行直方图 —— 页内 64 字节分桶的"取指异常"分布
sh00..sh56: 16 行直方图 —— 页内 64 字节分桶的"单步执行"分布
```

直方图解读: 目标地址 `xxx8F0` 落在第 `(0x8F0>>6)&63` = 35 桶(`fh32` 行第 4 列)。
命中桶 = 该地址确实被锁着执行过; 全空 = 该地址从未在锁页状态下执行。

- 判断"有没有命中"最快方式: `./pte_client -s`。
- 常见日志行对照:
  - `HIT pid=... far=... esr=...` = 精确命中断点地址(esr=0x8200000f 即 IABT+权限错);
  - `mod W8=0x2` = 已修改寄存器;
  - `STEP leave pid=... pc=...` = 线程离开断点页(页面锁工作正常);
  - `STEP slot full tid=...` = 单步槽位满(降级放行, 正常场景不应出现);
  - `step unlink done` = 卸载时手动摘单步 hook 成功。

## 十二、明确边界(本方案不做的事)

- 不用 ptrace(不 attach、不建立 PTRACE 关系)
- 不用影子页 / 幽灵页 / 克隆页(无任何额外可执行内存)
- 不改代码段(.text 零字节篡改, CRC 完全免疫)
- 不注入(无 dlopen、无 SO 注入)
- 不占硬件断点槽(BRP/WRP 全程不碰, 目标自占 4 槽不受影响)
- 唯一进程状态改动: 命中瞬间对目标线程的 `TIF_SINGLESTEP`(内核标准调试位,
  单步完成后清除; 属内核既有机制, 非持久状态位)

## 十三、许可

GPL v2(KPM 框架要求)。仅用于网络安全研究。

---

## 附录 A：开发心得与数据回顾（v1.0 收官）

> v1.0 是首个全链路稳定运行的版本，非最终版；后续若发现扩展点或 bug 可继续迭代。
> 本附录回顾完整开发历程、关键数据与方法论心得，供版本演进与同类项目参考。

### A.1 环境矩阵

| 角色 | 内核 | 平台 | 用途 |
|---|---|---|---|
| 编译 / 先行验证 | 6.1.118（android14-11） | KPatch-Next v0.01 | 编译、demo 全链路、仿真、压力测试 |
| 目标实测 | 5.15.180（android14 GKI） | KPatch-Next v0.01 | 游戏实战、与 stackplz 对测 |

### A.2 数据里程碑（按时间顺序）

| 阶段 | 关键数据 | 意义 |
|---|---|---|
| 全链路首次打通（6.1 demo） | `add=3 -> add=4662`，连续命中 15 次 | 布防->拦截->改寄存器->单步回放->撤防 闭环 |
| 5.15 demo（用户实测） | `add=4662` | 跨平台可用性验证 |
| 游戏场景首个非零计数 | 55 | 从 0 命中的突破 |
| 对照观测 | 59 vs 113（带 -r / 不带） | 首次发现"-r 改变目标行为"偏差 |
| 重锁门控修复后 | 103（带 -r）/ 185（不带） | 修复后计数大幅回升 |
| **同会话双断点对测** | **231 vs 231（stackplz）** | **零漏拦（收官判据）** |
| 游戏节奏仿真 | 两轮 250/250 | 游戏密度下捕获率 100% |
| 机制受控实验 | 150 -> 1（-r 后第 1 次即终止） | 改寄存器影响调用方逻辑的铁证 |
| 并发压力（修复前 -> 后） | 2 线程 5->143；4 线程 17->126 | "熄火" bug 修复的量化验证 |

### A.3 方法论心得

1. **崩溃必先尸检**：系统自带转储（pstore / `/data/persist_log/backup/SYSTEM_LAST_KMSG.txt`）是最权威的证据。先取现场（Comm 进程名 + 寄存器 + 调用栈 + 出错地址）再下结论；本项目数次设备崩溃、7 个根因定位全部依赖此流程。
2. **hook 逐个隔离验证**：把三种 hook 拆成独立测试模块（none/abort/step/prctl）分别加载，加载间写进度文件；卡死时"进度停在哪一步"直接指认元凶。
3. **先在可直连设备打通全链路，再上目标设备**：本机崩溃可自动恢复、日志易取；避免把未稳定版本带进目标机。
4. **平台偏移表须验证再用**：KPatch-Next 的 `task_struct_offset.pid/tgid/tasks` 恒为 -1（从不赋值）；没验证过的偏移视同陷阱，能用内核原生 API 就优先原生。
5. **警惕卸载路径 RCU 自死锁**：KP 卸载持 `rcu_read_lock`，官方 step hook 注销接口内部会 `synchronize_rcu()`；正确姿势是自持锁 + 手动摘链。
6. **失败路径是 bug 重灾区**：清理逻辑统一走单一出口、统一清指针（mmput 双释放教训）。
7. **指针运算先问语义**：`mm->pgd` 已是内核线性映射 VA（不要再加 phys->virt 偏移）；改指针算术前，源码与反汇编双重确认。
8. **多线程下共享标志不可信**：`armed` 滞后曾导致"页面永久解锁"；关键判定以实况为准（直接读 PTE），门控条件保持最小必要（`active && !disarming`）。
9. **小心观测者效应**：修改寄存器会改变目标行为（185 -> 103）；纯计数严禁带 `-r`；A/B 对照必须同会话、同地址、同终止时刻。
10. **压测分层**：最坏情况压测（2/4 线程死循环锤页）暴露架构极限（单指令窗口）；节奏仿真（5 线程 x 随机间隔）验证真实场景（100%）。两者结论分开表述。
11. **日志纪律**：异常上下文零文件 IO（仅内存环形缓冲）；每条带全局序号；只在布防 / 撤防 / 卸载 / flush 四个时刻落盘。

### A.4 版本备忘

- **v1.0 稳定判据**：双平台 demo 通过、同会话对测 231=231、仿真 250/250、修复后压力数据（见 8.5）、设备零残留（模块仅临时加载）。
- **投入**：约 1.3 亿 token 量级的持续调试会话；数次设备崩溃全部定位修复。
- **定位**：v1.0 为首个稳定运行版本，非最终版；扩展与 bug 修复继续迭代。
- **使用纪律**（务必保持）：只临时加载（绝不勾保存 / 绝不写入开机目录）；撤防优先于退出（进程已退用 `-p 0 -d`）；异常时直接卸载模块是最干净的撤销。
