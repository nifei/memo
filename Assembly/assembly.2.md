原文在这里: http://www.agner.org/optimize/
# 2 开始之前

## 2.1 开始编程之前要决定的事情

使用汇编开始编程之前, 你得好好想想为什么你要使用汇编语言, 你的程序的哪部分需要使用汇编来完成, 以及使用什么样的编程方法. 要是开发策略还不明了, 很快你就会发现自己在浪费时间, 优化程序中错误的部分, 使用汇编来干本来可以用c++干的事儿, 尝试优化不可能再优化的东西, 写出一堆一堆难以维护的代码, 或者充斥着错误而且无法调试的代码. 

下面列出了开始编程之前需要考虑的几件事: 

* Never make the whole program in assembly. That is a waste of time. Assembly code
should be used only where speed is critical and where a significant improvement in
speed can be obtained. Most of the program should be made in C or C++. These are
the programming languages that are most easily combined with assembly code.

+ 不要用汇编写整个程序. 那会浪费时间. 只应该在追求速度而且可以取得大幅速度提升的地方使用汇编代码. 要用c或者c++实现大部分的程序. 它们是最容易和汇编代码混合使用的语言了. 

* If the purpose of using assembly is to make system code or use special instructions
that are not available in standard C++ then you should isolate the part of the
program that needs these instructions in a separate function or class with a well
defined functionality. Use intrinsic functions (see p. 34) if possible.

+ 如果使用汇编的目的是完成系统代码或者使用某些标准c++不提供的指令, 应使用函数或者功能完备的类把汇编代码和程序的其他部分隔离开来. 可能的话使用伪指令函数(见34页)

* If the purpose of using assembly is to optimize for speed then you have to identify
the part of the program that consumes the most CPU time, possibly with the use of a
**profiler**. Check if the bottleneck is file access, memory access, CPU instructions, or
something else, as described in manual 1: "Optimizing software in C++". Isolate the
critical part of the program into a function or class with a well-defined functionality.

+ 如果使用汇编的目的是优化速度就得先定位出程序的哪一部分最消耗cpu时间, 可能得用**profiler** .看看瓶颈是访问文件, 访问内存, CPU指令还是别的什么, 就像手册1: "优化c++软件" 中讲的那样. 把程序的关键部分用函数或者功能完备的类隔离出来.  

* If the purpose of using assembly is to make a function library then you should clearly
define the functionality of the library. Decide whether to make a function library or a
class library. Decide whether to use static linking (.lib in Windows, .a in Linux) or
dynamic linking (.dll in Windows, .so in Linux). Static linking is usually more
efficient, but dynamic linking may be necessary if the library is called from languages
such as C# and Visual Basic. You may possibly make both a static and a dynamic
link version of the library.

+ 如果使用汇编是为了完成函数库, 要清楚地定义库的功能. 先定下是要完成函数库还是类库. 再定下是要使用静态链接(Windows的.lib, Linux的.a)还是动态链接(Windows的.dll, Linux的.so). 静态链接更高效, 但是从C#或者VB调用的话只能用动态链接库了. 可能动态和静态的版本都要事先. 

* If the purpose of using assembly is to optimize an embedded application for size or
speed then find a development tool that supports both C/C++ and assembly and
make as much as possible in C or C++.

+ 如果使用汇编是为了优化嵌入式应用的空间和速度, 找个支持C/C++和汇编的开发工具, 并且尽可能用C/C++实现. 

* Decide if the code is reusable or application-specific. Spending time on careful
optimization is more justified if the code is reusable. A reusable code is most
appropriately implemented as a function library or class library.

+ 代码是可重用的还是应用独有的. 用心优化可重用代码更合理. 可重用代码最合适的实现方式是函数库或者类库. 

* Decide if the code should support multithreading. A multithreading application can
take advantage of microprocessors with multiple cores. **Any data that must be
preserved from one function call to the next on a per-thread basis should be stored
in a C++ class or a per-thread buffer supplied by the calling program**.

+ 代码是否支持多线程. 多线程应用可以使用多核微处理器. **在线程之内需要保存以函数调用之间传递的数据应该由调用程序保存在C++类或者线程缓冲区内**

* Decide if portability is important for your application. Should the application work in
both Windows, Linux and Intel-based Mac OS? Should it work in both 32 bit and 64
bit mode? Should it work on non-x86 platforms? This is important for the choice of
compiler, assembler and programming method.

+ 可移植性对应用软件是否重要. 软件要在Windows, Linux和Intel-based Mac OS下都工作吗? 32位和64位模式都要支持吗? 非x86平台呢? 这些事情对编译器的选择, 汇编编译器的选择和编程方法都很重要. 

* Decide if your application should work on old microprocessors. If so, then you may
make one version for microprocessors with, for example, the SSE2 instruction set,
and another version which is compatible with old microprocessors. You may even
make several versions, each optimized for a particular CPU. It is recommended to
make **automatic CPU dispatching** (see page 137).

+ 软件是否要支持旧的微处理器? 如果是的话, 你可能要实现一个SSE2指令集之类的版本给微处理器用, 另一个版本用来和旧微处理器兼容. 甚至可以为每个特殊的CPU都实现一个优化版本. 建议使用**自动CPU分配**(见137页). 

* There are three assembly programming methods to choose between: (1) Use
intrinsic functions and vector classes in a C++ compiler. (2) Use inline assembly in a
C++ compiler. (3) Use an assembler. These three methods and their relative
advantages and disadvantages are described in chapter 5, 6 and 7 respectively
(page 34, 36 and 45 respectively).

+ 有三种汇编编程方法可选: (1) 在C++编译器中使用伪指令函数和向量类. (2) 在C++编译器中使用内联汇编. (3)使用汇编编译器. 这三种方法和它们的优缺点在第5, 6, 7章讲述(见34, 36, 45页). 

* If you are using an assembler then you have to choose between different syntax
dialects. It may be preferred to use an assembler that is compatible with the
assembly code that your C++ compiler can generate.

+ 使用汇编编译器的话还要在不同的语法中选择. (作者?)倾向使用和你的C++编译器生成的汇编代码兼容的汇编编译器. 

* Make your code in C++ first and optimize it as much as you can, using the methods
described in manual 1: "Optimizing software in C++". Make the compiler translate
the code to assembly. Look at the compiler-generated code and see if there are any
possibilities for improvement in the code.

+ 先使用C++实现并且尽可能用手册1: "优化C++软件"中介绍的方法优化它. 让编译器把代码翻译成汇编代码. 看看编译器生成的汇编代码, 看看你的(C++)代码是否还有改进余地. 

* Highly optimized code tends to be very difficult to read and understand for others
and even for yourself when you get back to it after some time. In order to make it
possible to maintain the code, it is important that you organize it into small logical
units (procedures or macros) with a well-defined interface and calling convention and
appropriate comments. Decide on a consistent strategy for code comments and
documentation.

+ 高度优化的代码对其他人来说是很难阅读和理解的,即使你自己过段时间回来看也这样. 为了让代码可维护, 把汇编代码组织成小一些的逻辑单位(过程或者宏), 遵循良好的接口定义和调用规范, 适当的注释都是很重要的. 制定一个可持续化的注释和文档策略. 

* Save the compiler, assembler and all other development tools together with the
source code and project files for later maintenance. Compatible tools may not be
available in a few years when updates and modifications in the code are needed.

+ 保存编译器, 汇编编译器和所有开发工具, 连同源代码和项目文件, 以备来日维护可用. 几年之后需要更新和修改代码的时候, 兼容的工具可能就没有了. (纳尼?)

## 2.2 Make a test strategy
Assembly code is error prone, difficult to debug, difficult to make in a clearly structured way,
difficult to read, and difficult to maintain, as I have already mentioned. A consistent test
strategy can ameliorate some of these problems and save you a lot of time.

## 2.2 制定测试策略

正如前文所述, 汇编代码爱出错, 难调试, 难写得条理清晰, 难读, 还难维护. 持续的测试策略可以改善这些问题, 节省很多时间. 

My recommendation is to make the assembly code as an isolated module, function, class or
library with a well-defined interface to the calling program. Make it all in C++ first. Then
make a test program which can test all aspects of the code you want to optimize. It is easier
and safer to use a test program than to test the module in the final application.

我的建议是让汇编代码自成一体, 模块, 方法, 类或者接口对调用者易用的库. 先全用C++实现. 然后写一个覆盖所有待优化代码的测试程序. 使用测试程序和在最终产品中测试模块相比更安全也更容易. 

The test program has two purposes. The first purpose is to verify that the assembly code
works correctly in all situations. And the second purpose is to test the speed of the
assembly code without invoking the user interface, file access and other parts of the final
application program that may make the speed measurements less accurate and less
reproducible.

测试程序有两个目的. 其一是验证汇编代码在所有分支下都正常工作. 其二是测试汇编代码的速度时, 可以不启动界面, 不访问文件, 也不调用应用程序的其他部分, 这样测试结果更准确并且可重现. 

You should use the test program repeatedly after each step in the development process and
after each modification of the code.

开发过程中的每一步, 每一次对代码的修改之后, 都应该使用测试程序. 

Make sure the test program works correctly. It is quite common to spend a lot of time
looking for an error in the code under test when in fact the error is in the test program.
There are different test methods that can be used for verifying that the code works correctly.
A white box test supplies a carefully chosen series of different sets of input data to make
sure that all branches, paths and special cases in the code are tested. A black box test
supplies a series of random input data and verifies that the output is correct. A very long
series of random data from a good random number generator can sometimes find rarely
occurring errors that the white box test hasn't found.

保证测试程序正确地工作. 花很多时间根据测试结果在代码里面找错误, 结果实际上是测试程序出错了这样的事情也不少. 

有各种测试方法可以用来验证代码是否工作正常. 白盒测试提供一系列精挑细选的输入数据集合保证代码中的各个分支路径和特例都会被测试到. 黑盒测试提供一系列随机输入数据并验证输出结果是否正确. 优秀并且足够长的随机生成机制有时可以发现白盒测试发现不了的极小概率错误.

The test program may compare the output of the assembly code with the output of a C++
implementation to verify that it is correct. The test should cover all boundary cases and
preferably also illegal input data to see if the code generates the correct error responses.
The speed test should supply a realistic set of input data. A significant part of the CPU time
may be spent on branch mispredictions in code that contains a lot of branches. The amount
of branch mispredictions depends on the degree of randomness in the input data. You may
experiment with the degree of randomness in the input data to see how much it influences
the computation time, and then decide on a realistic degree of randomness that matches a
typical real application.

测试程序可以通过对比汇编代码输出和C++代码输出来验证是否正确. 测试应覆盖所有边界条件并且最好有一些非法输入来验证代码生成的错误反馈是否正确. 

速度测试应提供真实输入数据. 在包含很多分支的代码中, CPU的时间很大一部分可能消耗在错误的分支预测上. 错误的分支预测数量取决于输入数据的随机程度. 你可以先试验一下输入数据的随机程度对计算时间的影响, 然后定下一个符合典型真实应用(场景)的随机度. 

An automatic test program that supplies a long stream of test data will typically find more
errors and find them much faster than testing the code in the final application. A good test
program will find most errors, but you cannot be sure that it finds all errors. It is possible that
some errors show up only in combination with the final application.

提供长测试数据流的自动化测试程序通常比在最终软件中测试更多更快地找到错误. 好的测试程序可以找到大部分错误, 但是你不能保证它找到所有错误. 有可能有些错误只有在最终产品中(模块)结合起来时才会出现. 

2.3 Common coding pitfalls
The following list points out some of the most common programming errors in assembly
code.
1. Forgetting to save registers. Some registers have callee-save status, for example
`EBX`. These registers must be saved in the prolog of a function and restored in the
epilog if they are modified inside the function. Remember that the order of `POP`
instructions must be the opposite of the order of `PUSH` instructions. See page 28 for a
list of callee-save registers.

## 2.3 常见编程陷阱
下面列出了一些汇编编程时最经常犯的错误.
1. 忘记保存寄存器. 有些寄存器保存调用者状态, 例如`EBX`. 如果函数中修改了这些寄存器的状态, 那要在函数开始时保存它们, 在退出前回复它们. 记住`POP`指令的顺序要和`PUSH`指令的顺序正好相反. 调用者会保存的寄存器列表见28页. 

2. Unmatched PUSH and POP instructions. The number of PUSH and POP instructions
must be equal for all possible paths through a function. Example:

Example 2.1. Unmatched push/pop
	<pre><code>
	push ebx
	test ecx, ecx
	jz Finished
	...
	pop ebx
	Finished: ; 错误! 标记要在pop ebx之前跳转. 
	ret
	</code></pre>
Here, the value of EBX that is pushed is not popped again if ECX is zero. The result is
that the RET instruction will pop the former value of EBX and jump to a wrong
address.

2. 不配对的`PUSH`和`POP`指令. 在方法的所有可能路径中, `PUSH`和`POP`指令的数目都必须相等. 例如:

例子2.1. 不配对的 push/pop
	<pre><code>
	push ebx
	test ecx, ecx
	jz Finished
	...
	pop ebx
	Finished: ; 错误! 标记要在pop ebx之前跳转. 
	ret
	</code></pre>
此处若`ECX`的值为0则被压栈的`EBX`没有出栈. 结果就是`RET`会弹出`EBX`之前的值从而跳到错误的地址去. 

3. Using a register that is reserved for another purpose. Some compilers reserve the
use of EBP or EBX for a **frame** pointer or other purpose. Using these registers for a
different purpose in inline assembly can cause errors.

3. 用了另有他用的寄存器. 有些编译器保留`EBX`或`EBP`用作**帧**指针或者其他用处. 在内联汇编中使用这些寄存器做别的事情可能会引起错误. 

4. Stack-relative addressing after push. When addressing a variable relative to the
stack pointer, you must take into account all preceding instructions that modify the
stack pointer. Example:
Example 2.2. Stack-relative addressing
	</code></pre>
		mov [esp+4], edi
		push ebp
		push ebx
		cmp esi, [esp+4] ; Probably wrong!
	</code></pre>
Here, the programmer probably intended to compare ESI with EDI, but the value of
ESP that is used for addressing has been changed by the two PUSH instructions, so
that ESI is in fact compared with EBP instead.

4. 压栈后的相对栈取址. 相对栈指针对变量取址时, 必须把之前修改栈指针的操作都考虑进去. 例如:

例 2.2. 相对栈取址
	</code></pre>
		mov [esp+4], edi
		push ebp
		push ebx
		cmp esi, [esp+4] ; 可能出错!
	</code></pre>
在此程序员可能意图比较`ESI`和`EDI`, 但是`ESP`寄存器的值已经被两次`PUSH`操作修改了, 所以`ESI`实际上在和`EBP`作比较. 

5. Confusing value and address of a variable. Example:
Example 2.3. Value versus address (MASM syntax)
	<pre><code>
	.data
	MyVariable DD 0 ; Define variable
	.code
	mov eax, MyVariable ; Gets value of MyVariable
	mov eax, offset MyVariable; Gets address of MyVariable
	lea eax, MyVariable ; Gets address of MyVariable
	mov ebx, [eax] ; Gets value of MyVariable through pointer
	mov ebx, [100] ; Gets the constant 100 despite brackets
	mov ebx, ds:[100] ; Gets value from address 100
	</code></pre>

5. 混淆变量的值和地址. 例如: 
例 2.3. 值与地址(MASM语法)
	<pre><code>
	.data
	MyVariable DD 0 ; 定义变量
	.code
	mov eax, MyVariable ; 读取变量MyVariable的值
	mov eax, offset MyVariable; 读取MyVariable的地址
	lea eax, MyVariable ; 读取MyVariable的地址
	mov ebx, [eax] ; 通过指针读取MyVariable的值
	mov ebx, [100] ; 忽略括号, 读取常量100
	mov ebx, ds:[100] ; 从地址100读取变量
	</code></pre>

6. Ignoring calling conventions. It is important to observe the calling conventions for
functions, such as the order of parameters, whether parameters are transferred on
the stack or in registers, and whether the stack is cleaned up by the caller or the
called function. See page 27.

6. 忽略调用惯例. 遵守调用惯例是很重要的, 比如参数的顺序, 参数是通过栈还是寄存器传递, 以及被调用者或者被调用函数有没有清空栈. 见27页. 

7. Function name **mangling**. A C++ code that calls an assembly function should use
extern "C" to avoid name mangling. Some systems require that an underscore (_)
is put in front of the name in the assembly code. See page 30.

7. 函数名称**压延**. C++代码调用汇编函数要使用extern "C"避免名字压延. 有些系统要求汇编函数名称前有下划线`_`. 见30页. 

8. Forgetting return. A function declaration must end with both RET and ENDP. Using
one of these is not enough. The execution will continue in the code after the
procedure if there is no RET.

8. 忘记返回. 函数声明必须以`RET`和`ENDP`结尾. 只使用一个是不够的. 没有`RET`的话过程调用之后操作会继续在代码中执行. 

9. Forgetting stack alignment. The stack pointer must point to an address divisible by
16 before any call statement, except in 16-bit systems and 32-bit Windows. See
page 27.

9. 忘记栈对齐. 在任何调用之前栈指针必须指向可以被16整除的地址, 除非在16位或者32位Windows下. 见27页. 

10. Forgetting **shadow space** in 64-bit Windows. It is required to reserve 32 bytes of
empty stack space before any function call in 64-bit Windows. See page 30.

10. 忘记64位Windows的**shadow space**. 在64位Windows下任何函数调用之前都要保留32 个字节的空栈控件. 见30页. 

11. Mixing calling conventions. The calling conventions in 64-bit Windows and 64-bit
Linux are different. See page 27.

11. 混用调用惯例. 64位Windows和64位Linux的调用惯例是不同的. 见27页. 

12. Forgetting to clean up floating point register stack. All floating point stack registers
that are used by a function must be cleared, typically with FSTP ST(0), before the
function returns, except for ST(0) if it is used for return value. It is necessary to keep
track of exactly how many floating point registers are in use. If a functions pushes
more values on the floating point register stack than it pops, then the register stack
will grow each time the function is called. An exception is generated when the stack
is full. This exception may occur somewhere else in the program.

12. 忘记清空浮点寄存器栈. 所有函数中用到的浮点栈寄存器都要被清空, 通常通过在返回前调用指令`FSTP ST(0)`来清空, 除了`ST(0)`, 如果它被用作存放返回值的话. 有必要记录有多少个浮点寄存器被使用. 如果一个函数压栈到浮点寄存器栈的次数比出栈的次数多, 寄存器栈就会每次函数调用都增长. 栈溢出的时候就会产生异常. 这个异常可能在程序其他地方冒出来. 

13. Forgetting to clear MMX state. A function that uses MMX registers must clear these
with the EMMS instruction before any call or return.

13. 忘记清空MMX状态. 使用MMx寄存器的函数需要用`EMMS`指令在调用或返回前清空它们. 

14. Forgetting to clear YMM state. A function that uses YMM registers must clear these
with the VZEROUPPER or VZEROALL instruction before any call or return.

14. 忘记清空YMM状态. 使用YMM寄存器的函数要在调用或返回前使用`VZEROUPPER`或`VZEROALL`清空它们. 

15. Forgetting to clear direction flag. Any function that sets the direction flag with STD
must clear it with CLD before any call or return.

15. 忘记清空方向标记位. 任何使用`STD`设置方向标记位的函数都要在调用或返回前使用`CLD`清空它. 

16. Mixing signed and unsigned integers. Unsigned integers are compared using the JB
and JA instructions. Signed integers are compared using the JL and JG instructions.
Mixing signed and unsigned integers can have unintended consequences.

16. 混用有符号和无符号整形. 无符号整形通过`JB`和`JA`来比较. 有符号整形通过`JL`和`JG`. 混用有符号和无符号整形数会有意想不到的后果. 

17. Forgetting to scale array index. An array index must be multiplied by the size of one
array element. For example mov eax, MyIntegerArray[ebx*4].

17. 忘记按比例缩放数组索引. 数组索引要乘以数组元素的大小. 例如`mov eax, MyIntegerArray[ebx*4]`. 

18. Exceeding array bounds. An array with n elements is indexed from 0 to n - 1, not
from 1 to n. A defective loop writing outside the bounds of an array can cause errors
elsewhere in the program that are hard to find.

18. 数组越界. n个元素的数组索引是0到n-1, 不是1到n. 有问题的循环在数组中越界写入, 会在程序中其他地方引起问题, 很难找到. 

19. Loop with ECX = 0. A loop that ends with the LOOP instruction will repeat 232 times if
ECX is zero. Be sure to check if ECX is zero before the loop.

19. 用`ECX=0`来做循环. `ECX`等于0的话以`LOOP`指令结尾的循环会被执行232次. 确保在循环调用之前`ECX`等于0. 

20. Reading carry flag after INC or DEC. The INC and DEC instructions do not change the
carry flag. Do not use instructions that read the carry flag, such as ADC, SBB, JC, JBE,
SETA, etc. after INC or DEC. Use ADD and SUB instead of INC and DEC to avoid this problem. 

20. 在`INC`或`DEC`之后读**carry**标记位. `INC`或`DEC`指令并不修改**carry** 标记位. 不要在调用它们之后使用诸如`ADC`, `SBB`, `JC`, `JBE`, `SETA`等指令读取**carry**标记位. 使用`ADD`和`SUB`代替`INC`和`DEC来避免这类问题. 