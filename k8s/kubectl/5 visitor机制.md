Table of Contents
=================

  * [1. 背景](#1-背景)
  * [2. visitor机制](#2-visitor机制)
     * [2.1 举例说明](#21-举例说明)
  * [3 总结](#3-总结)

### 1. 背景

之前分析到kubectl 的Factory机制时，牵扯到了 visitor机制，这里参考了皓叔的 GO 编程模式：K8S VISITOR 模式[https://coolshell.cn/articles/21263.html], 做的相关笔记，以加深对visitor机制的了解，方便后面的源码分析。

<br>

### 2. visitor机制

visitor机制看起来是更高级的装饰器模式。它的核心就是这个模式是一种将算法与操作对象的结构分离的一种方法。这种分离的实际结果是能够在不修改结构的情况下向现有对象结构添加新操作，是遵循开放/封闭原则的一种方法。

#### 2.1 举例说明

```
package main

import "fmt"

type VisitorFunc func(*Info, error) error

type Visitor interface {
	Visit(VisitorFunc) error
}

type Info struct {
	Namespace   string
	Name        string
	OtherThings string
}
func (info *Info) Visit(fn VisitorFunc) error {
	return fn(info, nil)
}

type NameVisitor struct {
	visitor Visitor
}

func (v NameVisitor) Visit(fn VisitorFunc) error {
	return v.visitor.Visit(func(info *Info, err error) error {
		fmt.Println("NameVisitor() before call function")
		err = fn(info, err)
		if err == nil {
			fmt.Printf("==> Name=%s, NameSpace=%s\n", info.Name, info.Namespace)
		}
		fmt.Println("NameVisitor() after call function")
		return err
	})
}


type OtherThingsVisitor struct {
	visitor Visitor
}

func (v OtherThingsVisitor) Visit(fn VisitorFunc) error {
	return v.visitor.Visit(func(info *Info, err error) error {
		fmt.Println("OtherThingsVisitor() before call function")
		err = fn(info, err)
		if err == nil {
			fmt.Printf("==> OtherThings=%s\n", info.OtherThings)
		}
		fmt.Println("OtherThingsVisitor() after call function")
		return err
	})
}

type LogVisitor struct {
	visitor Visitor
}

func (v LogVisitor) Visit(fn VisitorFunc) error {
	return v.visitor.Visit(func(info *Info, err error) error {
		fmt.Println("LogVisitor() before call function")
		err = fn(info, err)
		fmt.Println("LogVisitor() after call function")
		return err
	})
}

func main() {
	info := Info{}
	var v Visitor = &info
	fmt.Printf("v is  %+v\n", v)

	v = LogVisitor{v}

	fmt.Printf("logvistor is  %+v\n", v)

	v = NameVisitor{v}

	fmt.Printf("namevistor is  %+v\n", v)
	v = OtherThingsVisitor{v}

	fmt.Printf("oth is  %+v\n", v)

	loadFile := func(info *Info, err error) error {
		info.Name = "Hao Chen"
		info.Namespace = "MegaEase"
		info.OtherThings = "We are running as remote team."
		return nil
	}
	v.Visit(loadFile)
}
```

**上述代码的输出为：**

```
v is  &{Namespace: Name: OtherThings:}
logvistor is  {visitor:0xc000070480}
namevistor is  {visitor:{visitor:0xc000070480}}
oth is  {visitor:{visitor:{visitor:0xc000070480}}}
LogVisitor() before call function
NameVisitor() before call function
OtherThingsVisitor() before call function
==> OtherThings=We are running as remote team.
OtherThingsVisitor() after call function
==> Name=Hao Chen, NameSpace=MegaEase
NameVisitor() after call function
LogVisitor() after call function

Process finished with the exit code 0

```

<br>

从上述代码可以看出来，visitor机制的原理在于：

* 定义一个基础的visitor接口，并且规定，visitor接口对应的函数必须是对info对象操作的函数fn(info, err)。
* 定义一个数据结构info，并实现visitor接口
* 其他的想要对info进行处理的。实现visitor接口，在对应实现的函数里，实现自己对info的处理。同时为了能链起来。所以还要调用 fn(info, err)函数
* 根据main函数那样定义的串起来。这样v.Visit(loadFile)就可以 递归似的执行visitor函数

### 3 总结

（1）kubectl 根据各种的options对资源对象会做各种处理，利用visitor机制就可以一个一个的处理

（2）但是有一个疑问就是，为啥不用简单的装饰器模式更方便理解。类似这种，for循环一个个处理？

A: 这样其实就是for循环，起不到 v1-v2-v3-v2-v1这样递归的效果

```
// Visit implements Visitor
func (v DecoratedVisitor) Visit(fn VisitorFunc) error {
  return v.visitor.Visit(func(info *Info, err error) error {
    if err != nil {
      return err
    }
    if err := fn(info, nil); err != nil {
      return err
    }
    for i := range v.decorators {
      if err := v.decorators[i](info, nil); err != nil {
        return err
      }
    }
    return nil
  })
}
```

