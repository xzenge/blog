---
title: 自己动手写Java虚拟机 笔记2
date: 2019-09-10 21:47:52
tags: 
    - JVM
    - Java
    - Go
categories :
    - Technology
---

## 实现cp/classpath 参数搜索class文件

### java class path 搜索顺序：
1. 启动类路径（bootstrap classpath）:jre\lib （可通过-Xbootclasspath参数修改该路径）
2. 扩展类路径（extension classpath）:jre\lib\ext
3. 用户类路径（user classpath）:用户自己指定

<!-- more -->

### 实现
1. Entry接口用来定义启动类的基本功能

{% codeblock lang:go %}
const pathListSeparator = string(os.PathListSeparator) //获取当前操作系统分隔符

type Entry interface {
	readClass(classNme string) ([]byte,Entry,error) //定义按class名读取class文件的方
	String() string 
}

//根据classpath生成对应的启动类
func newEntry(path string) Entry {
	if strings.Contains(path,pathListSeparator){ //classpath路径包含系统分割符时，new组合启动类
		return newCompositeEntry(path)
	}

	if strings.HasSuffix(path,"*") {
		return newWildcardEntry(path) //classpath路径包含*时，new通配符启动类
	}

	if strings.HasSuffix(path, ".jar") || strings.HasSuffix(path, ".JAR") ||
      strings.HasSuffix(path, ".zip") || strings.HasSuffix(path, ".ZIP") {
      return newZipEntry(path) //classpath路径包含jar、JAR、zip、ZIP时，new文件夹启动类
   }
	return newDirEntry(path)

}
{% endcodeblock %}

2. DirEntry，按照文件路径读取文件二进制流
   
{% codeblock lang:golang %}
type DirEntry struct {
	absDir string //存储文件夹路径
}

func newDirEntry(path string) *DirEntry {
	absDir,err := filepath.Abs(path) //如果文件路径存在，获取文件夹路径
	if err != nil {
		panic(err) //文件路径不存在时提示错误
	}

	return &DirEntry{absDir} //返回DirEntry实例
}

func (self *DirEntry) readClass(className string) ([]byte,Entry,error){
	fileName := filepath.Join(self.absDir,className) //拼接文件夹路径和classname
	data,err := ioutil.ReadFile(fileName) //读取文件
	return data,self,err //返回文件内容
}

func(self *DirEntry) String() string {
	return self.absDir //返回文件夹路径
}
{% endcodeblock %}

