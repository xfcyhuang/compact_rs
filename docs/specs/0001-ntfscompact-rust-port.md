# Spec 0001: ntfscompact Rust 移植

compact_rs = [ntfscompact](https://github.com/xfcyhuang/ntfscompact)(C++)的全功能 Rust 重写。产物 `ntfscompact.exe`(x64 单文件,裸 exe ≤150KB),行为对齐 C++ 版 master 基准 commit `76deb95`,可直接替换旧版。

- 术语:`GLOSSARY.md`(八术语,自 C++ 版原样继承)
- 决策:`docs/adr/0001`(对齐基准与唯一差异)、`docs/adr/0002`(llvm-mingw 与体积预算)、`docs/adr/0003`(全动态 DLL 加载)
- C++ 参考源码(本地解包,可直读):`D:\Temp\ntfscompact_extract\xfcyhuang-ntfscompact-76deb95\`
- C++ 版完整行为事实提取(输出格式逐行样例、API 常量、参数表):`D:\Temp\compact_rs_notes\cpp-facts.md`

## 行为规范(摘要,细则以 cpp-facts.md 为准)

### CLI

```
ntfscompact <路径...> [-a xpress4k|xpress8k|xpress16k|lzx] [-u] [-n] [-q] [-v] [--force-close-all]
```

- 默认算法 xpress8k;`-a` 与 `-u` 互斥;任何以 `-` 开头的未识别 token 视为未知选项(退出码 2,stderr:`ntfscompact: <错误>\n\n` + 完整用法)
- `--help` / `--version`(输出 `ntfscompact 0.3.0`)→ stdout,退出码 0

### 退出码

| 码 | 含义 |
|---|---|
| 0 | 全部成功(跳过不算失败) |
| 1 | 部分失败;启动装配失败(SetConsoleCtrlHandler 注册、wofutil/rstrtmgr 加载失败,含干跑)同为 1 |
| 2 | 用法错误 |
| 3 | Ctrl+C/关窗/注销/关机中断(中断优先于失败码) |

### 压缩/解压机制

- wofutil.dll 动态加载 `WofSetFileDataLocation` / `WofIsExternalFile`;算法码:xpress4k=0、lzx=1、xpress8k=2、xpress16k=3;Provider=WOF_PROVIDER_FILE(2);结构体 `WofFileCompressionInfoV1 { u4_algorithm, u4_flags }`
- FSCTL 本地常量:GET_COMPRESSION=CTL_CODE(9,15,METHOD_BUFFERED,ANY)、SET_COMPRESSION=CTL_CODE(9,16,...)、DELETE_EXTERNAL_BACKING=CTL_CODE(9,197,METHOD_BUFFERED,SPECIAL_ACCESS)
- 瞬态重试:10 次 × 50ms,针对 ACCESS_DENIED/SHARING_VIOLATION/LOCK_VIOLATION/OPLOCK_NOT_GRANTED/OPLOCK_BREAK_IN_PROGRESS;HRESULT 0x8007xxxx 先 `& 0xFFFF` 解包
- 压缩后复核 `WofIsExternalFile`(WOF+provider FILE+算法匹配);解压后复核须为未压缩
- `ERROR_COMPRESSION_NOT_BENEFICIAL` → 跳过"无收益",非错误

### 候选过滤(scan)

- `kMinCompressibleSize = 4096`,条件 `<=`(即须 >4096);显式传入的文件同样过滤
- 拒绝:ENCRYPTED、非常规 reparse(例外:tag==`IO_REPARSE_TAG_WOF` 0x80000017 放行)、pagefile.sys/hiberfil.sys/swapfile.sys(文件名不分大小写)、路径任一组件名 == WinSxS
- 隐藏/系统属性不过滤;稀疏文件无专门处理
- 通配符匹配到目录:输出跳过行(**唯一刻意差异**,C++ 静默丢弃)
- 长路径:操作路径 `\\?\` 前缀(UNC → `\\?\UNC\`),显示路径剥前缀;GetFullPathNameW 缓冲 32768

### 升级释放链(release)

四级:Oplock 协商 → RM 优雅关 → RM 强制关 → 句柄强关;级间退避 {100,300,1000}ms、单文件总预算 5000ms;每级前重探测。入口先探测(GENERIC_READ CreateFile 失败且 SHARING_VIOLATION 才算占用);执行后错误码 32 再走链重试一次。

- Oplock:`FSCTL_REQUEST_OPLOCK`,RWH 三级,OVERLAPPED+事件,超时 2000ms,`CancelIoEx` 收场
- RM:rstrtmgr.dll 动态加载 `RmStartSession`/`RmRegisterResources`/`RmGetList`(两次调用)/`RmShutdown`(0 或 0x1)/`RmEndSession`
- 句柄强关:非提权止步;ntdll 动态取 `NtQuerySystemInformation`(SystemExtendedHandleInformation=64,缓冲 1<<19 起翻倍至 1<<27,STATUS_INFO_LENGTH_MISMATCH=0xC0000004 扩容);`DuplicateHandle` 复制 + `GetFinalPathNameByHandleW` 小写全等匹配;granted_access 无 FILE_READ_DATA 且无 FILE_WRITE_DATA 的句柄剔除;`DUPLICATE_CLOSE_SOURCE` 强关
- 低风险白名单 5 个(searchindexer/searchprotocolhost/searchfilterhost/msmpeng/nissrv.exe);高风险黑名单 41 个(explorer、Office、编辑器、数据库、虚拟机、系统关键、浏览器,全表见 cpp-facts.md §4);不在两表 = Unknown 同样不放行(须 --force-close-all)
- 绝不卷卸载、绝不 TerminateProcess

### 中断协议(interrupt)

`SetConsoleCtrlHandler`;三原子标志 requested/finishing/wrapup_complete。Ctrl+C/Break:首次置位继续,收尾中再收 → `ExitProcess(3)`。关窗/注销/关机:handler 内有界等待收尾(50ms 步进,预算 4000ms),超时/再收 → `ExitProcess(3)`。观察点:扫描枚举每项 + 流水线每项边界;在途项完整走完(含守卫恢复、字节统计、部分汇总+标注行)。

### 输出格式

骨架 `[标签] 显示路径：内容`,全角冒号,无对齐。路由:失败行→stderr(含 -q);其余逐文件行→stdout(-q 抑制);汇总与中断标注→stdout 无条件。顺序:目录标记处理(先)→扫描跳过行→逐文件项→扫描错误行(最后)→汇总→中断标注。逐行样例、-v 附加、汇总行拼接规则、升级释放审计后缀格式:cpp-facts.md §2。

## 工程约束

- target `x86_64-pc-windows-gnu`,linker llvm-mingw(`D:\software\LLVM\bin`,clang 23.1.0)
- profile:opt-level=z、lto=fat、codegen-units=1、panic=abort、strip=true
- 依赖:仅 `windows-sys`(按需 feature,只取常量/结构体);无 clap,CLI 手写;三 DLL(wofutil/rstrtmgr/ntdll)全动态加载
- 全代码 Result 化:禁 unwrap/expect/索引越界/panic 路径
- CI:tag 触发 Release + 体积断言(>150KB 失败);cargo test 含真实压缩用例(本机临时目录,勿在网络卷跑)

## 验收

1. `cargo build --release` 产物 ≤150KB(裸,无 UPX)
2. `cargo test` 全绿(含真实压缩用例)
3. 与 C++ 版输出逐行对照:压缩/重压/解压/跳过/失败/干跑/-q/-v/汇总/中断场景格式一致(样例集见 cpp-facts.md §2)
4. 退出码 0/1/2/3 语义一致
