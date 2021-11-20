Table of Contents
=================

  * [1. Visitor 接口](#1-visitor-接口)
  * [2. visitor种类](#2-visitor种类)
     * [2.1 StreamVisitor](#21-streamvisitor)
     * [2.2 FileVisitor](#22-filevisitor)
     * [2.3 URLvisitor](#23-urlvisitor)
     * [2.4 KustomizeVisitor](#24-kustomizevisitor)
     * [2.5 Selector](#25-selector)
     * [2.5 InfoListVisitor](#25-infolistvisitor)
     * [2.6 FilteredVisitor](#26-filteredvisitor)
     * [2.7 DecoratedVisitor](#27-decoratedvisitor)
     * [2.8 ContinueOnErrorVisitor](#28-continueonerrorvisitor)
     * [2.9 FlattenListVisitor](#29-flattenlistvisitor)
     * [2.10 EagerVisitorList](#210-eagervisitorlist)
     * [2.11 VisitorList](#211-visitorlist)
  * [3 总结](#3-总结)

### 1. Visitor 接口

visitor接口和上文描述的一致。一个visit函数，函数参数为VisitorFunc

```
// Visitor lets clients walk a list of resources.
type Visitor interface {
	Visit(VisitorFunc) error
}

// VisitorFunc implements the Visitor interface for a matching function.
// If there was a problem walking a list of resources, the incoming error
// will describe the problem and the function can decide how to handle that error.
// A nil returned indicates to accept an error to continue loops even when errors happen.
// This is useful for ignoring certain kinds of errors or aggregating errors in some way.
type VisitorFunc func(*Info, error) error
```

### 2. visitor种类

#### 2.1 StreamVisitor

StreamVisitor就是根据json, 或者yaml中的内容，生成Info信息。

```
// StreamVisitor reads objects from an io.Reader and walks them. A stream visitor can only be
// visited once.
// TODO: depends on objects being in JSON format before being passed to decode - need to implement
// a stream decoder method on runtime.Codec to properly handle this.
type StreamVisitor struct {
	io.Reader
	*mapper

	Source string   //这个source是yaml ,json这种对象来源的含义
	Schema ContentValidator    
}

// NewStreamVisitor is a helper function that is useful when we want to change the fields of the struct but keep calls the same.
func NewStreamVisitor(r io.Reader, mapper *mapper, source string, schema ContentValidator) *StreamVisitor {
	return &StreamVisitor{
		Reader: r,
		mapper: mapper,
		Source: source,
		Schema: schema,
	}
}


// Visit implements Visitor over a stream. StreamVisitor is able to distinct multiple resources in one stream.
func (v *StreamVisitor) Visit(fn VisitorFunc) error {
  // 1.从这里也能看出来，只支持yaml,json两种文件格式
	d := yaml.NewYAMLOrJSONDecoder(v.Reader, 4096)
	for {
		ext := runtime.RawExtension{}
		if err := d.Decode(&ext); err != nil {
			if err == io.EOF {
				return nil
			}
			return fmt.Errorf("error parsing %s: %v", v.Source, err)
		}
		// TODO: This needs to be able to handle object in other encodings and schemas.
		ext.Raw = bytes.TrimSpace(ext.Raw)
		if len(ext.Raw) == 0 || bytes.Equal(ext.Raw, []byte("null")) {
			continue
		}
		// 2.利用Factory的validator进行验证。staging/src/k8s.io/kubectl/pkg/validation/schema.go
		if err := ValidateSchema(ext.Raw, v.Schema); err != nil {
			return fmt.Errorf("error validating %q: %v", v.Source, err)
		}
		
		// 3.InfoForData用传入的数据生成一个Info object。会把json对象转换成对应的struct类型
		info, err := v.infoForData(ext.Raw, v.Source)
		if err != nil {
			if fnErr := fn(info, err); fnErr != nil {
				return fnErr
			}
			continue
		}
		// 4.调用fn函数，可以看出来是先执行该selector的处理逻辑，在执行其他visitor的逻辑
		if err := fn(info, nil); err != nil {
			return err
		}
	}
}


// InfoForData用传入的数据生成一个Info object。会把json对象转换成对应的struct类型
// InfoForData creates an Info object for the given data. An error is returned
// if any of the decoding or client lookup steps fail. Name and namespace will be
// set into Info if the mapping's MetadataAccessor can retrieve them.
func (m *mapper) infoForData(data []byte, source string) (*Info, error) {
	obj, gvk, err := m.decoder.Decode(data, nil, nil)
	if err != nil {
		return nil, fmt.Errorf("unable to decode %q: %v", source, err)
	}

	name, _ := metadataAccessor.Name(obj)
	namespace, _ := metadataAccessor.Namespace(obj)
	resourceVersion, _ := metadataAccessor.ResourceVersion(obj)

	ret := &Info{
		Source:          source,
		Namespace:       namespace,
		Name:            name,
		ResourceVersion: resourceVersion,

		Object: obj,
	}

	if m.localFn == nil || !m.localFn() {
		restMapper, err := m.restMapperFn()
		if err != nil {
			return nil, err
		}
		mapping, err := restMapper.RESTMapping(gvk.GroupKind(), gvk.Version)
		if err != nil {
			return nil, fmt.Errorf("unable to recognize %q: %v", source, err)
		}
		ret.Mapping = mapping

		client, err := m.clientFn(gvk.GroupVersion())
		if err != nil {
			return nil, fmt.Errorf("unable to connect to a server to handle %q: %v", mapping.Resource, err)
		}
		ret.Client = client
	}

	return ret, nil
}
```

#### 2.2 FileVisitor

FileVisitor封装了一个type StreamVisitor struct，用于处理open/close files

```
// FileVisitor is wrapping around a StreamVisitor, to handle open/close files
type FileVisitor struct {
	Path string
	*StreamVisitor
}

// Visit in a FileVisitor is just taking care of opening/closing files
func (v *FileVisitor) Visit(fn VisitorFunc) error {
	var f *os.File
	if v.Path == constSTDINstr {
		f = os.Stdin
	} else {
		var err error
		f, err = os.Open(v.Path)
		if err != nil {
			return err
		}
		defer f.Close()
	}

	// TODO: Consider adding a flag to force to UTF16, apparently some
	// Windows tools don't write the BOM
	utf16bom := unicode.BOMOverride(unicode.UTF8.NewDecoder())
	v.StreamVisitor.Reader = transform.NewReader(f, utf16bom)

	return v.StreamVisitor.Visit(fn)
}
```

#### 2.3 URLvisitor

URLVisitor下载URL的内容，如果成功，返回一个表示info object代表URL的信息。封装了一个StreamVisitor

```
// URLVisitor downloads the contents of a URL, and if successful, returns
// an info object representing the downloaded object.
type URLVisitor struct {
   URL *url.URL
   *StreamVisitor
   HttpAttemptCount int
}

func (v *URLVisitor) Visit(fn VisitorFunc) error {
   body, err := readHttpWithRetries(httpgetImpl, time.Second, v.URL.String(), v.HttpAttemptCount)
   if err != nil {
      return err
   }
   defer body.Close()
   v.StreamVisitor.Reader = body
   return v.StreamVisitor.Visit(fn)
}
```

#### 2.4 KustomizeVisitor

这个和file, url一样，也是一种输入的格式。例如：

通过  `kubectl apply -k <kustomization_directory>` 创建应用。

详见：https://kubernetes.io/zh/docs/tasks/manage-kubernetes-objects/kustomization/

这个最终也是调用了StreamVisitor进行了处理。

```
// KustomizeVisitor is wrapper around a StreamVisitor, to handle Kustomization directories
type KustomizeVisitor struct {
   Path string
   *StreamVisitor
}

// Visit in a KustomizeVisitor gets the output of Kustomize build and save it in the Streamvisitor
func (v *KustomizeVisitor) Visit(fn VisitorFunc) error {
   fSys := fs.MakeRealFS()
   var out bytes.Buffer
   err := kustomize.RunKustomizeBuild(&out, fSys, v.Path)
   if err != nil {
      return err
   }
   v.StreamVisitor.Reader = bytes.NewReader(out.Bytes())
   return v.StreamVisitor.Visit(fn)
}
```







#### 2.5 Selector

Selector是一个resources的Visitor，实现了label selector。

该selector就是填充info的 ListOptions，然后再执行其他的selector

```
// Selector is a Visitor for resources that match a label selector.
type Selector struct {
	Client        RESTClient
	Mapping       *meta.RESTMapping
	Namespace     string
	LabelSelector string
	FieldSelector string
	Export        bool
	LimitChunks   int64
}

// NewSelector创建一个资源选择器，它隐藏由标签选择器获取项目的细节。
// NewSelector creates a resource selector which hides details of getting items by their label selector.
func NewSelector(client RESTClient, mapping *meta.RESTMapping, namespace, labelSelector, fieldSelector string, export bool, limitChunks int64) *Selector {
	return &Selector{
		Client:        client,
		Mapping:       mapping,
		Namespace:     namespace,
		LabelSelector: labelSelector,
		FieldSelector: fieldSelector,
		Export:        export,
		LimitChunks:   limitChunks,
	}
}

// Visit implements Visitor and uses request chunking by default.
func (r *Selector) Visit(fn VisitorFunc) error {
	var continueToken string
	for {
		list, err := NewHelper(r.Client, r.Mapping).List(
			r.Namespace,
			r.ResourceMapping().GroupVersionKind.GroupVersion().String(),
			r.Export,
			&metav1.ListOptions{
				LabelSelector: r.LabelSelector,
				FieldSelector: r.FieldSelector,
				Limit:         r.LimitChunks,
				Continue:      continueToken,
			},
		)
		if err != nil {
			if errors.IsResourceExpired(err) {
				return err
			}
			if errors.IsBadRequest(err) || errors.IsNotFound(err) {
				if se, ok := err.(*errors.StatusError); ok {
					// modify the message without hiding this is an API error
					if len(r.LabelSelector) == 0 && len(r.FieldSelector) == 0 {
						se.ErrStatus.Message = fmt.Sprintf("Unable to list %q: %v", r.Mapping.Resource, se.ErrStatus.Message)
					} else {
						se.ErrStatus.Message = fmt.Sprintf("Unable to find %q that match label selector %q, field selector %q: %v", r.Mapping.Resource, r.LabelSelector, r.FieldSelector, se.ErrStatus.Message)
					}
					return se
				}
				if len(r.LabelSelector) == 0 && len(r.FieldSelector) == 0 {
					return fmt.Errorf("Unable to list %q: %v", r.Mapping.Resource, err)
				}
				return fmt.Errorf("Unable to find %q that match label selector %q, field selector %q: %v", r.Mapping.Resource, r.LabelSelector, r.FieldSelector, err)
			}
			return err
		}
		resourceVersion, _ := metadataAccessor.ResourceVersion(list)
		nextContinueToken, _ := metadataAccessor.Continue(list)
		info := &Info{
			Client:  r.Client,
			Mapping: r.Mapping,

			Namespace:       r.Namespace,
			ResourceVersion: resourceVersion,

			Object: list,
		}
    
    // 调用fn函数，可以看出来是先执行该selector的处理逻辑，在执行其他visitor的逻辑
		if err := fn(info, nil); err != nil {
			return err
		}
		if len(nextContinueToken) == 0 {
			return nil
		}
		continueToken = nextContinueToken
	}
}
```

#### 2.5 InfoListVisitor

InfoListVisitor就是多个info对对象的集合。就是该visitor同时对多个info进行处理。例如kubectl create -f yaml中。yaml定义了多个资源对象的情况

```
type InfoListVisitor []*Info

func (infos InfoListVisitor) Visit(fn VisitorFunc) error {
	var err error
	for _, i := range infos {
		err = fn(i, err)
	}
	return err
}

```

<br>

#### 2.6 FilteredVisitor

FilteredVisitor可以检查info是否满足某些条件。如果满足条件，则往下执行，否则返回err。

FilterFunc函数在初始化FilteredVisitor的时候定义好。

```
type FilterFunc func(info *Info, err error) (bool, error)

type FilteredVisitor struct {
	visitor Visitor
	filters []FilterFunc
}

func NewFilteredVisitor(v Visitor, fn ...FilterFunc) Visitor {
	if len(fn) == 0 {
		return v
	}
	return FilteredVisitor{v, fn}
}

func (v FilteredVisitor) Visit(fn VisitorFunc) error {
	return v.visitor.Visit(func(info *Info, err error) error {
		if err != nil {
			return err
		}
		for _, filter := range v.filters {
			ok, err := filter(info, nil)
			if err != nil {
				return err
			}
			if !ok {
				return nil
			}
		}
		// 最后在调用fn
		return fn(info, nil)
	})
}
```

#### 2.7 DecoratedVisitor

在调用visitor function之前，DecoratedVisitor将调用decorators。错误将终止visit函数。

NewDecoratedVisitor将在调用用户提供的visitor function之前，创建一个visitor来调用入参visitor functions，让他们有机会改变 Info对象或提前return error。

```
// DecoratedVisitor will invoke the decorators in order prior to invoking the visitor function
// passed to Visit. An error will terminate the visit.
type DecoratedVisitor struct {
	visitor    Visitor
	decorators []VisitorFunc
}

// NewDecoratedVisitor will create a visitor that invokes the provided visitor functions before
// the user supplied visitor function is invoked, giving them the opportunity to mutate the Info
// object or terminate early with an error.
func NewDecoratedVisitor(v Visitor, fn ...VisitorFunc) Visitor {
	if len(fn) == 0 {
		return v
	}
	return DecoratedVisitor{v, fn}
}

// Visit implements Visitor
func (v DecoratedVisitor) Visit(fn VisitorFunc) error {
	return v.visitor.Visit(func(info *Info, err error) error {
		if err != nil {
			return err
		}
		for i := range v.decorators {
			if err := v.decorators[i](info, nil); err != nil {
				return err
			}
		}
		// 也是最后才执行fn
		return fn(info, nil)
	})
}
```

#### 2.8 ContinueOnErrorVisitor

ContinueOnErrorVisitor访问每个item，如果任何一个item发生错误，则在访问所有item后返回一个聚合错误。

如果遍历期间没有发生错误，func (v ContinueOnErrorVisitor) Visit返回nil。
		如果发生错误，或者发生多个错误，则返回聚合错误。
		如果指定的visitor在任何单独的item上失败，它不会阻止其余的item被访问。
		visitor直接返回error，可能会导致一个items没有被访问到。
	收集子Visitor产生的错误，并返回。

```
// ContinueOnErrorVisitor visits each item and, if an error occurs on
// any individual item, returns an aggregate error after all items
// are visited.
type ContinueOnErrorVisitor struct {
	Visitor
}

// Visit returns nil if no error occurs during traversal, a regular
// error if one occurs, or if multiple errors occur, an aggregate
// error.  If the provided visitor fails on any individual item it
// will not prevent the remaining items from being visited. An error
// returned by the visitor directly may still result in some items
// not being visited.
func (v ContinueOnErrorVisitor) Visit(fn VisitorFunc) error {
	errs := []error{}
	
	// 从这里可以看出来，执行其他的visitor就算出错误了，也是返回nil
	err := v.Visitor.Visit(func(info *Info, err error) error {
		if err != nil {
			errs = append(errs, err)
			return nil
		}
		// 先执行的fn函数
		if err := fn(info, nil); err != nil {
			errs = append(errs, err)
		}
		return nil
	})
	
	
	if err != nil {
		errs = append(errs, err)
	}
	if len(errs) == 1 {
		return errs[0]
	}
	return utilerrors.NewAggregate(errs)
}
```

#### 2.9 FlattenListVisitor



```
FlattenListVisitor将任何runtime.ExtractList转化为一个list－拥有一个公共字段"Items"。
	"Items" 是一个runtime.Objects切片
	任何子item的错误（例如，如果列表中包含没有注册的客户端或资源的对象）将终止FlattenListVisitor的visit函数。
	
// FlattenListVisitor flattens any objects that runtime.ExtractList recognizes as a list
// - has an "Items" public field that is a slice of runtime.Objects or objects satisfying
// that interface - into multiple Infos. Returns nil in the case of no errors.
// When an error is hit on sub items (for instance, if a List contains an object that does
// not have a registered client or resource), returns an aggregate error.
type FlattenListVisitor struct {
   visitor Visitor
   typer   runtime.ObjectTyper
   mapper  *mapper
}

NewFlattenListVisitor创建一个visitor，它将list样式的runtime.Objects扩展成单独的items，然后单独访问它们。
// NewFlattenListVisitor creates a visitor that will expand list style runtime.Objects
// into individual items and then visit them individually.
func NewFlattenListVisitor(v Visitor, typer runtime.ObjectTyper, mapper *mapper) Visitor {
   return FlattenListVisitor{v, typer, mapper}
}

func (v FlattenListVisitor) Visit(fn VisitorFunc) error {
   return v.visitor.Visit(func(info *Info, err error) error {
      if err != nil {
         return err
      }
      if info.Object == nil {
         return fn(info, nil)
      }
      if !meta.IsListType(info.Object) {
         return fn(info, nil)
      }

      items := []runtime.Object{}
      itemsToProcess := []runtime.Object{info.Object}

      for i := 0; i < len(itemsToProcess); i++ {
         currObj := itemsToProcess[i]
         if !meta.IsListType(currObj) {
            items = append(items, currObj)
            continue
         }

         currItems, err := meta.ExtractList(currObj)
         if err != nil {
            return err
         }
         if errs := runtime.DecodeList(currItems, v.mapper.decoder); len(errs) > 0 {
            return utilerrors.NewAggregate(errs)
         }
         itemsToProcess = append(itemsToProcess, currItems...)
      }

      // If we have a GroupVersionKind on the list, prioritize that when asking for info on the objects contained in the list
      var preferredGVKs []schema.GroupVersionKind
      if info.Mapping != nil && !info.Mapping.GroupVersionKind.Empty() {
         preferredGVKs = append(preferredGVKs, info.Mapping.GroupVersionKind)
      }
      errs := []error{}
      for i := range items {
         item, err := v.mapper.infoForObject(items[i], v.typer, preferredGVKs)
         if err != nil {
            errs = append(errs, err)
            continue
         }
         if len(info.ResourceVersion) != 0 {
            item.ResourceVersion = info.ResourceVersion
         }
         if err := fn(item, nil); err != nil {
            errs = append(errs, err)
         }
      }
      return utilerrors.NewAggregate(errs)

   })
}
```

#### 2.10 EagerVisitorList

EagerVisitorList 实现其包含的子Visitor的Visit方法。在遍历其子Visitor的过程中，所有的error会被收集起来，在迭代结束后一起return

```
// EagerVisitorList implements Visit for the sub visitors it contains. All errors
// will be captured and returned at the end of iteration.
type EagerVisitorList []Visitor

// Visit implements Visitor, and gathers errors that occur during processing until
// all sub visitors have been visited.
func (l EagerVisitorList) Visit(fn VisitorFunc) error {
	errs := []error(nil)
	for i := range l {
		if err := l[i].Visit(func(info *Info, err error) error {
			if err != nil {
				errs = append(errs, err)
				return nil
			}
			if err := fn(info, nil); err != nil {
				errs = append(errs, err)
			}
			return nil
		}); err != nil {
			errs = append(errs, err)
		}
	}
	return utilerrors.NewAggregate(errs)
}
```

#### 2.11 VisitorList

VisitorList 实现其包含的子Visitor的Visit方法。在遍历其子Visitor的过程中，只要出现error，VisitorList的Visit立刻return

```
// VisitorList implements Visit for the sub visitors it contains. The first error
// returned from a child Visitor will terminate iteration.
type VisitorList []Visitor

// Visit implements Visitor
func (l VisitorList) Visit(fn VisitorFunc) error {
	for i := range l {
		if err := l[i].Visit(fn); err != nil {
			return err
		}
	}
	return nil
}
```

### 3 总结

Builder中的Do()函数返回一个Result，而Visitor是Result里面最重要的数据结构。本文梳理了kubectl定义的所有visitor。到这里就已经基本理清楚kubectl factory的主要逻辑：

（1）factory主要包含了builder结构体，该结构包含了所有的配置信息。builder中包含了visitor结构体，然后f.NewBuilder().xx.xx.xx.Do() 定义好了所有的visitor。

（2）最后再通过调用r.visit，使得整个链路的visitor都执行起来。

```
err = r.Visit(func(info *resource.Info, err error) error {
		if err != nil {
			return err
		}
		if err := util.CreateOrUpdateAnnotation(cmdutil.GetFlagBool(cmd, cmdutil.ApplyAnnotationsFlag), info.Object, scheme.DefaultJSONEncoder()); err != nil {
			return cmdutil.AddSourceToErr("creating", info.Source, err)
		}

		if err := o.Recorder.Record(info.Object); err != nil {
			klog.V(4).Infof("error recording current command: %v", err)
		}

		if !o.DryRun {
			if err := createAndRefresh(info); err != nil {
				return cmdutil.AddSourceToErr("creating", info.Source, err)
			}
		}

		count++

		return o.PrintObj(info.Object)
	})
```

