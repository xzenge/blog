---
title: 自己动手写Java虚拟机 笔记1
date: 2019-09-09 22:18:50
tags: 
    - JVM
    - Java
    - Go
categories :
    - Technology
---

## 环境配置
1. 安装Golang
   
{% blockquote %}
https://golang.google.cn/
{% endblockquote %}

2. 设置Go环境变量

{% asset_img gopath.png GOPATH %}

同时要注意，go的工作目录为该目录，如果需要切换需要改变此路径，不然可使用GoLand等ide切换。
<!-- more -->

## 简单实现java命令
java SDK自带很多命令，如java、javac、javap等。
java命令是其中最终要的一个，平时我们启动一个jar，如springboot工程。还是一个基于tomcat、weblogic、jboss的war包。其根本都是利用java命令执行main方法为入口。

{% codeblock %}
java [-options] class [args]
java [-options] -jar jarfile [args]
javaw [-options] class [args]
javaw [-options] -jar jarfile [args]
{% endcodeblock %}

{% blockquote %}
官方说明：  https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html
{% endblockquote %}

## 用Go实现java命令的参数读取
1. java命令入参结构

{% codeblock lang:go %}
type Cmd struct {
	helpFlag bool //命令为help 或 ? 时：true
	versionFlag bool //命令是否为version时：true
	cpOption string //命令为cp 或 classpath时，保存值
	XjreOption string //命令为Xjre时，保存值
	class string //保存class
	args []string //保存其他入参
}
{% endcodeblock %}

2. 解析入参

{% codeblock lang:go %}
func parseCmd() *Cmd {
	cmd := &Cmd{} //实例化入参对象
	flag.Usage = printUsage //设置用法说明
	flag.BoolVar(&cmd.helpFlag,"help",false,"print help message") //参数为help时，&cmd.helpFlag = true
	flag.BoolVar(&cmd.helpFlag,"?",false,"print help message") //参数为?时，&cmd.helpFlag = true
	flag.BoolVar(&cmd.versionFlag,"version",false,"print version and exit") //参数为version时，&cmd.versionFlag = true
	flag.StringVar(&cmd.cpOption,"cp","","classpath") //参数为version时，&cmd.versionFlag = true
	flag.StringVar(&cmd.cpOption,"classpath","","classpath")
	flag.StringVar(&cmd.XjreOption,"Xjre","","path to jres")
	flag.Parse()
	args := flag.Args()
	if len(args) > 0 {
		cmd.class = args[0]
		cmd.args = args[1:]
	}
	return cmd
}

func printUsage() {
	fmt.Printf("Usage: %s [-options] class [args...]\n",os.Args[0])
}
{% endcodeblock %}

3. 启动方法


{% codeblock lang:go %}
func main () {
	cmd := parseCmd() //解析方法
	if cmd.versionFlag {
		fmt.Println("version 0.0.1") //入参为version时打印版本号
	} else if cmd.helpFlag || cmd.class == "" {
		printUsage() //入参为help或?或classpath、cp为空时打印说明
	}else {
		startJVM(cmd) //启动JVM
	}
}

func startJVM(cmd *Cmd){
	fmt.Printf("classpath:%s class:%s args:%v",cmd.cpOption,cmd.class,cmd.args)
}
{% endcodeblock %}

4. 试运行

{% codeblock lang:shell %}
D:\work\workspace_go\src\jvmgo\ch01> go install

PS D:\work\workspace_go\bin> .\ch01.exe -version
version 0.0.1

PS D:\work\workspace_go\bin> .\ch01.exe -help
Usage: D:\work\workspace_go\bin\ch01.exe [-options] class [args...]

PS D:\work\workspace_go\bin> .\ch01.exe -cp test.class a b c
classpath:test.class class:a args:[b c]
{% endcodeblock %}