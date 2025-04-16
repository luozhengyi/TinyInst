# Notes

## knowledge

- Windows操作系统中，进程的模块（Modules）通常包含以下两类文件：
  - 主模块（.exe文件）: 由操作系统加载到内存中作为进程的入口点。
  - 动态链接库（.dll文件）
  - 可能还有其他类型的文件被加载为模块，比如.ocx（ActiveX控件）、.drv（驱动程序）等
- .exe 和 .dll 都是 PE 格式
  - 代码通常存储在.text section
  - 最小内存分配单位为页（4KB），相邻页在虚拟地址空间表现为连续区块
  - PE文件被映射到内存时，各个 section(.text、.rdata、.data) 会被加载到不同的内存区域，这些区域可能不连续
  - .text节区本身通常是连续的。不过，如果代码被分段或者有热补丁、Hook等情况，可能会导致代码区间不连续
  - 另外，需要考虑动态生成代码的情况，比如某些程序可能会在运行时生成代码并申请新的内存区域，这些区域虽然属于模块的一部分，但可能不在原始的.text节区中。
  - 此外，内存对齐和页面保护机制也可能导致代码区间被分割，比如不同的内存页设置不同的权限（如可执行、可读写）可能导致**物理上的不连续**。
- `.text` section
  - 基础连续性
    - 主代码段（.text节区）在内存中通常是连续的逻辑地址空间
    - PE文件加载时，.text节区按文件中的原始顺序映射到内存
    - 最小内存分配单位为页（4KB），相邻页在虚拟地址空间表现为连续区块
  - 典型分裂场景
    - 多代码段配置：
      - 编译器可通过 `#pragma code_seg` 创建多个可执行节区（如.text、.text2）
      - 安全加固技术可能分离敏感代码到独立段（如微软的Guard页隔离）
    - 动态代码注入：
      - 运行时通过 VirtualAlloc + WriteProcessMemory 注入的代码形成新内存区
      - 热补丁技术（HotPatching）预留的跳板空间会分割原始代码段
    - 内存保护机制：
      - 不同权限设置的页边界（如代码段与只读数据段交界处）
      - Control Flow Guard（CFG）添加的校验指令可能插入间隙
  - 现代系统特性影响
    - 地址空间布局随机化（ASLR）导致基址偏移，但保持内部相对连续性
    - 基于硬件的SMAP/SMEP防护可能强制插入隔离页
    - 增量链接（Incremental Linking）生成的填充间隙
- 跳转/调用
  - 近跳转: 模块(module) 内部调用: 调用模块内部函数
  - 远跳转: 模块(module) 外部调用: 调用模块外部函数, 比如其它 DLL 中的函数

- Win APIs

```c++
SIZE_T VirtualQueryEx(
  [in]           HANDLE                    hProcess,
  [in, optional] LPCVOID                   lpAddress,
  [out]          PMEMORY_BASIC_INFORMATION lpBuffer,
  [in]           SIZE_T                    dwLength
);

// 查询进程内存信息的函数，主要用于调试和分析目的，以页面为单位;
// 页面: 在计算机内存管理中，内存被分成固定大小的块，称为“页面”。不同的操作系统可能使用不同的页面大小，常见的页面大小是 4 KB（4096 字节）。
// 页面边界: 页面边界是指内存页面的开始地址。例如，如果页面大小是 4 KB，那么页面边界的地址是 0x0000, 0x1000, 0x2000, 等等。
// lpAddress: 该地址会被向下舍入到最近的页面边界, 这意味着，如果你传递的地址不是页面边界的开始地址，函数会将其调整到该页面的开始位置。
//    eg: lpAddress 的值为 0x00001234 或者 0x00001FFF 时,会被调整为 0x00001000，VirtualQueryEx 函数将查询从 0x00001000 开始的内存区域。
// VirtualQueryEx 函数确定区域中第一页的属性，然后扫描后续页面，直到扫描整个页面范围，或遇到具有非匹配属性集的页面。 
// 函数返回具有匹配属性的页区域的属性和大小（以字节为单位）。 
// 例如，如果有一个 40 兆字节 (MB) 可用内存区域，并在区域中 10 MB 的页面上调用 VirtualQueryEx ，则函数将获取 MEM_FREE 状态，大小为 30 MB。

typedef struct _MEMORY_BASIC_INFORMATION {
  PVOID  BaseAddress; // 指向页面区域基址的指针, 是 lpAddress 向下舍入后的页边界地址
  PVOID  AllocationBase;
  DWORD  AllocationProtect;
  WORD   PartitionId;
  SIZE_T RegionSize; // 以字节为单位, 页面大小的整数倍
  DWORD  State;
  DWORD  Protect;
  DWORD  Type;
} MEMORY_BASIC_INFORMATION, *PMEMORY_BASIC_INFORMATION;

// Debugger::ExtractCodeRanges(...) 参考此方法
```

## Debugger

- 只可以指定一个 target 函数（同时要指定target 所在的 module）
  - `-target_module`
  - `-target_method`

- DebugEvent
  - EXCEPTION
    - bp BREAKPOINT_ENTRYPOINT
      - `OnEntrypoint()`
    - bp BREAKPOINT_TARGET
      - `OnTargetMethodReached()`
  - CREATE_THREAD
  - CREATE_PROCESS
    - `OnProcessCreated()`
    - 在进程的 entrypoint 处设置 bp，类型为 BREAKPOINT_ENTRYPOINT
  - EXIT_THREAD
  - EXIT_PROCESS
  - LOAD_DLL
    - `OnModuleLoaded()`
    - 如果指定了 target 在 target 的入口处设置 bp，类型为 BREAKPOINT_TARGET
  - UNLOAD_DLL

```c++
class ModuleInfo { // 在 TinyInst::OnInstrumentModuleLoaded(...) 中被赋值
 public:
  ModuleInfo();
  void ClearInstrumentation();

  std::string module_name;
  void *module_header; // 等价于 lpBaseOfDll, 检索 `module_header =`
  size_t min_address;  // Debugger::GetImageSize(...) 中被赋值, 保存的是 module_header 的值
  size_t max_address;  // max_address = min_address + SizeOfImage
  size_t code_size;    // module 所有的代码字节数的总和
  bool loaded;
  bool instrumented;
  std::list<AddressRange> executable_ranges;
  // ...
}

```

## Instrument

- TinyInst: public Debugger

### [实现原理分析](https://paper.seebug.org/3053/?comefrom=https://blogread.cn/news/)

- 拷贝要插桩的 module 的代码，并将原始代码内存属性设为 protected
- 为插桩后的代码分配空间


## Coverage

- 这里的**覆盖率**最小单位是 BasicBlock, 而不是单条语句
  - 如何识别 BasicBlock
  - 如何统计单个函数/模块总共有多少个 BasicBlock
  - 如何统计单个函数/模块执行了多少个 BasicBlock
