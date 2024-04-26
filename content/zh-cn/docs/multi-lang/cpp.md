---
title: 使用 C++
description: 基于C++封装DUT硬件的运行环境，并编译为动态库。
categories: [教程]
tags: [docs]
weight: 1
---

## 原理介绍

### 基础库

在本章节中，我们将介绍如何使用Picker将RTL代码编译为C++ Class，并编译为动态库。

1. 首先，Picker工具会解析RTL代码，根据指定的 Top Module ，创建一个新的 Module 封装该模块的**输入输出端口**，并导出`DPI/API`以操作输入端口、读取输出端口。
    > *工具通过指定Top Module所在的文件和 Module Name来确定需要封装的模块。此时可以将 Top 理解为软件编程中的main。*

2. 其次，Picker工具会使用指定的 **仿真器** 编译RTL代码，并生成一个**DPI库文件**。该库文件内包含模拟运行RTL代码所需要的逻辑（即为**硬件模拟器**）。
    > *对于VCS，该库文件为.so（动态库）文件，对于Verilator，该库文件为.a（静态库）文件。*
    > *DPI的含义是 [Direct Programming Interface](https://www.chipverify.com/systemverilog/systemverilog-dpi)，可以理解为一种API规范。*

3. 接下来，Picker工具会根据配置参数，渲染源代码中定义的基类，生成用于对接仿真器并**隐藏仿真器细节**的基类（wrapper）。然后链接基类与DPI库文件，生成一个 **UT动态库文件**。
    > - *此时，该**UT库文件**使用了Picker工具模板中提供的统一API，相比于**DPI库文件**中与仿真器强相关的API，UT库文件为仿真器生成的硬件模拟器，提供了**统一的API接口**。*
    > - *截至这一步生成**UT库文件**在不同语言中是**通用**的！如果没有另行说明，其他高级语言均会通过**调用UT动态库**以实现对硬件模拟器的操作。*

4. 最后，Picker工具会根据配置参数和解析的RTL代码，生成一段 **C++ Class** 的源码。这段源码即是 RTL 硬件模块在软件中的**定义 (.hpp)** 及**实现 (.cpp)** 。实例化该类即相当于创建了一个硬件模块。
    > *该类继承自**基类**，并实现了基类中的纯虚函数，以用软件方式实例化硬件。*
    > *不将**类的实现**这一步也封装进动态库的原因有两点：*
    >   1. *由于**UT库文件**需要在不同语言中**通用**，而不同语言实现类的方式不同。为了通用性，不将`类的实现`封装进动态库。*
    >   2. *为了**便于调试**，提升代码可读性，方便用户进行二次封装和修改。*

### 生成可执行文件

在本章节中，我们将介绍如何基于上一章节生成的基础库（包含动态库，类的声明及定义），编写测试用例，生成可执行文件。

1. 首先，用户需要编写测试用例，即实例化上一章节生成的类，并调用类中的方法，以实现对硬件模块的操作。
详情可以参考[随机数生成器验证-配置测试代码](docs/quick-start/examples/rmg/#配置测试代码)中实例化及初始化的过程。

2. 其次，用户需要根据基础库所应用的**不同仿真器**，应用不同的链接参数以生成可执行文件。对应的参数在`template/cpp/cmake/*.cmake`中有定义。

3. 最终根据配置的链接参数，编译器会链接基础库，生成可执行文件。

    > 以 [加法器验证](docs/quick-start/examples/adder/#将rtl构建为c-class) 为例，`picker_out_adder/cpp/cmake/*.cmake`即是上述表项2所述模板的拷贝。
    > `vcs.cmake`定义了使用VCS仿真器生成的基础库的链接参数，`verilator.cmake`定义了使用Verilator仿真器生成的基础库的链接参数。

## 使用方案

- 参数 `--language cpp` 或 `-l cpp` 用于指定生成C++基础库。
- 参数 `-e` 用于生成包含示例项目的可执行文件。
- 参数 `-v` 用于保留生成项目时的中间文件。

```cpp
#include "UT_Adder.hpp"

int64_t random_int64()
{
    static std::random_device rd;
    static std::mt19937_64 generator(rd());
    static std::uniform_int_distribution<int64_t> distribution(INT64_MIN,
                                                            INT64_MAX);
    return distribution(generator);
}

int main()
{
#if defined(USE_VCS)
    UTAdder *dut = new UTAdder("libDPIAdder.so");
#elif defined(USE_VERILATOR)
    UTAdder *dut = new UTAdder();
#endif
    // dut->initClock(dut->clock);
    dut->xclk.Step(1);
    printf("Initialized UTAdder\n");

    struct input_t {
        uint64_t a;
        uint64_t b;
        uint64_t cin;
    };

    struct output_t {
        uint64_t sum;
        uint64_t cout;
    };

    for (int c = 0; c < 114514; c++) {
        input_t i;
        output_t o_dut, o_ref;

        i.a   = random_int64();
        i.b   = random_int64();
        i.cin = random_int64() & 1;

        auto dut_cal = [&]() {
            dut->a   = i.a;
            dut->b   = i.b;
            dut->cin = i.cin;
            dut->xclk.Step(1);
            o_dut.sum  = (uint64_t)dut->sum;
            o_dut.cout = (uint64_t)dut->cout;
        };

        auto ref_cal = [&]() {
            uint64_t sum = i.a + i.b;
            bool carry   = sum < i.a;

            sum += i.cin;
            carry = carry || sum < i.cin;

            o_ref.sum  = sum;
            o_ref.cout = carry ;
        };

        dut_cal();
        ref_cal();
        printf("[cycle %llu] a=0x%lx, b=0x%lx, cin=0x%lx\n", dut->xclk.clk, i.a,
            i.b, i.cin);
        printf("DUT: sum=0x%lx, cout=0x%lx\n", o_dut.sum, o_dut.cout);
        printf("REF: sum=0x%lx, cout=0x%lx\n", o_ref.sum, o_ref.cout);
        Assert(o_dut.sum == o_ref.sum, "sum mismatch");
    }

    delete dut;
    printf("Test Passed, destory UTAdder\n");
    return 0;
}
```