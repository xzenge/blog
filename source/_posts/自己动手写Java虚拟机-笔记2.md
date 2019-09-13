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

2. Entry接口的具体实现
 * DirEntry，按照文件路径读取文件二进制流 
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

 * ZipEntry,读取压缩文件夹内的文件
{% codeblock lang:golang %}
type ZipEnty struct {
	absPath string //文件夹路径
}

func newZipEntry(path string) *ZipEnty {
	absPath, err := filepath.Abs(path) //如果文件路径存在，获取文件夹路径
	if err != nil {
		panic(err)
	}
	return &ZipEnty{absPath}
}

func (self *ZipEnty) readClass(className string) ([]byte,Entry,error){
	r, err := zip.OpenReader(self.absPath) //以压缩文件格式解压
	if err != nil {
	   return nil, nil, err
	}
	defer r.Close() //最后关闭文件流
	for _, f := range r.File { //循环解压出来的文件
	   if f.Name == className { //如果当前文件为指定文件
		  rc, err := f.Open() //打开该文件
		  if err != nil {
			 return nil, nil, err
		  }
		  defer rc.Close() //最后关闭文件
		  data, err := ioutil.ReadAll(rc) //读取文件流
		  if err != nil {
			 return nil, nil, err
		  }
		  return data, self, nil //返回文件流
	   }
	}
	return nil, nil, errors.New("class not found: " + className)
}

func (self *ZipEnty) String() string {
	return self.absPath
}
{% endcodeblock %}

 * CompositeEntry,读取符合文件类型
{% codeblock lang:golang %}
type CompositeEntry []Entry //CompositeEntry为多种Entry的数组（Dir、Zip）

func newCompositeEntry(pathList string) CompositeEntry {
	compositeEntry := []Entry{} //实例Entry接口数组

	for _,path := range strings.Split(pathList,pathListSeparator){ //按照系统分割符截取path列表，并循环
		entry := newEntry(path) //按照每种path实例化不同的Entry(Dir、Zip、Wildcard)
		compositeEntry = append(compositeEntry,entry) //将实例化的不同Entry放入CompositeEntry数组中
	}
	return compositeEntry
}

func (self CompositeEntry) readClass(className string)([]byte,Entry,error){
	for _, entry := range self { //循环compositeEntry数组
		data, from, err := entry.readClass(className) //根据不同Entry(Dir、Zip、Wildcard)，按指定classname读取文件
		if err == nil {
		   return data, from, nil //返回读取结果
		}
	 }
	 return nil, nil, errors.New("class not found: " + className) //无结果是返回错误信息
}

func (self CompositeEntry) String() string {
	strs := make([]string, len(self)) //按照CompositeEntry的长度申明该长度的Sring数组
	for i, entry := range self { //循环CompositeEntry
	   strs[i] = entry.String() //调用每个Entry(Dir、Zip、Wildcard)实现类的String()方法,并放入String数组中
	}
	return strings.Join(strs, pathListSeparator) //将String数组中的结果以系统分隔符拼接并返回
}
{% endcodeblock %}

 * WildcardEntry，按照通配符解析文件
{% codeblock lang:golang %}
func newWildcardEntry(path string) CompositeEntry {
	baseDir := path[:len(path)-1] // remove *
	compositeEntry := []Entry{} //申明compositeEntry对象(Entry的数组)
	walkFn := func(path string, info os.FileInfo, err error) error {
		if err != nil {
		   return err
		}
		if info.IsDir() && path != baseDir { //如果当前目录是文件夹，并且是根目录
		   return filepath.SkipDir //返回SkipDir的错误
		}
		if strings.HasSuffix(path, ".jar") || strings.HasSuffix(path, ".JAR") { //如果当前路径是以.jar或.JAR结尾的
			jarEntry := newZipEntry(path) //实例化ZipEntry
			compositeEntry = append(compositeEntry, jarEntry) //将实例化的ZipEntry对象放入compositeEntry数组中
		 }
		 return nil
	  }
	filepath.Walk(baseDir, walkFn) //将walkFn方法传递给filepath.Walk
	return compositeEntry //返回结果
 }
{% endcodeblock %}

* Classpath，-cp|classpath参数解析实现的入口方法
{% codeblock lang:golang %}
type Classpath struct { //Classpath包含java的三种基础启动路径
	bootClasspath Entry 
	extClasspath Entry
	userClasspath Entry
}

func Parse(jreOption, cpOption string) * Classpath  {
	cp := &Classpath{} //实例化Classpath对象
	cp.parseBootAndExtClasspath(jreOption) //按照-Xjre参数解析Boot和Ext启动路径
	cp.parseUserClasspath(cpOption) //按照-cp|classpath参数解析User启动路径
	return cp //返回设置好启动路径的Classpath对象
}

func (self *Classpath) ReadClass(className string) ([]byte,Entry,error){
	className = className + ".class" //初始化className文件名
	if data, entry, err := self.bootClasspath.readClass(className); err == nil { //先扫描Boot启动路径下是否存在该className.class文件
	   return data, entry, err //存在则返回
	}
	if data, entry, err := self.extClasspath.readClass(className); err == nil { //再扫描Ext启动路径下是否存在该className.class文件
	   return data, entry, err //存在则返回
	}
	return self.userClasspath.readClass(className) //以上两个路径下都不存在，则扫描User启动路径
}

func (self *Classpath) String() string  {
	return self.userClasspath.String()
}

func (self *Classpath) parseBootAndExtClasspath(jreOption string) { //按照-Xjre参数解析Boot和Ext启动路径
	jreDir := getJreDir(jreOption) //获取jre路径
   // jre/lib/*
   jreLibPath := filepath.Join(jreDir, "lib", "*")
   self.bootClasspath = newWildcardEntry(jreLibPath)
   // jre/lib/ext/*
   jreExtPath := filepath.Join(jreDir, "lib", "ext", "*")
   self.extClasspath = newWildcardEntry(jreExtPath)
}

func getJreDir(jreOption string) string {
	if jreOption != "" && exists(jreOption) { //jreOption不为空且存在时返回该路径
	   return jreOption
	}
	if exists("./jre") { //./jre路径存在时，返回.jre
	   return "./jre"
	}
	if jh := os.Getenv("JAVA_HOME"); jh != "" { //环境变量JAVA_HOME存在并且不为空时
	   return filepath.Join(jh, "jre") //返回JAVA_HOME + "jre"
	}
	panic("Can not find jre folder!")
 }

 func exists(path string) bool {
	if _, err := os.Stat(path); err != nil {
	   if os.IsNotExist(err) {
		  return false
	   }
	}
	return true
 }

 func (self *Classpath) parseUserClasspath(cpOption string) { //按照-cp|classpath参数解析User启动参数
	if cpOption == "" { //cpOption为空时
	   cpOption = "." //返回相对路劲
	}
	self.userClasspath = newEntry(cpOption) //根据-cp|classpath参数实例化对应的Entry
 }
{% endcodeblock %}

 * main，修改启动类
{% codeblock lang:golang %}
func startJVM(cmd *Cmd){
	cp := classpath.Parse(cmd.XjreOption, cmd.cpOption) //根据-Xjre和-cp|classpath参数实例化classpath对象实例
	fmt.Printf("classpath:%v class:%v args:%v\n",
	   cp, cmd.class, cmd.args)
	className := strings.Replace(cmd.class, ".", "/", -1)
	classData, _, err := cp.ReadClass(className) //按照3个启动路径查找并读取className文件
	if err != nil {
	   fmt.Printf("Could not find or load main class %s\n", cmd.class)
	   return
	}
	fmt.Printf("class data:%v\n", classData)
}
{% endcodeblock %}

3. 试运行
{% codeblock lang:shell %}
PS D:\work\workspace_go\src\jvmgo\ch02> go install

PS D:\work\workspace_go\bin> .\ch02.exe -Xjre "C:\Program Files\Java\jre1.8.0_202" java.lang.Object
classpath:D:\work\workspace_go\bin class:java.lang.Object args:[]
class data:[202 254 186 190 0 0 0 52 0 75 3 0 15 66 63 8 0 16 8 0 36 8 0 40 1 0 3 40 41 73 1 0 20 40 41 76 106 97 118 97 47 108 97 110 103 47 79 98 106 101 99 116 59 1 0 20 40 41 76 106 97 118 97 47 108 97 110 103 47 83 116 114 105 110 103 59 1 0 3 40 41 86 1 0 21 40 73 41 76 106 97 118 97 47 108 97 110 103 47 83 116 114 105 110 103 59 1 0 4 40 74 41 86 1 0 5 40 74 73 41 86 1 0 21 40 76 106 97 118 97 47 108 97 110 103 47 79 98 106 101 99 116 59 41 90 1 0 21 40 76 106 97 118 97 47 108 97 110 103 47 83 116 114 105 110 103 59 41 86 1 0 8 60 99 108 105 110 105 116 62 1 0 6 60 105 110 105 116 62 1 0 1 64 1 0 4 67 111 100
101 1 0 10 69 120 99 101 112 116 105 111 110 115 1 0 9 83 105 103 110 97 116 117 114 101 1 0 13 83 116 97 99 107 77 97 112 84 97 98 108 101 1 0 6 97 112 112 101 110 100 1 0 5 99 108 111 110 101 1 0 6 101 113
117 97 108 115 1 0 8 102 105 110 97 108 105 122 101 1 0 8 103 101 116 67 108 97 115 115 1 0 7 103 101 116 78 97 109 101 1 0 8 104 97 115 104 67 111 100 101 1 0 15 106 97 118 97 47 108 97 110 103 47 67 108 97
115 115 1 0 36 106 97 118 97 47 108 97 110 103 47 67 108 111 110 101 78 111 116 83 117 112 112 111 114 116 101 100 69 120 99 101 112 116 105 111 110 1 0 34 106 97 118 97 47 108 97 110 103 47 73 108 108 101 103 97 108 65 114 103 117 109 101 110 116 69 120 99 101 112 116 105 111 110 1 0 17 106 97 118 97 47 108 97 110 103 47 73 110 116 101 103 101 114 1 0 30 106 97 118 97 47 108 97 110 103 47 73 110 116 101 114 114
117 112 116 101 100 69 120 99 101 112 116 105 111 110 1 0 16 106 97 118 97 47 108 97 110 103 47 79 98 106 101 99 116 1 0 23 106 97 118 97 47 108 97 110 103 47 83 116 114 105 110 103 66 117 105 108 100 101 114 1 0 19 106 97 118 97 47 108 97 110 103 47 84 104 114 111 119 97 98 108 101 1 0 37 110 97 110 111 115 101 99 111 110 100 32 116 105 109 101 111 117 116 32 118 97 108 117 101 32 111 117 116 32 111 102 32 114 97 110 103 101 1 0 6 110 111 116 105 102 121 1 0 9 110 111 116 105 102 121 65 108 108 1 0 15 114 101 103 105 115 116 101 114 78 97 116 105 118 101 115 1 0 25 116 105 109 101 111 117 116 32 118 97 108 117 101 32 105 115 32 110 101 103 97 116 105 118 101 1 0 11 116 111 72 101 120 83 116 114 105 110 103 1 0 8 116 111 83 116 114 105 110 103 1 0 4 119 97 105 116 7 0 28 7 0 29 7 0 30 7 0 31 7 0 32 7 0 33 7 0 34 7 0 35 1 0 19 40 41 76 106 97 118 97 47 108 97 110 103 47 67 108 97 115 115 59 1 0 22 40 41 76 106 97 118 97 47 108 97 110 103 47 67 108 97 115 115 60 42 62 59 1 0 45 40 76 106 97 118 97 47 108 97 110 103 47 83 116 114 105 110 103 59 41 76 106 97 118 97 47 108 97 110 103 47 83 116 114 105 110 103 66 117 105 108 100 101 114 59 12 0 27 0 5 12 0 15 0 8 12 0 39 0 8 12 0 43 0 10 12 0 25 0 52 12 0 26 0 7 12 0 42 0 7 12 0 41 0
9 12 0 15 0 13 12 0 21 0 54 10 0 44 0 60 10 0 46 0 63 10 0 47 0 62 10 0 49 0 55 10 0 49 0 57 10 0 49 0 58 10 0 49 0 59 10 0 50 0 56 10 0 50 0 61 10 0 50 0 64 0 33 0 49 0 0 0 0 0 0 0 14 0 1 0 15 0 8 0 1 0 17 0 0 0 13 0 0 0 1 0 0 0 1 177 0 0 0 0 1 10 0 39 0 8 0 0 1 17 0 25 0 52 0 1 0 19 0 0 0 2 0 53 1 1 0 27 0 5 0 0 0 1 0 23 0 12 0 1 0 17 0 0 0 34 0 2 0 2 0 0 0 11 42 43 166 0 7 4 167 0 4 3 172 0 0 0 1 0 20 0 0 0 5
0 2 9 64 1 1 4 0 22 0 6 0 1 0 18 0 0 0 4 0 1 0 45 0 1 0 42 0 7 0 1 0 17 0 0 0 48 0 2 0 1 0 0 0 36 187 0 50 89 183 0 72 42 182 0 71 182 0 65 182 0 74 18 2 182 0 74 42 182 0 68 184 0 67 182 0 74 182 0 73 176 0
0 0 0 1 17 0 37 0 8 0 0 1 17 0 38 0 8 0 0 1 17 0 43 0 10 0 1 0 18 0 0 0 4 0 1 0 48 0 17 0 43 0 11 0 2 0 17 0 0 0 74 0 4 0 4 0 0 0 50 31 9 148 156 0 13 187 0 46 89 18 4 183 0 66 191 29 155 0 9 29 18 1 164 0 13 187 0 46 89 18 3 183 0 66 191 29 158 0 7 31 10 97 64 42 31 182 0 70 177 0 0 0 1 0 20 0 0 0 6 0 4 16 9 9 7 0 18 0 0 0 4 0 1 0 48 0 17 0 43 0 8 0 2 0 17 0 0 0 18 0 3 0 1 0 0 0 6 42 9 182 0 70 177 0 0 0 0 0 18
0 0 0 4 0 1 0 48 0 4 0 24 0 8 0 2 0 17 0 0 0 13 0 0 0 1 0 0 0 1 177 0 0 0 0 0 18 0 0 0 4 0 1 0 51 0 8 0 14 0 8 0 1 0 17 0 0 0 16 0 0 0 0 0 0 0 4 184 0 69 177 0 0 0 0 0 0]
{% endcodeblock %}
