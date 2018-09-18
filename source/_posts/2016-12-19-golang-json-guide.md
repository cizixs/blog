---
layout: post
title: "Go 语言 JSON 简介"
excerpt: "文章介绍了如何使用 Go 语言和 JSON 打交道， 在 JSON 变得格外流行的今天，值得每个学习 Go 语言的初学者了解。"
categories: blog
tags: [Go, JSON, tutorial]
comments: true
share: true
---

## 简介

[JSON](http://json.org)（JavaScript Object Notation） 是一种轻量级的数据交换格式，因为易读性、机器容易处理而变得流行。

JSON 语言定义的内容非常简洁，主要分为三种类型：对象（object）、数组（array）和基本类型（value）。基本类型（value）包括：

- string 字符串，双引号括起来的 unciode 字符序列
- number 数字，可以是整数，也可以是浮点数，但是不支持八进制和十六进制表示的数字
- true，false 真值和假值，一般对应语言中的 bool 类型
- null 空值，对应于语言中的空指针等

数组（array）就是方括号括`[]`起来的任意值的序列，中间以逗号 `,` 隔开。对象（object）是一系列无序的键值组合，**键必须是字符串**，键值对之间以逗号 `,` 隔开，键和值以冒号 `:` 隔开。数组和对象中的值都可以是嵌套的。

[JSON 官网](http://json.org) 有非常易懂的图示，进一步了解可以移步。

JSON 不依赖于任何具体的语言，但是和大多数 C 家族的编程语言数据结构特别相似，所以 JSON 成了多语言之间数据交换的流行格式。Go 语言也不例外，标准库 `encoding/json` 就是专门处理 JSON 转换的。

这篇文章就专门介绍 Go 语言中怎么和 JSON 打交道，常用的模式以及需要注意的事项。

## 使用

Golang 的 `encoding/json` 库已经提供了很好的封装，可以让我们很方便地进行 JSON 数据的转换。

Go 语言中数据结构和 JSON 类型的对应关系如下表：

golang 类型     |   JSON 类型   |   注意事项
---             |   ---         |   ---
bool            |   JSON booleans   |   
浮点数、整数    |   JSON numbers    |   
字符串          |   JSON strings    |   字符串会转换成 UTF-8 进行输出，无法转换的会打印对应的 unicode 值。而且为了防止浏览器把 json 输出当做 html， "<"、">" 以及 "&" 会被转义为 "\u003c"、"\u003e" 和 "\u0026"。
array，slice    |   JSON arrays     |   []byte 会被转换为 base64 字符串，nil slice 会被转换为 JSON null
struct          |   JSON objects    |   只有导出的字段（以大写字母开头）才会在输出中

**NOTE**：Go 语言中一些特殊的类型，比如 Channel、complex、function 是不能被解析成 JSON 的。

### Encode 和 Decode

要把 golang 的数据结构转换成 JSON 字符串（encode），可以使用 `Marshal`函数：

```
func Marshal(v interface{}) ([]byte, error)
```

比如我们有结构体 `User`

```
type User struct {
    Name string
    IsAdmin bool
    Followers uint
}
```
以及一个实例：

```
user := User{
		Name:      "cizixs",
		IsAdmin:   true,
		Followers: 36,
	}
data, err := json.Marshal(user)
```
那么 data 就是 `[]byte` 类型的数组，里面包含了解析为 JSON 之后的数据：

```
data == []byte(`{"Name":"cizixs","IsAdmin":true,"Followers":36}`)
```

相对应的，要把 JSON 数据转换成 Go 类型的值（Decode）， 可以使用 `json.Unmarshal`。它的定义是这样的：

```
func Unmarshal(data []byte, v interface{}) error
```

data 中存放的是 JSON 值，v 会存放解析后的数据，所以必须是指针，可以保证函数中做的修改能保存下来。

下面看个例子：

```
data = []byte(`{"Name":"gopher","IsAdmin":false,"Followers":8900}`)
var newUser = new(User)
err = json.Unmarshal(data, &newUser)
if err != nil {
	fmt.Errorf("Can not decode data: %v\n", err)
}
fmt.Printf("%v\n", newUser)
```

那么 `Unmarshal` 是怎么找到结构体中对应的值呢？比如给定一个 JSON key `Filed`，它是这样查找的：

- 首先查找 tag 名字（关于 JSON tag 的解释参看下一节）为 `Field` 的字段
- 然后查找名字为 `Field` 的字段
- 最后再找名字为 `FiElD` 等大小写不敏感的匹配字段。
- 如果都没有找到，就直接忽略这个 key，也不会报错。这对于要从众多数据中只选择部分来使用非常方便。

### 更多控制：Tag

在定义 struct 字段的时候，可以在字段后面添加 tag，来控制 encode/decode 的过程：是否要 decode/encode 某个字段，JSON 中的字段名称是什么。

可以选择的控制字段有三种：

- `-`：不要解析这个字段
- `omitempty`：当字段为空（默认值）时，不要解析这个字段。比如 false、0、nil、长度为 0 的 array，map，slice，string
- `FieldName`：当解析 json 的时候，使用这个名字

举例来说吧：

```
// 解析的时候忽略该字段。默认情况下会解析这个字段，因为它是大写字母开头的
Field int   `json:"-"`

// 解析（encode/decode） 的时候，使用 `other_name`，而不是 `Field`
Field int   `json:"other_name"`

// 解析的时候使用 `other_name`，如果struct 中这个值为空，就忽略它
Field int   `json:"other_name,omitempty"`
```

### 解析动态内容: interface{}

上面的解析过程有一个假设——你要事先知道要解析的 JSON 内容格式，然后定义好对应的数据结构。如果你不知道要解析的内容呢？ Go 提供了 `interface{}` 的格式，这个接口没有限定任何的方法，因此所有的类型都是满足这个接口的。在解析 JSON 的时候，任意动态的内容都可以解析成 `interface{}`。

比如还是上面的数据，我们可以这样做：

```
data := []byte(`{"Name":"cizixs","IsAdmin":true,"Followers":36}`)

var f interface{}
json.Unmarshal(data, &f)
```

但是要使用 `f`，还是很麻烦的，我们要使用 [type assertion](https://golang.org/ref/spec#Type_assertions)：

```
name := f.(map[string]interface{})["Name"].(string)
```
对于比较复杂的结构，这样的访问很麻烦，也很容易出错。

如果已经知道 JSON 数据是对象，而不是基本类型（bool，number，string，array）等，因为 JSON 对象键都是字符串，所以可以把上面的例子修改为：

```
var f map[string]interface{}

// 省去了上面 f 的 type assertion 步骤
name := f["Name"].(string)
```
需要注意的是，尽管 `Followers` 字段没有小数点，我们希望它是整数值，解析的时候它还是会被解析成 `float64`，如果直接把它当做 `int` 访问，会出现错误：

```
followers := f["Followers"].(int)

// panic: interface conversion: interface is float64, not int
```

而必须自己做类型转换：

```
followers := int(f["Followers"].(float64))
```

### 延迟解析：json.RawMessage

在解析的时候，还可以把某部分先保留为 JSON 数据不要解析，等到后面得到更多信息的时候再去解析。继续拿 `User` 举例，比如我们要添加认证的信息，认证可以是用户名和密码，也可以是 token 认证。

```
type BasicAuth struct {
    Email string
    Password string
}

type TokenAuth struct {
    Token string
}

type User struct {
    Name string
    IsAdmin bool
    Followers uint
    Auth json.RawMessage
}
```

我们在定义 `User` 结构体的时候，把认证字段的类型定义为 `json.RawMessage`，这样解析 JSON 数据的时候，对应的字段会先不急着转换成 Go 数据结构。然后我们可以自己去再次调用 `Unmarshal` 去读取里面的值：

```
err := json.Unmarshal(data, &basicAuth)
if basicAuth.Email != "" {
    // 这是用户名/密码认证方式，在这里继续做一些处理
} else {
    json.Unmarshal(data, &tokenAuth)
    if tokenAuth.Token != "" {
        // 这是 token 认证方法
    }
}

```

### 自定义解析方法

如果希望自己控制怎么解析成 JSON，或者把 JSON 解析成自定义的类型，只需要实现对应的接口（interface）。`encoding/json` 提供了两个接口：`Marshaler` 和 `Unmarshaler`：

```
// Marshaler 接口定义了怎么把某个类型 encode 成 JSON 数据
type Marshaler interface {
        MarshalJSON() ([]byte, error)
}

// Unmarshaler 接口定义了怎么把 JSON 数据 decode 成特定的类型数据。如果后续还要使用 JSON 数据，必须把数据拷贝一份
type Unmarshaler interface {
        UnmarshalJSON([]byte) error
}
```

标准库 [`time.Time`](https://golang.org/src/time/time.go#L928) 就实现了这两个接口。另外一个简单的例子（这个例子来自于参考资料中 Go and JSON 文章）：

```
type Month struct {
    MonthNumber int
    YearNumber int
}

func (m Month) MarshalJSON() ([]byte, error){
    return []byte(fmt.Sprintf("%d/%d", m.MonthNumber, m.YearNumber)), nil
}

func (m *Month) UnmarshalJSON(value []byte) error {
    parts := strings.Split(string(value), "/")
    m.MonthNumber = strconv.ParseInt(parts[0], 10, 32)
    m.YearNumber = strconv.ParseInt(parts[1], 10, 32)

    return nil
}
```

### 和 stream 中 JSON 打交道

上面所有的 JSON 数据来源都是预先定义的 `[]byte` 缓存，在很多时候，如果能读取/写入其他地方的数据就好了。`encoding/json` 库中有两个专门处理这个事情的结构：`Decoder` 和 `Encoder`：

```
// Decoder 从 r io.Reader 中读取数据，`Decode(v interface{})` 方法把数据转换成对应的数据结构
func NewDecoder(r io.Reader) *Decoder

// Encoder 的 `Encode(v interface{})` 把数据结构转换成对应的 JSON 数据，然后写入到 w io.Writer 中
func NewEncoder(w io.Writer) *Encoder
```

下面的例子就是从标准输入流中读取数据，解析成数据结构，删除所有键不是 `Name` 的字段，然后再 encode 成 JSON 数据，打印到标准输出。

```
package main

import (
    "encoding/json"
    "log"
    "os"
)

func main() {
    dec := json.NewDecoder(os.Stdin)
    enc := json.NewEncoder(os.Stdout)
    for {
        var v map[string]interface{}
        if err := dec.Decode(&v); err != nil {
            log.Println(err)
            return
        }
        for k := range v {
            if k != "Name" {
                delete(v, k)
            }
        }
        if err := enc.Encode(&v); err != nil {
            log.Println(err)
        }
    }
}
```

## 参考资料

- [Go Blog: JSON and Go](https://blog.golang.org/json-and-go)
- [Dynamic JSON in Go](http://eagain.net/articles/go-dynamic-json/)
- [Go and JSON](https://eager.io/blog/go-and-json/)