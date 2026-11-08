# llvm-mingw 工具链与 150KB 体积预算

项目目标"单 exe 最小体积"落到工程约束:target `x86_64-pc-windows-gnu`,linker 用 llvm-mingw(与 C++ 版同源工具链);体积预算为裸 exe(不含 UPX)≤150KB(release x64),CI 断言防回归。

达成手段:`opt-level = "z"`、`lto = "fat"`、`codegen-units = 1`、`panic = "abort"`、`strip = true`;依赖仅 windows-sys(按需 feature,仅取常量与结构体定义,零运行时体积增量),CLI 手写解析不引 clap;帮助文本、错误文案、`-v` 明细不砍——体积让位于可用性。

## Considered Options

- MSVC 默认工具链——否决:与 C++ 版 llvm-mingw 工具链分裂;llvm-mingw 单一工具链覆盖两仓库。
- UPX 再压一层——否决:杀软误报代价高,本工具做系统级文件操作。
- panic 展开(unwind)保 Drop 守卫——否决:多 10–20KB;以"全代码 Result 化、禁 unwrap/expect/索引越界"纪律替代,panic=abort 下守卫恢复走显式控制流。

## Consequences

- panic 路径即 abort,Drop 不运行;任何 panic 都属 bug,修复责任在消除 panic 而非恢复展开。
- 预算超标时先查新增依赖与格式化路径,再谈放宽。
