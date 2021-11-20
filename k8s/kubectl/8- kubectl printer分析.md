Table of Contents
=================

  * [1. kubectl 强大的格式化输出](#1-kubectl-强大的格式化输出)
     * [1.1 常见的用法： kubectl -o/--output  json, yaml， wide](#11-常见的用法-kubectl--o--output--json-yaml-wide)
     * [1.2 custom-columns](#12-custom-columns)
     * [1.3 go-template， go-template-file](#13-go-template-go-template-file)
     * [1.4 jsonpath，jsonpath-file](#14-jsonpathjsonpath-file)
  * [2. kubectl Printer源码分析](#2-kubectl-printer源码分析)
     * [2.1 kubectl create定义printor的过程](#21-kubectl-create定义printor的过程)
     * [2.2 各种printor的实现](#22-各种printor的实现)
        * [2.2.1 JSONYamlPrint](#221-jsonyamlprint)
        * [2.2.2 NamePrinter](#222-nameprinter)
        * [2.2.3 GoTemplatePrinter](#223-gotemplateprinter)
  * [3.总结](#3总结)

### 1. kubectl 强大的格式化输出

Printer是kubectl 命令在输出的时候设置的显示格式。kubectl有着强大的输出格式。本文研究一下kubectl对Printer的实现。

从kubectl -h可以看出来，kubectl有着以下的定制化输出：

```
Usage:
  kubectl get
[(-o|--output=)json|yaml|wide|custom-columns=...|custom-columns-file=...|go-template=...|go-template-file=...|jsonpath=...|jsonpath-file=...]
(TYPE[.VERSION][.GROUP] [NAME | -l label] | TYPE[.VERSION][.GROUP]/NAME ...) [flags] [options]
```

#### 1.1 常见的用法： kubectl -o/--output  json, yaml， wide

#### 1.2 custom-columns

kubectl  custom-columns, custom-columns-file (这两个是一样的，只不过第二个把格式定义在了文件中)

```
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[*].restartCount,CONATAINER_NAME:.spec.containers[*].name,READY:.status.containerStatuses[*].ready
NAME     STATUS    RESTARTS   CONATAINER_NAME   READY
nginx    Running   5          nginx             true
nginx1   Running   0,0        nginx,nginx1      true,true
```

这个很经典，它将所有的字段都可以定制化输出。

其中类似containerStatuses，containers这种数组情况的，一定要加[]。 `[*]`表达所有的元素。就是上面的那个，没有`*`的表示只输出第一个，如下：

```
root@k8s-master:~# kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[].restartCount,CONATAINER_NAME:.spec.containers[].name,READY:.status.containerStatuses[].ready
NAME     STATUS    RESTARTS   CONATAINER_NAME   READY
nginx    Running   5          nginx             true
nginx1   Running   0          nginx             true
```

#### 1.3 go-template， go-template-file

 go-template， go-template-file效果是一样的

go-template的强大还在于可以if else语句

```
root@k8s-master:~# kubectl get pods -o go-template --template='{{range .items}}{{printf "%s %s\n" .metadata.name .metadata.creationTimestamp}}{{end}}'
nginx 2021-11-08T08:23:02Z
```

#### 1.4 jsonpath，jsonpath-file

这两个效果也是一样的

```
root@k8s-master:~# kubectl get pods -o=jsonpath="{range .items[*]}{.metadata.name}{'\t'}{.status.startTime}{'\n'}{end}"
nginx   2021-11-08T08:23:02Z
```

<br>

### 2. kubectl Printer源码分析

kubectl creat 和 get 在格式化输出的时候其实是一样的。总体的流程就是：

（1）通过cmd 获取 printFlags

（2）根据printFlags选择Printor

（3）不同的Printor对info对象进行不同的格式化输出

<br>

接下来继续从kubectl create命令出发，从源码角度看看是如何实现的。

#### 2.1 kubectl create定义printor的过程

（1）绑定PrintFlags参数

（2）在运行RunCreate之前，通过Complete函数，补全了CreateOptions。

（3）Complete函数根据参数，实例化printor

（4）根据不同的output指定的格式，选择不同的printor。可以看出来就是对应上面那几种printor

<br>

（1）绑定PrintFlags参数

```
// CreateOptions is the commandline options for 'create' sub command
type CreateOptions struct {
	PrintFlags  *genericclioptions.PrintFlags
}

// PrintFlags composes common printer flag structs
// used across all commands, and provides a method
// of retrieving a known printer based on flag values provided.
type PrintFlags struct {
	JSONYamlPrintFlags   *JSONYamlPrintFlags
	NamePrintFlags       *NamePrintFlags
	TemplatePrinterFlags *KubeTemplatePrintFlags

	TypeSetterPrinter *printers.TypeSetterPrinter

	OutputFormat *string

	// OutputFlagSpecified indicates whether the user specifically requested a certain kind of output.
	// Using this function allows a sophisticated caller to change the flag binding logic if they so desire.
	OutputFlagSpecified func() bool
}

NewCmdCreate函数中有这样一句语句
o.PrintFlags.AddFlags(cmd)

// 这里就直接绑定了参数
func (f *PrintFlags) AddFlags(cmd *cobra.Command) {
	f.JSONYamlPrintFlags.AddFlags(cmd)
	f.NamePrintFlags.AddFlags(cmd)
	f.TemplatePrinterFlags.AddFlags(cmd)

	if f.OutputFormat != nil {
		cmd.Flags().StringVarP(f.OutputFormat, "output", "o", *f.OutputFormat, fmt.Sprintf("Output format. One of: %s.", strings.Join(f.AllowedFormats(), "|")))
		if f.OutputFlagSpecified == nil {
			f.OutputFlagSpecified = func() bool {
				return cmd.Flag("output").Changed
			}
		}
	}
}
```

（2）在运行RunCreate之前，通过Complete函数，补全了CreateOptions。

```
// NewCmdCreate returns new initialized instance of create sub command
func NewCmdCreate(f cmdutil.Factory, ioStreams genericclioptions.IOStreams) *cobra.Command {
	o := NewCreateOptions(ioStreams)

	cmd := &cobra.Command{
		Use:                   "create -f FILENAME",
		DisableFlagsInUseLine: true,
		Short:                 i18n.T("Create a resource from a file or from stdin."),
		Long:                  createLong,
		Example:               createExample,
		Run: func(cmd *cobra.Command, args []string) {
			if cmdutil.IsFilenameSliceEmpty(o.FilenameOptions.Filenames, o.FilenameOptions.Kustomize) {
				ioStreams.ErrOut.Write([]byte("Error: must specify one of -f and -k\n\n"))
				defaultRunFunc := cmdutil.DefaultSubCommandRun(ioStreams.ErrOut)
				defaultRunFunc(cmd, args)
				return
			}
			// 调用了Complete函数，补全了CreateOptions
			cmdutil.CheckErr(o.Complete(f, cmd))
			cmdutil.CheckErr(o.ValidateArgs(cmd, args))
			cmdutil.CheckErr(o.RunCreate(f, cmd))
		},
	}
```

<br>

（3）Complete函数根据参数，实例化printor

```
// Complete completes all the required options
func (o *CreateOptions) Complete(f cmdutil.Factory, cmd *cobra.Command) error {
	var err error
	o.RecordFlags.Complete(cmd)
	o.Recorder, err = o.RecordFlags.ToRecorder()
	if err != nil {
		return err
	}

	o.DryRun = cmdutil.GetDryRunFlag(cmd)

	if o.DryRun {
		o.PrintFlags.Complete("%s (dry run)")
	}
	// 根据参数，实例化printer
	printer, err := o.PrintFlags.ToPrinter()
	if err != nil {
		return err
	}

	o.PrintObj = func(obj kruntime.Object) error {
		return printer.PrintObj(obj, o.Out)
	}

	return nil
}
```

(4)  根据不同的output指定的格式，选择不同的printor。可以看出来就是对应上面那几种printor

```
func (f *PrintFlags) ToPrinter() (printers.ResourcePrinter, error) {
	outputFormat := ""
	if f.OutputFormat != nil {
		outputFormat = *f.OutputFormat
	}
	// For backwards compatibility we want to support a --template argument given, even when no --output format is provided.
	// If no explicit output format has been provided via the --output flag, fallback
	// to honoring the --template argument.
	templateFlagSpecified := f.TemplatePrinterFlags != nil &&
		f.TemplatePrinterFlags.TemplateArgument != nil &&
		len(*f.TemplatePrinterFlags.TemplateArgument) > 0
	outputFlagSpecified := f.OutputFlagSpecified != nil && f.OutputFlagSpecified()
	if templateFlagSpecified && !outputFlagSpecified {
		outputFormat = "go-template"
	}

	if f.JSONYamlPrintFlags != nil {
		if p, err := f.JSONYamlPrintFlags.ToPrinter(outputFormat); !IsNoCompatiblePrinterError(err) {
			return f.TypeSetterPrinter.WrapToPrinter(p, err)
		}
	}

	if f.NamePrintFlags != nil {
		if p, err := f.NamePrintFlags.ToPrinter(outputFormat); !IsNoCompatiblePrinterError(err) {
			return f.TypeSetterPrinter.WrapToPrinter(p, err)
		}
	}

	if f.TemplatePrinterFlags != nil {
		if p, err := f.TemplatePrinterFlags.ToPrinter(outputFormat); !IsNoCompatiblePrinterError(err) {
			return f.TypeSetterPrinter.WrapToPrinter(p, err)
		}
	}

	return nil, NoCompatiblePrinterError{OutputFormat: f.OutputFormat, AllowedFormats: f.AllowedFormats()}
}
```

#### 2.2 各种printor的实现

##### 2.2.1 JSONYamlPrint

会再根据outputFormat 分为JSONPrinter, YAMLPrinter

```
// ToPrinter receives an outputFormat and returns a printer capable of
// handling --output=(yaml|json) printing.
// Returns false if the specified outputFormat does not match a supported format.
// Supported Format types can be found in pkg/printers/printers.go
func (f *JSONYamlPrintFlags) ToPrinter(outputFormat string) (printers.ResourcePrinter, error) {
	var printer printers.ResourcePrinter

	outputFormat = strings.ToLower(outputFormat)
	switch outputFormat {
	case "json":
		printer = &printers.JSONPrinter{}
	case "yaml":
		printer = &printers.YAMLPrinter{}
	default:
		return nil, NoCompatiblePrinterError{OutputFormat: &outputFormat, AllowedFormats: f.AllowedFormats()}
	}

	return printer, nil
}
```

<br>

**jsonPrinter**

用法：kubectl get pod -o json

```
// PrintObj is an implementation of ResourcePrinter.PrintObj which simply writes the object to the Writer.
func (p *JSONPrinter) PrintObj(obj runtime.Object, w io.Writer) error {
	// we use reflect.Indirect here in order to obtain the actual value from a pointer.
	// we need an actual value in order to retrieve the package path for an object.
	// using reflect.Indirect indiscriminately is valid here, as all runtime.Objects are supposed to be pointers.
	if InternalObjectPreventer.IsForbidden(reflect.Indirect(reflect.ValueOf(obj)).Type().PkgPath()) {
		return fmt.Errorf(InternalObjectPrinterErr)
	}

	switch obj := obj.(type) {
	case *metav1.WatchEvent:
		if InternalObjectPreventer.IsForbidden(reflect.Indirect(reflect.ValueOf(obj.Object.Object)).Type().PkgPath()) {
			return fmt.Errorf(InternalObjectPrinterErr)
		}
		// 调用"encoding/json" package对对象进行格式化
		data, err := json.Marshal(obj)
		if err != nil {
			return err
		}
		_, err = w.Write(data)
		if err != nil {
			return err
		}
		_, err = w.Write([]byte{'\n'})
		return err
	case *runtime.Unknown:
		var buf bytes.Buffer
		err := json.Indent(&buf, obj.Raw, "", "    ")
		if err != nil {
			return err
		}
		buf.WriteRune('\n')
		_, err = buf.WriteTo(w)
		return err
	}

	if obj.GetObjectKind().GroupVersionKind().Empty() {
		return fmt.Errorf("missing apiVersion or kind; try GetObjectKind().SetGroupVersionKind() if you know the type")
	}

	data, err := json.MarshalIndent(obj, "", "    ")
	if err != nil {
		return err
	}
	data = append(data, '\n')
	_, err = w.Write(data)
	return err
}
```

**YAMLPrinter**看起来是先将Obj转成了json，然后再从json转成yaml

用法：kubectl get pod -o yaml

```
// PrintObj prints the data as YAML.
func (p *YAMLPrinter) PrintObj(obj runtime.Object, w io.Writer) error {
	// we use reflect.Indirect here in order to obtain the actual value from a pointer.
	// we need an actual value in order to retrieve the package path for an object.
	// using reflect.Indirect indiscriminately is valid here, as all runtime.Objects are supposed to be pointers.
	if InternalObjectPreventer.IsForbidden(reflect.Indirect(reflect.ValueOf(obj)).Type().PkgPath()) {
		return fmt.Errorf(InternalObjectPrinterErr)
	}

	count := atomic.AddInt64(&p.printCount, 1)
	if count > 1 {
		if _, err := w.Write([]byte("---\n")); err != nil {
			return err
		}
	}

	switch obj := obj.(type) {
	case *metav1.WatchEvent:
		if InternalObjectPreventer.IsForbidden(reflect.Indirect(reflect.ValueOf(obj.Object.Object)).Type().PkgPath()) {
			return fmt.Errorf(InternalObjectPrinterErr)
		}
		// 看起来是先转成了json？
		data, err := json.Marshal(obj)
		if err != nil {
			return err
		}
		data, err = yaml.JSONToYAML(data)
		if err != nil {
			return err
		}
		_, err = w.Write(data)
		return err
	case *runtime.Unknown:
		data, err := yaml.JSONToYAML(obj.Raw)
		if err != nil {
			return err
		}
		_, err = w.Write(data)
		return err
	}

	if obj.GetObjectKind().GroupVersionKind().Empty() {
		return fmt.Errorf("missing apiVersion or kind; try GetObjectKind().SetGroupVersionKind() if you know the type")
	}

	output, err := yaml.Marshal(obj)
	if err != nil {
		return err
	}
	_, err = fmt.Fprint(w, string(output))
	return err
}
```

##### 2.2.2 NamePrinter

就是只打印resource/name

eg:

```
root@k8s-master:~# kubectl get pod -o name
pod/nginx
pod/nginx1
```

```
// PrintObj is an implementation of ResourcePrinter.PrintObj which decodes the object
// and print "resource/name" pair. If the object is a List, print all items in it.
func (p *NamePrinter) PrintObj(obj runtime.Object, w io.Writer) error {
	switch castObj := obj.(type) {
	case *metav1.WatchEvent:
		obj = castObj.Object.Object
	}

	// we use reflect.Indirect here in order to obtain the actual value from a pointer.
	// using reflect.Indirect indiscriminately is valid here, as all runtime.Objects are supposed to be pointers.
	// we need an actual value in order to retrieve the package path for an object.
	if InternalObjectPreventer.IsForbidden(reflect.Indirect(reflect.ValueOf(obj)).Type().PkgPath()) {
		return fmt.Errorf(InternalObjectPrinterErr)
	}

	if meta.IsListType(obj) {
		// we allow unstructured lists for now because they always contain the GVK information.  We should chase down
		// callers and stop them from passing unflattened lists
		// TODO chase the caller that is setting this and remove it.
		if _, ok := obj.(*unstructured.UnstructuredList); !ok {
			return fmt.Errorf("list types are not supported by name printing: %T", obj)
		}

		items, err := meta.ExtractList(obj)
		if err != nil {
			return err
		}
		for _, obj := range items {
			if err := p.PrintObj(obj, w); err != nil {
				return err
			}
		}
		return nil
	}

	if obj.GetObjectKind().GroupVersionKind().Empty() {
		return fmt.Errorf("missing apiVersion or kind; try GetObjectKind().SetGroupVersionKind() if you know the type")
	}

	name := "<unknown>"
	if acc, err := meta.Accessor(obj); err == nil {
		if n := acc.GetName(); len(n) > 0 {
			name = n
		}
	}

	return printObj(w, name, p.Operation, p.ShortOutput, GetObjectGroupKind(obj))
}
```



##### 2.2.3 GoTemplatePrinter

```
func (f *KubeTemplatePrintFlags) ToPrinter(outputFormat string) (printers.ResourcePrinter, error) {
   if f == nil {
      return nil, NoCompatiblePrinterError{}
   }

   if p, err := f.JSONPathPrintFlags.ToPrinter(outputFormat); !IsNoCompatiblePrinterError(err) {
      return p, err
   }
   return f.GoTemplatePrintFlags.ToPrinter(outputFormat)
}

// PrintObj formats the obj with the Go Template.
func (p *GoTemplatePrinter) PrintObj(obj runtime.Object, w io.Writer) error {
	if InternalObjectPreventer.IsForbidden(reflect.Indirect(reflect.ValueOf(obj)).Type().PkgPath()) {
		return fmt.Errorf(InternalObjectPrinterErr)
	}

	var data []byte
	var err error
	data, err = json.Marshal(obj)
	if err != nil {
		return err
	}

	out := map[string]interface{}{}
	if err := json.Unmarshal(data, &out); err != nil {
		return err
	}
	if err = p.safeExecute(w, out); err != nil {
		// It is way easier to debug this stuff when it shows up in
		// stdout instead of just stdin. So in addition to returning
		// a nice error, also print useful stuff with the writer.
		fmt.Fprintf(w, "Error executing template: %v. Printing more information for debugging the template:\n", err)
		fmt.Fprintf(w, "\ttemplate was:\n\t\t%v\n", p.rawTemplate)
		fmt.Fprintf(w, "\traw data was:\n\t\t%v\n", string(data))
		fmt.Fprintf(w, "\tobject given to template engine was:\n\t\t%+v\n\n", out)
		return fmt.Errorf("error executing template %q: %v", p.rawTemplate, err)
	}
	return nil
}
```

### 3.总结

（1）体验到了kubectl get的强大之处

（2）可以参考这种思路实现自己的客户端CIL。 通过flags填充option，然后再根据option定义不同的Printer，最后定制化输出