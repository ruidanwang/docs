## 2025-07-17 
## 01 aya-tool 生成的 vmlinux.rs

### 🧩 一、aya-tool 生成的 vmlinux.rs 可以修改吗？
**✅ 可以修改，但一般不建议手动修改原始文件。**
**为什么？**
- aya-tool（或 cargo xtask generate）是基于 BTF 自动生成 vmlinux.rs，下一次运行 generate 会完全覆盖你做的手动更改。

- 你修改的是一个机器生成的结构定义文件，和内核 ABI 强绑定，手改容易造成：

    - 字段偏移不对齐

    - 类型错配

    - panic 或 undefined behavior

**🧭 正确做法（建议）：**

**✅ 不要直接改 vmlinux.rs，而是：**
- **✅ 方式一：使用手动定义的结构体（推荐）**

自己在项目中创建一个 my_task_struct.rs，只复制你关心的字段，比如：

```rust
#[repr(C)]
pub struct MyTaskStruct {
    pub pid: u32,
    pub comm: [u8; 16],
    // 只关心你要 trace 的字段
}
```
然后用 BPF 的 unsafe 指针偏移方式从原始 task_struct 拿到字段。

- **✅ 方式二：用 aya-tool 生成后，在自动生成区域之外扩展 wrapper**
```rust
mod vmlinux {
    include!("vmlinux.rs");
}

pub struct MyWrapper {
    pub task: vmlinux::task_struct,
    // 你可以在 wrapper 中加辅助逻辑
}
```

### 🛠️ 二、什么情况会修改 vmlinux.rs？
在一些高级场景下，以下修改是有意为之：

**✅ 合理修改场景包括：**

| 修改内容      | 目的 |
| ----------- | ----------- |
| 删除不必要的结构体 |	减小编译时间、减小 ELF 体积 |
| 删除某些字段	| 规避编译器对复杂 union / padding 的抱怨 |
|手动添加 #[derive(Debug)] |	为了调试打印结构体 |
|修改字段名（如 __bindgen_padding_1）|	更易读，但可能出错 |
|删除 __bindgen_padding_...	| 让代码更简洁，但不建议 |

⚠️ 请确保你理解结构体对齐与内核 BTF 映射，否则建议不改。

### 📦 三、为什么不同项目生成的 vmlinux.rs 文件大小不一样？


**✅ 大小不一致的根本原因：**

| 差异来源	| 说明 |
| -------- |-----|
|✅ 内核版本不同	| BTF 是针对每个 Linux 内核构建的，字段/结构数量可能不同（比如 5.10 vs 6.1）|
|✅ 使用的 bpftool 不同	| 新版 bpftool 会保留更多类型或别名结构体 |
|✅ aya-tool 的配置不同	| 你可能指定了不同 --type task_struct 或无限制生成全部类型 |
|✅ 裁剪优化不同	 | 有的项目使用 bindgen 参数裁剪不必要的类型，另一些则全量生成 |
|✅ 自动清理无用结构体（新版本 aya-tool）|	某些版本会删除未被引用的结构体，有的不会 |

**📌 如何只生成自己想要的结构体（推荐）？**
如果你只关心 task_struct 和 cred，你可以：

```bash
aya-tool generate \
  --btf /sys/kernel/btf/vmlinux \
  --type task_struct \
  --type cred \
  > src/vmlinux.rs
```
这样生成的文件会非常小，只包含你指定的结构体及依赖。

**✅ 总结**

|问题	| 解答 |
|------|-----|
| vmlinux.rs 可以修改吗？|	可以，但不推荐直接改；更推荐包装或重定义关键结构 |
|修改有什么风险？	| 被重写、字段偏移错配、编译失败、运行错误 |
|如何正确修改？|	自己定义 repr(C) 结构体，只选用关心的字段 |
| 两个项目生成的文件为何不一样？	|由于内核版本、btf 内容、aya-tool 参数、是否裁剪类型等不同 |



## 02 eBPF LSM 和  AppArmor、SELinux、Smack 和 TOMOYO等对比

### 🧩 总览对比表
|特性/模块	|eBPF LSM	|SELinux	|AppArmor	|TOMOYO	 |Smack |
|---------|-----------|-----------|-----------|--------|------|
|⛓️ 强制访问控制 (MAC)	|✅ 是，可实现完整 LSM	|✅ 是，全面	|✅ 是，路径/程序级别控制	|✅ 是，过程路径控制	|✅ 是，标签系统 |
|⚙️ 实现方式 |	使用 eBPF 程序挂在 LSM hook 上 |	C 内核模块 + 策略|	内核模块 + 用户态配置文件 |	内核模块 + 学习模式 |	内核模块 + 简单策略系统 |
| 📦 用户态依赖	| 最少（可以内嵌策略）	| 需要完整的策略文件系统	| 需要 profile 文件	| 需要配置文件，学习路径记录	| 简单的标签文件| 
| 🧠 灵活性/可编程性	| ✅ 极高，自定义任意 hook 和逻辑	| ❌ 固定策略语言	| ❌ 相对简单（Profile 限定）	| ❌ 有限制	| ❌ 非程序化，固定策略| 
| 💡 动态性/热更新 |	✅ 可热加载 eBPF，无需重启	| ❌ 修改策略需重启或 reload	| ✅ 可 reload	| ✅ 可 reload	| ❌ 基本需重启策略系统| 
| 🧪 安全粒度	| ✅ 任意精度（函数、路径、UID）	| ✅ 非常高（类型/域）	| 中等（路径、程序、cap）	| 中等	| 较低| 
| 👷 用户学习成本	| 高（需要懂 eBPF）	| 极高（复杂策略语言）	| 中等（文件路径 + 权限）	| 中等（配置直观）	| 低| 
| 📈 性能	| 很高，近原生，灵活优化	| 高（但略低于 eBPF）	| 高	| 高	| 高| 
| 🧪 审计支持	| ✅ 可自定义日志	| ✅ SELinux Audit	| ✅ AppArmor Audit	| ❌ 较弱	| ❌ 较弱| 
| 🎯 应用场景	| 云原生、特定服务安全、细粒度控制	| 政府、金融、大型企业	| 桌面/服务器通用轻量级隔离	| 定制小型系统	| 简单强制策略（IoT）| 

### 🧬 1. 什么是 eBPF LSM？
eBPF LSM 是将 eBPF 程序挂载到 LSM hook 点 的能力，首次引入于 Linux 5.7，在 Linux 5.13 后完全主线支持。

它允许用户编写自定义的安全逻辑并 attach 到内核安全钩子上，如：

```rust
#[lsm_hook]
fn bpf_file_open(file: *const File, cred: *const Cred) -> i32 {
    if bpf_current_uid() == 0 {
        return -1; // 阻止 root 打开文件
    }
    0
}
```
这类程序在 内核事件触发时执行，用于审计、控制、拒绝等。

### 🔐 2. 与传统 LSM 的核心区别

| 项目	| 传统 LSM（SELinux, AppArmor 等）	| eBPF LSM| 
| ----- | ---- | -----| 
| 安全策略	| 配置文件 + 编译策略语言	| eBPF 程序| 
| 更新方式	| 修改后需 | reload、可能重启	| 动态加载、热更新| 
| 粒度	| 静态策略级别（路径/类型）	| 任意上下文（PID、UID、path、参数内容）| 
| 可组合性	| 多 LSM 组合受限（仅部分模块）	| 作为 stacked LSM 插件| 

### 🎯 3. 适用场景对比
| 场景	| 建议模块| 
| --- | ---| 
| 🏛️ 企业服务器（如 RHEL）	| SELinux（政府级安全）| 
| 🖥️ 桌面系统	| AppArmor（Ubuntu 默认）| 
| 🚀 云原生、Kubernetes	| eBPF LSM（灵活、热插拔）| 
| 🔧 IoT 嵌入式系统	| Smack / TOMOYO（轻量）| 
| 🧪 安全研究/定制监控	| eBPF LSM（Hook 级别控制）| 

### ⚠️ 4. 多 LSM 共存问题

Linux 支持 Stacked LSM，允许多个 LSM 并存，但：

- **SELinux 和 AppArmor 通常不能共存**

- **eBPF LSM 是作为 secondary module 插入，通常可以和现有 LSM 共存（非默认 deny-only）**

### 🛡️ 5. 开发成本和难度
| 模块	| 开发方式	| 学习难度| 
| --| --| --| 
| SELinux	| 写策略、使用 semanage, audit2allow	| 极高| 
| AppArmor	| 写 profile，简单规则	| 中等| 
| eBPF LSM	| 写 Rust/C eBPF，理解内核 Hook 点	| 高| 
| Smack / TOMOYO	| 写简单配置	| 中低| 

### 🧠 小结：你该选择哪个？
| 角色	| 推荐选择| 
| --| --| 
| 安全研究者	| eBPF LSM：灵活、高粒度、易实验| 
| 企业安全工程师	| SELinux：政府合规、集中策略| 
| 云安全工程师	| eBPF LSM + AppArmor（组合灵活）| 
| 桌面发行版开发者	| AppArmor：轻便、用户易用| 
| IoT 系统工程师| 	Smack / TOMOYO：超轻量| 



## 03 LSM 规则（hooks）

下面是一个全面的 Linux 安全模块（LSM, Linux Security Modules）的 规则（hooks）体系总览，包括：

- 内核中可供 LSM 模块挂钩的全部 hook 点（lsm_hooks）

- 各模块（SELinux / AppArmor / eBPF LSM / Smack / TOMOYO）通常使用的规则类别

- 覆盖文件、进程、网络、IPC、模块、内存等各个维度

- 适合安全策略开发、eBPF-LSM 插桩点选择、对比理解

### 🧩 1. LSM Hook 点（规则类型）总览

Linux 内核通过 security/security.c 公开了大量 LSM hook 点，分为以下大类：

| 类型	| 示例 Hook 函数	| 用途| 
| -- | -- | -- | 
| 进程控制	| security_bprm_check, task_kill, ptrace_access_check	| 控制进程执行、信号发送、调试权限等| 
| 文件操作	| inode_permission, file_open, file_ioctl	| 控制文件打开、访问、权限检查等| 
| 网络操作	| socket_create, socket_connect, socket_sendmsg	| 限制网络套接字创建、连接、数据发送| 
| IPC / 信号量	| ipc_permission, sem_associate	| 控制消息队列、信号量、共享内存等| 
| 模块加载	| kernel_module_request	| 控制模块加载与禁止指定模块| 
| 挂载操作	| sb_mount, sb_umount, sb_remount	| 限制文件系统挂载、卸载等| 
| 内存操作	| mmap_file, mmap_addr	| 控制 mmap 行为，防止可执行映射| 
| 设备控制	| dev_alloc_name, dev_create	| 控制字符块设备的使用与命名| 
| 任务生命周期	| task_create, task_free	| 控制任务创建和销毁行为| 
| BPF/Perf 相关	| perf_event_open, bpf	| 限制用户开启 perf/bpf 程序| 
| LSM 自定义	| security_add_hooks	| 插入自定义 hook（用于 eBPF LSM）| 

### 🧬 2. 按照资源维度分类的完整 LSM Hook 表

| 分类	| Hook 示例	| 功能说明| 
| --| --|- | 
| 🧵 进程相关	| bprm_check_security, task_create, task_fix_setuid, task_kill, ptrace_access_check	| 控制 exec、fork、信号发送、ptrace 调试| 
| 📁 文件与 inode	| inode_permission, inode_create, file_open, file_ioctl, file_permission, file_mmap	| 文件打开、创建、mmap 映射、ioctl 使用| 
| 🌐 网络	| socket_create, socket_connect, socket_bind, inet_conn_request, inet_csk_clone	| 套接字创建、连接、监听等| 
| 📦 挂载与超级块	| sb_mount, sb_remount, sb_umount, sb_kern_mount	| 文件系统挂载行为控制| 
| 🔗 符号链接和路径| path_truncate, path_unlink, path_symlink	| 限制软链接、删除行为| 
| 🔧 模块与系统控制	| kernel_module_request, capable, quotactl, syslog	| 控制内核模块、capabilities 使用| 
| 📥 IPC 控制	| ipc_permission, shm_shmat, sem_associate	| 控制 shm、sem、mq 等使用| 
| 🔬 BPF 和 perf	| bpf, perf_event_open, perf_event_write	| 限制加载 BPF 程序、开启 perf events| 
| 🔒 安全上下文	| cred_prepare, cred_transfer, task_setscheduler	| 控制进程身份、调度、用户切换| 
| 🗄️ 安全审计支持	| audit_rule_match, audit_rule_free, audit_rule_field	| 触发审计日志与规则匹配| 
| 🧠 LSM 基础控制	| security_add_hooks, init_task_security, cred_alloc_blank	| 初始化安全属性，自定义扩展| 

### 🧠 3. 各 LSM 使用的典型规则（按模块）
| 模块	| 常用 Hook 类型	| 示例| 
| -|- | -| 
| SELinux	| 几乎使用所有 LSM Hook，基于类型强制访问控制	| inode_permission, bprm_check, socket_create, capable, task_setpgid| 
| AppArmor	| 主要基于路径的文件访问和执行控制	| file_open, bprm_check_security, path_truncate, socket_bind| 
| Smack	| 使用简化标签系统，较少 hook	| inode_permission, socket_connect, file_open| 
| TOMOYO	| 使用路径/程序组合学习策略，文件与进程类 Hook	| bprm_check_security, file_open, file_ioctl, task_setuid| 
| eBPF LSM	| 用户可选择挂载任意 Hook 点	| lsm::file_open, lsm::socket_connect, lsm::task_kill 等自定义函数| 


### 📌 总结
| 功能	| 推荐 Hook| 
|- |- | 
| 阻止某些进程运行	| bprm_check_security| 
| 控制文件读取	| inode_permission, file_open| 
| 阻止特定 IP 连接	| socket_connect| 
| 禁止用户加载模块	| kernel_module_request| 
| 拦截 mmap 为可执行	| file_mmap| 
| 控制某些信号	| task_kill| 



## 04 需要生成 struct的 LSM Hook 

### 🧵 进程类（常涉及 task_struct, cred, linux_binprm）

| Hook 名称	| 参数	| 常用目的	| 需要结构体| 
| -|- |- | -| 
| bprm_check_security	| *const linux_binprm	| 拦截进程执行	| ✅ linux_binprm| 
| task_kill	| *const task_struct	| 信号控制	| ✅ task_struct| 
| task_fix_setuid	| *const cred	| UID 切换检测	| ✅ cred| 
| ptrace_access_check	| *const task_struct	| 调试权限	| ✅ task_struct| 
| task_create	| *const task_struct	| 进程克隆/创建	| ✅ task_struct| 

### 📁 文件类（涉及 file, dentry, inode）
| Hook 名称	| 参数	| 目的	| 需要结构体| 
|- | -|- | -| 
| file_open	| *const file	| 文件打开拦截	| ✅ file, dentry, inode| 
| file_permission	| *const file	| 权限检查	| ✅ file, cred| 
| inode_permission	| *const inode, *const cred	inode | 权限检查	| ✅ inode, cred| 
| file_ioctl	| *const file, cmd	| 限制某些 ioctl	| ✅ file| 

### 🌐 网络类（涉及 sock, sockaddr, socket）
| Hook 名称	| 参数	| 用途	| 需要结构体| 
|- | -|- |- | 
| socket_connect	| *mut sock, *const sockaddr	| 拦截连接请求	| ✅ sock, sockaddr| 
| socket_bind	| 同上	| 限制端口绑定	| ✅ sock, sockaddr| 
| inet_conn_request	| *mut sock, *mut sk_buff	| 入站连接请求	| ✅ sock, sk_buff| 
| socket_create	| int family, int type, int protocol	| 检查 socket 创建参数	| ❌（无指针参数，不需要 struct）| 

### 🧪 是否一定要生成结构体？
**不一定：**

- 如果你只依靠 UID、GID、capability，可以用 helper（如 bpf_get_current_uid_gid()）直接获取，不需要结构体。

- 但如果你需要访问结构体的字段（如文件路径、socket 地址、命名空间信息等），就必须通过 aya-tool 生成并安全读取。

### 🛠️ 如何生成结构体？
```bash
aya-tool generate \
  --path /sys/kernel/btf/vmlinux \
  --target x86_64 \
  --output src/vmlinux.rs \
  --objects task_struct cred file linux_binprm dentry inode sock sockaddr
```
你可以根据用到的 hook 参数选择性生成。

### 📌 总结
| 使用 Hook	| 是否需要生成结构体？	| 生成哪些？| 
|- |- |- | 
| bprm_check_security	| ✅	| linux_binprm| 
| task_kill	| ✅	| task_struct| 
| file_open	| ✅	| file, dentry| 
| socket_connect	| ✅	| sock, sockaddr| 
| socket_create	| ❌	| 无结构体指针，不必生成| 
| capable	| ❌	| 可通过 helper 获取 uid、cap，无需结构体| 



## 05 LSM eBPF 很强大,但还需要kprobe、tracepoint、uprobe 等 eBPF 类型

### 🧠 结论先行：

**是的，虽然 LSM eBPF 很强大，但它并不能覆盖所有内核可观测点。Aya 中还保留 kprobe、tracepoint、uprobe 等 eBPF 类型，是因为：**

**🔒 LSM hook 主要聚焦在“安全决策点” —— 安全策略、访问控制；而其他 eBPF 类型（kprobe、tracepoint、XDP、tc 等）聚焦在性能监控、网络过滤、行为观测等更广泛用途。**

### 🔍 为什么需要其他类型的 eBPF 程序？
我们来具体看一下每种类型的 eBPF 能力和 LSM 的局限。

#### 🛡️ LSM eBPF 的能力与边界
**LSM = Linux Security Module Hook**

**主要能力：**

| 功能	| 示例| 
| -| -| 
| 阻止某些行为（返回非 0）	| 拦截 root exec、写文件、kill 等| 
| 检查权限	| UID、GID、CAP、SELinux label、文件 inode| 
| 微观访问控制（MAC）	| 拦截 socket_bind、file_open、ptrace 等| 
| 最终结果影响系统行为	| 拒绝某行为| 

**LSM 的局限：**

| 不适合的场景	| 原因| 
|- | -| 
| 无安全语义的性能采样	| LSM hook 不存在于调度器、内存分配路径| 
| 网络包的早期处理（XDP 层）	| LSM hook 无法插入网卡驱动入口| 
| 用户态函数跟踪	| 无法 hook 到用户空间代码（需要 uprobe）| 
| 内核任意函数调用跟踪	| 只能 hook LSM 插桩点，hook 不到普通函数| 
| 原始事件观察（如 syscall latency）	| 无直接 hook，需使用 tracepoint/kprobe| 

### 🔩 为什么还需要 kprobe、tracepoint、uprobe、XDP 等？

| eBPF 类型	| 可 hook 场景	| 用途| 
| -|-| -| 
| kprobe	| 任意内核函数（如 do_sys_openat2）	| 内核函数行为观测、动态分析| 
| tracepoint	| 内核定义好的静态点（如 sched_process_exit）	| 稳定观测点，性能低开销| 
| uprobe	| 任意用户态函数（如 libc 中的 malloc）| 	追踪用户进程行为| 
| XDP	| 网络驱动层入口	| 高性能包过滤（DDoS 防护等）| 
| tc	| 网络协议栈层	| 更细粒度包处理| 
| perf_event	| 计时器 / 硬件事件	| 周期性采样、性能分析| 
| LSM	| LSM 安全接口（有限函数）	| 安全策略判断与拒绝操作| 

### 🧪 示例对比：你需要谁？

**✅ 目标：**
| 目标	| 类型| 
| -| -| 
| 禁止 | root 执行某些程序	| ✅ LSM| 
| 查看哪个进程占用 IO 高	| ❌ LSM 无法，需 tracepoint/kprobe| 
| 抓取 syscall 执行时间分布	| ❌ LSM 无法，需 tracepoint| 
| 捕获用户空间函数参数	| ❌ LSM 无法，需 uprobe| 
| 拦截网络包在驱动层	| ❌ LSM 无法，需 XDP| 
| 分析 TCP 重传	| ❌ LSM 无法，需 kprobe + sk_buff| 
| 记录所有 execve 的命令行	| ✅ LSM 可读 linux_binprm，但性能上 tracepoint 也可以| 

### 📌 总结一句话：
**LSM eBPF 程序是安全控制的王者，但无法观察/分析整个系统行为。kprobe、tracepoint 等其他类型让 eBPF 成为一把系统可观测性“瑞士军刀”。**

