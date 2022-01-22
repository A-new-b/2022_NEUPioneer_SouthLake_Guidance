# 标准C



## 宏



就我个人来看，C语言中的宏很像是一种字符处理工具。在预处理器阶段，将宏指令替换成对应的字符。

比如说，我们常用的包含头文件，我们利用如下源代码来观察：

```c
// main.c
#include <stdio.h>
```

我们用之前学到的指令，编译：

```shell
gcc main.c -E > txts
vim txts
```

可以看出，一个简单的包含头文件的预处理指令被替换成了很长一段代码。如果在 `main.c` 中写一些其他代码，结果应该是类似的。

> 我们想想，实际上这里我们没有用到任何的函数，但是它依旧将所有内容都导入了，包括那些我或许整个生涯里都不怎么会直接调用的函数。那么如果我们只想要其中一个函数，或许我们可以使用函数定义的方式，这就涉及一些后话了。不过无论怎么说，这样的导入头文件方式不是很理想，这或许就是C++里引入 `import` 和 `export` 的原因吧。

接下来，我们来尝试尝试黑魔法宏。

首先，C语言有几个内置宏，比如 `__LINE__`，`__FILE__` 这两个，一个用于表示该宏位于文件的第几行和源代码文件的名称。

接下来我们书写如下代码：

```c
// main.c
#define SAMPLE(x) __LINE__ " line in " __FILE__ x
SAMPLE("\n");
```

然后如法炮制获得预处理后的文件：

```shell
gcc main.c -E > txts
vim txts
```

可以看到文件内出现了这么几行字：

```
2 " line in " "main.c" "\n";
```

这就是宏展开后的结果。

这样操作之后，我们会发现，虽然语法和语义上，该文件都不应该能被编译，但是预处理器可以处理它。所以预处理器可以看作是在编译器前端之前工作的一个程序。它的行为仅仅是将源代码文件作为一个写满字符的文件处理，至于字符满不满足语法和语义，他不关心。

同时也一定程度上这印证我所认为的，预处理很多做的是字符的替换工作。毕竟预处理大部分都是在处理 `#define` 宏指令。

接下来，我们来使用如下两个宏指令的功能，**将内容转换成字符**和**拼接字符**。

不过，在此之前，我们先来考虑宏的展开顺序。

```c
// main.c
#define SAMPLE xx
SAMPLE
#define xx 1
SAMPLE
#define EXAMPLE xx
EXAMPLE
```

预处理后结果出现如下内容：

```

xx

1

1
```

可以看出，在宏替换中，宏的作用空间是从定义开始一直到文件底部，而定义顺序不影响他们的作用顺序。只要满足宏定义，就会尽可能的将字符替换。

那么接下来我们考虑之前说的两个宏指令。

```c
// main.c
#define EXCHANGE_TO_STRING(x) #x
#define STICK_TOGETHER(x, y) x ## y

EXCHANGE_TO_STRING(12345)

STICK_TOGETHER(xyg, 123)
```

预处理后结果出现如下内容：

```



"12345"

xyg123
```

简单说该在本例当中，一个 `#` 后面跟上一个宏变量会将宏变量转换成对应的字符串，而 `##` 会将两侧的宏变量结合在一起，而不关心两侧的空白符号。

接下来我们再做更多的测试：

```c
// main.c
#define EXCHANGE_TO_STRING(x) #x
#define STICK_TOGETHER(x, y) x ## y

EXCHANGE_TO_STRING(12345)
EXCHANGE_TO_STRING("12345")
EXCHANGE_TO_STRING(hfasjkhdfak)

STICK_TOGETHER(xyg, 123)
STICK_TOGETHER( , 123)
STICK_TOGETHER(xyz, )
STICK_TOGETHER(, )
STICK_TOGETHER("sdfa", fdsaf)
STICK_TOGETHER("sdfa", 132421)
STICK_TOGETHER("sdfa", "132421")
```

芜湖，在预处理的时候报错了。报错情况如下：

```
main.c:12:16: 错误：毗连“"sdfa"”和“fdsaf”不能给出一个有效的预处理标识符
   12 | STICK_TOGETHER("sdfa", fdsaf)
      |                ^~~~~~
main.c:2:30: 附注：in definition of macro ‘STICK_TOGETHER’
    2 | #define STICK_TOGETHER(x, y) x ## y
      |                              ^
main.c:13:16: 错误：毗连“"sdfa"”和“132421”不能给出一个有效的预处理标识符
   13 | STICK_TOGETHER("sdfa", 132421)
      |                ^~~~~~
main.c:2:30: 附注：in definition of macro ‘STICK_TOGETHER’
    2 | #define STICK_TOGETHER(x, y) x ## y
      |                              ^
main.c:14:16: 错误：毗连“"sdfa"”和“"132421"”不能给出一个有效的预处理标识符
   14 | STICK_TOGETHER("sdfa", "132421")
      |                ^~~~~~
main.c:2:30: 附注：in definition of macro ‘STICK_TOGETHER’
    2 | #define STICK_TOGETHER(x, y) x ## y
      |       
```

可以看出，`##` 用于将毗连两个内容转换成一个有效的预处理标识符，而不是简单的字符串拼接。这也可以看出，之前所说的也很片面，实际上预处理器也有自己一套规则。比如说 `##` 两侧需要能偶组成合法的预处理标识符。

我们删除不合法的部分继续预处理。

```



"12345"
"\"12345\""
"hfasjkhdfak"

xyg123
123
xyz


```

得到的结果如上面所示，非常的有趣。比如说第二个转换成字符串，就很类似于在字符串前面添加一个 `r` 或者 `R` 来说明该字符串需要用原始表达一样。而下方内容中，有一个宏变量为空的时候，就会拼接一个空的宏变量而不是报错。

看来宏里面还有很多怪怪的内容，我们再继续测试测试：

```c
#define LOG(fmt, ...) __builtin_printf(fmt, ##__VA_ARGS__)

LOG("%d", 12)
LOG("Hello")

#define FORM_NAME(i) X_##i
FORM_NAME(1)
FORM_NAME(2)

#define EXCHANGE(x) #x
#define UNPKG(x) EXCHANGE(x)
#define PI 3.1415926
#define EXAMPLE(x) UNPKG(x) EXCHANGE(x) UNPKG(PI) EXCHANGE(PI)
EXAMPLE(PI)

#define mstk_1 #__LINE__ " line in " __FILE__
mstk_1
EXCHANGE(__LINE__) " line in " __FILE__
UNPKG(__LINE__) " line in " __FILE__

#define mstk_2 __LINE__##__LINE__
mstk_2

#define mstk_3 #1231
mstk_3

#define mstk_3 123##123
mstk_3
```

这里再预处理的时候会警告，但是没有报错。

```
main.c:27: 警告：“mstk_3”重定义
   27 | #define mstk_3 123##123
      |
main.c:24: 附注：这是先前定义的位置
   24 | #define mstk_3 #1231
      |
```

很有趣的是，在只警告但是不报错的情况下，我们可以视作一种神奇的BUG特性。它不被推荐使用，但是可以用来做很多破坏性工作。这将在未来研究。

以下是结果里的内容：

```


__builtin_printf("%d", 12)
__builtin_printf("Hello")


X_1
X_2





"3.1415926" "3.1415926" "3.1415926" "PI"


 #17 " line in " "main.c"
"__LINE__" " line in " "main.c"
"19" " line in " "main.c"


__LINE____LINE__


 #1231


123123

```

实际上，这里就是宏很有趣的几个性质的展示。这些特性或许每个人都有自己的理解，有的人很讨厌，甚至抵制这些特性。有的人向初学者展示以后就直接放弃解释，并开始抨击预处理器奇怪的处理功能。

从这里我们简单总结如下：

- `##` 两侧不能出现字符串
- `##` 两侧可以都是常量，当然，不能是字符串
- 变参函数中使用的 `...` 可以为空，相对应的，应该在宏定义中使用类似 `##__VA_ARGS__` 的表达，这种表达可以在 `...` 为空的时候同时去除掉 `,`，防止函数调用的时候出问题
- `#` 的右侧应该接上宏变量，如果是常量则保留 `#` 不做任何改变
- `#` 会将后面跟的宏变元的字面转换成字符串，如果需要将某个宏的展开值转换成字符串，需要先定义另一个宏将其中的值展开以后再使用 `#` 将其转换成字符串
- 宏可以重复定义，在这个情况下，每个宏的值从第一次定义开始到下一次定义之前保持不变



















