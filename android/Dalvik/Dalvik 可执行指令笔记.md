# Dalvik 可执行指令学习笔记

结合 [Dalvik 可执行指令格式](https://source.android.com/devices/tech/dalvik/instruction-formats) 和 [Dalvik 字节码](https://source.android.com/devices/tech/dalvik/dalvik-bytecode.html) 两个文档来学习

## 按位描述

指令格式**每个单元**由 2 个字节组成（16 位）并且按照空格分隔，单元内还可以用 `|` 进一步分割（最小 4 位）

例如，`B|A|op CCCC` 格式表示其包含两个 16 位代码单元。第一个单元由低 8 位中的操作码和高 8 位中的两个四位值组成；第二个单元由单个 16 位值组成

## ID

通常由两个十进制加一个字母组成，例如上面的 `20bc`。第一个十进制数表示格式中 16 位代码单元的数量。第二个十进制数表示格式包含的最大寄存器数量，特殊标识 `r` 表示已对寄存器的数量范围进行编码，最后一个字母以半助记符的形式表示该格式编码的任何其他数据类型。例如，`21t` 格式的长度为 2，包含一个寄存器引用，另外还有一个分支目标(助记符 t 的含义)。

## op

操作符 `op` 都是位于首个 16bit 数据的低 8bit，根据找到的 `op` 可以参照[Dalvik 字节码](https://source.android.com/devices/tech/dalvik/dalvik-bytecode.html)找到对应的 `Syntax` 和 `format id`

例如指令 `0x0062 0000`,首先得到 `op` 是62，语法为 `sget-object`，两个参数 A: 值寄存器或寄存器对；可以是源寄存器，也可以是目标寄存器（8 位） B: 静态字段引用索引（16 位），62 对应的 `format id` 是 `21c`, 即 `AA|op BBBB` ，含义即读取 0000 这个 field_id 的是值到 A 寄存器（0 号寄存器）
