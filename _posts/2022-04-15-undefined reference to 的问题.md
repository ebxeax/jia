\## 引言

我们在Linux下用C/C++工作的时候，经常会遇到"undefined reference to XXX"的问题，直白地说就是在**链接**(从.cpp源代码到可执行的ELF文件，要经过预处理->编译->链接三个阶段，此时预处理和编译已经通过了)的时候，链接器找不到XXX这个函数的定义了。这个问题在网上随便搜搜就有很多网页提供解决思路，要么是错的，要么不全面，要么只给结果没有具体分析思路。偶尔没头脑也可以轻易解决，但有的时候又隐藏的很深很细，需要花很长时间去排查。船长这里通过几个小例子，试着总结一下"undefined reference to XXX"问题的直接原因和解决方法，以后大家遇到这个问题，可以按照本文一条一条Check，可以节省时间，提高工作效率。背后的深层原因牵扯到编译，链接的基础知识、计算机原理和Linux系统环境，先挖个坑，以后有时间再专门写文来埋。



**欢迎探讨，本文持续维护。**



\## 实验平台：



\+ 操作系统：Ubuntu 16.04 LTS，Ubuntu 18.04 LTS

\+ 编译器：g++ (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609



　　　　gcc (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609



\### Case 1, 链接时缺少定义了XXX的源文件或者目标文件或者库文件



实验代码：

![img](/images/v2-21f175a4a5d61ea213eb00ada674a91f_720w.jpg)

\+ 1.1 缺源文件。下面显示的是首先直接用g++编译main.cpp，出现了"undefined reference to foo()"的问题，未能编译出可执行程序a.out；然后在编译命令行加上foo.cpp(foo函数的定义文件)后，成功编译出a.out，而且执行起来更是非常顺滑～

![img](/images/v2-822709be74e850959deb92c523e077cd_720w.jpg)

\+ 1.2 缺目标文件。同样，首先把foo.cpp编译成目标文件foo.o之后，也可以用foo.o来编译，也可以解决"undefined reference to XXX"的问题：

![img](/images/v2-51180c1c46df48fc72168e54d335295d_720w.jpg)



\+ １.3 缺库文件。最后，把foo.cpp编译成动态库（静态库一样的）[文件foo.so](https://link.zhihu.com/?target=http%3A//xn--foo-tu9dq54h.so/)，我们来试试：



![img](/images/v2-cadace7476d5ec4e70058dc36dece1da_720w.jpg)



\### Case 2, 链接顺序不对



接上节1.3，请看下面的代码：

![img](/images/v2-90fc7952a69eb17462642b91ca896853_720w.jpg)



这段代码与1.3节相比，只是调换了foo.so库文件和main.cpp文件的前后顺序，就会出现"undefined reference to XXX"的问题。原因在[gcc手册]([Using the GNU Compiler Collection (GCC): Link Options](https://link.zhihu.com/?target=https%3A//gcc.gnu.org/onlinedocs/gcc-5.5.0/gcc/Link-Options.html%23Link-Options))里面有提及，*It makes a difference where in the command you write this option; the linker searches and processes libraries and object files in the order they are specified. Thus, ‘foo.o -lz bar.o’ searches library ‘z’ after file foo.o but before bar.o. **If bar.o refers to functions in ‘z’, those functions may not be loaded***。不难懂就不翻译了，总之在给编译器输入源文件，目标文件或者动态库静态库文件时，如果B文件依赖A文件中的内容，那么B文件应该放在A文件的**左边**。这个问题很隐蔽，而且莫名其妙，权且当个结论记住吧。



\### Case 3, 函数符号修饰不一样



先说一下符号修饰（Name Mangling），函数从源代码编译到目标文件时，函数在目标文件中的名字是会改变的（这个改变的规则是编译器厂商定的，一般会包含函数参数列表信息、name space信息等），后面在链接阶段，链接器是按照函数改变后的名字来索引函数的实现机器码，函数改变后的名字不一样的函数，在链接器看来就是不一样的函数。至于为什么要有符号修饰这个机制，能不能直接用函数名来在目标文件内表示不同函数呢？答案是，能，而且在远古时期，就是这么干的。但是随着语言各种特性的发展，渐渐地就不能这么简单粗暴了，继续这样搞很容易造成符号冲突，而符号修饰在函数名上加上了名字空间信息和参数列表等信息，就可以允许不同名字空间里面的同名函数不冲突，允许函数重载机制。



啰嗦这么多，记住一句话，链接器看到的名字和要找的名字，是经过符号修饰之后的名字，和你在源代码里看到的不一样。至于怎么看链接器要找的名字，和怎么看目标文件和库文件里有的名字，后面我会讲。



下面我们具体列举一下，函数签名不一样（导致符号修饰不一样，继而导致链接器懵逼报"undefined reference to XXX"）导致的"undefined reference to XXX"错误和解决方法。



\+ 3.1 函数定义和声明不一致



这个很好理解，我们在写代码时，都是用.cpp文件写函数的定义，用.h头文件来写函数的接口声明来实现和接口的分离（那些写#include "xxx.cpp"的牲口，应该祭天～）。但是如果我们在.h文件里面写的声明和.cpp文件里面写的函数定义不一样的话，那就会出"undefined reference to XXX"这个问题，实验代码请看：

![img](/images/v2-8d1b11c9dd97d7c7703214a855449cfb_720w.jpg)



在bar.h文件内bar函数的声明是void bar(void)，在实现文件内实现是void bar(int)，参数列表变了。



编译一下：



![img](https://pic3.zhimg.com/80/v2-f39670eff80cbb49e9285fadc9fc569a_720w.jpg)



怎么解决？把声明和定义整一致了就行，很简单就不赘述了。



\+ 3.2 C和C++混合编程



如果我们的开发的代码用的是C++语言，但是有些第三方库或者代码，用的是C语言编写编译，在应用C语言的库的时候，就要加上extern "C"，告诉C++的编译器按照C语言的符号修饰规则去找这些符号。



这是实验代码，baz.c是C源文件，里面定义了函数void baz(void)，在C++文件main.cpp里面，引用到了这个函数

![img](/images/v2-afb62cfa60d21137d20889f6e011b645_720w.jpg)



先用gcc编译C文件baz.c得到目标文件baz.o，然后用g++编译C++文件main.cpp和baz.o，就出现了错误：

![img](/images/v2-6c12000ce808b63c16e0518cf929eaa3_720w.jpg)



解决方案，加上extern "C"，就可以了：

![img](/images/v2-283e16a9a73b8fb1b63ace00e63a66e8_720w.jpg)



\+ 3.3 编译器版本或者编译器选项不一致



不同的编译器版本，或者编译器版本和链接器版本不一致，用的运行时库不一样，都不一定兼容；编译器选项不一样，也可能会影响到函数签名。在开发时，最好每个技术团队内都每个程序员都用一套标准的编译器和编译器选项配置。



\### Case 4，把模板函数写进了cpp文件中



如果把模板函数写进了实现文件.cpp中，那么编译器就会认为这是一个独立的编译单元进行编译。然而，因为它是模板函数，编译器不能确定到底要将它特化到哪个实现(是特化成int的，还是float的，还是别的什么类型)，编译器也不会搜索你的整个工程里面别的cpp来确定特化什么类型(要搜索得话，万一你有10000000个.cpp呢？)，所以干脆就没有特化，所以你在链接的时候就找不到该函数的定义了，也就是"undefined reference to XXX"。这个情况发生不多，就不单独介绍了，[imred]([https://blog.csdn.net/imred/article/details/80261632](https://link.zhihu.com/?target=https%3A//blog.csdn.net/imred/article/details/80261632))这里写的生动详细，还附送了一个关于编译时弱符号(.weak)的知识点，感兴趣的可以看这里。



\### Case 5，api hinden



如果把动态库libA.so的确实现了foo(...)，但是在编译成so的时候使用了-fvisibility=hidden类似的开关，那么使用者也是链接不了的（但是nm一下还是看得到），也会出现undefined reference to foo(...)这样的报错。代码实验过程请参考[D#0006]([Captain1986/CaptainBlackboard](https://link.zhihu.com/?target=https%3A//github.com/Captain1986/CaptainBlackboard/blob/master/D%230006-protect_my_function/D%230006.md))的Case 1, api hinden节。这种情况常见于一些开源库里面的不兼容，前面的版本这个api是可以用的，库升级后，这个版本变成了hinden了。





\## 杂项

有时候我们使用第三方的库文件，又找不到相应的文档；或者我们查到了函数修饰后的名字，但是太晦涩难懂，怎么翻译成源代码里面的函数声明呢？

\+ 怎么查看动态库文件里面提供了哪些函数符号？

首先，我们把bar.cpp文件编译成动态库bar.so文件，这个文件提供了函数void bar(int)的实现

![img](/images/v2-41a9ff1bd18b24571bc2b1c7f19ebbe5_720w.jpg)



要查看bar.so这个库里面提供了哪些函数符号，可以用nm命令



![img](/images/v2-77ca1a8c7aeff379459d2c9104312877_720w.jpg)



这里我们可以看到，在bar.so文件的代码段（.Text段），有函数_Z3bari的定义。这个_Z3bari就是bar(int)这个函数在名字修饰后在库文件中的名字，链接器就是看这个名字来进行链接活动。nm -C可以查看函数名字修饰之前的名字，大家可以试试看。



\+ 知道了函数修饰后的名字，怎么推函数声明？



那么，如果我们知道了某个库里面的函数修饰后的名字，有没有什么方法可以反推函数源文件中的声明呢？用c++filt这个命令就可以。还是以上文中_Z3bari为例：

![img](/images/v2-67da7c424fbc66ec761607bcc1d56baa_720w.jpg)



可以看到，_Z3bari这个符号是bar(int)这个函数修饰而成。可能有人会问，为什么不是void bar(int)，是不是由于是void型的返回值，所以略去了？这里要注意，函数修饰不会管函数的返回值是什么。函数的返回值不同，而函数名和参数列表一样，经过函数修饰产生的符号是一样的，这也为什么C++规定，函数只有返回值不一样的话，是不能算重载的原因。



\+ 题外话，有时候我们不愿意让别人看到我们发布的库文件的符号表（泄露内部实现信息），我们也可以将符号信息从库文件中删除（strip命令，还有-fvisibility=hidden)。



\## 总结



以上差不多就是"undefined reference to XXX"的这个问题的常见原因和解决方案了，总结起来就是三点：１．是不是编译器找不到定义了XXX的文件；２．是不是定义了XXX的文件，由于函数修饰的原因里面没有想要的XXX符号;3．找到了想要的符号，但是该符号是隐藏属性，不能链接使用。如果不确定库里面有没有这个XXX符号，用nm找，用c++filt可以从修饰后的符号找函数声明。



\## 参考资料

\+ [《How to write a shared library》]([https://www.akkadia.org/drepper/dsohowto.pdf](https://link.zhihu.com/?target=https%3A//www.akkadia.org/drepper/dsohowto.pdf))

\+ [https://www.cprogramming.com/tutorial/shared-libraries-linux-gcc.html](https://link.zhihu.com/?target=https%3A//www.cprogramming.com/tutorial/shared-libraries-linux-gcc.html)

\+ [《程序员的自我修养：链接、装载与库》]([程序员的自我修养 (豆瓣)](https://link.zhihu.com/?target=https%3A//book.douban.com/subject/3652388/))

\+ [《深入理解计算机系统》]([深入理解计算机系统 (豆瓣)](https://link.zhihu.com/?target=https%3A//book.douban.com/subject/1896753/))

\+ [关于模板函数为什么要写进.h文件的解释]([https://blog.csdn.net/imred/article/details/80261632](https://link.zhihu.com/?target=https%3A//blog.csdn.net/imred/article/details/80261632))