+++
date = '2026-04-04T18:00:00+08:00'
draft = false
title = 'C++ 学习笔记：内存底层机制与网络协议解析'
tags = ["C++", "网络编程", "内存管理"]
summary = "重新审视 C++ 基础体系，记录从变量内存实质、指针操作规范到连续内存游走逻辑，以及网络协议解析中的结构体对齐与大小端转换。"

+++

> 之前觉得自己有写过几个项目，找工作应该不会太难，结果面试的时候被几个基础八股给问住了。之后越想越难受，想翻了一下自己记的东西，发现好多概念都是囫囵吞枣混过来的，发现自己并没有之前觉得那么厉害，或者说被ai给惯出幻觉了，所以希望能重新把相关的理论知识过一遍，边学边练，这篇文章就是第一章。

## 1. 变量与内存：物理空间到底长什么样

C++ 里声明一个变量，本质上就是向系统要一块连续的内存空间，至于这块内存具体落在哪里、占多大，取决于类型和声明位置。

在 64 位系统下，`char` 占用 1 字节，`int` 占用 4 字节。指针变量（存储内存地址）固定占用 8 字节。

```cpp
#include <iostream>

int main() {
    int a = 10;
    char b = 'A';
    double c = 3.14;

    std::cout << "int 占用字节数: " << sizeof(int) << "\n";      // 4
    std::cout << "变量 a 占用字节数: " << sizeof(a) << "\n";      // 4
    std::cout << "char 占用字节数: " << sizeof(char) << "\n";    // 1
    std::cout << "指针的占用字节数: " << sizeof(&a) << "\n";      // 8 (64位)
    return 0;
}
```

输出：

```tex
int 占用字节数: 4
变量 a 占用字节数: 4
char 占用字节数: 1
指针的占用字节数: 8
```

> 以前总觉得`int`变量可以为`null`，甚至写过`if (a == null)` 。后来才搞明白，内存分配好后，那4个字节就一直存在，只是里面存的是什么值的问题。`0`是合法数值，`-1`也是合法数值，根本不存在`null`这个概念。所谓的`null`只对指针有意义，因为指针可以指向`nullptr`，但指针本身占的那 8 个字节也从来没空过。

**栈、堆、全局变量的地址特征**（64位Linux/macOS典型值）：

- 栈变量地址：`0x7fff16085624`（高地址）
- 堆变量地址：`0x57ae6d713920`（低地址）
- 全局变量地址：`0x57ae4f92d038`（低地址，与堆相近）

> 此前一度觉得可以通过地址值来判断变量在栈还是堆，比如看到 `0x7fff` 开头的就说是栈，然后在代码里写 `if (addr > 0x70000000)` 之类的判断逻辑。但 ASLR 一开、换个系统版本、或者程序跑在不同架构上，地址范围完全不一样。生产代码里真要区分，必须在分配的时候自己记录下来，比如用带来源标记的封装类型，别靠地址猜。

变量在栈上、堆上还是全局区，由它的声明方式决定，不是地址告诉你的。

---

## 2. 指针与引用：严格区分语义与防御边界

`*` 和 `&` 在声明和操作里含义不一样，这个坑踩过几次之后就记住了。

- **指针 (`*`)**：声明时表示"这是个指针类型"，使用时表示解引用去取地址里的值。指针可能指向无效地址，所以解引用之前最好检查一下。
- **引用 (`&`)**：声明时表示"给变量起个别名"，使用时就跟原变量一样，不需要额外操作。语法上强制要求初始化时必须绑定一个有效实体，所以不存在空引用。

```cpp
// 指针版本：面临空指针风险，需要判空
void swap_by_pointer(int* a, int* b) {
    if (a == nullptr || b == nullptr) return; 
    int temp = *a;
    *a = *b;
    *b = temp;
}

// 引用版本：语法层面保证安全，无需判空
void swap_by_reference(int& a, int& b) {
    int temp = a;
    a = b;
    b = temp;
}
```

> 有一阵子写函数只要是传参就习惯性加 `if (a == nullptr)` 的防御，包括参数是引用的时候也这么干。但引用根本不可能为空，这种检查编译都过不去（除非强行转成指针再比，但那更蠢了）。后来想通了，引用本身就保证了有效性，少写一行判空就少一个出错的可能。

能用引用就尽量别用指针，代码清晰是一方面，安全是另一方面。

---

## 3. 数组和双指针：直接游走比下标计算舒服

数组名在绝大多数情况下会退化成指向第一个元素的指针。操作连续内存的时候，用指针移动比算下标更直观、也更贴近底层。

**物理边界法则**：连续内存块的最后一个合法元素地址为 `start + size - 1`。

```cpp
void reverse_memory(int* start, int size) {
    if (start == nullptr || size <= 1) return;

    int* left = start; 
    int* right = start + size - 1; // 指向最后一个元素

    while (left < right) {
        swap_by_reference(*left, *right); 
        left++;   // 自动步进 sizeof(int)
        right--;  
    }
}
```

> **以前写这种逻辑的时候**，习惯用 `for (int i = 0; i < size / 2; ++i)` 然后 `swap(start[i], start[size - 1 - i])`。虽然也能跑，但每次循环都要算一次 `size - 1 - i`，而且读起来总得多想一下。改成首尾指针相向移动之后，`left++` 和 `right--` 就是直接的地址加减，逻辑更接近内存的物理结构，也省了循环里那些多余的计算。

操作连续内存，指针游走比下标计算更顺手。

---

## 4. 堆内存分配

### 4.1 基本语法与初始化

```cpp
#include <iostream>

int main() {
    int* p1 = new int;        // 未初始化（值不确定）
    int* p2 = new int();      // 值初始化为 0
    int* p3 = new int(100);   // 初始化为 100
    int* p4 = new int[5];     // 5 个元素，未初始化
    int* p5 = new int[5]();   // 5 个元素，全部初始化为 0
    int* p6 = new int[5]{10,20,30}; // 前三个 10,20,30，后两个 0

    std::cout << "*p1 = " << *p1 << " (可能是随机值)" << std::endl;
    std::cout << "*p2 = " << *p2 << std::endl;
    std::cout << "*p3 = " << *p3 << std::endl;
    std::cout << "p6[0]=" << p6[0] << " p6[1]=" << p6[1] << " p6[2]=" << p6[2]
              << " p6[3]=" << p6[3] << " p6[4]=" << p6[4] << std::endl;

    delete p1; delete p2; delete p3;
    delete[] p4; delete[] p5; delete[] p6;
    return 0;
}
```

输出：

```tex
*p1 = 0 (可能是随机值)
*p2 = 0
*p3 = 100
p6[0]=10 p6[1]=20 p6[2]=30 p6[3]=0 p6[4]=0
```

> 以前觉得 `new int` 和 `new int()` 是一回事，反正内存分配了就行。但实际上前者不初始化，读它的值是未定义行为，后者会清零。同样，数组初始化列表里没明确指定的元素也会被置零。所以除非你明确想写垃圾值，不然一律用带初始化的版本，省得后面出一些莫名其妙的 bug。

### 4.2 堆地址间隔

连续多次 `new` 返回的地址之间并无固定关系，有时候下一个地址反而比上一个更小。

```cpp
#include <iostream>
#include <vector>

int main() {
    std::vector<void*> addrs;

    // 连续 new int
    for (int i = 0; i < 5; ++i) {
        addrs.push_back(new int(i));
    }
    std::cout << "连续 new int 的地址间隔:" << std::endl;
    for (size_t i = 1; i < addrs.size(); ++i) {
        std::cout << (char*)addrs[i] - (char*)addrs[i-1] << " ";
    }
    std::cout << std::endl;
    for (void* p : addrs) delete static_cast<int*>(p);
    addrs.clear();

    // 交错分配不同大小
    for (int i = 0; i < 3; ++i) {
        addrs.push_back(new int(i));
        addrs.push_back(new double(i));
        addrs.push_back(new char[100]);
    }
    std::cout << "交错分配不同大小的地址间隔:" << std::endl;
    for (size_t i = 1; i < addrs.size(); ++i) {
        std::cout << (char*)addrs[i] - (char*)addrs[i-1] << " ";
    }
    std::cout << std::endl;

    return 0;
}
```

输出：

```
连续 new int 的地址间隔:
64 -32 64 80 
交错分配不同大小的地址间隔:
-80 1232 -1296 32 1376 -1440 1552 32
```

> 堆分配器内部有元数据、空闲链表、各种对齐策略，不同大小的对象交错分配之后地址分布完全是乱的。所以不能假设两次 `new` 的地址有什么关系，需要连续内存就老老实实用数组 `new T[N]` 或者 `std::vector`。

---

## 5. 结构体对齐（内存对齐）

CPU 按字读取内存，编译器默认会在结构体成员之间插入一些空洞（填充字节），让每个成员都落在自然对齐的地址上。这样访存效率高，代价是多占一点空间。

> **有个概念得先说清楚**：`sizeof` 和结构体的内存布局在编译期就定死了，运行时改不了。我以前想过能不能在程序里根据网络数据包的情况动态决定要不要对齐，这方向从一开始就不对。

做网络协议解析的时候，通常两种做法：

1. **边界映射**：用 `#pragma pack(1)` 强制取消对齐，定义一个紧凑的结构体直接映射收到的字节流。这样结构体大小就是各成员大小之和，没有空洞。
2. **反序列化（空间换时间）**：但对齐取消之后，成员可能落在未对齐的地址上，直接频繁读取会有性能问题。所以一般从紧凑结构体里把数据读出来之后，马上赋给本地对齐的变量，后续逻辑都用本地变量操作。

```cpp
#pragma pack(push, 1) 
struct PackedNetworkHeader {
    char type;
    int length;
};
#pragma pack(pop)

void process_network_packet(char* raw_buffer) {
    PackedNetworkHeader* header = (PackedNetworkHeader*)raw_buffer;

    // 拿出来之后立刻转存到对齐的本地变量
    int local_length = header->length; 
    char local_type = header->type;

    // 后面所有逻辑都基于 local_length 和 local_type 走
}
```

简单说就是：用紧凑结构体去映射报文，但访问完成员马上拷贝出来，别在紧凑结构体上做大量计算。

---

## 6. 大小端字节序

在处理跨网络的多字节数据（如 `int`、`short`）时，内存中字节的排列顺序决定了数据的正确性。

- **大端序**：高位字节存放在低地址。**TCP/IP 协议标准规定的网络字节序**。
- **小端序**：低位字节存放在低地址。**x86 及现代 ARM 主机的默认字节序**。

> **以前写过字节序判断和翻转函数**，用 `char*` 强转去读内存，然后根据结果手动位移拼接。后来查了一下才发现这活儿有现成的标准库函数，而且还针对不同平台做了优化——如果源端序和目标端序一致，直接变成空操作；不一致就调用高效的汇编指令（比如 x86 的 `bswap`），比自己手写靠谱多了。

### 标准转换函数族

函数名遵循 `[源端]to[目的端][类型]` 的缩写规则：

- **h**: host (主机字节序)
- **n**: network (网络字节序)
- **s**: short (16 位，2 字节，常用于 Port 端口)
- **l**: long (32 位，4 字节，常用于 IPv4 地址、报文长度)

常用 API：

- `ntohl()`: Network to Host Long (32位网络序转主机序)
- `ntohs()`: Network to Host Short (16位网络序转主机序)
- `htonl()`: Host to Network Long (32位主机序转网络序)
- `htons()`: Host to Network Short (16位主机序转网络序)

### 举个例子：解析网络报文长度

```cpp
#include <iostream>
#include <arpa/inet.h>

#pragma pack(push, 1)
struct RawPacket {
    char flag;      // 1 字节
    int data_len;   // 4 字节，网络序（大端）
};
#pragma pack(pop)

void parse_packet(char* buffer) {
    if (buffer == nullptr) return;

    RawPacket* pkt = (RawPacket*)buffer;
    int safe_len = ntohl(pkt->data_len);   // 转成主机序
    char* payload = buffer + 5;            // 跳过头部

    std::cout << "主机端识别的长度: " << safe_len << "\n";
}

int main() {
    // 模拟网络收到的原始数据：flag='A', len=1（大端表示为 0x00 0x00 0x00 0x01）
    unsigned char network_data[] = {'A', 0x00, 0x00, 0x00, 0x01};
    parse_packet((char*)network_data);     // 小端主机上输出 1
    return 0;
}
```

输出：

```tex
主机端识别的长度: 1
```

凡是跨网络传输的多字节整数，从报文里取出来之后马上用 `ntoh*` 或 `hton*` 转一下再参与业务逻辑，别偷懒。

