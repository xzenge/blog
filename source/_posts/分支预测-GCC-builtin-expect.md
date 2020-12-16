---
title: 分支预测 GCC __builtin_expect 有用吗？
date: 2020-12-14 22:57:18
tags: 
    - C
    - GCC
    - 代码优化
categories :
    - Technology
---

我们日常编写的代码中常见的分支语句，解释行语言能够根据代码运行期间收集分支的跳转的趋势动态优化分支跳转。而编译言语由于编译后直接生成可执行文件，因此很难在编译期推断出最好的优化条件。GCC为开发者提供了手动优化分支预测的方法__builtin_expect，通过显示编程向编译器提供优化建议从而优化分支的走向。
<!-- more -->
编译器是如何处理分支条件的？使用__builtin_expect的分支是如何得到优化的？使用GCC默认的优化级别会有什么不同？

为了解开这些疑惑，我们直接上代码。

* 平台 Ubuntu 20.04.1 LTS
* 内核 Linux b13a1870ebc6 5.4.39-linuxkit #1 SMP Fri May 8 23:03:06 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
* 编译工具 gcc (Ubuntu 9.3.0-17ubuntu1~20.04) 9.3.0

``` C
#include <stdio.h>
#include <stdlib.h>

//x很可能为true
#define likely(x) __builtin_expect(!!(x), 1)  
//x很可能为false
#define unlikely(x) __builtin_expect(!!(x), 0)

void foo(){
    printf("foo\n");
}
void foo1(){
    printf("foo1\n");
}

void doLikely(int a){
    
    if(likely(a)){
        foo();    
    }else {
        foo1();
    }
}

void doLikelyNocall(int a){
    int x;
    if(likely(a)){
        x = a*a;
    }else {
        x = a+a;
    }
}

void doUnlikely(int a){
    
    if(unlikely(a)){
        foo();
    }else {
        foo1();
    }
}

void doUnlikelyNocall(int a){
    int x;
    if(unlikely(a)){
        x = a*a;
    }else {
        x = a+a;
    }
}

void normal(int a){
    
    if(a){
        foo();
    }else {
        foo1();
    }
}

void normalNocall(int a){
    int x;
    if(a){
        x = a*a;
    }else {
        x = a+a;
    }
}


int main(int argc, char **argv){
    //为了排除常量导致编译器的优化，这里使用不定参数作为入参，防止编译器优化相关代码
    int a = atoi(argv[1]);
    doLikely(a);
    doUnlikely(a);
    normal(a);
    doLikelyNocall(a);
    doUnlikelyNocall(a);
    normalNocall(a);
    return 0;
}
```

开始编辑，这里显示的指定-O0 -O1 -O2 -O3这四个优化等级。
``` bash
root@b13a1870ebc6:/workspace_c/builtin_expect# gcc -O0 -o test_0 builtin_expect.c
root@b13a1870ebc6:/workspace_c/builtin_expect# gcc -O1 -o test_1 builtin_expect.c
root@b13a1870ebc6:/workspace_c/builtin_expect# gcc -O2 -o test_2 builtin_expect.c
root@b13a1870ebc6:/workspace_c/builtin_expect# gcc -O3 -o test_3 builtin_expect.c
```

将编译后的目标文件反汇编
``` bash
root@b13a1870ebc6:/workspace_c/builtin_expect# objdump -d test_0 >test_0.a
root@b13a1870ebc6:/workspace_c/builtin_expect# objdump -d test_1 >test_1.a
root@b13a1870ebc6:/workspace_c/builtin_expect# objdump -d test_2 >test_2.a
root@b13a1870ebc6:/workspace_c/builtin_expect# objdump -d test_3 >test_3.a
```

最终我们得到了4个优化等级不同的汇编代码，并依次对比相关逻辑。

1.-O0等级下，编译器默认不执行任何优化，我们对比原始分支代码和经过__builtin_expect优化后的代码在汇编码上有何区别。

``` x86asm
0000000000001197 <doLikely>:
    1197:	f3 0f 1e fa          	endbr64 
    119b:	55                   	push   %rbp
    119c:	48 89 e5             	mov    %rsp,%rbp
    119f:	48 83 ec 10          	sub    $0x10,%rsp
    11a3:	89 7d fc             	mov    %edi,-0x4(%rbp)      #参数a存放至-0x4(%rbp)处
    11a6:	83 7d fc 00          	cmpl   $0x0,-0x4(%rbp)      #参数a和0比较
    11aa:	0f 95 c0             	setne  %al                  #条件设置，al中保存~ZF。if a = 1 then al = 1 ; if a = 0 then al = 0
    11ad:	0f b6 c0             	movzbl %al,%eax             #eax = 0000000000000000 -> 00000000000000000000000000000000 or 0000000000000001 -> 00000000000000000000000000000001
    11b0:	48 85 c0             	test   %rax,%rax            #判断eax是否为0
    11b3:	74 0c                	je     11c1 <doLikely+0x2a> #eax = 0(a = 0)时，跳转至11c1，call foo1
    11b5:	b8 00 00 00 00       	mov    $0x0,%eax
    11ba:	e8 aa ff ff ff       	callq  1169 <foo>           #eax!=0(a = 0)时，call foo
    11bf:	eb 0a                	jmp    11cb <doLikely+0x34> #跳转至11cb，结束
    11c1:	b8 00 00 00 00       	mov    $0x0,%eax
    11c6:	e8 b5 ff ff ff       	callq  1180 <foo1>
    11cb:	90                   	nop
    11cc:	c9                   	leaveq 
    11cd:	c3                   	retq   


0000000000001265 <normal>:
    1265:	f3 0f 1e fa          	endbr64 
    1269:	55                   	push   %rbp
    126a:	48 89 e5             	mov    %rsp,%rbp
    126d:	48 83 ec 10          	sub    $0x10,%rsp
    1271:	89 7d fc             	mov    %edi,-0x4(%rbp)      #参数a存放至-0x4(%rbp)处
    1274:	83 7d fc 00          	cmpl   $0x0,-0x4(%rbp)      #参数a和0比较
    1278:	74 0c                	je     1286 <normal+0x21>   #如果a=0，则跳转至1286，执行foo1
    127a:	b8 00 00 00 00       	mov    $0x0,%eax
    127f:	e8 e5 fe ff ff       	callq  1169 <foo>           #如果a!=0,则执行foo
    1284:	eb 0a                	jmp    1290 <normal+0x2b>   #跳转至1290，结束
    1286:	b8 00 00 00 00       	mov    $0x0,%eax
    128b:	e8 f0 fe ff ff       	callq  1180 <foo1>
    1290:	90                   	nop
    1291:	c9                   	leaveq 
    1292:	c3                   	retq   
```

通过对比，结果令人惊讶。虽然执行结果一直，但是没有经过_builtin_expect优化的分支判断，反而比标识_builtin_expect的汇编代码更加简洁。

分析Unlikely的汇编代码，也存在同样的问题，编译器貌似会多此一举增加一步的判断
``` x86asm
00000000000011fe <doUnlikely>:
    11fe:	f3 0f 1e fa          	endbr64 
    1202:	55                   	push   %rbp
    1203:	48 89 e5             	mov    %rsp,%rbp
    1206:	48 83 ec 10          	sub    $0x10,%rsp
    120a:	89 7d fc             	mov    %edi,-0x4(%rbp)
    120d:	83 7d fc 00          	cmpl   $0x0,-0x4(%rbp)
    1211:	0f 95 c0             	setne  %al
    1214:	0f b6 c0             	movzbl %al,%eax
    1217:	48 85 c0             	test   %rax,%rax
    121a:	74 0c                	je     1228 <doUnlikely+0x2a>
    121c:	b8 00 00 00 00       	mov    $0x0,%eax
    1221:	e8 43 ff ff ff       	callq  1169 <foo>
    1226:	eb 0a                	jmp    1232 <doUnlikely+0x34>
    1228:	b8 00 00 00 00       	mov    $0x0,%eax
    122d:	e8 4e ff ff ff       	callq  1180 <foo1>
    1232:	90                   	nop
    1233:	c9                   	leaveq 
    1234:	c3                   	retq   
```

此外，我们也写了个非内部调用的版本，结果也是同样。编译器没有对_builtin_expect做出不一样的优化。

``` x86asm
00000000000011ce <doLikelyNocall>:
    11ce:	f3 0f 1e fa          	endbr64 
    11d2:	55                   	push   %rbp
    11d3:	48 89 e5             	mov    %rsp,%rbp
    11d6:	89 7d ec             	mov    %edi,-0x14(%rbp)
    11d9:	83 7d ec 00          	cmpl   $0x0,-0x14(%rbp)
    11dd:	0f 95 c0             	setne  %al
    11e0:	0f b6 c0             	movzbl %al,%eax
    11e3:	48 85 c0             	test   %rax,%rax
    11e6:	74 0b                	je     11f3 <doLikelyNocall+0x25>
    11e8:	8b 45 ec             	mov    -0x14(%rbp),%eax
    11eb:	0f af c0             	imul   %eax,%eax
    11ee:	89 45 fc             	mov    %eax,-0x4(%rbp)
    11f1:	eb 08                	jmp    11fb <doLikelyNocall+0x2d>
    11f3:	8b 45 ec             	mov    -0x14(%rbp),%eax
    11f6:	01 c0                	add    %eax,%eax
    11f8:	89 45 fc             	mov    %eax,-0x4(%rbp)
    11fb:	90                   	nop
    11fc:	5d                   	pop    %rbp
    11fd:	c3                   	retq   
```

2.为了了解_builtin_expect是否真的有用，我们再看看其他优化等级 -O1。

``` x86asm
000000000000119b <doLikely>:
    119b:	f3 0f 1e fa          	endbr64 
    119f:	48 83 ec 08          	sub    $0x8,%rsp
    11a3:	85 ff                	test   %edi,%edi
    11a5:	74 0f                	je     11b6 <doLikely+0x1b>
    11a7:	b8 00 00 00 00       	mov    $0x0,%eax
    11ac:	e8 b8 ff ff ff       	callq  1169 <foo>
    11b1:	48 83 c4 08          	add    $0x8,%rsp
    11b5:	c3                   	retq   
    11b6:	b8 00 00 00 00       	mov    $0x0,%eax
    11bb:	e8 c2 ff ff ff       	callq  1182 <foo1>
    11c0:	eb ef                	jmp    11b1 <doLikely+0x16>

00000000000011f3 <normal>:
    11f3:	f3 0f 1e fa          	endbr64 
    11f7:	48 83 ec 08          	sub    $0x8,%rsp
    11fb:	85 ff                	test   %edi,%edi
    11fd:	74 0f                	je     120e <normal+0x1b>
    11ff:	b8 00 00 00 00       	mov    $0x0,%eax
    1204:	e8 60 ff ff ff       	callq  1169 <foo>
    1209:	48 83 c4 08          	add    $0x8,%rsp
    120d:	c3                   	retq   
    120e:	b8 00 00 00 00       	mov    $0x0,%eax
    1213:	e8 6a ff ff ff       	callq  1182 <foo1>
    1218:	eb ef                	jmp    1209 <normal+0x16>

00000000000011ee <doUnlikelyNocall>:
    11ee:	f3 0f 1e fa          	endbr64 
    11f2:	c3                   	retq   
```

观察-O1优化后的汇编代码，代码风格有了质的飞越。去除了公式化的参数传递，费劲的值比较。但是依然没有体现出_builtin_expect有没有的区别。
同样的，我们的非调用版本由于无返回值，已经被编译器完全优化掉了。

我们再看看-O2。

``` x86asm
0000000000001210 <doLikely>:
    1210:	f3 0f 1e fa          	endbr64 
    1214:	85 ff                	test   %edi,%edi
    1216:	74 10                	je     1228 <doLikely+0x18>
    1218:	48 8d 3d e5 0d 00 00 	lea    0xde5(%rip),%rdi        # 2004 <_IO_stdin_used+0x4>
    121f:	e9 3c fe ff ff       	jmpq   1060 <puts@plt>
    1224:	0f 1f 40 00          	nopl   0x0(%rax)
    1228:	48 8d 3d d9 0d 00 00 	lea    0xdd9(%rip),%rdi        # 2008 <_IO_stdin_used+0x8>
    122f:	e9 2c fe ff ff       	jmpq   1060 <puts@plt>
    1234:	66 66 2e 0f 1f 84 00 	data16 nopw %cs:0x0(%rax,%rax,1)
    123b:	00 00 00 00 
    123f:	90                   	nop

0000000000001290 <normal>:
    1290:	f3 0f 1e fa          	endbr64 
    1294:	85 ff                	test   %edi,%edi
    1296:	74 10                	je     12a8 <normal+0x18>
    1298:	48 8d 3d 65 0d 00 00 	lea    0xd65(%rip),%rdi        # 2004 <_IO_stdin_used+0x4>
    129f:	e9 bc fd ff ff       	jmpq   1060 <puts@plt>
    12a4:	0f 1f 40 00          	nopl   0x0(%rax)
    12a8:	48 8d 3d 59 0d 00 00 	lea    0xd59(%rip),%rdi        # 2008 <_IO_stdin_used+0x8>
    12af:	e9 ac fd ff ff       	jmpq   1060 <puts@plt>
    12b4:	66 66 2e 0f 1f 84 00 	data16 nopw %cs:0x0(%rax,%rax,1)
    12bb:	00 00 00 00 
    12bf:	90                   	nop
```

代码进一步的被优化，从汇编代码中可以看出，方法调用完全优化掉了栈上调用。去除了栈上传递，将call指令变为了jmp指令，进一步减少了指令上的开销。
但结果仍然没有改变，_builtin_expect依然没有发光发热。

最后我们只能把希望寄托于-O3上看看了。

``` x86asm
0000000000001210 <doLikely>:
    1210:	f3 0f 1e fa          	endbr64 
    1214:	85 ff                	test   %edi,%edi
    1216:	74 10                	je     1228 <doLikely+0x18>
    1218:	48 8d 3d e5 0d 00 00 	lea    0xde5(%rip),%rdi        # 2004 <_IO_stdin_used+0x4>
    121f:	e9 3c fe ff ff       	jmpq   1060 <puts@plt>
    1224:	0f 1f 40 00          	nopl   0x0(%rax)
    1228:	48 8d 3d d9 0d 00 00 	lea    0xdd9(%rip),%rdi        # 2008 <_IO_stdin_used+0x8>
    122f:	e9 2c fe ff ff       	jmpq   1060 <puts@plt>
    1234:	66 66 2e 0f 1f 84 00 	data16 nopw %cs:0x0(%rax,%rax,1)
    123b:	00 00 00 00 
    123f:	90                   	nop

0000000000001290 <normal>:
    1290:	f3 0f 1e fa          	endbr64 
    1294:	85 ff                	test   %edi,%edi
    1296:	74 10                	je     12a8 <normal+0x18>
    1298:	48 8d 3d 65 0d 00 00 	lea    0xd65(%rip),%rdi        # 2004 <_IO_stdin_used+0x4>
    129f:	e9 bc fd ff ff       	jmpq   1060 <puts@plt>
    12a4:	0f 1f 40 00          	nopl   0x0(%rax)
    12a8:	48 8d 3d 59 0d 00 00 	lea    0xd59(%rip),%rdi        # 2008 <_IO_stdin_used+0x8>
    12af:	e9 ac fd ff ff       	jmpq   1060 <puts@plt>
    12b4:	66 66 2e 0f 1f 84 00 	data16 nopw %cs:0x0(%rax,%rax,1)
    12bb:	00 00 00 00 
    12bf:	90                   	nop
```

可见，O3已经没有任何优化空间了，_builtin_expect的存在感也已然是0。

这次的实验是否失败呢，结果上看起来的确如此。但是我们并没有尝试更多的平台，同时各个平台的版本或GCC的版本都可能对编译的结果造成不同的影响。
_builtin_expect到底有没有用？我能确信的是，再更加激进的优化等级下，_builtin_expect的存在感必然是稀薄的。但是GCC在遇到_builtin_expect时具体做了些什么，希望某一天阅读完GCC相关的源码后找到一个满意的答案吧！~
