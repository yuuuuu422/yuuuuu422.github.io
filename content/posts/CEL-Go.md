---
title: Preliminary Study on CEL-Go
date: 2021-10-26T23:24:32+08:00
toc: On
---

## 前言

XRAY很牛逼，但是其并不开源，最近也是想写点东西，就自己也造个轮子，有时间的话就把**Thinkphp，structs2，weblogic**都整上了。

poc的话当然希望可以直接融合XRAY的poc，采用YAML格式，但有一个问题，比如我们一般用`gopkg.in/yaml.v2`把YAML解析为结构体，但如果YAML中出现一些表达式，就很难直接用结构体解决。(无脑正则当然ok)

```yaml
name: poc-yaml-thinkphp5023-method-rce
rules:
  - method: POST
    path: /index.php?s=captcha
    headers:
      Content-Type: application/x-www-form-urlencoded
    body: |
      _method=__construct&filter[]=printf&method=GET&server[REQUEST_METHOD]=TmlnaHQgZ2F0aGVycywgYW5%25%25kIG5vdyBteSB3YXRjaCBiZWdpbnMu&get[]=1
    expression: |
      response.status==200&&response.body.bcontains(b"TmlnaHQgZ2F0aGVycywgYW5%kIG5vdyBteSB3YXRjaCBiZWdpbnMu1")
detail:
  links:
    - https://github.com/vulhub/vulhub/tree/master/thinkphp/5.0.23-rce
```

上面的`expression`其实很好理解，长得比较像python表达式，在python中eval可以直接对表达式求解，而我们现在也就需要一个高效的方法可以自定义求解表达式。

在[XRAY-如何编写expression表达式](https://docs.xray.cool/#/guide/poc?id=%e5%a6%82%e4%bd%95%e7%bc%96%e5%86%99expression%e8%a1%a8%e8%be%be%e5%bc%8f)中其实有提到一个库 ==> [CEL-Go](https://github.com/google/cel-go),官网介绍看了半天说实话不知道在干嘛...查了查国内这方面的教程也几乎为0，不过好在Google大爹的[codelabs](https://codelabs.developers.google.com/codelabs/cel-go/#0)实在良心，自己也跟着过一遍，算是入了个门，在这简单记录一下。

## Introduction

>CEL是为了安全的执行用户代码而设计的一门“语言”，就像用户在Python上盲目调用`eval`是危险的，但CEL可以安全的执行。
>
>CEL多用于求解表达式，类似single line functions或者lambda表达式，并且通常用于计算bool值，但它也可用于构造更复杂的对象，如JSON或Protobuf。

## Key concepts

- Variable bindings
- Function bindings for any custom extensions
- An AST to evaluate

## Declare the variables

cel中数据类型是绑定在protobuf上的，所以我们先简单写一个proto文件

```protobuf
syntax = "proto3";
package celDemo;

option go_package = "celDemo/bp";

message Response{
  string url = 1;
  int32 status =2;
  bytes body = 3;
}
```

生成对应go文件 == >`protoc --go_out=. celDemo/http.proto`

```
celDemo
├── bp
│   └── http.pb.go
├── cel.go 
├── cel_test.go 
└── http.proto
```
cel是通过env来解析表达式，我们可以通过`env, err := cel.NewEnv()`创建一个独立的环境。

> NewEnv creates a program environment configured with the standard library of CEL functions and macros. The Env value returned can parse and check any CEL program which builds upon the core features documented in the CEL specification.

之前提到要想让cel识别这个Response，就需要在创建env时加入我们的options

>The environment can be customized by providing the options cel.EnvOption to the call. Those options are able to disable macros, declare custom variables and functions, etc.

```go
type options struct {
	envOptions  []cel.EnvOption
	programOptions  []cel.ProgramOption
}

func newEnvOption() *options{
	opt:= &options{}
	opt.envOptions = []cel.EnvOption{
		cel.Types(
			&bp.Response{}),
		cel.Declarations(
			decls.NewVar("response",
				decls.NewObjectType("celDemo.Response")),
			),
	}
	opt.programOptions= []cel.ProgramOption{}
	return opt
}
```

之后就能直接创建env

```go
	options:= newEnvOption()
	env,_ := cel.NewEnv(cel.Lib(options))

//注意cel.Lib
func Lib(l Library) EnvOption {
	return func(e *Env) (*Env, error) {
		var err error
		for _, opt := range l.CompileOptions() {
			e, err = opt(e)
			if err != nil {
				return nil, err
			}
		}
		e.progOpts = append(e.progOpts, l.ProgramOptions()...)
		return e, nil
	}
}

//发现这里调用了l.CompileOptions(),l.ProgramOptions() 所以我们需要写入options的方法中：

func (opt *options) CompileOptions() []cel.EnvOption{
	return opt.envOptions
}
func (opt *options) ProgramOptions() []cel.ProgramOption{
	return opt.programOptions
}
```

简单测试一下：

```go
package celDemo

import (
	"celDemo/bp"
	"github.com/google/cel-go/cel"
	"testing"
)

func TestPoc(t *testing.T) {
	poc := map[string]interface{}{}

	poc["response"]= &bp.Response{
		Url: "theoyu.top",
		Status: 200,
		Body: []byte("who is theoyu"),
	}

	options:= newEnvOption()
	env,_ := cel.NewEnv(cel.Lib(options))
	ast,iss:=env.Compile("response.status == 200")
	if iss.Err() != nil {
		t.Fatal(iss.Err())
	}
	prg,err:=env.Program(ast)
	if err != nil{
		t.Fatal(err)
	}
	out,_,_:=prg.Eval(poc)
	t.Log(out)
}
```

输出：

```go
=== RUN   TestPoc
    cel_test.go:29: true
--- PASS: TestPoc (0.01s)
PASS
```


## Custom Functions

和变量一样，函数也在`EnvOption`中定义

```go
cel.Declarations(
			decls.NewFunction("bcontains",decls.NewInstanceOverload(
				"bytes_contains_bytes",
				[]*exprpb.Type{decls.Bytes,decls.Bytes},
				decls.Bool)),
			),
```

实现在`opt.programOptions`中完成

```go
	opt.programOptions= []cel.ProgramOption{
		cel.Functions(
			&functions.Overload{
				Operator:     "bcontains",
				Binary: func(lhs ref.Val, rhs ref.Val) ref.Val{
					return types.Bool(bytes.Contains(lhs.(types.Bytes),rhs.(types.Bytes)))
				},
			},
		),
	}
```

测试一下：

```go
ast,iss:=env.Compile(`response.body.bcontains(b"theoyu")`)

output:
=== RUN   TestPoc
    cel_test.go:29: true
--- PASS: TestPoc (0.01s)
PASS
```

