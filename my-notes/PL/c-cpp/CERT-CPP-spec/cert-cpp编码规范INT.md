

## 整型数

### INT50-CPP. 不要转换超出枚举值的范围

严重程度：中等。由于转换带来的未指定值导致的buffer溢出，从而给攻击者制造运行任意恶意代码的场景。数据完整性违例等等。

在C++中枚举有两种形式： 区域枚举，底层的类型是固定的。非区域枚举底层的类型可能是也可能不是固定的。枚举类型的合法值的范围在C++标准[dcl.enum]中有描述：

> For an enumeration whose underlying type is fixed, the values of the enumeration are the values of the underlying type. Otherwise, for an enumeration where emin is the smallest enumerator and emax is the largest, the values of the enumeration are the values in the range bmin to bmax, defined as follows: Let K be 1 for a two’s complement representation and 0 for a one’s complement or sign-magnitude representation. bmax is the smallest value greater than or equal to max(|emin| − K, |emax|) and equal to 2M − 1, where M is a non-negative integer. bmin is zero if emin is non-negative and −(bmax + K) otherwise. The size of the smallest bit-field large enough to hold all the values of the enumeration type is max(M, 1) if bmin is zero and M + 1 otherwise. It is possible to define an enumeration that has values not defined by any of its enumerators. If the enumeratorlist is empty, the values of the enumeration are as if the enumeration had a single enumerator with value 0.

然后对于枚举类型的转换，C++标准[expr.static.cast]也有描述：

> A value of integral or enumeration type can be explicitly converted to an enumeration type. The value is unchanged if the original value is within the range of the enumeration values (7.2). Otherwise, the resulting value is unspecified (and might not be in that range). A value of floating-point type can also be explicitly converted to an enumeration type. The resulting value is the same as converting the original value to the underlying type of the enumeration (4.9), and subsequently to the enumeration type.

#### 代码样例对比

``` cpp
enum EnumType {
  First,
  Second,
  Third
};

void f(int intVar) {
  EnumType enumVar = static_cast<EnumType>(intVar);
  if (enumVar < First || enumVar > Third) {
    // Handle error
  }
}
```

上面的代码中，函数f()试图检查intVar是否在枚举值的范围以内，但是对一个未知的int值转换成枚举类型，会导致未指定的值，也就是说enumVar是一个未指定的值，一旦对其进行if判断，就会导致未指定行为。需要用以下方式解决：

``` cpp
enum EnumType {
  First,
  Second,
  Third
};

void f(int intVar) {
  if (intVar < First || intVar > Third) {
  // Handle error
  }
  EnumType enumVar = static_cast<EnumType>(intVar);
}
```

把if判断前置，不会导致未指定行为了。

``` cpp
enum class EnumType {
  First,
  Second,
  Third
};

void f(int intVar) {
  EnumType enumVar = static_cast<EnumType>(intVar);
  if (intVar < First || intVar > Third) {
    // Handle error
  }
}
```
用枚举类类型。也就是之前所说的区域枚举。默认底层的类型是int类型表述的值。允许任何int类型的参数合法转换为枚举值。

``` cpp
enum EnumType : int {
  First,
  Second,
  Third
};

void f(int intVar) {
  EnumType enumVar = static_cast<EnumType>(intVar);
}
```

用固定非区域枚举。如果没有以上的int声明，那么这个枚举类型就是非固定非区域枚举。 现在该枚举类型的底层类型为固定的int类型，允许任何int值转换为合法枚举值。