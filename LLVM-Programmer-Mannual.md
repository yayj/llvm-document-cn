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

LLVM有一些自定义模板来表示运行时类型信息(RTTI)，并在源码中广泛使用。这些模板跟C++的dynamic_cast<>操作符很相似，但却没有它的一些缺点(dynamic_cast<>只能对有v-table的类使用)。所以，了解这些模板并清楚它们的工作原理是非常有必要的。所有的模板的定义都在文件[llvm/Support/Casting.h]("http://llvm.org/doxygen/Casting_8h-source.html")中，不过你通常不用直接include它。

isa<>:
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

dyn_cast<>:
The dyn_cast<> operator is a "checking cast" operation. It checks to see if the operand is of the specified type, and if so, returns a pointer to it (this operator does not work with references). If the operand is not of the correct type, a null pointer is returned. Thus, this works very much like the dynamic_cast<> operator in C++, and should be used in the same circumstances. Typically, the dyn_cast<> operator is used in an if statement or some other flow control statement like this:

* dyn_cast<>: dyn_cast<>是一个带检查的类型转换，它先检查操作数的类型是否匹配，若匹配则返回类型转换后的指针(该操作符不能用于引用)，否则返回空指针。由此可见，它和C++中的dynamic_cast<>非常像，可以用于相同的场景。通常来说，dyn_cast<>一般用在类似于以下例子中的流程控制语句中：

		if (AllocationInst *AI = dyn_cast<AllocationInst>(Val)) {
		  // ...
		}
		This form of the if statement effectively combines together a call to isa<> and a call to cast<> into one statement, which is very convenient.
Note that the dyn_cast<> operator, like C++'s dynamic_cast<> or Java's instanceof operator, can be abused. In particular, you should not use big chained if/then/else blocks to check for lots of different variants of classes. If you find yourself wanting to do this, it is much cleaner and more efficient to use the InstVisitor class to dispatch over the instruction type directly.

**注意**: dyn_cast<>可能会像C++的dynamic_cast<>和Java中的instanceof一样被滥用。通常来说，我们不应该用一长串if/then/else来进行变量的类型判断。如果非如此不可，那么使用InstVisitor类来实现…

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

可以使用C风格的0结尾字符串或std::string来隐式创建一个StringRef，也用一个字符指针和长度来显式创建。例如，假设这是查找字符串的函数的申明：

`iterator find(StringRef Key);`

and clients can call it using any one of:

那么调用者可以用以下形式调用它：

	  Map.find("foo");                 // 查找"foo"
	  Map.find(std::string("bar"));    // 查找"bar"
	  Map.find(StringRef("\0baz", 4)); // 查找"\0baz"
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

	$ opt < a.bc > /dev/null -mypass
	<no output>
	$ opt < a.bc > /dev/null -mypass -debug
	I am here!
Using the DEBUG() macro instead of a home-brewed solution allows you to not have to create "yet another" command line option for the debug output for your pass. Note that DEBUG() macros are disabled for optimized builds, so they do not cause a performance impact at all (for the same reason, they should also not contain side-effects!).

用DEBUG()宏取代其它的方式，则可以不用在命令行上使用其它参数来控制输出。值得注意的是，使用优化参数编译程序时，DEBUG()宏会被禁用，所以完全不用担心它会影响程序性能(同样，它们也不会导致[副效应]("http://en.wikipedia.org/wiki/Side_effect_(computer_science)"))。

One additional nice thing about the DEBUG() macro is that you can enable or disable it directly in gdb. Just use "set DebugFlag=0" or "set DebugFlag=1" from the gdb if the program is running. If the program hasn't been started yet, you can always just run it with -debug.

DEBUG()宏的另一个好处是，在gdb中运行程序时，可以直接使用"set DebugFlag=0"或"set DebugFlag=1"来开启或关闭。如果没有运行时，使用-debug参数也可以开启调试信息。

#### 用DEBUG_TYPE和-debug-only选项整理调试信息

Sometimes you may find yourself in a situation where enabling -debug just turns on too much information (such as when working on the code generator). If you want to enable debug information with more fine-grained control, you define the DEBUG_TYPE macro and the -debug only option as follows:

有时候我们发现开启-debug选项后，会产生过多的调试信息(例如在开发代码生成器时)。如果使用DEBUG_TYPE宏和-debug-only选项对调试信息进行分类，则可以很好地整理这些调试信息了：

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
Then you can run your pass like this:

之后可以这样运行程序：

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
Of course, in practice, you should only set DEBUG_TYPE at the top of a file, to specify the debug type for the entire module (if you do this before you #include "llvm/Support/Debug.h", you don't have to insert the ugly #undef's). Also, you should use names more meaningful than "foo" and "bar", because there is no system in place to ensure that names do not conflict. If two different modules use the same string, they will all be turned on when the name is specified. This allows, for example, all debug information for instruction scheduling to be enabled with -debug-type=InstrSched, even if the source lives in multiple files.

当然，在实际操作当中，只需要在文件开始设置DEBUG_TYPE宏即可，这样可以让它对整个模块都生效(前提是要把它放在#include "llvm/Support/Debug.h"之前)。由于LLVM没有为DEBUG_TYPE提供冲突检测机制，所以我们应该使用比"foo"或"bar"更有意义的分类名。如果两个不同的模块使用了相同的分类名，则它们的调试信息会同时输出。...

The DEBUG_WITH_TYPE macro is also available for situations where you would like to set DEBUG_TYPE, but only for one specific DEBUG statement. It takes an additional first parameter, which is the type to use. For example, the preceding example could be written as:

如果只想为单独的一句设置分类名，则可以使用DEBUG_WITH_TYPE宏，该宏的第一个参数即为分类名。上一个例子可以用该宏简化成如下：

	DEBUG_WITH_TYPE("", errs() << "No debug type\n");
	DEBUG_WITH_TYPE("foo", errs() << "'foo' debug type\n");
	DEBUG_WITH_TYPE("bar", errs() << "'bar' debug type\n"));
	DEBUG_WITH_TYPE("", errs() << "No debug type (2)\n");

#### Statistic类和-stats选项

The "[llvm/ADT/Statistic.h]("http://llvm.org/doxygen/Statistic_8h-source.html")" file provides a class named Statistic that is used as a unified way to keep track of what the LLVM compiler is doing and how effective various optimizations are. It is useful to see what optimizations are contributing to making a particular program run faster.

"[llvm/ADT/Statistic.h]("http://llvm.org/doxygen/Statistic_8h-source.html")"声明了Statistic类，它提供了一种统一的方式对LLVM编译过程和优化工作进行统计。通过Statistic类可以方便地看到编译器对程序做了哪些优化。

Often you may run your pass on some big program, and you're interested to see how many times it makes a certain transformation. Although you can do this with hand inspection, or some ad-hoc method, this is a real pain and not very useful for big programs. Using the Statistic class makes it very easy to keep track of this information, and the calculated information is presented in a uniform manner with the rest of the passes being executed.



There are many examples of Statistic uses, but the basics of using it are as follows: