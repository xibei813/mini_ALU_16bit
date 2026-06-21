# 🔬 mini_ALU_16bit

<div align="center">

**一个从零手写的 16 位算术逻辑单元**

[![Verilog](https://img.shields.io/badge/Verilog-HDL-8B0000?style=flat-square&logo=verilog)](https://en.wikipedia.org/wiki/Verilog)
[![Course Project](https://img.shields.io/badge/Type-课程设计-blue?style=flat-square)]()
[![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)](LICENSE)

*所有操作数仅适用于无符号数*

</div>

---

## 📖 什么是 ALU？

ALU（Arithmetic Logic Unit）是 CPU 的运算核心。你写的每行代码最终都会变成 ALU 能理解的指令——加法、减法、逻辑与或非、移位、比较。**本项目用 Verilog 从最基本的逻辑门开始，一步步搭建了一个完整的 16 位 ALU。**

---

## 🏗 架构总览

```
                        ┌─────────────────────────────┐
  data0[15:0] ─────────┤                             ├──── result[31:0]
  data1[15:0] ─────────┤                             ├──── overflow
  OP[7:0]     ─────────┤      mini_ALU_16bit         ├──── valid
  num_shift[4:0] ──────┤        (顶层模块)            │
  clk / rst ───────────┤                             │
  div_start ───────────┤                             │
                        └──┬────┬────┬────┬──────────┘
                           │    │    │    │
                    ┌──────┘    │    │    └──────┐
                    ▼           ▼    ▼           ▼
                 ADD模块    SUB模块 MUL模块   DIV模块
              (行波进位)  (补码减法) (部分积) (恢复余数)
```

### 模块职责

| 模块 | 文件 | 核心算法 | 输出位宽 |
|------|------|----------|----------|
| **顶层** | `mini_ALU_16bit.v` | 操作码译码 + 结果选择 | 32-bit |
| **加法器** | `mini_ALU_16bit_ADD.v` | 行波进位 (Ripple Carry) | 16-bit |
| **减法器** | `mini_ALU_16bit_SUB.v` | 二进制补码 (`~b + 1`) | 16-bit |
| **乘法器** | `mini_ALU_16bit_MUL.v` | 部分积累加（手算模拟） | 32-bit |
| **除法器** | `mini_ALU_16bit_DIV.v` | 恢复余数法 + 状态机 | 16-bit 商 + 16-bit 余 |

---

## 🔧 支持的 15 种操作

### 算术运算

| 操作码 | 指令 | 功能 | 公式 |
|--------|------|------|------|
| `0000_0001` | **ADD** | 加法 | `result = data0 + data1` |
| `0000_0010` | **SUB** | 减法 | `result = data0 - data1` |
| `0000_0011` | **MUL** | 乘法 | `result = data0 × data1` |
| `0000_0100` | **DIV** | 除法 | `result = {商, 余数}` |

### 逻辑运算

| 操作码 | 指令 | 功能 | 真值表口诀 |
|--------|------|------|-----------|
| `0000_0101` | **AND** | 按位与 | 有 0 则 0 |
| `0000_0110` | **OR** | 按位或 | 有 1 则 1 |
| `0000_0111` | **XOR** | 按位异或 | 相异为 1 |
| `0000_1000` | **both_NOT** | 两数取反拼接 | `{~data1, ~data0}` |
| `0000_1001` | **data0_NOT** | data0 按位取反 | `~data0` |
| `0000_1010` | **data1_NOT** | data1 按位取反 | `~data1` |

### 移位运算

| 操作码 | 指令 | 移位量由 `num_shift` 决定 |
|--------|------|--------------------------|
| `0000_1011` | **data0_SHIFT_LEFT** | data0 左移 |
| `0000_1100` | **data1_SHIFT_RIGHT** | data1 右移 |
| `0000_1101` | **data0_SHIFT_RIGHT** | data0 右移 |
| `0000_1110` | **data1_SHIFT_LEFT** | data1 左移 |

### 比较运算

| 操作码 | 指令 | 返回值 |
|--------|------|--------|
| `0000_1111` | **CMP** | `1` = data0 > data1, `2` = data0 < data1, `3` = 相等 |

---

## 🧠 深入各模块

### 1. 加法器 — 行波进位 (Ripple Carry)

```
  data0[15]  data1[15]         data0[1]  data1[1]         data0[0]  data1[0]
      │          │                 │         │                 │         │
   ┌──┴──────────┴──┐         ┌──┴─────────┴──┐          ┌──┴─────────┴──┐
   │   Full Adder   │◄────···──│   Full Adder  │◄─────────│   Full Adder  │◄─ 0
   └──────┬──────┬───┘         └─────┬─────┬────┘          └─────┬─────┬────┘
          │      │                   │     │                     │     │
       sum[15]  cout              sum[1] carry[0]            sum[0] carry

   全加器真值表:           全加器逻辑:
   ┌───┬───┬───┬───┬───┐   sum  = a ⊕ b ⊕ cin
   │ a │ b │cin│sum│cou│   cout = (a·b) + (a·cin) + (b·cin)
   ├───┼───┼───┼───┼───┤
   │ 0 │ 0 │ 0 │ 0 │ 0 │
   │ 0 │ 0 │ 1 │ 1 │ 0 │
   │ 0 │ 1 │ 0 │ 1 │ 0 │
   │ 0 │ 1 │ 1 │ 0 │ 1 │   ⚠️ 串行进位意味着必须等低位算出进位，
   │ 1 │ 0 │ 0 │ 1 │ 0 │      高位才能开始计算——这是性能瓶颈所在。
   │ 1 │ 0 │ 1 │ 0 │ 1 │      工业界用超前进位(Carry Lookahead)解决。
   │ 1 │ 1 │ 0 │ 0 │ 1 │
   │ 1 │ 1 │ 1 │ 1 │ 1 │
   └───┴───┴───┴───┴───┘
```

### 2. 减法器 — 补码减法

一个优雅的数学事实：**减去一个数 = 加上它的补码**

```
  data0 - data1  =  data0 + (~data1 + 1)
```

实现极简：复用加法器，对 data1 取反加一后相加。若 `data0 < data1`，结果取绝对值并将 `overflow` 置 1。

### 3. 乘法器 — 手算模拟

乘法本质是「移位加」。手算 `1011 × 1101` 的过程：

```
          1 0 1 1    (被乘数 data0)
        × 1 1 0 1    (乘数 data1)
        ─────────
          1 0 1 1    ← data0 × 1，不移位
        0 0 0 0      ← data0 × 0，左移 1 位
      1 0 1 1        ← data0 × 1，左移 2 位
    1 0 1 1          ← data0 × 1，左移 3 位
  ─────────────────
  1 0 0 0 1 1 1 1    ← 16 个部分积逐个累加
```

代码中 16 个 `partial_product` 就是这 16 行，然后通过 **15 个 31-bit 串行加法器**逐个相加。最后的进位 `temp_cout[14]` 与和拼接形成 32-bit 结果。

> 工业优化方向：**Booth 编码**减少部分积数量 → **Wallace Tree** 压缩器并行求和。

### 4. 除法器 — 恢复余数法 + 状态机

除法器是唯一需要多个时钟周期的模块，使用两段式状态机：

```
              ┌─────────┐
     start=0  │         │  start=1
    ┌─────────┤  IDLE   ├──────────┐
    │         │         │          │
    │         └─────────┘          ▼
    │                          ┌─────────┐
    │         计数 < 16         │         │
    └──────────────────────────┤  START  │
               计数 = 16       │         │
              (自动返回 IDLE)   └─────────┘
```

**恢复余数算法**核心循环（每次）：

```
  1. 余数拼接当前被除数位 → 左移 1 位
  2. 试减除数
  3. 够减 → 商上 1，保留新余数
     不够 → 商上 0，恢复旧余数
  4. 重复 16 次
```

耗时 16 个时钟周期（4-bit 计数器 `&count` 检测全 1 即完成）。除数 = 0 时 `overflow` 置 1。

---

## 📡 接口定义

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `data0` | input | 16 | 操作数 A |
| `data1` | input | 16 | 操作数 B |
| `OP` | input | 8 | 操作码（仅低 4 位有效） |
| `num_shift` | input | 5 | 移位量（仅移位指令有效） |
| `clk` | input | 1 | 时钟 |
| `rst` | input | 1 | 异步复位（低有效） |
| `div_start` | input | 1 | 除法启动脉冲 |
| `result` | output | 32 | 运算结果 |
| `overflow` | output | 1 | 溢出/异常标志 |
| `valid` | output | 1 | 结果有效标志 |

---

## ⚡ 仿真波形

`wave_picture/` 目录下包含所有操作的仿真截图，以下是关键波形说明：

| 截图 | 测试内容 |
|------|----------|
| `ADD.png` / `ADD_alu.png` | 加法及顶层验证 |
| `SUB.png` / `SUB_alu.png` | 减法（含小减大取绝对值） |
| `MUL.png` / `MUL_alu.png` | 乘法部分积累加 |
| `div*.png` | 除法：16 周期时序、除零检测、边界用例 |
| `and/OR/xor_alu.png` | 逻辑运算 |
| `cmp_alu.png` | 三值比较 |
| `data0_right.png` 等 | 移位运算验证 |

---

## 📁 项目结构

```
mini_ALU_16bit/
├── source/                          # Verilog 源码
│   ├── mini_ALU_16bit.v             # 顶层模块
│   ├── mini_ALU_16bit_ADD.v         # 加法器 + 全加器
│   ├── mini_ALU_16bit_SUB.v         # 减法器（补码）
│   ├── mini_ALU_16bit_MUL.v         # 乘法器（部分积）
│   └── mini_ALU_16bit_DIV.v         # 除法器（恢复余数）
├── testbench/                       # 测试平台
│   ├── mini_ALU_16bit_tb.v          # 顶层测试
│   ├── mini_ALU_16bit_ADD_tb.v      # 加法测试
│   ├── mini_ALU_16bit_SUB_tb.v      # 减法测试
│   ├── mini_ALU_16bit_MUL_tb.v      # 乘法测试
│   └── mini_ALU_16bit_DIV_tb.v      # 除法测试
├── wave_picture/                    # 仿真波形截图
│   ├── ADD*.png, SUB*.png
│   ├── MUL*.png, div*.png
│   └── *_alu.png, *.png
└── README.md
```

---

## 🎯 已知局限与改进方向

| 局限 | 说明 | 改进方案 |
|------|------|----------|
| 仅支持无符号数 | 无法处理负数 | 添加符号位检测 + 补码转换 |
| 行波进位 | 16 级门延迟，速度慢 | 超前进位加法器 (CLA) |
| 乘法链式累加 | 15 个串行加法器，面积大 | Booth 编码 + Wallace Tree |
| 除法固定 16 周期 | 即使商已确定也要跑完 | 提前终止检测 |
| 无流水线 | 一条指令未完成不能开始下一条 | 流水线分级 |
| 无时序约束 | 未做时序分析 | SDC 约束 + STA |

---

## 🚀 快速开始

### 仿真环境

任一 Verilog 仿真工具均可运行，例如：

```bash
# Icarus Verilog (开源)
iverilog -o alu_tb source/*.v testbench/mini_ALU_16bit_tb.v
vvp alu_tb

# 或用 GTKWave 查看波形
gtkwave dump.vcd
```

### 开发环境

- **Vivado / Quartus**：直接添加 `source/` 下所有 `.v` 文件
- **Verilator**：支持 lint 检查和 SystemC 协同仿真

---

## 📚 学习路线建议

1. 先读懂 `full_adder` → 理解组合逻辑的本质
2. 再看 `mini_ALU_16bit_ADD` → 理解 `generate` 语句和模块复用
3. 对比 `SUB` 和 `ADD` → 理解补码的妙处
4. 手算一遍 MUL 的部分积 → 理解硬件乘法的原理
5. 画出 DIV 的状态转移图 → 理解时序逻辑
6. 最后看顶层 `case` 分支 → 理解指令译码

---

<div align="center">

**从 AND 门到 ALU，从 ALU 到 CPU——这是计算机最浪漫的递进 🚀**

made with ❤️ by [xibei813](https://github.com/xibei813) | 2024 课程设计

</div>
