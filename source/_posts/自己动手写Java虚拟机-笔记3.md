---
title: 自己动手写Java虚拟机 笔记3
date: 2019-09-13 18:44:38
tags: 
    - JVM
    - Java
    - Go
categories :
    - Technology
---

<!-- more -->

### JVM中class文件结构(ClassFile)
Length|name
---|---
u4|magic
u2|minor_version
u2|major_version
u2|constant_pool_count
cp_info|constant_pool[constant_pool_count-1]
u2|access_flags
u2|this_class
u2|super_class
u2|interfaces_count
u2|interfaces[interfaces_count]
u2|fields_count
field_info|fields[fields_count]
u2|methods_count
method_info|methods[methods_count]

### JVM中字段结构(field_info)
Length|name
---|---
u2|access_flags
u2|name_index
u2|descriptor_index
u2|attributes_count
attribute_info|attributes[attributes_count]

### JVM中访问控制符结构(access_flags)
标志名|标志值|标志含义|针对的对象
---|---|---|---
ACC_PUBLIC|0x0001|public类型|所有类型
ACC_FINAL|0x0010|final类型|类
ACC_SUPER|0x0020|使用新的invokespecial语义|类和接口
ACC_INTERFACE|0x0200|接口类型|接口
ACC_ABSTRACT|0x0400|抽象类型|类和接口
ACC_SYNTHETIC|0x1000|该类不由用户代码生成|所有类型
ACC_ANNOTATION|0x2000|注解类型|注解
ACC_ENUM|0x4000|枚举类型|枚举

### 实现
1. 定义Class文件的录取类(ClassReader)
{% codeblock lang:golang %}
type ClassReader struct {
	data []byte //class文件字节流
}

// 读取u1
func (self *ClassReader) readUint8() uint8 {
	val := self.data[0] //读取1Byte
	self.data = self.data[1:]
	return val
}

// u2
func (self *ClassReader) readUint16() uint16 {
	val := binary.BigEndian.Uint16(self.data) //读取2Byte并转为uint16
	self.data = self.data[2:]
	return val
}

// u4
func (self *ClassReader) readUint32() uint32 {
	val := binary.BigEndian.Uint32(self.data) //读取4Byte并转为uint32
	self.data = self.data[4:]
	return val
}

func (self *ClassReader) readUint64() uint64 {
	val := binary.BigEndian.Uint64(self.data) //读取8Byte并转为uint64
	self.data = self.data[8:]
	return val
}

func (self *ClassReader) readUint16s() []uint16 {
	n := self.readUint16() //读取2Byte，该值为后续数组的长度
	s := make([]uint16, n) //生产长度为n的uint16类型数组
	for i := range s {
		s[i] = self.readUint16() //读取class文件，每读取2Byte便放入数组
	}
	return s
}

func (self *ClassReader) readBytes(n uint32) []byte {
	bytes := self.data[:n] //读取长度为n的字节
	self.data = self.data[n:]
	return bytes
}
{% endcodeblock %}

2. 定义Class文件的格式(ClassFile)
{% codeblock lang:golang %}
type ClassFile struct { //按照jvm规范，定义class文件结构
	//magic      uint32
	minorVersion uint16
	majorVersion uint16
	constantPool ConstantPool
	accessFlags  uint16
	thisClass    uint16
	superClass   uint16
	interfaces   []uint16
	fields       []*MemberInfo
	methods      []*MemberInfo
	attributes   []AttributeInfo
}

//文件解析入口方法，入参class文件流，返回class文件对象
func Parse(classData []byte) (cf *ClassFile, err error) {
	defer func() { //捕获异常
		if r := recover(); r != nil {
			var ok bool
			err, ok = r.(error)
			if !ok {
				err = fmt.Errorf("%v", r)
			}
		}
	}()

	cr := &ClassReader{classData} //实例化classReader类
	cf = &ClassFile{} //实例化ClassFile类
	cf.read(cr) //读取class文件
	return
}

//读取class文件的具体方法
func (self *ClassFile) read(reader *ClassReader) {
	self.readAndCheckMagic(reader) //读取魔数
	self.readAndCheckVersion(reader) //读取主次版本号
	self.constantPool = readConstantPool(reader) //读取常量池
	self.accessFlags = reader.readUint16() //读取当前类（或者接口）的访问修饰符
	self.thisClass = reader.readUint16() //读取当前类索引
	self.superClass = reader.readUint16() //读取父类索引
	self.interfaces = reader.readUint16s() //读取所有接口索引
	self.fields = readMembers(reader, self.constantPool) //从常量池中读取所有成员
	self.methods = readMembers(reader, self.constantPool) //从常量池中读取所有方法
	self.attributes = readAttributes(reader, self.constantPool) //从常量池中读取所有属性
}

func (self *ClassFile) readAndCheckMagic(reader *ClassReader) {
	magic := reader.readUint32() //读取4Byte魔数
	if magic != 0xCAFEBABE { //判断魔数是否为0xCAFEBABE
		panic("java.lang.ClassFormatError: magic!")
	}
}

func (self *ClassFile) readAndCheckVersion(reader *ClassReader) {
	self.minorVersion = reader.readUint16() //高2Byte为次版本号
	self.majorVersion = reader.readUint16() //低2Byte为主版本号
	switch self.majorVersion { //检查版本号是否为45~52
	case 45:
		return
	case 46, 47, 48, 49, 50, 51, 52:
		if self.minorVersion == 0 {
			return
		}
	}

	panic("java.lang.UnsupportedClassVersionError!")
}

//对外暴露get方法
func (self *ClassFile) MinorVersion() uint16 {
	return self.minorVersion
}
//对外暴露get方法
func (self *ClassFile) MajorVersion() uint16 {
	return self.majorVersion
}
//对外暴露get方法
func (self *ClassFile) ConstantPool() ConstantPool {
	return self.constantPool
}
//对外暴露get方法
func (self *ClassFile) AccessFlags() uint16 {
	return self.accessFlags
}
//对外暴露get方法
func (self *ClassFile) Fields() []*MemberInfo {
	return self.fields
}
//对外暴露get方法
func (self *ClassFile) Methods() []*MemberInfo {
	return self.methods
}
//对外暴露get方法，根据thisClass索引在常量池中查找名称
func (self *ClassFile) ClassName() string {
	return self.constantPool.getClassName(self.thisClass)
}
//对外暴露get方法，根据superClass索引在常量池中查找名称
func (self *ClassFile) SuperClassName() string {
	if self.superClass > 0 {
		return self.constantPool.getClassName(self.superClass)
	}
	return ""
}
//对外暴露get方法，根据interfaces索引在常量池中查找名称
func (self *ClassFile) InterfaceNames() []string {
	interfaceNames := make([]string, len(self.interfaces))
	for i, cpIndex := range self.interfaces {
		interfaceNames[i] = self.constantPool.getClassName(cpIndex)
	}
	return interfaceNames
}
{% endcodeblock %}