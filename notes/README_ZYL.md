# Notes

## knowledge

- module: 可以是 .exe 或者是 .dll

## Debugger

- 只可以指定一个 target 函数（同时要指定target 所在的 module）
  - `-target_module`
  - `-target_method`

- DebugEvent
  - EXCEPTION
    - bp BREAKPOINT_ENTRYPOINT
      - OnProcessEntrypoint()
    - bp BREAKPOINT_TARGET
      - OnTargetMethodReached()
  - CREATE_THREAD
  - CREATE_PROCESS
    - OnProcessCreated()
    - 在进程的 entrypoint 处设置 bp，类型为 BREAKPOINT_ENTRYPOINT
  - EXIT_THREAD
  - EXIT_PROCESS
  - LOAD_DLL
    - OnModuleLoaded()
    - 如果指定了 target 在 target 的入口处设置 bp，类型为 BREAKPOINT_TARGET
  - UNLOAD_DLL

## Instrument

- TinyInst: public Debugger

## Coverage

- 这里的**覆盖率**最小单位是 BasicBlock, 而不是单条语句
  - 如何识别 BasicBlock
  - 如何统计单个函数/模块总共有多少个 BasicBlock
  - 如何统计单个函数/模块执行了多少个 BasicBlock
