# Exercise 8

我们丢弃了一小部分代码---即当我们在printf中指定输出"%o"格式的字符串，即八进制格式的代码。尝试去完成这部分程序。



　在这个练习中我们首先要阅读以下三个源文件的代码，弄清楚他们三者之间的关系：

　　三个文件分别为 **\kern\printf.c，\kern\console.c, \lib\printfmt.c**

　　首先大致浏览三个源文件，其中粗略的观察到3点：

　　　　　1.**\kern\printf.c**中的cprintf，vcprintf子程序调用了**\lib\printfmt.c**中的**vprintfmt**子程序。

　　　　  2.**\kern\printf.c**中的putch子程序中调用了**cputchar**，这个程序是定义在**\kern\console.c**中的。

　　　　　3.**\lib\printfmt.c**中的某些程序也依赖于**cputchar**子程序

　　　　所以得出结论，**\kern\printf.c**，**\lib\printfmt.c**两个文件的功能依赖于**\kern\console.c**的功能。所以我们就先探究一下**\kern\console.c**。



1. \kern\console.c

　　 这个文件中定义了如何把一个字符显示到console上，即我们的显示屏之上，里面包括很多对IO端口的操作。

　　 其中我们最感兴趣的自然就是**cputchar**子程序了。下面是这个程序的代码

```c
// `High'-level console I/O.  Used by readline and cprintf.

void
cputchar(int c)
{
        cons_putc(c);
}

// output a character to the console
static void
cons_putc(int c)
{
        serial_putc(c);
        lpt_putc(c);
        cga_putc(c);
}
```

在上面的代码中我发现两点，1.cputchar代码的注释中说：这个程序时最高层的console的IO控制程序，2.cputchar的实现其实是通过调用cons_putc完成的。

　　 cons_putc程序的功能在它的备注中已经被叙述的很清楚了，即输出一个字符到控制台(计算机的屏幕)。所以我们就知道了cputchar的功能也是向屏幕上输出一个字符。



2. \lib\printfmt.c

　　首先看一下这个文件刚开头的注释：

```
// Stripped-down primitive printf-style formatting routines,
// used in common by printf, sprintf, fprintf, etc.
// This code is also used by both the kernel and user programs.
```

　　"打印各种样式的字符串的子程序，经常被printf，sprintf，fprintf函数所调用，这些代码是同时被内核和用户程序所使用的。"

　　通过这个注释我们知道，这个文件中定义的子程序是我们能在编程时直接利用printf函数向屏幕输出信息的关键。

　　那么我们把目光锁定到被其他文件依赖的**vprintfmt**子程序，下面的这个版本是我已经加过注释，并且补充了一部分的版本.

```c
  1 void
  2 vprintfmt(void (*putch)(int, void*), void *putdat, const char *fmt, va_list ap)
  3 {
  4     register const char *p;
  5     register int ch, err;
  6     unsigned long long num;
  7     int base, lflag, width, precision, altflag;
  8     char padc;
  9 
 10     while (1) {
 11         while ((ch = *(unsigned char *) fmt++) != '%') {
 12             if (ch == '\0')
 13                 return;
 14             putch(ch, putdat);
 15         }
 16 
 17         // Process a %-escape sequence
 18         padc = ' ';
 19         width = -1;
 20         precision = -1;
 21         lflag = 0;
 22         altflag = 0;
 23     reswitch:
 24         switch (ch = *(unsigned char *) fmt++) {
 25 
 26         // flag to pad on the right
 27         case '-':
 28             padc = '-';
 29             goto reswitch;
 30 
 31         // flag to pad with 0's instead of spaces
 32         case '0':
 33             padc = '0';
 34             goto reswitch;
 35 
 36         // width field
 37         case '1':
 38         case '2':
 39         case '3':
 40         case '4':
 41         case '5':
 42         case '6':
 43         case '7':
 44         case '8':
 45         case '9':
 46             for (precision = 0; ; ++fmt) {
 47                 precision = precision * 10 + ch - '0';
 48                 ch = *fmt;
 49                 if (ch < '0' || ch > '9')
 50                     break;
 51             }
 52             goto process_precision;
 53 
 54         case '*':
 55             precision = va_arg(ap, int);
 56             goto process_precision;
 57 
 58         case '.':
 59             if (width < 0)
 60                 width = 0;
 61             goto reswitch;
 62 
 63         case '#':
 64             altflag = 1;
 65             goto reswitch;
 66 
 67         process_precision:
 68             if (width < 0)
 69                 width = precision, precision = -1;
 70             goto reswitch;
 71 
 72         // long flag (doubled for long long)
 73         case 'l':
 74             lflag++;
 75             goto reswitch;
 76 
 77         // character
 78         case 'c':
 79             putch(va_arg(ap, int), putdat);
 80             break;
 81 
 82         // error message
 83         case 'e':
 84             err = va_arg(ap, int);
 85             if (err < 0)
 86                 err = -err;
 87             if (err >= MAXERROR || (p = error_string[err]) == NULL)
 88                 printfmt(putch, putdat, "error %d", err);
 89             else
 90                 printfmt(putch, putdat, "%s", p);
 91             break;
 92 
 93         // string
 94         case 's':
 95             if ((p = va_arg(ap, char *)) == NULL)
 96                 p = "(null)";
 97             if (width > 0 && padc != '-')
 98                 for (width -= strnlen(p, precision); width > 0; width--)
 99                     putch(padc, putdat);
100             for (; (ch = *p++) != '\0' && (precision < 0 || --precision >= 0); width--)
101                 if (altflag && (ch < ' ' || ch > '~'))
102                     putch('?', putdat);
103                 else
104                     putch(ch, putdat);
105             for (; width > 0; width--)
106                 putch(' ', putdat);
107             break;
108 
109         // (signed) decimal
110         case 'd':
111             num = getint(&ap, lflag);
112             if ((long long) num < 0) {
113                 putch('-', putdat);
114                 num = -(long long) num;
115             }
116             base = 10;
117             goto number;
118 
119         // unsigned decimal
120         case 'u':
121             num = getuint(&ap, lflag);
122             base = 10;
123             goto number;
124 
125         // (unsigned) octal
126         case 'o':
127             // Replace this with your code.
128             putch('X', putdat);
129             putch('X', putdat);
130             putch('X', putdat);
131             break;
132 
133         // pointer
134         case 'p':
135             putch('0', putdat);
136             putch('x', putdat);
137             num = (unsigned long long)
138                 (uintptr_t) va_arg(ap, void *);
139             base = 16;
140             goto number;
141 
142         // (unsigned) hexadecimal
143         case 'x':
144             num = getuint(&ap, lflag);
145             base = 16;
146         number:
147             printnum(putch, putdat, num, base, width, padc);
148             break;
149 
150         // escaped '%' character
151         case '%':
152             putch(ch, putdat);
153             break;
154 
155         // unrecognized escape sequence - just print it literally
156         default:
157             putch('%', putdat);
158             for (fmt--; fmt[-1] != '%'; fmt--)
159                 /* do nothing */;
160             break;
161         }
162     }
163 }

```

具体子程序中大致每段代码中在做什么我在上面的代码中已经注释了，这里总结下

　　这个程序包含**4个输入参数：**

　　(1)void (*putch)(int, void*)：

　　这个参数是一个函数指针，这类函数包含两个输入参数int, void*，int参数代表一个要输出的字符的值。void* 则代表要把这个字符输出的位置的地址，但是这里void *参数的值并不是这个地址，而是这个地址的值被存放到的存储单元的地址。比如我想把一个字符值为0x30的字符('0')输出到地址0x01处，此时我们的程序应该如下图所示：

```c
1 int addr = 0x01; 
2 int ch = 0x30;
3 putch(ch, &addr);
```

之所以这样做，就是因为这个子程序能够实现，把值存放到这个地址后，地址数自动增加1，即上面的代码执行完后，0x01内存处的值变为0x30，addr的值变为0x02.

　　(2)void *putdat

　　这个参数就是输入的字符要存放在的内存地址的指针，就是和上面putch函数的第二个输入参数是一个含义。

　　(3)const char *fmt

　　这个参数代表你在编写类似于printf这种格式化输出程序时，你指定格式的字符串，即printf函数的第一个输入参数，比如printf("This is %d test", n)，这个子程序中，fmt就是"This is %d test"。

　　(4)va_list ap

　　这个参数代表的是多个输入参数，即printf子程序中从第二个参数开始之后的参数，比如("These are %d test and %d test", n, m)，那么ap指的就是n，m



那么这个函数的执行过程主要是一个while循环，分为以下几个步骤：

　　(1)(源文件中第92~96行) 首先一个一个的输出格式字符串fmt中所有'%'之前的字符，因为它们就是要直接输出的，比如"This is %d test"中的"This is "。当然如果在把这些字符一个个输出中遇到结束符'\0'，则结束输出。

　　(2)(源文件中第98~243行) 剩余的代码都是在处理'%'符号后面的格式化输出，比如是%d，则按照十进制输出对应参数。另外还有一些其他的特殊字符比如'%5d'代表显示5位，其中的5要特殊处理。具体的含义在上面的代码中有备注。

  而这个程序也是正是这个练习让我们补充的地方，在源程序的第207行~212行，这里是要处理显示八进制的格式的时候的代码：

　　我们可以参照上面显示无符号十进制的情况'u'，或者十六进制的'x'，来书写八进制的，具体原理可以看上面代码的备注，我填写代码如下：

```c
...
                // (unsigned) octal
                case 'o':
                        // Replace this with your code.
                        putch('0',putdat);
                        num = getuint(&ap, lflag);
                        base = 8;
                        goto number;

...
```

注：这个子程序里面涉及到一个非常重要的子函数va_arg()，其实与这个函数类似的还有2个，va_start()，va_end()，以及一个数据类型va_list。这个4个东西是为了计算机能够处理输入参数不固定的程序。比如下面这种程序的声明方式

　　　　　void fun(int arg_num, ...)

 　其中arg_num，代表这个程序输入参数的个数(不包含arg_num本身)，而后面的省略号则指代后续所有的输入参数，我们可以在程序中调用，如下

　　　　　fun(3, 10, 20, 30)；

 这种能够处理可变个数输入参数的功能就是有va_list, va_arg(), va_start(), va_end()来实现的，大家可以看这篇博文 http://www.cnblogs.com/justinzhang/archive/2011/09/29/2195969.html 学习一下~



　**3. \kern\printf.c**

 　　　下面查看一下最后一个文件，这个文件中定义的就是我们在编程中会用到的最顶层的一些格式化输出子程序，比如printf，sprintf等等。而在这个文件中定义了三个子程序。

　　　　 首先看一下最下面的cprintf子程序，它的输入是最接近于我们在编程中使用格式化输出子程序时的输入了，比如printf("This is %d test", n)，第一个参数为输出的格式字符串，而后面就是我要输出的一些参数。

```c
1 int
 2 cprintf(const char *fmt, ...)
 3 {
 4     va_list ap;
 5     int cnt;
 6 
 7     va_start(ap, fmt);
 8     cnt = vcprintf(fmt, ap);
 9     va_end(ap);
10 
11     return cnt;
12 }
```

 它是如何实现的呢，我们在它的内部可以看到，va_list，va_arg(), va_start(), va_end()这组操作的使用，前面我们刚刚说过，他们专门是来处理这种输入参数的个数不确定的情况。你最好还是先弄懂这个几个操作是如何配合使用的~

　　　　 在cprintf中我们发现，它利用va_list，va_arg(), va_start(), va_end()这些操作，把cprintf的fmt之后的输入参数都转换为va_list类型的一个参数，然后把fmt，和这个新生成的ap作为参数传递给vcprintf

　　　　 在vcprintf中我们发现，它就是调用了我们在上面仔细分析过的vprintfmt子程序，回顾一下，介绍vprintfmt子程序时，我们说过它有4个参数，如下

　　　　 (1)void (*putch)(int, void*)：

　　　　 这个参数是一个函数指针，这类函数包含两个输入参数int, void*，int参数代表一个要输出的字符的值。void* 则代表要把这个字符输出的位置的地址

 　　　(2)void *putdat

　   　这个参数就是输入的字符要存放在的内存地址的指针，就是和上面putch函数的第二个输入参数是一个含义。

 　　　(3)const char *fmt

　　　　这个参数代表你在编写类似于printf这种格式化输出程序时，你指定格式的字符串，即printf函数的第一个输入参数，比如printf("This is %d test", n)，这个子程序中，fmt就是"This is %d test"。

　　　　 (4)va_list ap

　　　　 这个参数代表的是多个输入参数，即printf子程序中从第二个参数开始之后的参数，比如("These are %d test and %d test", n, m)，那么ap指的就是n，m

　　　　我们可以发现，刚刚得到的fmt和ap正好可以被放在第3和第4个输入参数处！

　　　　另外再看头两个参数，第一个参数是一个函数指针，这个函数必须能够实现把一个字符输出到某个地址处的功能。再看一下vcprintf中它赋给vprintfmt子程序的第一个参数是这个文件中的第一个子程序putch。

我们再看一下这个putch程序的功能，

```c
1 static void
2 putch(int ch, int *cnt)
3 {
4     cputchar(ch);
5     *cnt++;
6 }
```

　它调用了我们最开始分析的子程序，cputchar，这个子程序可以把字符输出到屏幕上。所以这个putch子程序是满足vprintfmt子程序的要求的~可以作为参数传递给它。

　　　　最后再看第二个参数，这个参数在这里就不具备内存地址的含义了，我们看到在putch里面，它只是把字符输出给屏幕，然后把这个cnt加1，并没有把字符存放到cnt所指向的地址处，所以这个cnt就变成了一个计数器。记录已经输出了多少的字符。

