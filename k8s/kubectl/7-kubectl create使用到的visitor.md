Table of Contents
=================

  * [1. 背景说明](#1-背景说明)
     * [1.1 get](#11-get)
     * [1.2 delete](#12-delete)
     * [1.3 create](#13-create)
     * [1.4 apply](#14-apply)
  * [2. kubectl create -f pod.yaml](#2-kubectl-create--f-podyaml)
     * [2.1 kubectl代码中定义visitor](#21-kubectl代码中定义visitor)
     * [2.2 再次看kubectl create的输出结果](#22-再次看kubectl-create的输出结果)
  * [3. 总结](#3-总结)

### 1. 背景说明

为了了解不同的kubectl操作，使用了哪些visitor, 给每个visitor打印了如下的日志。然后使用各种kubectl 命令进行操作。

```
func (v *URLVisitor) Visit(fn VisitorFunc) error {
	klog.Errorf("in URLVisitor")
	defer klog.Errorf("after URLVisitor")
```

#### 1.1 get 

可以看出来最简单的get 也用到了selector，应该是默认加了-n default的缘故

```
root@k8s-master:~# ./kubectl get pods 
E1108 15:07:03.521469    6873 visitor.go:335] in DecoratedVisitor
E1108 15:07:03.521496    6873 visitor.go:364] in ContinueOnErrorVisitor
E1108 15:07:03.521506    6873 visitor.go:404] in FlattenListVisitor
E1108 15:07:03.521518    6873 visitor.go:216] in EagerVisitorList
E1108 15:07:03.521530    6873 selector.go:55] in Selector
E1108 15:07:03.525540    6873 selector.go:107] after Selector
E1108 15:07:03.525564    6873 visitor.go:233] after EagerVisitorList
E1108 15:07:03.525573    6873 visitor.go:406] after FlattenListVisitor
E1108 15:07:03.525582    6873 visitor.go:383] after ContinueOnErrorVisitor
E1108 15:07:03.525589    6873 visitor.go:337] after DecoratedVisitor
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   70         2d22h


root@k8s-master:~# ./kubectl get pods --field-selector status.phase=Running
E1108 15:10:11.485035    8174 visitor.go:335] in DecoratedVisitor
E1108 15:10:11.485076    8174 visitor.go:364] in ContinueOnErrorVisitor
E1108 15:10:11.485081    8174 visitor.go:404] in FlattenListVisitor
E1108 15:10:11.485087    8174 visitor.go:216] in EagerVisitorList
E1108 15:10:11.485093    8174 selector.go:55] in Selector
E1108 15:10:11.489887    8174 selector.go:107] after Selector
E1108 15:10:11.489924    8174 visitor.go:233] after EagerVisitorList
E1108 15:10:11.489937    8174 visitor.go:406] after FlattenListVisitor
E1108 15:10:11.489947    8174 visitor.go:383] after ContinueOnErrorVisitor
E1108 15:10:11.489954    8174 visitor.go:337] after DecoratedVisitor
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   71         2d23h
```

<br>

#### 1.2 delete

可以看到delete是先删除完后，再对对象进行处理

```
root@k8s-master:~# ./kubectl delete  pods  nginx
E1108 15:32:47.413503   17665 visitor.go:335] in DecoratedVisitor
E1108 15:32:47.413531   17665 visitor.go:364] in ContinueOnErrorVisitor
E1108 15:32:47.413539   17665 visitor.go:404] in FlattenListVisitor
E1108 15:32:47.413544   17665 visitor.go:199] in VisitorList
E1108 15:32:47.413554   17665 visitor.go:96] in Info
pod "nginx" deleted
E1108 15:32:47.427400   17665 visitor.go:98] after Info
E1108 15:32:47.427477   17665 visitor.go:206] after VisitorList
E1108 15:32:47.427608   17665 visitor.go:406] after FlattenListVisitor
E1108 15:32:47.427669   17665 visitor.go:383] after ContinueOnErrorVisitor
E1108 15:32:47.427682   17665 visitor.go:337] after DecoratedVisitor
E1108 15:32:47.427695   17665 visitor.go:780] in InfoListVisitor
E1108 15:33:03.414515   17665 visitor.go:786] after InfoListVisitor
```

#### 1.3 create

```
root@k8s-master:~# ./kubectl create -f pod.yaml -v 3
E1108 15:34:00.125162   18176 visitor.go:335] in DecoratedVisitor
E1108 15:34:00.125198   18176 visitor.go:364] in ContinueOnErrorVisitor
E1108 15:34:00.125206   18176 visitor.go:404] in FlattenListVisitor
E1108 15:34:00.125210   18176 visitor.go:404] in FlattenListVisitor
E1108 15:34:00.125214   18176 visitor.go:216] in EagerVisitorList
E1108 15:34:00.125226   18176 visitor.go:526] in FileVisitor
E1108 15:34:00.125256   18176 visitor.go:592] in StreamVisitor
I1108 15:34:00.131293   18176 create.go:272] post data to apiserver
pod/nginx created
E1108 15:34:00.141895   18176 visitor.go:599] after StreamVisitor
E1108 15:34:00.141923   18176 visitor.go:545] after FileVisitor
E1108 15:34:00.141934   18176 visitor.go:233] after EagerVisitorList
E1108 15:34:00.141941   18176 visitor.go:406] after FlattenListVisitor
E1108 15:34:00.141946   18176 visitor.go:406] after FlattenListVisitor
E1108 15:34:00.141951   18176 visitor.go:383] after ContinueOnErrorVisitor
E1108 15:34:00.141960   18176 visitor.go:337] after DecoratedVisitor
```

#### 1.4 apply

```
root@k8s-master:~# ./kubectl apply -f pod.yaml -v 3
E1108 15:34:33.097456   18406 visitor.go:335] in DecoratedVisitor
E1108 15:34:33.097490   18406 visitor.go:364] in ContinueOnErrorVisitor
E1108 15:34:33.097494   18406 visitor.go:404] in FlattenListVisitor
E1108 15:34:33.097501   18406 visitor.go:404] in FlattenListVisitor
E1108 15:34:33.097505   18406 visitor.go:216] in EagerVisitorList
E1108 15:34:33.097516   18406 visitor.go:526] in FileVisitor
E1108 15:34:33.097551   18406 visitor.go:592] in StreamVisitor
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
pod/nginx configured
E1108 15:34:33.110716   18406 visitor.go:599] after StreamVisitor
E1108 15:34:33.110748   18406 visitor.go:545] after FileVisitor
E1108 15:34:33.110756   18406 visitor.go:233] after EagerVisitorList
E1108 15:34:33.110764   18406 visitor.go:406] after FlattenListVisitor
E1108 15:34:33.110771   18406 visitor.go:406] after FlattenListVisitor
E1108 15:34:33.110778   18406 visitor.go:383] after ContinueOnErrorVisitor
E1108 15:34:33.110786   18406 visitor.go:337] after DecoratedVisitor
```

<br>

### 2. kubectl create -f pod.yaml

从上诉日志中，可以看出来create 一共用了 DecoratedVisitor，ContinueOnErrorVisitor，FlattenListVisitor， FlattenListVisitor，EagerVisitorList，FileVisitor， StreamVisitor 7个visitor。

接下来从代码角度看看是怎么实现的，为什么这么实现。

#### 2.1 kubectl代码中定义visitor

```
	r := f.NewBuilder().       // 和visitor无关，只赋值了一些变量,ToRESTConfig等等
		Unstructured().          // 因为是create,所以不确定是创建哪种对象，所以要用Unstructured
		Schema(schema).          // 进行schema赋值，方便校验
		ContinueOnError().       // 设置ContinueOnError=true
		NamespaceParam(cmdNamespace).DefaultNamespace().   // 设置命名空间
		FilenameParam(enforceNamespace, &o.FilenameOptions).    // 设置path
		LabelSelectorParam(o.Selector).                         // 设置label
		Flatten().                                              // 设置flatten=true
		Do()                             
```

<br>

```
func (b *Builder) Do() *Result {
  // 初始化visitor，kubectl create -f的情况下有：DecoratedVisitor，FlattenListVisitor，EagerVisitorList
	r := b.visitorResult()
	r.mapper = b.Mapper()
	if r.err != nil {
		return r
	}
	if b.flatten {
		r.visitor = NewFlattenListVisitor(r.visitor, b.objectTyper, b.mapper)
	}
	helpers := []VisitorFunc{}
	if b.defaultNamespace {
		helpers = append(helpers, SetNamespace(b.namespace))
	}
	if b.requireNamespace {
		helpers = append(helpers, RequireNamespace(b.namespace))
	}
	helpers = append(helpers, FilterNamespace)
	if b.requireObject {
		helpers = append(helpers, RetrieveLazy)
	}
	// 增加了ContinueOnErrorVisitor
	if b.continueOnError {
		r.visitor = NewDecoratedVisitor(ContinueOnErrorVisitor{r.visitor}, helpers...)
	} else {
		r.visitor = NewDecoratedVisitor(r.visitor, helpers...)
	}
	return r
}

```



```
func (b *Builder) visitorResult() *Result {
  。。。
	// visit items specified by paths
	// create -f pod.yaml走这条路线
	if len(b.paths) != 0 {
		return b.visitByPaths()
	}
  。。。
}

unc (b *Builder) visitByPaths() *Result {
	

	var visitors Visitor
	// 1.定义了EagerVisitorList,错误收集后统一返回
	if b.continueOnError {
		visitors = EagerVisitorList(b.paths)
	} else {
		visitors = VisitorList(b.paths)
	}
  
  // 2.定义了FlattenListVisitor，FlattenListVisitor将任何runtime.ExtractList转化为一个list－拥有一个公共字段"Items"
  // 如果create -f xx.yaml中有多个字段，那输出的objectlist就会有 Items这个字段
	if b.flatten {
		visitors = NewFlattenListVisitor(visitors, b.objectTyper, b.mapper)
	}
   
  // 3.定义了DecoratedVisitor，这如果有defaultNamespace，就设置默认的ns
	// only items from disk can be refetched
	if b.latest {
		// must set namespace prior to fetching
		if b.defaultNamespace {
			visitors = NewDecoratedVisitor(visitors, SetNamespace(b.namespace))
		}
		visitors = NewDecoratedVisitor(visitors, RetrieveLatest)
	}
	
   // 创建时没有selector,所以没有这个
	if b.labelSelector != nil {
		selector, err := labels.Parse(*b.labelSelector)
		if err != nil {
			return result.withError(fmt.Errorf("the provided selector %q is not valid: %v", *b.labelSelector, err))
		}
		visitors = NewFilteredVisitor(visitors, FilterByLabelSelector(selector))
	}
	result.visitor = visitors
	result.sources = b.paths
	return result
}
```

可以看出来，上面定义了DecoratedVisitor，ContinueOnErrorVisitor，FlattenListVisitor， FlattenListVisitor，EagerVisitorList。但是没有 FileVisitor， StreamVisitor 。

FileVisitor在最开始FilenameParam中定义了。从这里可以看出来StreamVisitor最先开始定义。

```
// FilenameParam groups input in two categories: URLs and files (files, directories, STDIN)
// If enforceNamespace is false, namespaces in the specs will be allowed to
// override the default namespace. If it is true, namespaces that don't match
// will cause an error.
// If ContinueOnError() is set prior to this method, objects on the path that are not
// recognized will be ignored (but logged at V(2)).
func (b *Builder) FilenameParam(enforceNamespace bool, filenameOptions *FilenameOptions) *Builder {

   for _, s := range paths {
      switch {
      case s == "-":
         // 文件方式
         b.Stdin()
}


// Stdin will read objects from the standard input. If ContinueOnError() is set
// prior to this method being called, objects in the stream that are unrecognized
// will be ignored (but logged at V(2)).
func (b *Builder) Stdin() *Builder {
	b.stream = true
	b.paths = append(b.paths, FileVisitorForSTDIN(b.mapper, b.schema))
	return b
}

// FileVisitorForSTDIN return a special FileVisitor just for STDIN
func FileVisitorForSTDIN(mapper *mapper, schema ContentValidator) Visitor {
	return &FileVisitor{
		Path:          constSTDINstr,
		StreamVisitor: NewStreamVisitor(nil, mapper, constSTDINstr, schema),
	}
}
```

#### 2.2 再次看kubectl create的输出结果

```
root@k8s-master:~# ./kubectl create -f pod.yaml -v 3
E1108 16:23:02.756637    6420 visitor.go:335] in DecoratedVisitor
E1108 16:23:02.756683    6420 visitor.go:364] in ContinueOnErrorVisitor
E1108 16:23:02.756688    6420 visitor.go:404] in FlattenListVisitor
E1108 16:23:02.756695    6420 visitor.go:404] in FlattenListVisitor
E1108 16:23:02.756698    6420 visitor.go:216] in EagerVisitorList
E1108 16:23:02.756703    6420 visitor.go:526] in FileVisitor
E1108 16:23:02.756740    6420 visitor.go:592] in StreamVisitor
I1108 16:23:02.760038    6420 create.go:260] info is {Client:0xc00132e000 Mapping:0xc001326000 Namespace:default Name:nginx Source:pod.yaml Object:0xc000ea1b68 ResourceVersion: Export:false}:
I1108 16:23:02.760354    6420 create.go:273] post data to apiserver
I1108 16:23:02.760375    6420 create.go:274] info is {Client:0xc00132e000 Mapping:0xc001326000 Namespace:default Name:nginx Source:pod.yaml Object:0xc000ea1b68 ResourceVersion: Export:false}:
pod/nginx created
E1108 16:23:02.772496    6420 visitor.go:599] after StreamVisitor
E1108 16:23:02.772523    6420 visitor.go:545] after FileVisitor
E1108 16:23:02.772531    6420 visitor.go:233] after EagerVisitorList
E1108 16:23:02.772538    6420 visitor.go:406] after FlattenListVisitor
E1108 16:23:02.772544    6420 visitor.go:406] after FlattenListVisitor
E1108 16:23:02.772551    6420 visitor.go:383] after ContinueOnErrorVisitor
E1108 16:23:02.772557    6420 visitor.go:337] after DecoratedVisitor
```

<br>

DecoratedVisitor -> ContinueOnErrorVisitor -> FlattenListVisitor -> EagerVisitorList -> FileVisitor ->  StreamVisitor -> info(post data to apiserver)

**整体的流程/处理顺序为：**

（1）DecoratedVisitor先执行DecoratedVisitor，这里是给默认的info，增加了默认的命名空间

（2）ContinueOnErrorVisitor是先执行下面的visitor，然后处理返回的错误

（3）FlattenListVisitor这里将info 的 runtime.ExtractList转化为一个list－拥有一个公共字段"Items"，先执行自己的

（4）EagerVisitorList 统一收集错误，先执行下面的visitor

（5）FileVisitor先执行自己的，再执行下面的

（6）StreamVisitor先执行自己的，再执行下面的

（7）发送到 apiserver

### 3. 总结

（1）下面这一套是关键，定义好了各种visitor，然后利用r.visit执行。感觉如果要自己定制化一个的话，第一是需要明确各种visitor，第二就是想清楚处理顺序。

```
	r := f.NewBuilder().
		Unstructured().
		Schema(schema).
		ContinueOnError().
		NamespaceParam(cmdNamespace).DefaultNamespace().
		FilenameParam(enforceNamespace, &o.FilenameOptions).
		LabelSelectorParam(o.Selector).
		Flatten().
		Do()
```

（2）目前看了kubectl create都是在发送数据到apiserver之前对 info进行了处理。还没有利用到返回了数据，再额外处理的情况。