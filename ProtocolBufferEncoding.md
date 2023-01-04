#Protocol Buffer编解码

###Varint编码
>varint是一种可变长编码，使用1个或多个字节对整数进行编码，可编码任意大的整数，小整数占用的字节少，大整数占用的字节多，如果小整数更频繁出现，则通过varint可实现压缩存储。
>
>varint中每个字节的最高位bit称之为most significant bit (MSB)，如果该bit为0意味着这个字节为表示当前整数的最后一个字节，如果为1则表示后面还有至少1个字节，可见，varint的终止位置其实是自解释的。除MSB后的7个bit是补码表示的值
>
>在Protobuf中，tag和length都是使用varint编码的。tag.field_number和length都是正整数int32，这里提一下tag，它的低3位bit为wire type，如果只用1个字节表示的话，最高位bit为0，则留给field_number只有4个bit位，1到15，如果field_number大于等于16，就需要用2个字节，所以对于频繁使用的field其field_number应设置为1到15。

举例：

- 0000 0001 = 1
- 10101100 00000010 = 300

  10101100 00000010  
  ↓去掉MSB  
  0101100 0000010  
  ↓反转  
  0000010 0101100  
  ↓去掉高位0  
   100101100 = 256+32+8+4 = 300
- 10000000 11111000 11111111 11111111 11111111 11111111 11111111 11111111 11111111 00000001 = -1024  

   10000000 11111000 11111111 11111111 11111111 11111111 11111111 11111111 11111111 00000001  
   ↓去掉MSB  
   0000000 1111000 1111111 1111111 1111111 1111111 1111111 1111111 1111111 0000001  
   ↓反转  
   0000001 1111111 1111111 1111111 1111111 1111111 1111111 1111111 1111000 0000000  
   ↓补码转原码（取反加1）  
     \-  0000000 0000000 0000000 0000000 0000000 0000000 0000000 0001000 0000000  
   ↓去掉高位0  
     \- 1000 0000000 = -1024


###消息结构

> https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/ch04.html

```json
{
    "userName": "Martin",
    "favoriteNumber": 1337,
    "interests": ["daydreaming", "hacking"]
}
```
```protobuf

message Person {
    required string user_name       = 1;
    optional int64  favorite_number = 2;
    repeated string interests       = 3;
}
```

- JSON
![](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0401.png)
- Thrift BinaryProtocol
![](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0402.png)
- Thrift CompactProtocol
![](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0403.png)
- Protocol Buffer
![](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/assets/ddia_0404.png)

####Tag(FieldNumber+WireType)

```java
  static final int TAG_TYPE_BITS = 3;
  static final int TAG_TYPE_MASK = (1 << TAG_TYPE_BITS) - 1;//0000 0111

  /** Given a tag value, determines the wire type (the lower 3 bits). */
  public static int getTagWireType(final int tag) {
    return tag & TAG_TYPE_MASK;
  }

  /** Given a tag value, determines the field number (the upper 29 bits). */
  public static int getTagFieldNumber(final int tag) {
    return tag >>> TAG_TYPE_BITS;
  }

  /** Makes a tag value given a field number and wire type. */
  static int makeTag(final int fieldNumber, final int wireType) {
    return (fieldNumber << TAG_TYPE_BITS) | wireType;
  }
```
####WireType参考

|Type|Meaning|Used For|
|---|---|---|
|0|	Varint|	int32, int64, uint32, uint64, sint32, sint64, bool, enum
|1|	64-bit|	fixed64, sfixed64, double
|2|	Length-delimited|	string, bytes, embedded messages, packed repeated fields
|3|	Start group|	groups (deprecated)
|4|	End group|	groups (deprecated)
|5|	32-bit|	fixed32, sfixed32, float

e.g. 

在Java生成的代码中 判断要给哪个字段赋值直接用tag，而C++生成的代码则是转成fieldNumber再判断

有符号的类型(sint32, sint64)使用ZigZag编码 

###版本兼容

增加减少字段是允许的，但是变换字段FieldNumber会导致新旧版本解析不一致  
未识别的tag/fieldNumber会存储到unknownFields，

###思考

- varint为什么要反转低位?