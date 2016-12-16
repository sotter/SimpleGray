---
layout: post
title: golang Reflect 源码简析
category: golang
---


## golang Reflect 源码简析

2016-12-15 第一稿， 文章内部难免有疏漏或者错误的地方，欢迎指正！

### 0. GO变量的存储结构	

通过分析Golang Reflect的源码的实现，可以比较清晰的看出golang中变量类型的内部实现。

> 同大多数高级语言一样，Go每一个变量前面都记录了关于这个变量的大量信息，而Interface本质上类似c中的指针，（内部实现相当于变量首部变量的一个引用）， interface的地址即为变量的首地址；
> Go通过获取到获取变量的地址首部的各种信息，来获取Value和Type；

下面具体分析Golang变量的内部实现，一个变量隐形存储的Header主要有3部分组成， rtype， impltype(针对不同的类型，结构有所不同）， uncomontype；

![](http://gitlab.mogujie.org/tianshan/tianshan/raw/master/Pictrue/golang/golang-total.png)

其中 rtype是所有类型的通用结构， 数据结构定义如下：

![](http://gitlab.mogujie.org/tianshan/tianshan/raw/master/Pictrue/golang/golang-rtype.png)

```go
type rtype struct {
	size       uintptr
	ptrdata    uintptr
	hash       uint32   // hash of type; avoids computation in hash tables
	tflag      tflag    // extra type information flags
	align      uint8    // alignment of variable with this type
	fieldAlign uint8    // alignment of struct field with this type
	kind       uint8    // enumeration for C
	alg        *typeAlg // algorithm table
	gcdata     *byte    // garbage collection data
	str        nameOff  // string form  (nameOff -> int32)
	ptrToThis  typeOff  // type for pointer to this type, may be zero (typeOff -> int32)
}
```

其中kind即表示了golang所能支持的26中类型： 

```go
const (
	Invalid Kind = iota
	Bool
	Int
	Int8
	Int16
	Int32
	Int64
	Uint
	Uint8
	Uint16
	Uint32
	Uint64
	Uintptr
	Float32
	Float64
	Complex64
	Complex128
	Array
	Chan
	Func
	Interface
	Map
	Ptr
	Slice
	String
	Struct
	UnsafePointer
)
```

### 1. 核心方法的实现：

#### 1.1 获取变量Name

调用链：

1. 根据&i找到变量i的rtype -> 2. 根据rtype的nameoff找到此类型的name段 -> 3. 从name段中获取的Name的值 

```go
//根据rtype找到Name段，t.nameOff(t.str) ， 至于如何通过nameoff找到name端，内部涉及到各种具体的细节，resolveNameOff的实现在runtime中；
func (t *rtype) String() string {
	s := t.nameOff(t.str).name()
	if t.tflag&tflagExtraStar != 0 {
		return s[1:]
	}
	return s
}
```

![](http://gitlab.mogujie.org/tianshan/tianshan/raw/master/Pictrue/golang/golang-name.png)

获取name段的name长度
```go
//byte的第2 第3个字节，然后大端一下
func (n name) nameLen() int {
	return int(uint16(*n.data(1))<<8 | uint16(*n.data(2)))
}
```

```go
func (n name) name() (s string) {
	if n.bytes == nil {
		return
	}
	b := (*[4]byte)(unsafe.Pointer(n.bytes))
	
	//把s进行一次强制类型转后，再直接赋值给Data，可以减少一次内存拷贝；
	hdr := (*stringHeader)(unsafe.Pointer(&s))
	//为什么是&b[3], 在上图中的数据结构里，flag占一个字节，namelen占2个字节，所有Name是从3开始的；
	hdr.Data = unsafe.Pointer(&b[3])
	hdr.Len = int(b[1])<<8 | int(b[2])
	return s
}
```

下面看一下ResoveNameOff的实现： 

```go
//1. 先从firstmoduledata中查询TODO： firstmoduledata 是干啥使的？
//2. 从全局管理的维护reflectOffs中寻找，reflectOffs维护了所有rtype object与offset的映射关系；
//后续： Runtime packet包

func resolveNameOff(ptrInModule unsafe.Pointer, off nameOff) name {
	if off == 0 {
		return name{}
	}
	base := uintptr(ptrInModule)
	for md := &firstmoduledata; md != nil; md = md.next {
		if base >= md.types && base < md.etypes {
			res := md.types + uintptr(off)
			if res > md.etypes {
				println("runtime: nameOff", hex(off), "out of range", hex(md.types), "-", hex(md.etypes))
				throw("runtime: name offset out of range")
			}
			return name{(*byte)(unsafe.Pointer(res))}
		}
	}

	// No module found. see if it is a run time name.
	reflectOffsLock()
	res, found := reflectOffs.m[int32(off)]
	reflectOffsUnlock()
	if !found {
		println("runtime: nameOff", hex(off), "base", hex(base), "not in ranges:")
		for next := &firstmoduledata; next != nil; next = next.next {
			println("\ttypes", hex(next.types), "etypes", hex(next.etypes))
		}
		throw("runtime: name offset base pointer out of range")
	}
	return name{(*byte)(res)}
}
```

rtype实现了Type的所有接口

问题1： rtype和其他type的关系是什么样的 ？
rtype的基本是数据类型，对于其他Type来讲，如Interface和Map，变量头部还有扩展， 比如InterfaceTime是这样的：

```go
// interfaceType represents an interface type.
type interfaceType struct {
	rtype   `reflect:"interface"`
	pkgPath name      // import path
	methods []imethod // sorted by hash
}
```

**引申问题1**： Interface的本质究竟为何物？ 
从对象模型的角度来看，变量interface相当于赋值给它值的变量首部的一个引用。 如 i Interface  = &StructVar, i即为StructVar首部的一个引用，地址&i即为地址&StructVar；

**引申问题2**： Go是传值调用的， Interface传递是否有指向对象的内存拷贝

```
i := &StructVar
func test( i Interface{}) b Interface {}  {
	//接口的赋值，都涉及到具体对象的内存拷贝的，这一点类似于C++中的基类指针；
	return &StructVar{}
}
```
**引申问题3**： 接口查询的效率如何？

如IFile.(ImlFile), 从上面的原理可以容易的看出，接口查询的只需要找到C中的Kind对比一下就行，效率是比较高的；

**引申问题4**： Name接口的效能如何？

相对于接口查询来说，获取Name要复杂不少，先找到Name段，在找寻的过程中resolveNameOff又需要查询全局的Map等；   
***同时需要注意的是***，调用TypeOf(Interface) 返回的Type， 并不是type的name已经已字符串的方式返回回来了，而是每一调用Name()都要进行一次计算。 即：如果只调用`type := TypeOf(i)`, 性能是比较高的，但如果针对type，调用它的一些获取接口，效率是比较低的；

下文中Method获取的分支，会知道它的性能会更低一些，甚至需要借助于Cache来实现；


#### 1.2 获取Method

Method的获取，主要是通过变量存储结构中uncommType字段来获取的

以MethodByName的实现为例：

```go
func (t *rtype) MethodByName(name string) (m Method, ok bool) {
	//如果是Interface的类型，强制转换成Interface的Type， 为什么interface的存储跟普通类型是不一样的；
	if t.Kind() == Interface {
		tt := (*interfaceType)(unsafe.Pointer(t))
		return tt.MethodByName(name)
	}
	ut := t.uncommon()
	if ut == nil {
		return Method{}, false
	}
	//查询取得method， 总共ut.mcount个，然后挨个遍历；
	utmethods := ut.methods()
	for i := 0; i < int(ut.mcount); i++ {
		p := utmethods[i]
		pname := t.nameOff(p.name)
		if pname.isExported() && pname.name() == name {
			return t.Method(i), true
		}
	}
	return Method{}, false
}
```

现在来看`utmethods := ut.methods()`的实现：   
-> 第一步： 先从缓存中获取，GO程序中有一个&rtype到methods的缓存；  
-> 第二步： 如果缓存中获取，那么根据uncommonType获取；   

> 引申问题： 缓存中是否超时释放的机制？ 通过GC，还是通过其他的？

```
// A MethodSetCache records the method set of each type T for which
// MethodSet(T) is called so that repeat queries are fast.
// The zero value is a ready-to-use cache instance.
type MethodSetCache struct {
	mu     sync.Mutex
	named  map[*types.Named]struct{ value, pointer *types.MethodSet } // method sets for named N and *N
	others map[types.Type]*types.MethodSet                            // all other types
}

//MethodCache中维护了一个类型名到它的Methond的一个映射；

func (t *rtype) exportedMethods() []method {
	//从methodCahce中获取变量的所有的method
	methodCache.RLock()
	methods, found := methodCache.m[t]
	methodCache.RUnlock()
	
	//如果找到就回家了
	if found {
		return methods
	}

	ut := t.uncommon()
	if ut == nil {
		return nil
	}
	allm := ut.methods()
	allExported := true
	for _, m := range allm {
		name := t.nameOff(m.name)
		if !name.isExported() {
			allExported = false
			break
		}
	}
	if allExported {
		methods = allm
	} else {
		methods = make([]method, 0, len(allm))
		for _, m := range allm {
			name := t.nameOff(m.name)
			if name.isExported() {
				methods = append(methods, m)
			}
		}
		methods = methods[:len(methods):len(methods)]
	}

	methodCache.Lock()
	if methodCache.m == nil {
		methodCache.m = make(map[*rtype][]method)
	}
	methodCache.m[t] = methods
	methodCache.Unlock()

	return methods
}
```

首先来看一下uncommon的字段结构

```go
type uncommonType struct {
	pkgPath nameOff // import path; empty for built-in types like int, string
	mcount  uint16  // number of methods
	_       uint16  // unused
	moff    uint32  // offset from this uncommontype to [mcount]method
	_       uint32  // unused
}
```

![](http://gitlab.mogujie.org/tianshan/tianshan/raw/master/Pictrue/golang/golang-uncommon.png)

获取过程也很明显了：
首先根据uncommon段中的moff字段找到mthod端 -> 根据Method端对应的method；



