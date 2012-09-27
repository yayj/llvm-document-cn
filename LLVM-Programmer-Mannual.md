# LLVM开发手册

## 简介

This document is meant to highlight some of the important classes and interfaces available in the LLVM source-base. This manual is not intended to explain what LLVM is, how it works, and what LLVM code looks like. It assumes that you know the basics of LLVM and are interested in writing transformations or otherwise analyzing or manipulating the code.

这篇文档的目的不是阐述LLVM是什么、LLVM的工作方式以及LLVM的源代码，而是介绍LLVM所提供的重要的类和接口。在开始之前，我们假设读者有一定的LLVM基础知识，并有兴趣去分析和操纵LLVM的源码，甚至去改进它。

This document should get you oriented so that you can find your way in the continuously growing source code that makes up the LLVM infrastructure. Note that this manual is not intended to serve as a replacement for reading the source code, so if you think there should be a method in one of these classes to do something, but it's not listed, check the source. Links to the [doxygen]("http://llvm.org/doxygen/") sources are provided to make this as easy as possible.

千万不要认为阅读该文档后就了解LLVM的源码了，因为这篇文档目的只是让读者有一个逐步了解LLVM源码的方向，而不是详细手册。如果读者确信某个类中应该有一个实现某项功能的方法，而该方法并没有在文档中列出，那么应该去源码中确认。我们也提供了[Doxygen]("http://llvm.org/doxygen/")文档，这使读者阅读起来更加方便。

The first section of this document describes general information that is useful to know when working in the LLVM infrastructure, and the second describes the Core LLVM classes. In the future this manual will be extended with information describing how to use extension libraries, such as dominator information, CFG traversal routines, and useful utilities like the [InstVisitor]("http://llvm.org/doxygen/InstVisitor_8h-source.html") template.

文档的第一部分描述了一些在LLVM开发相关的信息，第二部分描述了一些LLVM的关键类。将来，我们还会加入如何使用一些扩展库，例如处理必经结点信息、遍历控制流图，以及介绍一些像[InstVisitor]("http://llvm.org/doxygen/InstVisitor_8h-source.html")模板的实用工具。

## 基本信息

This section contains general information that is useful if you are working in the LLVM source-base, but that isn't specific to any particular API.

这部分介绍了LLVM开发相关的信息，但这不是

### 标准模板库(STL)

LLVM makes heavy use of the C++ Standard Template Library (STL), perhaps much more than you are used to, or have seen before. Because of this, you might want to do a little background reading in the techniques used and capabilities of the library. There are many good pages that discuss the STL, and several books on the subject that you can get, so it will not be discussed in this document.

LLVM大量使用了C++的标准模板库，其范围可能超出你的想象。正因为如此，你可能需要一些STL的背景知识。现在已经有许多网页都有关于STL的讨论，也有一些介绍STL的书，所以这里就不复述了。

Here are some useful links:

给出一些关于STL的链接：

1. Dinkumware C++ Library reference - an excellent reference for the STL and other parts of the standard C++ library.
1. C++ In a Nutshell - This is an O'Reilly book in the making. It has a decent Standard Library Reference that rivals Dinkumware's, and is unfortunately no longer free since the book has been published.
1. C++ Frequently Asked Questions
SGI's STL Programmer's Guide - Contains a useful Introduction to the STL.
1. Bjarne Stroustrup's C++ Page
1. Bruce Eckel's Thinking in C++, 2nd ed. Volume 2 Revision 4.0 (even better, get the book).

同时，如果能阅读[LLVM编码规范]("")就更好了，该文档阐述了应该如何写出具有可维护性的代码，而不仅仅是告诉你大括号应该摆在哪儿。

### 其它引用

1. Using static and shared libraries across platforms

## LLVM APIs

Here we highlight some LLVM APIs that are generally useful and good to know about when writing transformations.

### isa<>, cast<>和dyn_cast<>模板

The LLVM source-base makes extensive use of a custom form of RTTI. These templates have many similarities to the C++ dynamic_cast<> operator, but they don't have some drawbacks (primarily stemming from the fact that dynamic_cast<> only works on classes that have a v-table). Because they are used so often, you must know what they do and how they work. All of these templates are defined in the [llvm/Support/Casting.h]("http://llvm.org/doxygen/Casting_8h-source.html") file (note that you very rarely have to include this file directly).

LLVM有一些自定义模板来实现运行时类型识别(RTTI)，并在源码中广泛使用。这些模板跟C++的dynamic_cast<>操作符很相似，但却没有它的一些缺点(dynamic_cast<>只能对有v-table的类使用)。所以，了解这些模板并清楚它们的工作原理是非常有必要的。所有的模板的定义都在文件[llvm/Support/Casting.h]("http://llvm.org/doxygen/Casting_8h-source.html")中，不过你通常不用直接include它。

* isa<>:
The isa<> operator works exactly like the Java "instanceof" operator. It returns true or false depending on whether a reference or pointer points to an instance of the specified class. This can be very useful for constraint checking of various sorts (example below).

* **isa<>**: 和Java语言里的instanceof操作符一样，isa<>用于判断一个引用或指针的类型是否和给出的类型一致，这在需要准确判断类型时非常有用。

cast<>:
The cast<> operator is a "checked cast" operation. It converts a pointer or reference from a base class to a derived class, causing an assertion failure if it is not really an instance of the right type. This should be used in cases where you have some information that makes you believe that something is of the right type. An example of the isa<> and cast<> template is:

* **cast<>**: cast<>把基类的指针或引用转换成继承类的指针或引用，若类型不匹配，则导致断言错误，所以它主要用在"先检查后转换"的操作当中。

		static bool isLoopInvariant(const Value *V, const Loop *L) {
		  if (isa<Constant>(V) || isa<Argument>(V) ||isa<GlobalValue>(V))
		    return true;
   
		  // Otherwise, it must be an instruction...
		  return !L->contains(cast<Instruction>(V)->getParent());
		}
Note that you should not use an isa<> test followed by a cast<>, for that use the dyn_cast<> operator.  
**注意**: 不能在cast<>后才用isa<>去检查类型，这种情况应当使用dyn_cast<>。

* dyn_cast<>:
The dyn_cast<> operator is a "checking cast" operation. It checks to see if the operand is of the specified type, and if so, returns a pointer to it (this operator does not work with references). If the operand is not of the correct type, a null pointer is returned. Thus, this works very much like the dynamic_cast<> operator in C++, and should be used in the same circumstances. Typically, the dyn_cast<> operator is used in an if statement or some other flow control statement like this:

* dyn_cast<>: dyn_cast<>是一个带检查的类型转换，它先检查操作数的类型是否匹配，若匹配则返回类型转换后的指针(该操作符不能用于引用)，否则返回空指针。由此可见，它和C++中的dynamic_cast<>非常像，可以用于相同的场景。通常来说，dyn_cast<>一般用在类似于以下例子中的流程控制语句中：

		if (AllocationInst *AI = dyn_cast<AllocationInst>(Val)) {
		  // ...
		}
		This form of the if statement effectively combines together a call to isa<> and a call to cast<> into one statement, which is very convenient.
Note that the dyn_cast<> operator, like C++'s dynamic_cast<> or Java's instanceof operator, can be abused. In particular, you should not use big chained if/then/else blocks to check for lots of different variants of classes. If you find yourself wanting to do this, it is much cleaner and more efficient to use the InstVisitor class to dispatch over the instruction type directly.  
**注意**: dyn_cast<>可能会像C++的dynamic_cast<>和Java中的instanceof一样被滥用。通常来说，我们不应该用一长串if/then/else来进行变量的类型判断。如果非如此不可，那么使用InstVisitor类来实现__TBD__

cast_or_null<>:
The cast_or_null<> operator works just like the cast<> operator, except that it allows for a null pointer as an argument (which it then propagates). This can sometimes be useful, allowing you to combine several null checks into one.

* **cast_or_null<>**: cast_or_null<>和cast<>很相似，但它可以允许使用空指针作为参数。这个特性可以把多个空指针检查合并到一句里。

dyn_cast_or_null<>:
The dyn_cast_or_null<> operator works just like the dyn_cast<> operator, except that it allows for a null pointer as an argument (which it then propagates). This can sometimes be useful, allowing you to combine several null checks into one.

* **dyn_cast_or_null<>**: dyn_cast_or_null<>和dyn_cast<>类似，但它允许使用空指针作为参数。这个特性可以把多个空指针检查合并到一句里。

These five templates can be used with any classes, whether they have a v-table or not. To add support for these templates, you simply need to add classof static methods to the class you are interested casting to. Describing this is currently outside the scope of this document, but there are lots of examples in the LLVM source base.

这五个模板可以与任意类一起使用，无论它们是否有v-table。为了支持这些模板，类必须要实现几个名为classof的静态方法，具体的实现方式已经超出了本文的范围，不过LLVM源码中有大量classof的实例。

### 字符串传递(StringRef和Twine)Passing strings (the StringRef and Twine classes)

Although LLVM generally does not do much string manipulation, we do have several important APIs which take strings. Two important examples are the Value class -- which has names for instructions, functions, etc. -- and the StringMap class which is used extensively in LLVM and Clang.

虽然LLVM通常不会做太多的字符串操作，但确实有一些重要的API使用了字符串。以两个被大量使用的类作为例子：用于保存指令名、函数名的Value类；在LLVM和Clang中大量使用的StrinngMap类。

These are generic classes, and they need to be able to accept strings which may have embedded null characters. Therefore, they cannot simply take a const char *, and taking a const std::string& requires clients to perform a heap allocation which is usually unnecessary. Instead, many LLVM APIs use a StringRef or a const Twine& for passing strings efficiently.

这些API都是通用类，而且要求它们要有能力处理那些包含空字符的串。因此，简单地使用const char *是不能满足需求的，而让调用者自己在堆上分配空间，并用const std::string&作为参数类型传递字符串的方式也是不必要的。于是，LLVM使用StringRef或const Twine&来实现高效的字符串传递。

#### StringRef

The StringRef data type represents a reference to a constant string (a character array and a length) and supports the common operations available on std:string, but does not require heap allocation.

StringRef表示一个常量字符串(一串字符和它的长度)的引用，它支持std::string类的常用操作，但却不需要从堆上分配空间。

It can be implicitly constructed using a C style null-terminated string, an std::string, or explicitly with a character pointer and length. For example, the StringRef find function is declared as:

可以使用C风格的0结尾字符串或std::string来隐式创建一个StringRef，也用一个字符指针和长度来显式创建。例如，假设这是查找字符串的函数的声明：

`iterator find(StringRef Key);`

and clients can call it using any one of:

那么调用者可以用以下形式调用它：

```
Map.find("foo");                 // 查找"foo"
Map.find(std::string("bar"));    // 查找"bar"
Map.find(StringRef("\0baz", 4)); // 查找"\0baz"
```
Similarly, APIs which need to return a string may return a StringRef instance, which can be used directly or converted to an std::string using the str member function. See "[llvm/ADT/StringRef.h]("http://llvm.org/doxygen/classllvm_1_1StringRef_8h-source.html")" for more information.

与此类似，当需要返回字符串时，可以返回StringRef实例，调用者可以直接使用StringRef实例，也可以调用它的str函数来转成std::string。更多信息可以查看"[llvm/ADT/StringRef.h]("http://llvm.org/doxygen/classllvm_1_1StringRef_8h-source.html")"。

You should rarely use the StringRef class directly, because it contains pointers to external memory it is not generally safe to store an instance of the class (unless you know that the external storage will not be freed). StringRef is small and pervasive enough in LLVM that it should always be passed by value.

实际上我们很少直接使用StringRef，因为它包含了一些指向类外部的指针，这类指针通常情况下是不安全的，除非你确定这些指针所指的外部空间不会被释放(如指向常量字符串)。StringRef非常小巧，在LLVM里的使用也很广泛。StringRef总是使用传值方式传递。

#### Twine

The Twine class is an efficient way for APIs to accept concatenated strings. For example, a common LLVM paradigm is to name one instruction based on the name of another instruction with a suffix, for example:

Twine类可以高效地完成多个字符串的连接。举例来说，LLVM里通常会在一个指令名后面追加另一个指令名作为后缀，比如

`New = CmpInst::Create(..., SO->getName() + ".cmp");`

The Twine class is effectively a lightweight rope which points to temporary (stack allocated) objects. Twines can be implicitly constructed as the result of the plus operator applied to strings (i.e., a C strings, an std::string, or a StringRef). The twine delays the actual concatenation of strings until it is actually required, at which point it can be efficiently rendered directly into a character array. This avoids unnecessary heap allocation involved in constructing the temporary results of string concatenation. See "llvm/ADT/Twine.h" and here for more information.

Twine的原理是把多个临时对象(栈对象)捆绑起来，这是一个高效且轻量级的实现。用+把多个字符串(例如c风格、std::string或StringRef)连接起来后，即可隐式地创建Twine对象。Twine实现了"需时连接"，即仅在需要的时候才把多个串组合成一个。这样可以避免连接过程中的那些不必要的堆空间分配。查看[llvm/ADT/Twine.h]("http://llvm.org/doxygen/Twine_8h_source.html")和[这里]("http://localhost/~matt/llvm/ProgrammersManual.html#dss_twine")以查看更多信息。

As with a StringRef, Twine objects point to external memory and should almost never be stored or mentioned directly. They are intended solely for use when defining a function which should be able to efficiently accept concatenated strings.

和StringRef一样，Twine对象也有指向外部的指针，所以一般来说不要保存或直接引用，而仅当在函数参数是多个字符串连接而成时，才用作参数的类型声明。

### DEBUG()宏和-debug选项

Often when working on your pass you will put a bunch of debugging printouts and other code into your pass. After you get it working, you want to remove it, but you may need it again in the future (to work out new bugs that you run across).

在开发过程中，我们会根据情况反复地开启和关闭一些调试信息输出，

在调试一段代码时可能需要在代码中加入一些语句来打印调试信息，而调试结束后则可以不用输出它们了，不过在将来可能这

Naturally, because of this, you don't want to delete the debug printouts, but you don't want them to always be noisy. A standard compromise is to comment them out, allowing you to enable them if you need them in the future.

所以，我们希望让这些调试信息呼之则来，挥之则去。一个标准的折中方案是在代码中把它们注释掉，将来需要时再恢复回来。

The "llvm/Support/Debug.h" file provides a macro named DEBUG() that is a much nicer solution to this problem. Basically, you can put arbitrary code into the argument of the DEBUG macro, and it is only executed if 'opt' (or any other tool) is run with the '-debug' command line argument:

基于这种尴尬的局面，[llvm/Support/Debug.h]("http://localhost/doxygen/Debug_8h-source.html")给出了DEBUG()宏，它可以很好地解决上述问题。你几乎可以把任何代码塞给它，这些代码是否执行，取决于程序运行时命令行上是否有-debug参数。

`DEBUG(errs() << "I am here!\n");`

Then you can run your pass like this:

这样，可以这样运行程序：

```
$ opt < a.bc > /dev/null -mypass
<no output>
$ opt < a.bc > /dev/null -mypass -debug
I am here!
```
Using the DEBUG() macro instead of a home-brewed solution allows you to not have to create "yet another" command line option for the debug output for your pass. Note that DEBUG() macros are disabled for optimized builds, so they do not cause a performance impact at all (for the same reason, they should also not contain side-effects!).

用DEBUG()宏取代其它的方式，则可以不用在命令行上使用其它参数来控制输出。值得注意的是，使用优化参数编译程序时，DEBUG()宏会被禁用，所以完全不用担心它会影响程序性能(同样，它们也不会导致[副效应]("http://en.wikipedia.org/wiki/Side_effect_(computer_science)"))。

One additional nice thing about the DEBUG() macro is that you can enable or disable it directly in gdb. Just use "set DebugFlag=0" or "set DebugFlag=1" from the gdb if the program is running. If the program hasn't been started yet, you can always just run it with -debug.

DEBUG()宏的另一个好处是，在gdb中运行程序时，可以直接使用"set DebugFlag=0"或"set DebugFlag=1"来开启或关闭。如果没有运行时，使用-debug参数也可以开启调试信息。

#### 用DEBUG_TYPE和-debug-only选项整理调试信息

Sometimes you may find yourself in a situation where enabling -debug just turns on too much information (such as when working on the code generator). If you want to enable debug information with more fine-grained control, you define the DEBUG_TYPE macro and the -debug only option as follows:

有时候我们发现开启-debug选项后，会产生过多的调试信息(例如在开发代码生成器时)。如果使用DEBUG_TYPE宏和-debug-only选项对调试信息进行分类，则可以很好地整理这些调试信息了：

```
#undef  DEBUG_TYPE
DEBUG(errs() << "No debug type\n");
#define DEBUG_TYPE "foo"
DEBUG(errs() << "'foo' debug type\n");
#undef  DEBUG_TYPE
#define DEBUG_TYPE "bar"
DEBUG(errs() << "'bar' debug type\n"));
#undef  DEBUG_TYPE
#define DEBUG_TYPE ""
DEBUG(errs() << "No debug type (2)\n");
```
Then you can run your pass like this:

之后可以这样运行程序：

```
$ opt < a.bc > /dev/null -mypass
<no output>
$ opt < a.bc > /dev/null -mypass -debug
No debug type
'foo' debug type
'bar' debug type
No debug type (2)
$ opt < a.bc > /dev/null -mypass -debug-only=foo
'foo' debug type
$ opt < a.bc > /dev/null -mypass -debug-only=bar
'bar' debug type
```
Of course, in practice, you should only set DEBUG_TYPE at the top of a file, to specify the debug type for the entire module (if you do this before you #include "llvm/Support/Debug.h", you don't have to insert the ugly #undef's). Also, you should use names more meaningful than "foo" and "bar", because there is no system in place to ensure that names do not conflict. If two different modules use the same string, they will all be turned on when the name is specified. This allows, for example, all debug information for instruction scheduling to be enabled with -debug-type=InstrSched, even if the source lives in multiple files.

当然，在实际操作当中，只需要在文件开始设置DEBUG_TYPE宏即可，这样可以让它对整个模块都生效(前提是要把它放在#include "llvm/Support/Debug.h"之前)。由于LLVM没有为DEBUG_TYPE提供冲突检测机制，所以我们应该使用比"foo"或"bar"更有意义的分类名。如果两个不同的模块使用了相同的分类名，则它们的调试信息会同时输出。...

The DEBUG_WITH_TYPE macro is also available for situations where you would like to set DEBUG_TYPE, but only for one specific DEBUG statement. It takes an additional first parameter, which is the type to use. For example, the preceding example could be written as:

如果只想为单独的一句设置分类名，则可以使用DEBUG_WITH_TYPE宏，该宏的第一个参数即为分类名。上一个例子可以用该宏简化成如下：

```
DEBUG_WITH_TYPE("", errs() << "No debug type\n");
DEBUG_WITH_TYPE("foo", errs() << "'foo' debug type\n");
DEBUG_WITH_TYPE("bar", errs() << "'bar' debug type\n"));
DEBUG_WITH_TYPE("", errs() << "No debug type (2)\n");
```

### Statistic类和-stats选项

The "[llvm/ADT/Statistic.h]("http://llvm.org/doxygen/Statistic_8h-source.html")" file provides a class named Statistic that is used as a unified way to keep track of what the LLVM compiler is doing and how effective various optimizations are. It is useful to see what optimizations are contributing to making a particular program run faster.

"[llvm/ADT/Statistic.h]("http://llvm.org/doxygen/Statistic_8h-source.html")"声明了Statistic类，它提供了一种统一的方式对LLVM编译过程和优化工作进行统计。通过Statistic类可以方便地看到编译器对程序做了哪些优化。

Often you may run your pass on some big program, and you're interested to see how many times it makes a certain transformation. Although you can do this with hand inspection, or some ad-hoc method, this is a real pain and not very useful for big programs. Using the Statistic class makes it very easy to keep track of this information, and the calculated information is presented in a uniform manner with the rest of the passes being executed.

通常来说，我们可能只是开发一个大的程序中的某个部件，那么在开发过程中，我们会关心对自己部件做了多少改进。虽然可以使用手动检查或自定义方法等方式来观察，但这个过程是很繁琐的，而且对较大的程序来说也不实用。而Statistic类可以简化这个过程，并提供一个统一的方式...

There are many examples of Statistic uses, but the basics of using it are as follows:

Statistic的例子非常多，不过最基本的用法是这样的：

1. 用如下方式自定义Statistic:

		#define DEBUG_TYPE "mypassname"   // This goes before any #include		s.
		STATISTIC(NumXForms, "The # of times I did stuff
The STATISTIC macro defines a static variable, whose name is specified by the first argument. The pass name is taken from the DEBUG_TYPE macro, and the description is taken from the second argument. The variable defined ("NumXForms" in this case) acts like an unsigned integer.  
DEBUG_TYPE宏定义了调试信息的分类，STATISTIC宏的第一个参数定义了一个静态变量，第二个参数是统计描述信息。静态变量(本例中的NumXForms)的类型是无符号整数。

1. Whenever you make a transformation, bump the counter:
1. 在任何有改动的地方都增加计数：

`++NumXForms;   // I did stuff!`

That's all you have to do. To get 'opt' to print out the statistics gathered, use the '-stats' option:

搞定。使用-stats选项，让程序输出收集到的统计信息：

```
$ opt -stats -mypassname < program.bc > /dev/null
... statistics output ...
```
When running opt on a C file from the SPEC benchmark suite, it gives a report that looks like this:

对一个SPEC benchmark suite里的C文件进行编译，结果如下：

```
   7646 bitcodewriter   - Number of normal instructions
    725 bitcodewriter   - Number of oversized instructions
 129996 bitcodewriter   - Number of bitcode bytes written
   2817 raise           - Number of insts DCEd or constprop'd
   3213 raise           - Number of cast-of-self removed
   5046 raise           - Number of expression trees converted
     75 raise           - Number of other getelementptr's formed
    138 raise           - Number of load/store peepholes
     42 deadtypeelim    - Number of unused typenames removed from symtab
    392 funcresolve     - Number of varargs functions resolved
     27 globaldce       - Number of global variables removed
      2 adce            - Number of basic blocks removed
    134 cee             - Number of branches revectored
     49 cee             - Number of setcc instruction eliminated
    532 gcse            - Number of loads removed
   2919 gcse            - Number of instructions removed
     86 indvars         - Number of canonical indvars added
     87 indvars         - Number of aux indvars removed
     25 instcombine     - Number of dead inst eliminate
    434 instcombine     - Number of insts combined
    248 licm            - Number of load insts hoisted
   1298 licm            - Number of insts hoisted to a loop pre-header
      3 licm            - Number of insts hoisted to multiple loop preds (bad, no loop pre-header)
     75 mem2reg         - Number of alloca's promoted
   1444 cfgsimplify     - Number of blocks simplified
```
Obviously, with so many optimizations, having a unified framework for this stuff is very nice. Making your pass fit well into the framework makes it more maintainable and useful.

显而易见，这种统一的框架很好的显示了如此多的优化结果。使用该框架将会使我们的程序更具可维护性。

### Viewing graphs while debugging code 调试时显示图

Several of the important data structures in LLVM are graphs: for example CFGs made out of LLVM BasicBlocks, CFGs made out of LLVM MachineBasicBlocks, and Instruction Selection DAGs. In many cases, while debugging various parts of the compiler, it is nice to instantly visualize these graphs.

LLVM中几种重要的数据结构是以图表示的，例如由[BasicBlock]("#BasicBlock")、[MachineBasicBlock]("#machinebasicblock")构成的流程控制图，[指令选择的有向无环图]("http://llmv.org/~matt/llvm/CodeGenerator.html#selectiondag_intro")。如果能够在编译器的调试阶段能够以图形方式展示这些图就好了。

LLVM provides several callbacks that are available in a debug build to do exactly that. If you call the Function::viewCFG() method, for example, the current LLVM tool will pop up a window containing the CFG for the function where each basic block is a node in the graph, and each node contains the instructions in the block. Similarly, there also exists Function::viewCFGOnly() (does not include the instructions), the MachineFunction::viewCFG() and MachineFunction::viewCFGOnly(), and the SelectionDAG::viewGraph() methods. Within GDB, for example, you can usually use something like call DAG.viewGraph() to pop up a window. Alternatively, you can sprinkle calls to these functions in your code in places you want to debug.

LLVM提供了几个回调函数来实现该功能，它们只在调试模式下有效。比如当调用Function::viewCFG()方法时，当前的LLVM工具会弹出一个窗口来显示流程控制图，该图__TBD__。同样，LLVM还提供了Function::viewCFGOnly()(不显示指令)、MachineFunction::viewCFG()、MachineFunction::viewCFGOnly()、SelectionDAG::viewGraph()这些方法。在GDB中，通常可以使用call DAG.viewGraph()来弹出窗口。当然，也可以它们放在代码里面，以便调试时使用。

Getting this to work requires a small amount of configuration. On Unix systems with X11, install the graphviz toolkit, and make sure 'dot' and 'gv' are in your path. If you are running on Mac OS/X, download and install the Mac OS/X Graphviz program, and add /Applications/Graphviz.app/Contents/MacOS/ (or wherever you install it) to your path. Once in your system and path are set up, rerun the LLVM configure script and rebuild LLVM to enable this functionality.

实现该功能只需要做少量的配置即可。如果是使用X11的Unix系统，需要安装graphviz工具包，并确保$PATH变量中包括了dot和gv命令。如果是Mac OS/X，需要下载并安装Mac OS/X版的Graphviz，并将/Applications/Graphviz.app/Contents/MacOS(或者自定义的路径)添加到路径中。设置完毕后，重新运行LLVM的配置脚本并重新编译LLVM即可开启上述功能。

SelectionDAG has been extended to make it easier to locate interesting nodes in large complex graphs. From gdb, if you call DAG.setGraphColor(node, "color"), then the next call DAG.viewGraph() would highlight the node in the specified color (choices of colors can be found at colors.) More complex node attributes can be provided with call DAG.setGraphAttrs(node, "attributes") (choices can be found at Graph Attributes.) If you want to restart and clear all the current graph attributes, then you can call DAG.clearGraphAttrs().

SelectionDAG提供的功能可以很容易在一张巨大、复杂的图里找到我们感兴趣的结点。在gdb中，使用call DAG.setGraphColor(node, "color")，那以后的DAG.viewGraph()则可以用指定的颜色显示这些结点(colors提供了所有的颜色)。一些比较复杂的结点属性则可以使用call DAG.setGraphAttrs(node, "attributes")(Graph Attributes提供了所有的属性)来显示。如果想清除所有的图属性或重头再来，可以使用call DAG.clearGraphAttrs()。

Note that graph visualization features are compiled out of Release builds to reduce file size. This means that you need a Debug+Asserts or Release+Asserts build to use these features.

需要注意的是，为了减小文件大小，在Release build中是不包含上述功能的。也就是说，如果要使用这些图形化功能，则必须使用Debug+Asserts或Release+Asserts选项编译LLVM。

## Picking the Right Data Structure for a Task 选择正确的数据结构

LLVM has a plethora of data structures in the llvm/ADT/ directory, and we commonly use STL data structures. This section describes the trade-offs you should consider when you pick one.

LLVM通常是使用STL里的数据结构，不过在`llvm/ADT`里也提供了大量的数据结构。这一节描述了如何选择一种合适的数据结构。

The first step is a choose your own adventure: do you want a sequential container, a set-like container, or a map-like container? The most important thing when choosing a container is the algorithmic properties of how you plan to access the container. Based on that, you should use:

首先需要确定一个分类：是需要一个有序类容器、还是一个集合类容器亦或是map类容器？打算如何访问数据是确定选择哪类容器的决定性因素。所以，我们可以使用：

* a [map-like]("http://llvm.org/~matt/llvm/ProgrammersManual.html#ds_map") container if you need efficient look-up of an value based on another value. Map-like containers also support efficient queries for containment (whether a key is in the map). Map-like containers generally do not support efficient reverse mapping (values to keys). If you need that, use two maps. Some map-like containers also support efficient iteration through the keys in sorted order. Map-like containers are the most expensive sort, only use them if you need one of these capabilities.  
如果需要通过一个关键字去快速查找对应值，则可以使用[map类容器]("http://llvm.org/~matt/llvm/ProgrammersManual.html#ds_map")。Map类容器检查关键字是否存在的效率也很高。但通常来说，反向映射的效率就相对差一些(通过值查找关键字)，如果需要反向映射，则可以使用两个map。部分map类容器也支持通过排序后的关键字进行快速遍历。Map类容器的开销是所有容器里最大的，请在确实需要以上功能时才使用map类容器。

* a set-like container if you need to put a bunch of stuff into a container that automatically eliminates duplicates. Some set-like containers support efficient iteration through the elements in sorted order. Set-like containers are more expensive than sequential containers.  
在把一堆元素存入容器时，如果希望容器能自动剔除重复的元素的话，则可以使用[集合类容器]("http://llvm.org/~matt/llvm/ProgrammersManual.html#ds_set")。部分集合类容器也支持通过排序后的元素进行快速遍历。集合类容器比有序类容器的开销要大一些。

* a sequential container provides the most efficient way to add elements and keeps track of the order they are added to the collection. They permit duplicates and support efficient iteration, but do not support efficient look-up based on a key.  
如果要让元素在容器中的位置跟放入时的顺序保持一致，那么[有序类容器]("http://llvm.org/~matt/llvm/ProgrammersManual.html#ds_sequential")是最佳选择。有序类容器允许有重复的元素，并支持快速遍历，但不支持通过关键字查找。

* a string container is a specialized sequential container or reference structure that is used for character or byte arrays.  
[字符串容器]("http://llvm.org/~matt/llvm/ProgrammersManual.html#ds_string")一个特殊的有序类容器或引用结构，它用于存放字符数组或字节数组。

* a bit container provides an efficient way to store and perform set operations on sets of numeric id's, while automatically eliminating duplicates. Bit containers require a maximum of 1 bit for each identifier you want to store.  
[比特位容器]("http://llvm.org/~matt/llvm/ProgrammersManual.html#ds_bit")__TBD__

Once the proper category of container is determined, you can fine tune the memory use, constant factors, and cache behaviors of access by intelligently picking a member of the category. Note that constant factors and cache behavior can be a big deal. If you have a vector that usually only contains a few elements (but could contain many), for example, it's much better to use SmallVector than vector . Doing so avoids (relatively) expensive malloc/free calls, which dwarf the cost of adding the elements to the container.

选定分类后，就可以在分类里根据内存使用，常量因素和缓存模式来选择具体的容器了，其中常量因素和缓存模式是__TBD__。如果容器中只需要存放少量的元素，那么使用SmallVector要比vector好，因为相对而言，SmallVector能减少对malloc/free的调用，

### Sequential Containers (std::vector, std::list, etc) 有序类容器(std::vector, std::list等)

There are a variety of sequential containers available for you, based on your needs. Pick the first in this section that will do what you want.

有序类容器非常多，可以根据本节开始部分的方法来选择合适的容器。

#### llvm/ADT/ArrayRef.h

The llvm::ArrayRef class is the preferred class to use in an interface that accepts a sequential list of elements in memory and just reads from them. By taking an ArrayRef, the API can be passed a fixed size array, an std::vector, an llvm::SmallVector and anything else that is contiguous in memory.

llvm::ArrayRef是只读不写的有序类容器，所以它很适合用作函数的参数类型，这样函数可以接受固定大小的数组、std::vector、llvm::SmallVector，甚至可以是内存中任意一段连续的元素。

#### Fixed Size Arrays 固定大小的数组

Fixed size arrays are very simple and very fast. They are good if you know exactly how many elements you have, or you have a (low) upper bound on how many you have.

固定大小数组非常简单，访问速度也很快。如果元素个数固定，或者有个数上限的时候，固定大小的数组是一个很好的选择。

#### Heap Allocated Arrays 堆数组

Heap allocated arrays (new[] + delete[]) are also simple. They are good if the number of elements is variable, if you know how many elements you will need before the array is allocated, and if the array is usually large (if not, consider a SmallVector). The cost of a heap allocated array is the cost of the new/delete (aka malloc/free). Also note that if you are allocating an array of a type with a constructor, the constructor and destructors will be run for every element in the array (re-sizable vectors only construct those elements actually used).

堆数组(new[] + delete[])也很简单。当元素个数是变量，并且能在分配空间前得知个数，同时元素个数比较大(否则请考虑SmallVector)时，则可以使用堆数组。堆数组的分配开销即为net/delete(aka malloc/free)的开销。

...同时，如果数组元素有构造函数，那么每个函数的构造函数和析构函数都会在必要时调用()。

#### "llvm/ADT/TinyPtrVector.h"

TinyPtrVector<Type> is a highly specialized collection class that is optimized to avoid allocation in the case when a vector has zero or one elements. It has two major restrictions: 1) it can only hold values of pointer type, and 2) it cannot hold a null pointer.

TinyPtrVector<Type>是一种非常特殊的容器，它专门对那些只有一个或没有元素的容器进行了优化，使其不再额外申请内存空间。所以它有两个限制：1)元素的类型必须是指针，2)不能存放空指针。

Since this container is highly specialized, it is rarely used.

因为该容器过于特殊，所以很少使用。

#### "llvm/ADT/SmallVector.h"

SmallVector<Type, N> is a simple class that looks and smells just like vector<Type>: it supports efficient iteration, lays out elements in memory order (so you can do pointer arithmetic between elements), supports efficient push_back/pop_back operations, supports efficient random access to its elements, etc.

SmallVector<Type, N>和vector<Type>非常地像，它支持快速遍历，元素在内存连续排列(这样可以用指针运算计算元素地址)，同时它的push_back/pop_back和对元素的随机访问的效率都很高。

The advantage of SmallVector is that it allocates space for some number of elements (N) in the object itself. Because of this, if the SmallVector is dynamically smaller than N, no malloc is performed. This can be a big win in cases where the malloc/free call is far more expensive than the code that fiddles around with the elements.

SmallVector的优势在于它会为自己预先申请指定数量(N)的元素所需的内存空间。正因为如此，只要元素个数动态变化时不超过N，那么SmallVector就不需要调用malloc。在某些情况下，malloc/free的开销要比移动元素的开销大，那么这时SmallVector就非常有用了。

This is good for vectors that are "usually small" (e.g. the number of predecessors/successors of a block is usually less than 8). On the other hand, this makes the size of the SmallVector itself large, so you don't want to allocate lots of them (doing so will waste a lot of space). As such, SmallVectors are most useful when on the stack.

同时，在元素个数比较少时使用SmallVector也很好(比如块结构的前后继个数通常是小于8的)。另一方面，该特点导致SmallVector的大小比实际大小要大，所以最好不要过多地使用(这样会浪费内存空间)。就这点来说，SmallVector很适合用在栈上。

SmallVector also provides a nice portable and efficient replacement for alloca.

...

#### <vector>

std::vector is well loved and respected. It is useful when SmallVector isn't: when the size of the vector is often large (thus the small optimization will rarely be a benefit) or if you will be allocating many instances of the vector itself (which would waste space for elements that aren't in the container). vector is also useful when interfacing with code that expects vectors :).

std::vector是非常受欢迎的容器，它能做一些SmallVector不能做的事情：例如元素个数比较大(这样SmallVector的优势就丧失殆尽)的时候，或者需要很多容器实例(大量SmallVector会浪费内存空间)。当然了，vector的优势还体现在接口要求必须是vector时:)。

One worthwhile note about std::vector: avoid code like this:

一个很值得我们注意的std::vector的错误用法是：

```
for ( ... ) {
   std::vector<foo> V;
   // make use of V.
}
```

Instead, write this as:

而实际上应该这样做：

```
std::vector<foo> V;
for ( ... ) {
   // make use of V.
   V.clear();
}
```

Doing so will save (at least) one heap allocation and free per iteration of the loop.

这样做可以(至少)省去一次内存分配的开销，并且可以避免每次循环结束时都会释放容器的内存空间。


#### \<deque>

std::deque is, in some senses, a generalized version of std::vector. Like std::vector, it provides constant time random access and other similar properties, but it also provides efficient access to the front of the list. It does not guarantee continuity of elements within memory.

从某种意义上说，std::deque是std::vector更一般化的版本。所以std::deque也可以像vector一样提供性能为O(1)的随机访问和一些其它类似的特性，不同的是，std::deque还能提供快速访问数组头部(译注：push_front/pop_front)，并且它不会保证数据在内存中的连续性。

In exchange for this extra flexibility, std::deque has significantly higher constant factor costs than std::vector. If possible, use std::vector or something cheaper.

相比std::vector来说，std::deque虽然提供了上述灵活性，但它的开销也相对大得多。如果可以的话，尽量使用std::vector或其它开销更小的容器。

#### \<list>

std::list is an extremely inefficient class that is rarely useful. It performs a heap allocation for every element inserted into it, thus having an extremely high constant factor, particularly for small data types. std::list also only supports bidirectional iteration, not random access iteration.

std::list是一个效率非常低下的容器类，所以用得很少。__TBD__ 

In exchange for this high cost, std::list supports efficient access to both ends of the list (like std::deque, but unlike std::vector or SmallVector). In addition, the iterator invalidation characteristics of std::list are stronger than that of a vector class: inserting or removing an element into the list does not invalidate iterator or pointers to other elements in the list.

虽然std::list的开销比较高，但却能够快速访问容器的首部和尾部(类似于std::deque，而std::vector和SmallVector不行)。同时，__TBD__

#### llvm/ADT/ilist.h

ilist<T> implements an 'intrusive' doubly-linked list. It is intrusive, because it requires the element to store and provide access to the prev/next pointers for the list.

ilist\<T>实现了一个"侵入式"的双向链表，之所以这样说，是因为它要求每个元素都必须保存相邻元素的指针，同时还必须把这两个指针暴露给ilist\<T>。

ilist has the same drawbacks as std::list, and additionally requires an ilist_traits implementation for the element type, but it provides some novel characteristics. In particular, it can efficiently store polymorphic objects, the traits class is informed when an element is inserted or removed from the list, and ilists are guaranteed to support a constant-time splice operation.

ilist不仅有跟std::list一样的缺点，而且还要求__TBD__，不过这一点却使得ilist与众不同：可以存放__TBD__，因为当添加或删除元素时会触发相应的trait类。同时，ilist也支持常量时间的连接操作。

These properties are exactly what we want for things like Instructions and basic blocks, which is why these are implemented with ilists.

对于像Instruction和基本语句块来说，使用ilist作容器是再合适不过了，这也是为什么它们都是使用ilist实现的。

Related classes of interest are explained in the following subsections:

如果有兴趣的话可以参考以下相关类：

* ilist_traits
* iplist
* llvm/ADT/ilist_node.h
* Sentinels

#### llvm/ADT/PackedVector.h

Useful for storing a vector of values using only a few number of bits for each value. Apart from the standard operations of a vector-like container, it can also perform an 'or' set operation.

PackedVector是一个用于存放少量比特位的容器，它不仅提供了类似于vector容器的标准方法外，还提供了__TBD__

For example:

```
enum State {
    None = 0x0,
    FirstCondition = 0x1,
    SecondCondition = 0x2,
    Both = 0x3
};

State get() {
    PackedVector<State, 2> Vec1;
    Vec1.push_back(FirstCondition);

    PackedVector<State, 2> Vec2;
    Vec2.push_back(SecondCondition);

    Vec1 |= Vec2;
    return Vec1[0]; // returns 'Both'.
}
```

#### ilist_traits

ilist_traits<T> is ilist<T>'s customization mechanism. iplist<T> (and consequently ilist<T>) publicly derive from this traits class.

ilist_traits\<T>为ilist\<T>提供了可定制的机制。iplist\<T>(当然还有ilist\<T>)就是继承自这个特性类。

#### iplist

iplist<T> is ilist<T>'s base and as such supports a slightly narrower interface. Notably, inserters from T& are absent.

iplist\<T>是ilist\<T>的基类，所以它提供了相对较少__TBD__

ilist_traits<T> is a public base of this class and can be used for a wide variety of customizations.

ilist_traits\<T>是iplist\<T>的其中一个public基类，它能提供非常多的可定制性。

#### llvm/ADT/ilist_node.h

ilist_node<T> implements a the forward and backward links that are expected by the ilist<T> (and analogous containers) in the default manner.

ilist_node\<T>内包括了指向前/后继的的指针，这正是ilist\<T>(以及类似的容器) __TBD__

ilist_node<T>s are meant to be embedded in the node type T, usually T publicly derives from ilist_node<T>.

ilist_node\<T>用于存放__TBD__

#### Sentinels

ilists have another specialty that must be considered. To be a good citizen in the C++ ecosystem, it needs to support the standard container operations, such as begin and end iterators, etc. Also, the operator-- must work correctly on the end iterator in the case of non-empty ilists.

ilist还有一个特点是需要注意的。作为C++生态系统中的良好市民，它需要支持标准的容器操作，例如取得头部和尾部的迭代器等。同时，在非空的ilist中，对它的尾部迭代器执行--运算也必须得到正确的结果(译注：即最后一个元素)。

The only sensible solution to this problem is to allocate a so-called sentinel along with the intrusive list, which serves as the end iterator, providing the back-link to the last element. However conforming to the C++ convention it is illegal to operator++ beyond the sentinel and it also must not be dereferenced.

要解决这个问题，唯一明智的方法是在容器尾部插入一个所谓的"哨兵"元素，并用它作为容器的尾部迭代器。但是，为了使其使用更简便，对Sentinel元素的++运算是非法的，且不能derefenrence。

These constraints allow for some implementation freedom to the ilist how to allocate and store the sentinel. The corresponding policy is dictated by ilist_traits<T>. By default a T gets heap-allocated whenever the need for a sentinel arises.

上述约定给予了ilist一些自由，让其自行决定如何分配和保存哨兵元素。正常的策略是使用ilist_traits\<T>，在默认情况下，当需要使用哨兵元素时，容器会为其分配空间。

While the default policy is sufficient in most cases, it may break down when T does not provide a default constructor. Also, in the case of many instances of ilists, the memory overhead of the associated sentinels is wasted. To alleviate the situation with numerous and voluminous T-sentinels, sometimes a trick is employed, leading to ghostly sentinels.

虽然默认的策略已经能够大多数情况了，不过当T没有默认构造函数时就行不通了。另外，当程序创建了大量的ilist实例时，为每个实例都分配一个哨兵元素的话就会显得很浪费。所以，为了缓解过多的T类型的哨兵元素，通常会巧妙地使用幽灵哨兵元素。

Ghostly sentinels are obtained by specially-crafted ilist_traits<T> which superpose the sentinel with the ilist instance in memory. Pointer arithmetic is used to obtain the sentinel, which is relative to the ilist's this pointer. The ilist is augmented by an extra pointer, which serves as the back-link of the sentinel. This is the only field in the ghostly sentinel which can be legally accessed.

幽灵哨兵是一个精心设计的ilist_traits\<T>，不同的实例可以重复使用。__TBD__

#### Other Sequential Container options 其它的有序类容器

Other STL containers are available, such as std::string.

其它的STL容器也是可用的，比如std::string。

There are also various STL adapter classes such as std::queue, std::priority_queue, std::stack, etc. These provide simplified access to an underlying container but don't affect the cost of the container itself.

当然还有一些STL适配类，比如std::queue、std::priority_queue、std::stack等。__TBD__

### String-like containers 字符串类容器

There are a variety of ways to pass around and use strings in C and C++, and LLVM adds a few new options to choose from. Pick the first option on this list that will do what you need, they are ordered according to their relative cost.

C/C++里有很多字符串的用法和传递方式，LLVM在其基础上增加了一些新选择。本节开始部分的描述了如何选择合适的容器，而下面将以开销的大小顺序来介绍它们。

Note that is is generally preferred to not pass strings around as "const char*"'s. These have a number of problems, including the fact that they cannot represent embedded nul ("\0") characters, and do not have a length available efficiently. The general replacement for 'const char*' is StringRef.

需要注意的是通常不要直接使用"const char *"类型来传递字符串，这可能会导致一大堆问题，例如字符串中不能包含空字符("\0")，不能快速取得字符串长度。一般来说可以使用StringRef来替代'const char *'.

For more information on choosing string containers for APIs, please see Passing strings.

更多信息可以跳至Passing strings。

#### llvm/ADT/StringRef.h

The StringRef class is a simple value class that contains a pointer to a character and a length, and is quite related to the ArrayRef class (but specialized for arrays of characters). Because StringRef carries a length with it, it safely handles strings with embedded nul characters in it, getting the length does not require a strlen call, and it even has very convenient APIs for slicing and dicing the character range that it represents.

StringRef的实现非常简单，它包含了一个字符串指针和字符串的长度，它是ArrayRef的特殊形式(数组元素是字符)。因为StringRef保存了字符串长度，所以可以安全地把空字符插入到字符串中，也不需要通过strlen函数来取得长度，而且还提供了一些非常方便的方法实现了__TBD__

StringRef is ideal for passing simple strings around that are known to be live, either because they are C string literals, std::string, a C array, or a SmallVector. Each of these cases has an efficient implicit conversion to StringRef, which doesn't result in a dynamic strlen being executed.

如果要传递一个已存在的字符串，StringRef将是非常理想的方式，无论它们是C风格字符串、std::string、C数组或是SmallVector。__TBD__

StringRef has a few major limitations which make more powerful string containers useful:

StringRef有一些限制：

1. You cannot directly convert a StringRef to a 'const char*' because there is no way to add a trailing nul (unlike the .c_str() method on various stronger classes).  
不能直接把StringRef转换成'const char\*'，因为__TBD__

1. StringRef doesn't own or keep alive the underlying string bytes. As such it can easily lead to dangling pointers, and is not suitable for embedding in datastructures in most cases (instead, use an std::string or something like that).  
StringRef不能自行保存字符串。这样它极有可能导致悬空指针，所以在大多数情况下是不适合放入到具体的数据结构里的(相反，std::string却很适合)。

1. For the same reason, StringRef cannot be used as the return value of a method if the method "computes" the result string. Instead, use std::string.  
同理，当函数需要"计算"出一个字符串结果时，StringRef也不也作为其返回值的类型。这个时候请使用std::string。

1. StringRef's do not allow you to mutate the pointed-to string bytes and it doesn't allow you to insert or remove bytes from the range. For editing operations like this, it interoperates with the Twine class.  
StringRef不允许修改它所保存的字符串。如果需要修改，它需要和Twine类配合使用。

Because of its strengths and limitations, it is very common for a function to take a StringRef and for a method on an object to return a StringRef that points into some string that it owns.

综上所述，StringRef通常作为函数的参数类型，也可以用作类成员函数的返回值，且返回的字符串必须是由类自身维护的。

#### llvm/ADT/Twine.h

The Twine class is used as an intermediary datatype for APIs that want to take a string that can be constructed inline with a series of concatenations. Twine works by forming recursive instances of the Twine datatype (a simple value object) on the stack as temporary objects, linking them together into a tree which is then linearized when the Twine is consumed. Twine is only safe to use as the argument to a function, and should always be a const reference, e.g.:

如果函数的字符串参数需要通过一系列的字符串连接而得，那么Twine可以用作一种中间的数据类型。Twine在栈空间上生成一些临时对象，并把它们连接成树状结构，这样可以让Twine在使用的时候看上去是一个连续的字符串。Twine只能作为函数的参数类型，并且总是一个常量引用(const reference)，例如：

````
void foo(const Twine &T);
...
StringRef X = ...
unsigned i = ...
foo(X + "." + Twine(i));
````
  
This example forms a string like "blarg.42" by concatenating the values together, and does not form intermediate strings containing "blarg" or "blarg.".

该例子创建了一个形如"blarg.42"的字符串，该字符串是把3个字符串连接而得。但实际上，它并没有立即生成包含"blarg"或"blarg."的字符串。

Because Twine is constructed with temporary objects on the stack, and because these instances are destroyed at the end of the current statement, it is an inherently dangerous API. For example, this simple variant contains undefined behavior and will probably crash:

因为Twine是在栈上创建的临时对象，并且在当前语句结束时就会被销毁，所以它带有与生俱来的危险性。例如，以下代码片断的行为将是不可预知的，并且很可能导致程序崩溃：

```
void foo(const Twine &T);
...
StringRef X = ...
unsigned i = ...
const Twine &Tmp = X + "." + Twine(i);
foo(Tmp);
```
  
... because the temporaries are destroyed before the call. That said, Twine's are much more efficient than intermediate std::string temporaries, and they work really well with StringRef. Just be aware of their limitations.

因为在函数调用之前，临时变量就会被销毁。也就是说，比起使用std::string作为临时的中间字符串来说，Twine显得更加高效，并且它能够很好地和StringRef配合使用。唯一的缺点就是需要注意以上所述的限制。

#### llvm/ADT/SmallString.h

SmallString is a subclass of SmallVector that adds some convenience APIs like += that takes StringRef's. SmallString avoids allocating memory in the case when the preallocated space is enough to hold its data, and it calls back to general heap allocation when required. Since it owns its data, it is very safe to use and supports full mutation of the string.

SmallString是SmallVector的一个子类，它加入了一些很方便的API，比如+=操作，该操作可以连接StringRef。当SmallString预先分配的空间足够时，连接操作就可以避免二次内存分配。因为SmallString是自己保存字符串，所以它相对来说比较安全，并且可以支持字符串的修改。

Like SmallVector's, the big downside to SmallString is their sizeof. While they are optimized for small strings, they themselves are not particularly small. This means that they work great for temporary scratch buffers on the stack, but should not generally be put into the heap: it is very rare to see a SmallString as the member of a frequently-allocated heap data structure or returned by-value.

和SmallVector类似，SmallString的问题在于不能对其使用sizeof。同时，它们是专为短小的字符串进行了优化，而它们自身通常并不短小，这就意味着最好是在栈上使用它，而不是在堆上。所以，SmallString很少出现在需要在堆上频繁出现的数据结构里，也很少以by-value的方式作为函数的返回值。

#### std::string

The standard C++ std::string class is a very general class that (like SmallString) owns its underlying data. sizeof(std::string) is very reasonable so it can be embedded into heap data structures and returned by-value. On the other hand, std::string is highly inefficient for inline editing (e.g. concatenating a bunch of stuff together) and because it is provided by the standard library, its performance characteristics depend a lot of the host standard library (e.g. libc++ and MSVC provide a highly optimized string class, GCC contains a really slow implementation).

C++标准库中的std::string是应用非常广泛的类，整个字符串会保存在std::string对象里(类似于SmallString)。对std::string使用sizeof是可行的，所以它完全可以嵌入到堆里的数据结构里，也可以作为以by-value方式的返回值。另一方面，当需要内联(inline)编辑(比如连接好几个字符串)时，使用std::string是非常不明智的。同时，std::string作为标准库的一部分，其性能几乎取决于本地的标准库(例如，libc++和MSVC提供了经过高度优化的string类，而GCC提供的版本运行起来却非常缓慢)。

The major disadvantage of std::string is that almost every operation that makes them larger can allocate memory, which is slow. As such, it is better to use SmallVector or Twine as a scratch buffer, but then use std::string to persist the result.

当然了，std::string最大的问题是，任何增长字符串的操作都会导致很慢的内存分配。在这种情况下，使用SmallVector或Twine更加理想，而std::string仅用于保存最终的结果。

### Set-Like Containers (std::set, SmallSet, SetVector, etc)集合类容器(std::set, SmallSet, SetVector等)

Set-like containers are useful when you need to canonicalize multiple values into a single representation. There are several different choices for how to do this, providing various trade-offs.

#### A sorted 'vector' 已排序的vector

If you intend to insert a lot of elements, then do a lot of queries, a great approach is to use a vector (or other sequential container) with std::sort+std::unique to remove duplicates. This approach works really well if your usage pattern has these two distinct phases (insert then query), and can be coupled with a good choice of sequential container.

如果是先向向容器中插入大量元素，之后再执行大量的查询操作的话，一个不错的方法是使用vector(或其它的有序类容器)作为容器，并通过std::sort和std::unique算法删除重复的元素。如果整个过程能分成两个清晰的阶段(先插入后查询)，那么这种方式会非常有效，而具体的容器可以选用有序类容器。

This combination provides the several nice properties: the result data is contiguous in memory (good for cache locality), has few allocations, is easy to address (iterators in the final vector are just indices or pointers), and can be efficiently queried with a standard binary or radix search.

这种组合有这样几个优点：最终结果在内存中是连续存放的(适合本地缓存__TBD__)，很少的内存分配，容易寻址(可以直接使用最终结果里的iterator进行遍历)，通过标准的二分法查找和基数查找实现高效率的查询。

#### "llvm/ADT/SmallSet.h"

If you have a set-like data structure that is usually small and whose elements are reasonably small, a SmallSet<Type, N> is a good choice. This set has space for N elements in place (thus, if the set is dynamically smaller than N, no malloc traffic is required) and accesses them with a simple linear search. When the set grows beyond 'N' elements, it allocates a more expensive representation that guarantees efficient access (for most types, it falls back to std::set, but for pointers it uses something far better, SmallPtrSet).

如果容器的容量通常不大，而且每个元素也相对较小，那个SmallSet<Type, N>将是个很好的选择。这个集合最多可以存放N个元素(如果实际使用量小于N，就不需要内存分配了)，其搜索方式是简单的线性搜索。如果元素的个数超过了N，它会申请__TBD__(对大多数类型来说，这样就退化到std::set了，不过对指针类型来说，SmallPtrSet会更好)。

The magic of this class is that it handles small sets extremely efficiently, but gracefully handles extremely large sets without loss of efficiency. The drawback is that the interface is quite small: it supports insertion, queries and erasing, but does not support iteration.

SmallSet不可思议的地方是，当处理少量元素时，它的效率非常地高，而处理大量元素时，效率也不会有损失。这个容器的缺点是它的接口面很小，只支持插入、查询和删除，但不支持迭代。

#### "llvm/ADT/SmallPtrSet.h"

SmallPtrSet has all the advantages of SmallSet (and a SmallSet of pointers is transparently implemented with a SmallPtrSet), but also supports iterators. If more than 'N' insertions are performed, a single quadratically probed hash table is allocated and grows as needed, providing extremely efficient access (constant time insertion/deleting/queries with low constant factors) and is very stingy with malloc traffic.

SmallPtrSet具有SmallSet的所有优点(__TBD__)，另外它还提供了迭代器。如果插入的元素超过了N，SmallPtrSet会生成一个使用二次探测算法的哈希表，并借此提供快速访问能力(常数级的插入/删除/查询)，同时，它也几乎没有malloc带来的开销。

Note that, unlike std::set, the iterators of SmallPtrSet are invalidated whenever an insertion occurs. Also, the values visited by the iterators are not visited in sorted order.

和std::set不一样的是，在每次插入操作后，SmallPtrSet的迭代器就会无效。所以，通过迭代器访问的顺序和排序后的顺序是不一致的。

#### "llvm/ADT/DenseSet.h"

DenseSet is a simple quadratically probed hash table. It excels at supporting small values: it uses a single allocation to hold all of the pairs that are currently inserted in the set. DenseSet is a great way to unique small values that are not simple pointers (use SmallPtrSet for pointers). Note that DenseSet has the same requirements for the value type that DenseMap has.

DenseSet实际上是一个使用二次探测算法的哈希表。它对小元素的支持很好，因为在插入元素时，它只进行一次内存分配。__TBD__

#### "llvm/ADT/SparseSet.h"

SparseSet holds a small number of objects identified by unsigned keys of moderate size. It uses a lot of memory, but provides operations that are almost as fast as a vector. Typical keys are physical registers, virtual registers, or numbered basic blocks.

SparseSet is useful for algorithms that need very fast clear/find/insert/erase and fast iteration over small sets. It is not intended for building composite data structures.

#### "llvm/ADT/FoldingSet.h"

FoldingSet is an aggregate class that is really good at uniquing expensive-to-create or polymorphic objects. It is a combination of a chained hash table with intrusive links (uniqued objects are required to inherit from FoldingSetNode) that uses SmallVector as part of its ID process.

Consider a case where you want to implement a "getOrCreateFoo" method for a complex object (for example, a node in the code generator). The client has a description of *what* it wants to generate (it knows the opcode and all the operands), but we don't want to 'new' a node, then try inserting it into a set only to find out it already exists, at which point we would have to delete it and return the node that already exists.

To support this style of client, FoldingSet perform a query with a FoldingSetNodeID (which wraps SmallVector) that can be used to describe the element that we want to query for. The query either returns the element matching the ID or it returns an opaque ID that indicates where insertion should take place. Construction of the ID usually does not require heap traffic.

Because FoldingSet uses intrusive links, it can support polymorphic objects in the set (for example, you can have SDNode instances mixed with LoadSDNodes). Because the elements are individually allocated, pointers to the elements are stable: inserting or removing elements does not invalidate any pointers to other elements.

<set>

std::set is a reasonable all-around set class, which is decent at many things but great at nothing. std::set allocates memory for each element inserted (thus it is very malloc intensive) and typically stores three pointers per element in the set (thus adding a large amount of per-element space overhead). It offers guaranteed log(n) performance, which is not particularly fast from a complexity standpoint (particularly if the elements of the set are expensive to compare, like strings), and has extremely high constant factors for lookup, insertion and removal.

The advantages of std::set are that its iterators are stable (deleting or inserting an element from the set does not affect iterators or pointers to other elements) and that iteration over the set is guaranteed to be in sorted order. If the elements in the set are large, then the relative overhead of the pointers and malloc traffic is not a big deal, but if the elements of the set are small, std::set is almost never a good choice.

"llvm/ADT/SetVector.h"

LLVM's SetVector<Type> is an adapter class that combines your choice of a set-like container along with a Sequential Container. The important property that this provides is efficient insertion with uniquing (duplicate elements are ignored) with iteration support. It implements this by inserting elements into both a set-like container and the sequential container, using the set-like container for uniquing and the sequential container for iteration.

The difference between SetVector and other sets is that the order of iteration is guaranteed to match the order of insertion into the SetVector. This property is really important for things like sets of pointers. Because pointer values are non-deterministic (e.g. vary across runs of the program on different machines), iterating over the pointers in the set will not be in a well-defined order.

The drawback of SetVector is that it requires twice as much space as a normal set and has the sum of constant factors from the set-like container and the sequential container that it uses. Use it *only* if you need to iterate over the elements in a deterministic order. SetVector is also expensive to delete elements out of (linear time), unless you use it's "pop_back" method, which is faster.

SetVector is an adapter class that defaults to using std::vector and a size 16 SmallSet for the underlying containers, so it is quite expensive. However, "llvm/ADT/SetVector.h" also provides a SmallSetVector class, which defaults to using a SmallVector and SmallSet of a specified size. If you use this, and if your sets are dynamically smaller than N, you will save a lot of heap traffic.

"llvm/ADT/UniqueVector.h"

UniqueVector is similar to SetVector, but it retains a unique ID for each element inserted into the set. It internally contains a map and a vector, and it assigns a unique ID for each value inserted into the set.

UniqueVector is very expensive: its cost is the sum of the cost of maintaining both the map and vector, it has high complexity, high constant factors, and produces a lot of malloc traffic. It should be avoided.

"llvm/ADT/ImmutableSet.h"

ImmutableSet is an immutable (functional) set implementation based on an AVL tree. Adding or removing elements is done through a Factory object and results in the creation of a new ImmutableSet object. If an ImmutableSet already exists with the given contents, then the existing one is returned; equality is compared with a FoldingSetNodeID. The time and space complexity of add or remove operations is logarithmic in the size of the original set.

There is no method for returning an element of the set, you can only check for membership.

Other Set-Like Container Options

The STL provides several other options, such as std::multiset and the various "hash_set" like containers (whether from C++ TR1 or from the SGI library). We never use hash_set and unordered_set because they are generally very expensive (each insertion requires a malloc) and very non-portable.

std::multiset is useful if you're not interested in elimination of duplicates, but has all the drawbacks of std::set. A sorted vector (where you don't delete duplicate entries) or some other approach is almost always better.

### Bit storage containers (BitVector, SparseBitVector)比特位容器(BitVector, SparseBitVector)

Unlike the other containers, there are only two bit storage containers, and choosing when to use each is relatively straightforward.

和其它容器不一样的是，比特位容器只有两个选择，而且选择的理由也相对直接。

One additional option is std::vector<bool>: we discourage its use for two reasons 1) the implementation in many common compilers (e.g. commonly available versions of GCC) is extremely inefficient and 2) the C++ standards committee is likely to deprecate this container and/or change it significantly somehow. In any case, please don't use it.

当然，std::vector\<bool>也是可行的，不过我们基于以下理由而不用它：1) 一些使用它的常见编译器(GCC)的效率非常低下，2) C++标准委员会可能会废除它，或者提供改进版。所以任何情况下都不要用它。

#### BitVector

The BitVector container provides a dynamic size set of bits for manipulation. It supports individual bit setting/testing, as well as set operations. The set operations take time O(size of bitvector), but operations are performed one word at a time, instead of one bit at a time. This makes the BitVector very fast for set operations compared to other containers. Use the BitVector when you expect the number of set bits to be high (IE a dense set).

BitVector可容纳数量可变的比特位，并支持对它们的写入/检测操作。写入操作的时间复杂度是O(n)，但一次写入是写入一个词，而非一比特，这使得BitVector的写入速度比其它的容器要快。__TBD__

#### SmallBitVector

The SmallBitVector container provides the same interface as BitVector, but it is optimized for the case where only a small number of bits, less than 25 or so, are needed. It also transparently supports larger bit counts, but slightly less efficiently than a plain BitVector, so SmallBitVector should only be used when larger counts are rare.

SmallBitVector提供的接口和BitVector一样，唯一不同的是它针对只有少量比特位的情况(少于25)做了优化。它同样也支持更多的比特位，只不过效率会不及BitVector，所以处理少量比特位时才使用SmallBitVector。

At this time, SmallBitVector does not support set operations (and, or, xor), and its operator[] does not provide an assignable lvalue.

当时？SmallBitVector不支持写入操作(and, or, xor)，并且它的[]运算结果也不能成为左值。

#### SparseBitVector

The SparseBitVector container is much like BitVector, with one major difference: Only the bits that are set, are stored. This makes the SparseBitVector much more space efficient than BitVector when the set is sparse, as well as making set operations O(number of set bits) instead of O(size of universe). The downside to the SparseBitVector is that setting and testing of random bits is O(N), and on large SparseBitVectors, this can be slower than BitVector. In our implementation, setting or testing bits in sorted order (either forwards or reverse) is O(1) worst case. Testing and setting bits within 128 bits (depends on size) of the current bit is also O(1). As a general statement, testing/setting bits in a SparseBitVector is O(distance away from last set bit).

SparseBitVector和BitVector很相似，唯一不同的是，只有被设置了的比特位才会被存入容器。这使得在存储稀疏比特位时，SparseBitVector要比BitVector节省空间，同时，它的写入时间复杂度是O(已写入的比特数)，要低于O(n)。它的缺点是，对随机比特位的写入和检测的时间复杂度是O(n)，另外，对于较大的SparseBitVector来说，它们通常会比BitVecotr要慢一些。__TBD__

## Helpful Hints for Common Operations 奇技淫巧

This section describes how to perform some very simple transformations of LLVM code. This is meant to give examples of common idioms used, showing the practical side of LLVM transformations.

本节描述了如何对LLVM代码进行简单的修改。本节会给出一些例子，它们通过一些常规手法来展现修改LLVM的一些特殊性。

Because this is a "how-to" section, you should also read about the main classes that you will be working with. The Core LLVM Class Hierarchy Reference contains details and descriptions of the main classes that you should know about.

因为本节的性质是how-to，那么要求读者对将会用到的主要类有一定的了解。LLVM核心类体系结构参考手册包括了它们的描述和细节。

### Basic Inspection and Traversal Routines 基本的检查和遍历过程

The LLVM compiler infrastructure have many different data structures that may be traversed. Following the example of the C++ standard template library, the techniques used to traverse these various data structures are all basically the same. For a enumerable sequence of values, the XXXbegin() function (or method) returns an iterator to the start of the sequence, the XXXend() function returns an iterator pointing to one past the last valid element of the sequence, and there is some XXXiterator data type that is common between the two operations.

LLVM提供了许多不同的数据结构，这些数据结构背后的实现技术和C++ STL基本上是一样的。对容器来说，XXXbegin()函数(或方法)返回第一个元素所对应的迭代器，XXXend()返回代表容器结束的迭代器，__TBD__

Because the pattern for iteration is common across many different aspects of the program representation, the standard template library algorithms may be used on them, and it is easier to remember how to iterate. First we show a few common examples of the data structures that need to be traversed. Other data structures are traversed in very similar ways.

由于LLVM中的迭代器模式是独立于不同容器的，所以它们也同样适用于STL算法，而且也易于使用。首先，让我们来看一些关于数据结构的例子，未提到的数据结构也可以采用相似的方式。

#### Iterating over the BasicBlocks in a Function 遍历函数内的所有BasicBlock

It's quite common to have a Function instance that you'd like to transform in some way; in particular, you'd like to manipulate its BasicBlocks. To facilitate this, you'll need to iterate over all of the BasicBlocks that constitute the Function. The following is an example that prints the name of a BasicBlock and the number of Instructions it contains:

对一个Function实例进行某种操作应该是非常常见的了，特别是对它所包含的BasickBlock进行操作。简单起见，可能需要使用迭代器来遍历Function实例里所有的BasicBlock。下面的例子会依次打印出BasicBlock的名字和它所包含的指令数：

```
// func is a pointer to a Function instance
for (Function::iterator i = func->begin(), e = func->end(); i != e; ++i)
  // Print out the name of the basic block if it has one, and then the
  // number of instructions that it contains
  errs() << "Basic block (name=" << i->getName() << ") has "
             << i->size() << " instructions.\n";
```

Note that i can be used as if it were a pointer for the purposes of invoking member functions of the Instruction class. This is because the indirection operator is overloaded for the iterator classes. In the above code, the expression i->size() is exactly equivalent to (*i).size() just like you'd expect.

需要注意的是，我们可以把i想像成__TBD__，因为迭代器已经重载了指针运算符。在上面的代码中，i->size()和(*i).size()是等效的，这也正是我们所期望的。

#### Iterating over the Instructions in a BasicBlock 遍历BasicBlock里的所有Instruction

Just like when dealing with BasicBlocks in Functions, it's easy to iterate over the individual instructions that make up BasicBlocks. Here's a code snippet that prints out each instruction in a BasicBlock:

和遍历BasicBlock一样，对BasicBlock里的所有指令进行遍历也是很简单的。以下代码片断会依次打印出一个BasicBlock里的每个指令：

```
// blk is a pointer to a BasicBlock instance
for (BasicBlock::iterator i = blk->begin(), e = blk->end(); i != e; ++i)
   // The next statement works since operator<<(ostream&,...)
   // is overloaded for Instruction&
   errs() << *i << "\n";
```

However, this isn't really the best way to print out the contents of a BasicBlock! Since the ostream operators are overloaded for virtually anything you'll care about, you could have just invoked the print routine on the basic block itself: errs() << *blk << "\n";.

不过，这不是打印BasicBlock内容的最好方式！因为ostream的运算符是经过重载的，所以直接打印语句块本身即可：`errs() << *blk << "\n";`。

#### Iterating over the Instructions in a Function 遍历函数内的所有指令

If you're finding that you commonly iterate over a Function's BasicBlocks and then that BasicBlock's Instructions, InstIterator should be used instead. You'll need to include llvm/Support/InstIterator.h, and then instantiate InstIterators explicitly in your code. Here's a small example that shows how to dump all instructions in a function to the standard error stream:

如果经常要通过Function的BasicBlock迭代器跟BasicBlock的Instruction迭代器嵌套的方式遍历函数里的所有指令，那么完全可以使用InstIterator来代替。只需要在源文件里包含"llvm/Support/InstIterator.h"，就可以在代码里直接使用它了。这个简短的例子将在标准错误上打印出函数里所有的指令：

```
#include "llvm/Support/InstIterator.h"

// F is a pointer to a Function instance
for (inst_iterator I = inst_begin(F), E = inst_end(F); I != E; ++I)
  errs() << *I << "\n";
```

Easy, isn't it? You can also use InstIterators to fill a work list with its initial contents. For example, if you wanted to initialize a work list to contain all instructions in a Function F, all you would need to do is something like:

很简单吧，不是吗？当然你也可以把InstIterator存入一个列表中，并将它们作为列表的初始内容。举例来说，如果要把F函数里的所有指令作为一个列表的初始数据，只需要这样做：

```
std::set<Instruction*> worklist;
// or better yet, SmallPtrSet<Instruction*, 64> worklist;

for (inst_iterator I = inst_begin(F), E = inst_end(F); I != E; ++I)
   worklist.insert(&*I);
```

The STL set worklist would now contain all instructions in the Function pointed to by F.

这样，worklist集合就包含了F函数里的所有指令了。

#### Turning an iterator into a class pointer (and vice-versa) 把迭代器转为类指针

Sometimes, it'll be useful to grab a reference (or pointer) to a class instance when all you've got at hand is an iterator. Well, extracting a reference or a pointer from an iterator is very straight-forward. Assuming that i is a BasicBlock::iterator and j is a BasicBlock::const_iterator:

有时候，我们手上持有一个迭代器，而实际上真正需要的是它对应类实例的引用(或指针)。在LLVM里，从迭代器里抽取出相应的引用或指针是非常简单的。假设i是一个BasicBlock::iterator，j是一个BasicBlock::const_iterator：

```
Instruction& inst = *i;   // Grab reference to instruction reference
Instruction* pinst = &*i; // Grab pointer to instruction reference
const Instruction& inst = *j;
```

However, the iterators you'll be working with in the LLVM framework are special: they will automatically convert to a ptr-to-instance type whenever they need to. Instead of dereferencing the iterator and then taking the address of the result, you can simply assign the iterator to the proper pointer type and you get the dereference and address-of operation as a result of the assignment (behind the scenes, this is a result of overloading casting mechanisms). Thus the last line of the last example,

不过，LLVM框架里的迭代器是很特殊的：当需要的时候它们会自动转成实例的指针。所以实际上不需要采用先提领迭代器，之后再对其取地址这种方式，而是可以直接把迭代器赋给相应的指针类型即可(这背后其实是一个重载了的类型转换)。对于上例的中间一句(译注：原文是最后一句)：

```
Instruction *pinst = &*i;
```

is semantically equivalent to

等价于：

```
Instruction *pinst = i;
```

It's also possible to turn a class pointer into the corresponding iterator, and this is a constant time operation (very efficient). The following code snippet illustrates use of the conversion constructors provided by LLVM iterators. By using these, you can explicitly grab the iterator of something without actually obtaining it via iteration over some structure:

把类指针转为相应的迭代器也是可以的，而且这个转换过程的时间开销还是常数级的(非常快)。下面的代码片断展示了使用指针来初始化迭代器，通过这种方式，我们可以显式地取得一个对象所对应的迭代器，而不需要到它所在的容器里去查找。

```
void printNextInstruction(Instruction* inst) {
  BasicBlock::iterator it(inst);
  ++it; // After this line, it refers to the instruction after *inst
  if (it != inst->getParent()->end()) errs() << *it << "\n";
}
```

Unfortunately, these implicit conversions come at a cost; they prevent these iterators from conforming to standard iterator conventions, and thus from being usable with standard algorithms and containers. For example, they prevent the following code, where B is a BasicBlock, from compiling:

不幸的是，这些隐式类型转换带来了一个问题：因为通过这种方式创建的迭代器实际上是不符合迭代器的规范的，所以对它们不能参与标准算法，也没有相应的容器。在下例中，B是一个BasicBlock，而该句代码却不能通过编译：

```
llvm::SmallVector<llvm::Instruction *, 16>(B->begin(), B->end());
```

Because of this, these implicit conversions may be removed some day, and operator* changed to return a pointer instead of a reference.

正因为如此，以上隐式类型转换在将来可能会被移除，*运算符的返回值也可能会用指针代替引用。

#### Finding call sites: a slightly more complex example 查找函数调用：一个稍微复杂点的例子

Say that you're writing a FunctionPass and would like to count all the locations in the entire module (that is, across every Function) where a certain function (i.e., some Function*) is already in scope. As you'll learn later, you may want to use an InstVisitor to accomplish this in a much more straight-forward manner, but this example will allow us to explore how you'd do it if you didn't have InstVisitor around. In pseudo-code, this is what we want to do:

__TBD__在后面的章节会看到，使用InstVisitor可以更直接地完成这项任务。不过这个例子只是说明如何在不使用InstVisitor的情况下如何做到。下面的伪代码描述了整个过程：

```
initialize callCounter to zero
for each Function f in the Module
  for each BasicBlock b in f
    for each Instruction i in b
      if (i is a CallInst and calls the given function)
        increment callCounter
```

And the actual code is (remember, because we're writing a FunctionPass, our FunctionPass-derived class simply has to override the runOnFunction method):

以下是实际代码(请记住，因为我们是在编写一个FunctionPass，所以只需要继承FunctionPass类，并简单地重载runOnFunction方法即可)：

```
Function* targetFunc = ...;

class OurFunctionPass : public FunctionPass {
  public:
    OurFunctionPass(): callCounter(0) { }

    virtual runOnFunction(Function& F) {
      for (Function::iterator b = F.begin(), be = F.end(); b != be; ++b) {
        for (BasicBlock::iterator i = b->begin(), ie = b->end(); i != ie; ++i) {
          if (CallInst* callInst = dyn_cast<CallInst>(&*i)) {
            // We know we've encountered a call instruction, so we
            // need to determine if it's a call to the
            // function pointed to by m_func or not.
            if (callInst->getCalledFunction() == targetFunc)
              ++callCounter;
          }
        }
      }
    }

  private:
    unsigned callCounter;
};
```

#### Treating calls and invokes the same way 统一call和invoke

You may have noticed that the previous example was a bit oversimplified in that it did not deal with call sites generated by 'invoke' instructions. In this, and in other situations, you may find that you want to treat CallInsts and InvokeInsts the same way, even though their most-specific common base class is Instruction, which includes lots of less closely-related things. For these cases, LLVM provides a handy wrapper class called CallSite. It is essentially a wrapper around an Instruction pointer, with some methods that provide functionality common to CallInsts and InvokeInsts.

在上例中，我们过于简化了一点，即没有处理由'invoke'指令产生的call。那么无论是这里，还是其它的情况下，我们都有可能希望通过一种统一的方式处理CallInst和InvokeInst，虽然它们的公共基类仅仅是最最基本的Instruction类。所以，LLVM提供了一个名为CallSite的包装类，它保存了一个Instruction指针，并向外提供一些CallInst和InvokeInst相同的方法。

This class has "value semantics": it should be passed by value, not by reference and it should not be dynamically allocated or deallocated using operator new or operator delete. It is efficiently copyable, assignable and constructable, with costs equivalents to that of a bare pointer. If you look at its definition, it has only a single pointer member.

__TBD__：它只能以传值的方式进行参数传递，而不能以引用方式。不能使用new或delete运算符进行动态内存分配或释放。它可以进行快速复制、赋值或创建，代价是多出一个悬挂指针，因为从它的定义可以知道，它实际上只有一个指针成员。

#### Iterating over def-use & use-def chains 通过定义-使用映射和使用-定义映射进行迭代

Frequently, we might have an instance of the Value Class and we want to determine which Users use the Value. The list of all Users of a particular Value is called a def-use chain. For example, let's say we have a Function* named F to a particular function foo. Finding all of the instructions that use foo is as simple as iterating over the def-use chain of F:

在编译器里，通过一个具体的Value实例去查找所有使用它的User是非常频繁的，这个映射关系称为定义-使用映射。举例来说，Function *F指向函数foo，要查找所有调用foo的指令，只需要简单地使用F的定义-使用映射即可：

```
Function *F = ...;

for (Value::use_iterator i = F->use_begin(), e = F->use_end(); i != e; ++i)
  if (Instruction *Inst = dyn_cast<Instruction>(*i)) {
    errs() << "F is used in instruction:\n";
    errs() << *Inst << "\n";
  }
```

Note that dereferencing a Value::use_iterator is not a very cheap operation. Instead of performing *i above several times, consider doing it only once in the loop body and reusing its result.

需要注意的是，提领Value::use_iterator的运算开销并不小，所以不要多次执行该运算，可以在循环体内运行一次后把结果保存下来，供后面语句使用。

Alternatively, it's common to have an instance of the User Class and need to know what Values are used by it. The list of all Values used by a User is known as a use-def chain. Instances of class Instruction are common Users, so we might want to iterate over all of the values that a particular instruction uses (that is, the operands of the particular Instruction):

另外，通过User实例查找它所使用的Value也非常常见，这称为使用-定义映射。以下代码是通过一个指令得到它所有的操作数：

```
Instruction *pi = ...;

for (User::op_iterator i = pi->op_begin(), e = pi->op_end(); i != e; ++i) {
  Value *v = *i;
  // ...
}
```

Declaring objects as const is an important tool of enforcing mutation free algorithms (such as analyses, etc.). For this purpose above iterators come in constant flavors as Value::const_use_iterator and Value::const_op_iterator. They automatically arise when calling use/op_begin() on const Value*s or const User*s respectively. Upon dereferencing, they return const Use*s. Otherwise the above patterns remain unchanged.

把对象声明为const，可以防止一些算法(例如分析算法等)去修改它们，所以也就有了const版的迭代器，Value::const_use_iterator和User::const_op_iterator(译注：原文为Value::const_op_iterator)。当迭代器被声明为const版本时，use/op_begin()的返回值会自动转成const Value \*s或const User \*s，如果提领这些迭代器，将得到const User \*s。这样可以保证对象不会被修改。

#### Iterating over predecessors & successors of blocks 遍历语句块的前后继

Iterating over the predecessors and successors of a block is quite easy with the routines defined in "llvm/Support/CFG.h". Just use code like this to iterate over all predecessors of BB:

使用"llvm/Support/CFG.h"中的例程，我们可以非常简单地遍历语句块的前后继。下例遍历了BB的所有前继：

```
#include "llvm/Support/CFG.h"
BasicBlock *BB = ...;

for (pred_iterator PI = pred_begin(BB), E = pred_end(BB); PI != E; ++PI) {
  BasicBlock *Pred = *PI;
  // ...
}
```

Similarly, to iterate over successors use succ_iterator/succ_begin/succ_end.

同样，使用succ_iterator/succ_begin/succ_end可以遍历所有后继。

### Making simple changes 简单的修改

There are some primitive transformation operations present in the LLVM infrastructure that are worth knowing about. When performing transformations, it's fairly common to manipulate the contents of basic blocks. This section describes some of the common methods for doing so and gives example code.

__TBD__

#### Creating and inserting new Instructions 创建并插入新指令

_Instantiating Instructions_

_初始化指令_

Creation of Instructions is straight-forward: simply call the constructor for the kind of instruction to instantiate and provide the necessary parameters. For example, an AllocaInst only requires a (const-ptr-to) Type. Thus:

创建指令的过程是很直接的：只需要调用特定指令类型的构造函数，并提供必要的参数即可。例如可以这样创建AllocaInst：

```
AllocaInst* ai = new AllocaInst(Type::Int32Ty);
```

will create an AllocaInst instance that represents the allocation of one integer in the current stack frame, at run time. Each Instruction subclass is likely to have varying default parameters which change the semantics of the instruction, so refer to the doxygen documentation for the subclass of Instruction that you're interested in instantiating.

这样就创建了一个AllocaInst指令，它表示运行时在栈上为一个整数分配空间。由于创建不同的Instruction类所需的参数可能不一样，所以在使用之前需要参考doxygen文档。

_Naming values_

_命名变量_

It is very useful to name the values of instructions when you're able to, as this facilitates the debugging of your transformations. If you end up looking at generated LLVM machine code, you definitely want to have logical names associated with the results of instructions! By supplying a value for the Name (default) parameter of the Instruction constructor, you associate a logical name with the result of the instruction's execution at run time. For example, say that I'm writing a transformation that dynamically allocates space for an integer on the stack, and that integer is going to be used as some kind of index by some other code. To accomplish this, I place an AllocaInst at the first point in the first BasicBlock of some Function, and I'm intending to use it within the same Function. I might do:

如果可以的话，给指令变量命名是非常有用的，因为可以方便调试。在查看已生成好的LLVM机器码时，你一定会希望每个指令的结果都有对应的名称！只要在调用Instruction的构造函数时提供了Name参数，这条指令执行的结果就会与此名称关联起来。例如，当我需要在栈上为一个整数动态分配空间，且该整数将会被其它代码作为某种索引使用，那么，我需要在函数的第一个BasicBlock的开始位置放置一条AllocaInst指令，这样可以在该函数的其它地方使用它了。具体代码：

```
AllocaInst* pa = new AllocaInst(Type::Int32Ty, 0, "indexLoc");
```

where indexLoc is now the logical name of the instruction's execution value, which is a pointer to an integer on the run time stack.

indexLoc现在就是该条指令结果的名称，即在栈上分配的那个整数。

_Inserting instructions_

_插入指令_

There are essentially two ways to insert an Instruction into an existing sequence of instructions that form a BasicBlock:

往指令序列中插入指令有两种方式：

* Insertion into an explicit instruction list 显式  
Given a BasicBlock* pb, an Instruction* pi within that BasicBlock, and a newly-created instruction we wish to insert before *pi, we do the following:  
给定一个BasicBlock \*pb和一个已经在pb中的Instruction \*pi，需要把一个新创建的指令插入到pi前面，我们可以这样做：

		BasicBlock *pb = ...;
		Instruction *pi = ...;
		Instruction *newInst = new Instruction(...);
		pb->getInstList().insert(pi, newInst); // Inserts newInst before pi in pb
Appending to the end of a BasicBlock is so common that the Instruction class and Instruction-derived classes provide constructors which take a pointer to a BasicBlock to be appended to. For example code that looked like:  
由于在BasicBlock的尾部添加指令也很普遍，所以Instruction及其子类的构造函数提供了一个参数，以标明该指令将被添加到哪个BasicBlock的尾部。以下代码

		BasicBlock *pb = ...;
		Instruction *newInst = new Instruction(...);

		pb->getInstList().push_back(newInst); // Appends newInst to pb
becomes:  
将变成：

		BasicBlock *pb = ...;
		Instruction *newInst = new Instruction(..., pb);
which is much cleaner, especially if you are creating long instruction streams.  
后者显得更加清楚，特别是在创建一个很长的指令流时。

* Insertion into an implicit instruction list 隐式  
Instruction instances that are already in BasicBlocks are implicitly associated with an existing instruction list: the instruction list of the enclosing basic block. Thus, we could have accomplished the same thing as the above code without being given a BasicBlock by doing:  
在BasicBlock中的指令实际上已经隐性地存入了一个已存在的指令序列中，即基本语句块的指令序列。这样，我们以下面的代码来完成与上例中同样的事情：

		Instruction *pi = ...;
		Instruction *newInst = new Instruction(...);

		pi->getParent()->getInstList().insert(pi, newInst);
In fact, this sequence of steps occurs so frequently that the Instruction class and Instruction-derived classes provide constructors which take (as a default parameter) a pointer to an Instruction which the newly-created Instruction should precede. That is, Instruction constructors are capable of inserting the newly-created instance into the BasicBlock of a provided instruction, immediately before that instruction. Using an Instruction constructor with a insertBefore (default) parameter, the above code becomes:  
事实上，插入指令的步骤出现得非常频繁，于是Instruction及其子类还提供了另外一个版本的构造函数，其参数是在这个指令的位置插入新的指令。这意味着，在创建指令时，向其提供一个已在BasicBlock中的指令，即可在此位置插入新创建的指令。采用该方式后代码将变成：

		Instruction* pi = ...;
		Instruction* newInst = new Instruction(..., pi);
which is much cleaner, especially if you're creating a lot of instructions and adding them to BasicBlocks.  
这个版本比上例更加清楚，特别是需要创建一大堆指令并需要将其放入BasicBlock中时。

#### Deleting Instructions 删除指令

Deleting an instruction from an existing sequence of instructions that form a BasicBlock is very straight-forward: just call the instruction's eraseFromParent() method. For example:

从一个已知的指令序列中删除一条指令也是很直接的：只需要调用指令的eraseFromParent()方法即可。比如：

```
Instruction *I = .. ;
I->eraseFromParent();
```

This unlinks the instruction from its containing basic block and deletes it. If you'd just like to unlink the instruction from its containing basic block but not delete it, you can use the removeFromParent() method.

这段代码解开了指令与它所在的语句块的关联，并将自己删除。若仅仅只是解除两者之间的关联，那么可以使用removeFromParent()方法。

#### Replacing an Instruction with another Value 为指令更名

_Replacing individual instructions_

_替换独立的指令_

Including "llvm/Transforms/Utils/BasicBlockUtils.h" permits use of two very useful replace functions: ReplaceInstWithValue and ReplaceInstWithInst.

"llvm/transforms/Utils/BasicBlockUtils.h"提供了两个非常有用的替换函数：ReplaceInstWithValue和ReplaceInstWithInst。

* ReplaceInstWithValue  
This function replaces all uses of a given instruction with a value, and then removes the original instruction. The following example illustrates the replacement of the result of a particular AllocaInst that allocates memory for a single integer with a null pointer to an integer.  
该函数__TBD__

		AllocaInst* instToReplace = ...;
		BasicBlock::iterator ii(instToReplace);

		ReplaceInstWithValue(instToReplace->getParent()->getInstList(), ii,
                     Constant::getNullValue(PointerType::getUnqual(Type::Int32Ty)));

* ReplaceInstWithInst  
This function replaces a particular instruction with another instruction, inserting the new instruction into the basic block at the location where the old instruction was, and replacing any uses of the old instruction with the new instruction. The following example illustrates the replacement of one AllocaInst with another.  
该函数实现指令之间的替换，它将新指令放到旧指令的位置，并__TBD__

		AllocaInst* instToReplace = ...;
		BasicBlock::iterator ii(instToReplace);

		ReplaceInstWithInst(instToReplace->getParent()->getInstList(), ii,
		                    new AllocaInst(Type::Int32Ty, 0, "ptrToReplacedInt"));
		                    
_Replacing multiple uses of Users and Values_

You can use Value::replaceAllUsesWith and User::replaceUsesOfWith to change more than one use at a time. See the doxygen documentation for the Value Class and User Class, respectively, for more information.

#### Deleting GlobalVariables

Deleting a global variable from a module is just as easy as deleting an Instruction. First, you must have a pointer to the global variable that you wish to delete. You use this pointer to erase it from its parent, the module. For example:

GlobalVariable *GV = .. ;

GV->eraseFromParent();
