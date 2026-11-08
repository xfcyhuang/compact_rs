# wofutil / rstrtmgr / ntdll 全动态加载

`WofSetFileDataLocation`、`WofIsExternalFile`(wofutil.dll),Restart Manager 五 API(rstrtmgr.dll),`NtQuerySystemInformation`(ntdll.dll)全部经 `LoadLibraryW` + `GetProcAddress` 动态解析,与 C++ 版一致;windows-sys 只提供常量与结构体定义,不做静态链接。

理由:llvm-mingw 环境无 wofutil.lib / rstrtmgr.lib 导入库,静态导入需手工生成 .def,无收益;`NtQuerySystemInformation` 本就无官方导入库;动态加载保留"缺 DLL 时输出 `ntfscompact: <错误>` 并退出码 1"的友好行为(含干跑模式,对齐 C++ 版)。
