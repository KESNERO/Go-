# Protobuf3学习笔记

[TOC]


## 定义消息类型

- 第一行需要声明使用的是 `proto3`
- 正文以 `message` 关键字开头， 定义消息的名称， 里面的内容包含字段的类型和字段的名称

```protobuf3
message SearchRequest {
    string query = 1;
    int32 page_number = 2;
    int32 result_per_page = 3;
}
```

### 定义字段类型
- 上述例子中的字段都是标量类型，两个整数（`page_number`, `result_per_page`），和字符串(`query`)，你也可以定义一个组合的字段类型

### 给字段赋值数字
- 每个字段定义都有一个唯一的数字标示，这些数字用于表示二进制编码中的字段，因此不能在正式使用该消息之后随意更改这个数字，1到15的数字会被编码为一个字节（包括字段的数字标示符和字段类型），16到2047会使用两个字节来编码，所以1到15的数字要分配给最常用的元素，要记住给未来可能会非常频繁使用的元素留下一些空闲空间（可能是为了减少消息传输带来的流量开销）
- 最小的字段数字是1，最大的是2^29-1（即536870911)，但是你不能使用 19000到19999之间的数字，因为这些被保留用语Protocol Buffers的实现， protocol buffer的编译器如果发现你使用了这些范围内的数字的话可能会报错，你也不能使用之前定义过的[保留](https://developers.google.com/protocol-buffers/docs/proto3#reserved)数字

### 定义字段规则
- 消息的字段可以是以下当中的一个
    - singular: 一个组织良好的消息可以有零个或1个这种类型的字段， 这是proto3的基本字段规则
    - repeated: 这种字段在组织良好的消息中可以重复任何次数（包括0次），重复值的顺序是被保留的（在proto3中repeated字段默认使用packed方式编码）

### 增加更多的消息类型

- 多个消息可以在同一份`.proto`文件中定义，这有助于定义多个关联的消息

```proto3
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message SearchResponse {
 ...
}
```

### 增加注释

- 可以使用C/C++方式给消息添加注释

```proto3
/* SearchRequest represents a search query, with pagination options to
 * indicate which results to include in the response. */

message SearchRequest {
  string query = 1;
  int32 page_number = 2;  // Which page number do we want?
  int32 result_per_page = 3;  // Number of results to return per page.
}
```

### 保留字段

- 如果从消息体当中彻底删除某个字段的话，未来的使用者是可以重复利用已删字段的代表数字的，但是如果未来的使用者不小心接收到了以前的消息体，那么会产生很多不符合预期的错误（包括数据腐败，隐私漏洞等等），所以我们需要保留已删字段的数字，这样protocol buffer编译器可以在未来有人使用这些保留字段的时候可以报出异常

```proto3
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

- 在这里你不能混用字段名称和字段数字

### 你的.proto文件会产生什么

- 当你运行protocol buffer编译器的编译`.proto`文件的时候，编译器会产生你所使用的语言的代码（包括获取和设置字段值，序列化消息到输出流，从输入流解析出消息等等）
    - 对于Go, 编译器会生成一个`.pb.go`文件

## 标量类型

- 一个标量消息字段可以是以下几种当中的一个

|.proto Type|Notes|Go Type|
|-----------|-----|-------|
|double||float64|
|float||float32|
|int32|Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead|int32|
|int64|Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead|int64|
|uint32|Uses variable-length encoding|uint32|
|uint64|Uses variable-length encoding|uint64|
|sint32|Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s|int32|
|sint64|Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s|int64|
|fixed32|Always four bytes. More efficient than uint32 if values are often greater than 2^28|uint32|
|fixed64|Always eight bytes. More efficient than uint64 if values are often greater than 2^56|uint64|
|sfixed32|Always four bytes|int32|
|sfixed64|Always eight bytes|int64|
|bool||bool|
|string|A string must always contain UTF-8 encoded or 7-bit ASCII text, and cannot be longer than 2^32|string|
|bytes|May contain any arbitrary sequence of bytes no longer than 2^32|[]byte|

## 默认值

- 当一个消息被解析之后，如果编码后的消息没有包含一个确切的singular元素，对应的字段在解析后的对象中会被赋予一个默认值，以下是几个字段类型的默认值
    - 字符串：空字符串
    - bytes: 空bytes
    - bool: false
    - numeric types: 0
    - enums: 第一个定义的enum值（必须是0）
    - repeated字段的默认值是空的（比如一个空列表）
- 要注意的是当一个消息被解析之后是无法感知到它是显式的赋予了默认值，还是因为没有设置而被赋予了默认值，所以你要在定义自己的消息体的时候要考虑这方面的问题（比如不要使用bool类型字段的false作为一个判断行为的条件），如果一个标量消息字段被设置成默认值，这个值是不会被序列化到传输流

## 枚举

```proto3
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Corpus corpus = 4;
}
```

- 如你所见，`Corpus`枚举的第一个常数是被映射到0，每个枚举类型都必须存在一个常数映射到0，且必须是第一个元素， 因为
    - 必须存在0值，我们才能为枚举类型设置一个默认值
    - 0值元素必须是第一个元素，因为第一个元素始终是默认值
- 你可以定义一个别名给枚举类型，且必须包含 `option allow_alias = true`关键字，否则protocol compiler发现有别名的时候会产生一个报错信息

```proto3
message MyMessage1 {
  enum EnumAllowingAlias {
    option allow_alias = true;
    UNKNOWN = 0;
    STARTED = 1;
    RUNNING = 1;
  }
}
message MyMessage2 {
  enum EnumNotAllowingAlias {
    UNKNOWN = 0;
    STARTED = 1;
    // RUNNING = 1;  // 如果去掉注释，谷歌内部版本的proto3会产生error，外部的版本的proto3会产生warning
  }
}
```

- 枚举常数必须使用32位的数字，因为`enum`值是使用`varint encoding`
- 如果enum定义在消息体外部，那多个消息体可以重复利用enum定义，如果enum定义在一个消息体内部，那么其他消息体也可以通过 `_MessageType_._EnumType_`的方式使用该enum
- 在反序列化期间，无法识别的枚举值会被消息保留，如何解析这些未知值取决于每个语言如何处理支持，开放枚举类型的语言（比如C++/Go)会赋予未知的枚举值为底层的数字，像Java一样封闭枚举类型的语言，需要通过特定的accessor 来访问未知的枚举值，其他情况一般未知的枚举值不会被反序列化

### 保留字段

- 可以使用`to`关键字来表示保留数值的范围
- 可以使用`max`来表示可以使用的最大的数值

```proto3
enum Foo {
  reserved 2, 15, 9 to 11, 40 to max;
  reserved "FOO", "BAR";
}
```

## 使用其他消息类型

- 你可以在定义字段中使用其他消息类型

```proto3
message SearchResponse {
  repeated Result results = 1;
}

message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```

### 导入定义

- 上述例子中 `Result`消息类型被定义在与`SearchResponse`同一个文件当中，如果我们发现消息体来行已经在其他文件定了该怎么办？
- 你可以通过**导入**来使用其他`.proto`文件定义的消息类型，语法如下

```proto3
import "myproject/other_protos.proto";
```

- 默认的话你只能导入指定文件中直接定义的消息类型，但是有时需要导入指定文件中导入过的消息类型（比如旧的`.proto`文件要迁移了，预期修改所有使用者的导入路径，不如在旧文件中留一个导入语句），这时可以使用`import public`关键字

```proto3
// new.proto
// All definitions are moved here

// old.proto
// This is the proto that all clients are importing.
import public "new.proto";
import "other.proto";

// client.proto
import "old.proto";
// You use definitions from old.proto and new.proto, but not other.proto
```

- 编译的时候如果添加 `-I/--proto_path` 参数表明导入要搜索的一连串路径，那么编译器会去指定路径搜索`.proto`文件，如果没有指定，那么编译器会在包含`.proto`文件的目录中寻找被import的`.proto`文件

## 嵌套类型

- 你可以定一个嵌套着消息类型的消息类型

```proto3
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```

- 如果想在另外一个消息类型中使用一个消息类型里面嵌套着的消息类型可以使用`_Parent_._Type_`

```proto3
message SomeOtherMessage {
  SearchResponse.Result result = 1;
}
```

- 可以任何次数的反复嵌套

```proto3
message Outer {                  // Level 0
  message MiddleAA {  // Level 1
    message Inner {   // Level 2
      int64 ival = 1;
      bool  booly = 2;
    }
  }
  message MiddleBB {  // Level 1
    message Inner {   // Level 2
      int32 ival = 1;
      bool  booly = 2;
    }
  }
}
```

## 更新一个消息类型

- 如果一个已存在的消息类型不再满足需求 - 比如你需要消息体定义拥有额外的字段 - 但是如果想继续使用旧格式生成的代码，不必担心！不用破坏已有的代码也可以安全更新消息类型，记住以下几个规则
    - 不要修改已有字段的数字标示
    - 如果增加新的字段，所有的旧字段仍然可以被新格式生成的代码解析，要记住所有旧代码中的默认值新生成的代码才能运作正常，类似的，新生成的代码也可以被旧代码解析，旧代码在解析的时候默认会忽略新增加的字段
    - 字段可以被移除，如果字段不再使用了。你需要重新命名不使用的字段名称，通过增加`OBSOLETE_`前缀，或者让这些字段数字编程`reserved`，保证未来的使用者不再偶然的重复使用这些被移除的字段数字
    - `int32, uint32, int64, uint64, bool`这些都是兼容的，所以可以保证向前兼容性和向后兼容性的情况更改这些字段类型到另外一个（比如64位的数字，被读成int32，就会被截断成32位）
    - `sint32, sint64` 互相兼容，但和其他数值不兼容
    - `string` 和 `bytes` 互相兼容， 如果`bytes`是有效的`UTF-8`
    - 嵌入的消息可以和`bytes`兼容，如果bytes是一个消息的编码
    - `fixed32`和`sfixed32, fixed64, sfixed64`兼容
    - 对于`string, bytes`类型， `optional`和`repeated`兼容，但通常情况下这是不安全的
    - `enum` 与 `int32,uint32,int64,uint64`兼容， 然而需要知道的是客户端代码也许在反序列化的时候区别对待他们，比如：未知的`enum`类型会被保留在消息中，但是如何标示这些是依赖语言的，Int字段通常会被保留它们的值
    - 如果更改一个single值到`oneof`类型是安全的，把多个字段改成`oneof`也是安全的，只要保证任何代码不会设置多个`oneof`值，多个字段添加到已有的`oneof`字段是不安全的

## Unknown字段

- 如果解析器不认识一个序列化的字段，那么该字段会被标记成Unknown字段，举例来说：一个新的二进制传输的数据中包含新的字段，旧的二进制无法识别这些新字段，就会标记成Unkonw字段
- 原则上proto3的消息会把所有Unkonwn字段抛弃，但是3.5版本之后为了兼容proto2，所以所有的unknown字段在解析和序列化的过程中都会保留

## Any

- `Any`消息类型可以让你在未定义嵌入类型的时候使用这些类型， 一个`Any`包含任意序列化的`bytes`， 想要使用`Any`类型，你需要导入`google/protobuf/any.proto`

```proto3
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
}
```

- 默认的类型URL是`type.googleapis.com/_packagename_._messagename_`
- 不用的语言实现将支持helpers运行库来pack和unpack`Any`值（保证类型安全的规则下），Java有`pack()`和`unpack()`accessor，C++有`PackFrom()`和`UnpackTo()`方法

```proto3
// Storing an arbitrary message type in Any.
NetworkErrorDetails details = ...;
ErrorStatus status;
status.add_details()->PackFrom(details);

// Reading an arbitrary message from Any.
ErrorStatus status = ...;
for (const Any& detail : status.details()) {
  if (detail.Is()) {
    NetworkErrorDetails network_error;
    detail.UnpackTo(&network_error);
    ... processing network_error ...
  }
}
```

## Oneof

- 如果你有一个多字段的消息，一次只有一个值会被赋值，你可以强迫使用`oneof`来节省内存
- Oneof字段有点像共享内存中排它的常规字段，一次只有一个字段被赋值，oneof任何成员被赋值的时候其他成员的值自动清空，你可以通过`case()`或`WhichOnfof()`来检测oneof中哪个字段正在使用，取决于你使用什么语言

### Oneof使用

- 为了定义oneof，你需要使用`oneof`关键字接一个oneof名称

```proto3
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

- 然后你可以增加你的oneof成员，但要排除`map`字段和`repeated`字段
- 在你生成的代码中，oneof字段拥有与其他常规字段相同的getters和setters，初次之外还有一个特殊的方法检测当前oneof中的是哪个成员， [API链接](https://developers.google.com/protocol-buffers/docs/reference/overview)

### Oneof特性

- 设置一个oneof成员会自动情况其他所有成员，所以设置多个oneof成员，只有最后一个生效

```proto3
SampleMessage message;
message.set_name("name");
CHECK(message.has_name());
message.mutable_sub_message();   // Will clear name field.
CHECK(!message.has_name());
```

- 如果解析器知道多个成员在流中，只有最后看到的成员才会被用于解析的消息当中
- oneof成员不能是`repeated`类型
- Reflection APIs work for oneof fields
- 如果一个oneof成员被设置成默认值，那么"case"方法会被设置成该成员，值也会被序列化到流中
- 如果在使用C++， 那么要注意任何造成内存崩溃的情况， 下面的代码就会崩溃，因为`sub_message`在调用`set_name()`的时候被清空了

```proto3
SampleMessage message;
SubMessage* sub_message = message.mutable_sub_message();
message.set_name("name");      // Will delete sub_message
sub_message->set_...            // Crashes here
```

- 在C++，如果使用`Swap()`两个oneof消息，每个消息会进入对方的oneof case，`msg1`会拥有`sub_message`，`msg2`会有`name`

```proto3
SampleMessage msg1;
msg1.set_name("name");
SampleMessage msg2;
msg2.mutable_sub_message();
msg1.swap(&msg2);
CHECK(msg1.has_sub_message());
CHECK(msg2.has_name());
```

### 向后兼容问题

- 在新增或者移除oneof字段的时候要小心，检测oneof值的时候如果得到了`None/NOT_SET`返回，这也许意味着这个oneof字段未设置，或者设置到了另外一个oneof里，我们没法知道这两者的区别，也没有方法知道一个数据流中的未知变量是否是oneof成员

- Tag重复利用的话题
    - 迁移字段到oneof或者从oneof移除字段： 也许会在序列化和解析之后丢失部分的消息，然而你可以安全的转移一个字段到新的oneof，也可以转移多个字段到一个新的oneof（只要保证每次只有一个字段在使用）
    - 删除一个oneof字段后再加回来：有可能在消息被序列化和解析之后丢失当前设置的值
    - 拆分或合并oneof: 类似迁移字段

## 映射

- 如果你想创建关联类型的映射，可以使用下面这种语法

```proto3
map<key_type, value_type> map_field = N;
```

- `key_type`可以是任何整数或者字符串类型（不能是float或者bytes，enum）
- `value_type`可以是任何类型（除了另一个map）
```proto3
map<string, Project> projects = 3;
```
- 注意
    - Map字段不能是`repeated`
    - 流中的map迭代顺序是未知的，所以不能依赖字段定义中的顺序
    - 当为`.proto`生成文本格式时，maps会以key来排序，数字类型的key会以数字方式进行排序
    - 如果在解析或者合并map的时候发现存在重复的key，那么以最后看到的key为准，如果从文本格式中解析map，那么在发现存在重复key的时候解析会失败
    - 如果你提供一个map的key，但是没有设置map的值，那么在序列化的时候的行为是依据语言不通的
    
- [API reference](https://developers.google.com/protocol-buffers/docs/reference/overview)

### 向后兼容性

- map语法在数据流中与下面定义的消息一致，所以不支持map的protocol buffer仍然可以从数据流中解析这些数据

```proto3
message MapFieldEntry {
  key_type key = 1;
  value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```

- 任何支持maps的protocol buffers的实现，必须既能产生和接受上述定义的数据类型

## 包

- 你可以添加 `package` 标示符来防止不同的`.proto`文件之间相同的消息类型发生冲突

```proto3
package foo.bar
message Open {...}
```

- 你可以在其他包中定义字段类型的时候可以引用包标示符

```proto3
message Foo {
    ...
    foo.bar.Open open = 1;
    ...
}
```

- 根据所选择的语言，包标示符对生成的代码的影响会有不同
    - 在 C++， 生成的类会被封装在命名空间内， 即上述例子中的`Open`函数会在foo::bar命名空间内
    - 在 Go， 包名会被用作Go的包名，除非在`.proto`文件提供了`option go_package`

### 包与命名规则

- 类型名称的命名规则在protocol buffer语言中有点像C++： 搜索范围是从里到外，即`.foo.bar.Baz`中最左边的是最外层
- protocol buffer编译器通过导入`.proto`文件来覆盖所有类型名称，每种语言的代码生成器都懂得如何指向每个类型，即使在不同的作用域内

## 定义服务

- 如果你想使用消息类型定义RPC系统，你可以在`.proto`文件中定义RPC服务接口，编译器会帮助你生成服务的相关接口和stub，举个例子，如果你想定义一个接收`SearchRequest`返回`SearchResponse`的RPC服务的话，你可以按以下方式定义在`.proto`文件中

```proto3
service SearchService {
    rpc Search(SearchRequest) returns (SearchResponse);
}
```

- protocol buffers使用的最直接的RPC系统是gRPC：是一个由谷歌开发的跨语言和平台的开源RPC系统，gRPC与protoco buffers工作地非常契合，只要你给protocol buffer编译器一个插件，就可以让你直接从`.proto`文件中生成一个相关的RPC代码

## JSON映射

- Proto3支持权威的JSON，可以让这在不同的系统之间更好的共享数据， 对应的类型如下表所示
- 如果一个值在JSON编码中是空的，或者值是null， 那么这会在protocol buffer解析的时候被转译成一个合适的默认值，如果一个字段拥有默认值，那么在输出成JSON编码的时候会被省略以节省空间，一个特殊的选项可以标记转换成JSON编码的时候是否要丢弃某些字段

|proto3|JSON|JSON example|Notes|
|------|----|------------|-----|
|message|object|{"fooBar": v, "g": null, ...}|Generates JSON objects. Message field names are mapped to lowerCamelCase and become JSON object keys. If the `json_name` field option is specified, the specified value will be used as the key instead. Parsers accept both the lowerCamelCase name (or the one specified by the `json_name` option) and the original proto field name. `null` is an accepted value for all field types and treated as the default value of the corresponding field type.|
|enum|string|"FOO_BAR"|The name of the enum value as specified in proto is used. Parsers accept both enum names and integer values.|
|map<K,V>|object|{"k": v, ...}|All keys are converted to strings.|
|repeated V|array|[v,...]|`null` is accepted as the empty list [].|
|bool|true,false|`true,false`||
|string|string|"Hello World"||
|bytes|base64 string|"YWFEWAFfewafjoi"|JSON value will be the data encoded as a string using standard base64 encoding with paddings. Either standard or URL-safe base64 encoding with/without paddings are accepted.|
|int32,fixed32,uint32|number|1,-10,0|JSON value will be a decimal number. Either numbers or strings are accepted.|
|int64,fixed64,uint64|string|"1","-10"|JSON value will be a decimal string. Either numbers or strings are accepted.|
|float, double|number|`1.1, -10.0, 0, "NaN", "Infinity"`|JSON value will be a number or one of the special string values "NaN", "Infinity", and "-Infinity". Either numbers or strings are accepted. Exponent notation is also accepted.|
|Any|object|{"@type": "url", "f": v}|If the Any contains a value that has a special JSON mapping, it will be converted as follows: `{"@type": xxx, "value": yyy}`. Otherwise, the value will be converted into a JSON object, and the `"@type"` field will be inserted to indicate the actual data type.|
|Timestamp|string|"1972-01-01T10:00:20.021Z"|Uses RFC 3339, where generated output will always be Z-normalized and uses 0, 3, 6 or 9 fractional digits. Offsets other than "Z" are also accepted.|
|Duration|string|"1.000340012s", "1s"|Generated output always contains 0, 3, 6, or 9 fractional digits, depending on required precision, followed by the suffix "s". Accepted are any fractional digits (also none) as long as they fit into nano-seconds precision and the suffix "s" is required.|
|Struct|object|{...}|Any JSON object. See `struct.proto`|
|Wrapper types|various types|`2, "2", "foo", true, "true", null, 0, …`|Wrappers use the same representation in JSON as the wrapped primitive type, except that `null` is allowed and preserved during data conversion and transfer.|
|FieldMask|string|"f.fooBar,h"||
|ListValue|array|[foo, bar]||
|Value|value||Any JSON value|
|NullValue|null||JSON null|
|Empty|object|{}|An empty JSON object|

### JSON选项

- A proto3 JSON 实现提供以下选项
    - 原样输出默认值的字段：当转换成JSON的时候拥有默认值的字段默认是被proto3丢弃的，一个特殊的选项可以重写这个行为，让拥有默认值的字段以默认值输出成JSON
    - 忽略未知字段：Proto3 JSON解析器应该拒绝未知字段，但是也有选项让其在解析的时候忽略未知字段
    - 使用proto字段名称代替lowerCamelCase名称：默认的proto3 JSON打印器应该把字段名称转换成lowerCamelCase并以此作为JSON名称，但是一个选项可以直接使用字段名称来代替JSON名称，也有选项让解析器兼容两者
    - 输出enum的数值来代替enum成员名称：默认会使用enum成员名称，但也有选项可以使用enum的数值

## 选项

- 可以在包含独立声明的`.proto`文件中附带一些注释形式的选项，选项并不会改变定义的语义，但是会在上下文中产生影响，所有已实现的选项都定义在`google/protobuf/descriptor.proto`中
- 一些选项是文件级别的选项，因此要写在文件的顶端，而非消息体、enum、服务定义的内部，一些选项是消息级别的选项，需要写到消息体内部，一些选项是字段级别的选项，要写在字段定义的内部，选项也可以写在enum类型、enum 值、oneof字段、服务类型、服务方法内，但是目前并没有类似可用的选项

- `optimize_for`(file_option): 可以被设置为`SPEED`，`CODE_SIZE`, 或者`LITE_RUNTIME`
    - `SPEED`(default)：protocol buffer 编译器会生成序列化、解析、表现其他通用操作的代码，代码是高度优化的
    - `CODE_SIZE`: protocol buffer 编译器会生成最小的类，依靠的是共享,反射为基础的代码来实现序列化，解析，各种各样其他的操作，生成的代码大小因此比`SPEED`更小，但是操作会变得更慢，生成的类的API和`SPEED`模式一模一样，这种模式很适合用在编译大量的`.proto`文件但不需要盲目的追求它的速度的时候
    - `LITE_RUNTIME`: protocol 编译器会基于轻量级`libprotobuf-lite`而非`libprotobuf`来生成类，轻量级运行时会比普通的库更小，但是会舍弃一些像描述符和反射一样的特性，这在一些像手机一样封闭的平台很有用，编译器会生成像`SPEED`模式一样快速的类实现，生成的类仅实现了一些`MessageLite`接口（一个`Message`接口的子集）

```
在计算机学中，反射（英語：reflection）是指计算机程序在运行时（runtime）可以访问、检测和修改它本身状态或行为的一种能力。 用比喻来说，反射就是程序在运行的时候能够“观察”并且修改自己的行为
```

### 自定义选项

- Protocol Buffers 也允许你自定义你自己的 options， 这是一个基本不会有人使用的高级特性，如果你需要生成你自己的选项

```proto3
import "google/protobuf/descriptor.proto";

extend google.protobuf.MessageOptions {
  optional string my_option = 51234;
}

message MyMessage {
  option (my_option) = "Hello world!";
}
```

## 生成你的类

- 为了生成你所需要的类代码， 你需要运行protocol buffer编译器`protoc`来编译`.proto`文件，对于Go语言，你需要安装特定的代码生成插件给编译器
- Protocol Compiler 以下面这种方式来运作

```bash
protoc --proto_path=_IMPORT_PATH_ --go_out=_DST_DIR_ _path/to/file_.proto
```

- `IMPORT_PATH` 标明`import`语句搜索`.proto`文件的路径，如果未指定，那就会使用当前的路径，多个搜索路径可以使用多个重复的 `--proto_path` 来指定，它们会以声明的顺序去搜索 `.proto`文件
- `--go_out`： 生成的代码存放的路径
- 你需要提供一个或多个`.proto`文件作为输入，多个`.proto`文件可以一次性传入，可以使用相对路径来指定这些文件，每个文件需要存在于`IMPORT_PATH`中的一个目录里，这样编译器才能找到这些文件
- go代码中的编码方法
```go
book := &pb.AddressBook{}
// ...

// Write the new address book back to disk.
out, err := proto.Marshal(book)
if err != nil {
        log.Fatalln("Failed to encode address book:", err)
}
if err := ioutil.WriteFile(fname, out, 0644); err != nil {
        log.Fatalln("Failed to write address book:", err)
}
```
- go代码中的解析方法
```go
// Read the existing address book.
in, err := ioutil.ReadFile(fname)
if err != nil {
        log.Fatalln("Error reading file:", err)
}
book := &pb.AddressBook{}
if err := proto.Unmarshal(in, book); err != nil {
        log.Fatalln("Failed to parse address book:", err)
}
```

- Summary of the packages provided by this module:
    - proto: Package proto provides functions operating on protobuf messages such as cloning, merging, and checking equality, as well as binary serialization and text serialization.
    - jsonpb: Package jsonpb serializes protobuf messages as JSON.
    - ptypes: Package ptypes provides helper functionality for protobuf well-known types.
    - ptypes/any: Package any is the generated package for google/protobuf/any.proto.
    - ptypes/empty: Package empty is the generated package for google/protobuf/empty.proto.
    - ptypes/timestamp: Package timestamp is the generated package for google/protobuf/timestamp.proto.
    - ptypes/duration: Package duration is the generated package for google/protobuf/duration.proto.
    - ptypes/wrappers: Package wrappers is the generated package for google/protobuf/wrappers.proto.
    - ptypes/struct: Package structpb is the generated package for google/protobuf/struct.proto.
    - protoc-gen-go/descriptor: Package descriptor is the generated package for google/protobuf/descriptor.proto.
    - protoc-gen-go/plugin: Package plugin is the generated package for google/protobuf/compiler/plugin.proto.
    - protoc-gen-go: The protoc-gen-go binary is a protoc plugin to generate a Go protocol buffer package.