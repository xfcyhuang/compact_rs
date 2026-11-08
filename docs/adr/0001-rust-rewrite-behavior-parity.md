# Rust 重写,全行为对齐 ntfscompact C++ 版

compact_rs 是 [ntfscompact](https://github.com/xfcyhuang/ntfscompact)(C++)的全功能 Rust 重写,产物同名 `ntfscompact.exe`,可直接替换旧版。所有可观察行为——退出码 0/1/2/3、输出行格式与 stdout/stderr 路由、输出顺序、升级释放链及其全部参数(退避 100/300/1000ms、单文件预算 5000ms、Oplock 超时 2000ms、收尾预算 4000ms、低风险白名单 5 进程、高风险黑名单 41 进程)、瞬态重试 10×50ms、只读属性守卫、长路径 `\\?\` 规范、WIM/Cloud 黑名单、pagefile/hiberfil/swapfile 与 WinSxS 排除、≤4096 字节阈值(显式文件同样过滤)——逐项对齐 C++ 版 master 基准 commit `76deb95`。

**唯一刻意差异**:通配符匹配到目录时,C++ 版静默丢弃(无任何输出);Rust 版输出跳过行。对齐不等于复刻缺陷,此差异为已知且有意。

## Considered Options

- 继续维护 C++ 版——否决:目标是 Rust 单 exe 最小体积,重写即目的。
- 重写时顺手改进各处行为——否决:行为对齐使两版可无缝替换、测试用例可复用;改进需求先记差异清单再单独决策。

## Consequences

- C++ 版成熟后 archive,两仓库 README 互指。
- 术语表(GLOSSARY.md 八术语)自 C++ 版原样继承。
- 未来任何行为偏离基准的需求,须扩充上述"刻意差异"清单。
