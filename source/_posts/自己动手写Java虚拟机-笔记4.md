---
title: 自己动手写Java虚拟机 笔记4
date: 2019-09-26 21:49:11
tags: 
    - JVM
    - Java
    - Go
categories :
    - Technology
---

### 运行时数据区
JVM将需要用到的数据存放入内存中，这个内存区就叫运行时数据区
可分为两种：
1. 一类是多线程共享的，该数据区需要在Java虚拟机启动时创建好，在Java虚拟机退出时销毁。
2. 一类则是线程私有的，该数据区则在创建线程时才创建，线程退出时销毁。

<!-- more -->

{% asset_img 运行时数据区.png 800 800 运行时数据区 %}

### 实现
#### 定义对象结构

{% codeblock lang:golang %}
type Object struct {
	// todo
}

{% endcodeblock %}

#### 运行时数据区-线程

{% codeblock lang:golang %}
type Thread struct {
	pc    int // the address of the instruction currently being executed
	stack *Stack //Java虚拟机栈指针
	// todo
}

//创建Thread实例
//java命令提供了-Xss选项来设置Java虚拟机栈大小
func NewThread() *Thread {
	return &Thread{
		stack: newStack(1024), //最多容纳1024帧
	}
}

func (self *Thread) PC() int {
	return self.pc
}
func (self *Thread) SetPC(pc int) {
	self.pc = pc
}

//当前帧进栈
func (self *Thread) PushFrame(frame *Frame) {
	self.stack.push(frame)
}
//当前帧出栈
func (self *Thread) PopFrame() *Frame {
	return self.stack.pop()
}
//获取当前帧
func (self *Thread) CurrentFrame() *Frame {
	return self.stack.top()
}




{% endcodeblock %}


#### 运行时数据区-Java虚拟机栈

{% codeblock lang:golang %}
type Stack struct {
	maxSize uint //最多可以容纳多少帧
	size    uint //栈的当前大小
	_top    *Frame //保存栈顶指针
}

//实例一个栈
func newStack(maxSize uint) *Stack {
	return &Stack{
		maxSize: maxSize,
	}
}

//当前帧进栈
func (self *Stack) push(frame *Frame) {
	if self.size >= self.maxSize { //判断当前栈大小是否超过最大容量
		panic("java.lang.StackOverflowError") 
	}

	if self._top != nil {  //当前栈顶不为空时
		frame.lower = self._top //将插入帧尾指向栈顶指针
	}

	self._top = frame //当前栈顶指向当前帧
	self.size++ //当前栈大小加1
}

//当前帧出栈
func (self *Stack) pop() *Frame {
	if self._top == nil { //当前栈顶为空时报错
		panic("jvm stack is empty!")
	}

	top := self._top //取出当前栈顶帧
	self._top = top.lower //将当前栈顶指向top的下一帧
	top.lower = nil //断开top的尾指针
	self.size-- //当前栈大小减1

	return top //返回top
}

//获取当前栈顶帧
func (self *Stack) top() *Frame {
	if self._top == nil { ////当前栈顶为空时报错
		panic("jvm stack is empty!")
	}

	return self._top //返回当前栈顶帧
}

{% endcodeblock %}

#### 运行时数据区-帧

{% codeblock lang:golang %}
type Frame struct {
	lower        *Frame //指向下一帧
	localVars    LocalVars //保存局部变量表指针
	operandStack *OperandStack //保存操作数栈指针
	// todo
}

//实例化帧
func NewFrame(maxLocals, maxStack uint) *Frame {
	return &Frame{
		localVars:    newLocalVars(maxLocals), //实例化局部变量对象
		operandStack: newOperandStack(maxStack), //实例化操作数栈对象
	}
}

// getters
func (self *Frame) LocalVars() LocalVars {
	return self.localVars
}
func (self *Frame) OperandStack() *OperandStack {
	return self.operandStack
}

{% endcodeblock %}

#### 运行时数据区-局部变量表

{% codeblock lang:golang %}
type LocalVars []Slot

//创建局部变量实例
func newLocalVars(maxLocals uint) LocalVars {
	if maxLocals > 0 {
		return make([]Slot, maxLocals) //申明大小为maxLocals的Slot数组
	}
	return nil
}

//设置int类型
func (self LocalVars) SetInt(index uint, val int32) {
	self[index].num = val
}
//取int类型
func (self LocalVars) GetInt(index uint) int32 {
	return self[index].num
}
//设置float32类型
func (self LocalVars) SetFloat(index uint, val float32) {
	bits := math.Float32bits(val)//转为uint32
	self[index].num = int32(bits) //转为int32
}
//取float32类型
func (self LocalVars) GetFloat(index uint) float32 {
	bits := uint32(self[index].num)//转为unit32
	return math.Float32frombits(bits)//转为float
}

//设置long类型
func (self LocalVars) SetLong(index uint, val int64) {
	self[index].num = int32(val)//取低位32位存储在index
	self[index+1].num = int32(val >> 32) //右移32位取高位32位存储在index+1
}
//取long类型
func (self LocalVars) GetLong(index uint) int64 {
	low := uint32(self[index].num) //取index低32位
	high := uint32(self[index+1].num) //取index+1高32位
	return int64(high)<<32 | int64(low) //左移高位32位，拼接高低位返回
}

//设置float64
func (self LocalVars) SetDouble(index uint, val float64) {
	bits := math.Float64bits(val) //转为uint64
	self.SetLong(index, int64(bits)) //用long型设置方法设置int64
}
//取float64
func (self LocalVars) GetDouble(index uint) float64 {
	bits := uint64(self.GetLong(index)) //按long型取值方法取出uint64
	return math.Float64frombits(bits) //转为float64
}
//设置引用
func (self LocalVars) SetRef(index uint, ref *Object) {
	self[index].ref = ref
}
//取引用
func (self LocalVars) GetRef(index uint) *Object {
	return self[index].ref
}

{% endcodeblock %}

#### 运行时数据区-操作数栈

{% codeblock lang:golang %}
type OperandStack struct {
	size  uint //记录栈顶位置
	slots []Slot
}
//实例化操作数栈
func newOperandStack(maxStack uint) *OperandStack {
	if maxStack > 0 {
		return &OperandStack{
			slots: make([]Slot, maxStack),//申明maxStack大小的Slot数组
		}
	}
	return nil
}

func (self *OperandStack) PushInt(val int32) {
	self.slots[self.size].num = val
	self.size++
}
func (self *OperandStack) PopInt() int32 {
	self.size--
	return self.slots[self.size].num
}

func (self *OperandStack) PushFloat(val float32) {
	bits := math.Float32bits(val)
	self.slots[self.size].num = int32(bits)
	self.size++
}
func (self *OperandStack) PopFloat() float32 {
	self.size--
	bits := uint32(self.slots[self.size].num)
	return math.Float32frombits(bits)
}

// long consumes two slots
func (self *OperandStack) PushLong(val int64) {
	self.slots[self.size].num = int32(val)
	self.slots[self.size+1].num = int32(val >> 32)
	self.size += 2
}
func (self *OperandStack) PopLong() int64 {
	self.size -= 2
	low := uint32(self.slots[self.size].num)
	high := uint32(self.slots[self.size+1].num)
	return int64(high)<<32 | int64(low)
}

// double consumes two slots
func (self *OperandStack) PushDouble(val float64) {
	bits := math.Float64bits(val)
	self.PushLong(int64(bits))
}
func (self *OperandStack) PopDouble() float64 {
	bits := uint64(self.PopLong())
	return math.Float64frombits(bits)
}

func (self *OperandStack) PushRef(ref *Object) {
	self.slots[self.size].ref = ref
	self.size++
}
func (self *OperandStack) PopRef() *Object {
	self.size--
	ref := self.slots[self.size].ref
	self.slots[self.size].ref = nil
	return ref
}
{% endcodeblock %}

#### 运行时数据区-操作数栈

{% codeblock lang:golang %}

{% endcodeblock %}