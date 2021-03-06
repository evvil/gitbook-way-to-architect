# Java中的位运算

Java中的位运算主要有以下几种

| 运算描述 | 运算符 | 运算规则 | 例子 |
| :--- | :--- | :--- | :--- |
| 与 | & | 都是1，结果才为1；其他情况结果为0 | 1&1=1 1&0=0 0&1=0 0&0=0 |
| 或 | \| | 只要有1个1，结果就为1；都是0的时候，结果才为0； | 1\|1=1 1\|0=1 0\|1=1 0\|0=0 |
| 取反 | ~ | 1变0，0变1 | ~0=1 ~1=0 |
| 异或 | ^ | 相同为0，不同为1 | 1^1=0 0^0=01 1^0=1 0^1=1 |
| 右移 | &gt;&gt; | a &gt;&gt; b ：所有位右移b位，高位补0 | 5 &gt;&gt; 1 ===&gt;  0000 0000 0000 0101 &gt;&gt; 1  = 0000 0000 0000 0010 = 2  |
| 左移 | &lt;&lt; | a &lt;&lt; b：所有位左移b位，低位补0 | 5 &lt;&lt; 1 ===&gt;  0000 0000 0000 0101 &lt;&lt; 1  = 0000 0000 0000 1010 = 10 |
| 无符号右移 | &gt;&gt;&gt; | a &gt;&gt;&gt; b ：除符号位外，其他位右移b位，高位补0 |  |
| 无符号左移 | &lt;&lt;&lt; | a &lt;&lt;&lt; b：除符号位外，其他位左移b位，低位补0 |  |

注：位运算时，进行运算的两个数，要转换为二进制，然后从最低位到最高位，一一对应。上面所述的运算规则，均为某位上的两个值的运算规则。例子均是某个位的数值。

**要注意** ①位操作只能用于整形数据，对float和double类型进行位操作会被编译器报错。 ②位操作符的运算优先级比较低，因为尽量使用括号来确保运算顺序，否则很可能会得到莫明其妙的结果。 \*\*

## 位运算的技巧或应用

**1、乘或除2（或2的倍数4/8/16/32...）的情形，可以考虑使用左移或右移，即**`<<`**或** `>>`

```java
int a = 9;
int b = a << 1 = 18
int c = a >> 1 = 4
```

**2、判断奇偶** 这里利用一个特性：偶数的二进制最低位必为0，奇数的二进制最低位必为1。

```java
//判断i是否为偶数
boolean even = (i & 1) == 0;
//这里有一个小小的常犯的错误，如下代码判断是否为奇数是有问题的：负的奇数运算结果为-1。
boolean odd = (i & 1) == 1;
//正确的判断奇数
boolean odd = ((i & 1) == 1) || ((i & 1) == -1);
```

解释：1的二进制为0000 0000 0000 0001，对任意的偶数（无论正负），其形式为xxxx xxxx xxxx xxx0（x表示0或1），两数进行与运算，从高位起的前31位必为0，最后一位是1&0=0，所以结果必然是0000 0000 0000 0000。

**3、交换两个数的值** 利用的特性： ①任意一个数a，a^a=0，且a^0=a，则a^a^a=a ②a^b=b^a

```java
int a = 10, b = 20;
a = a^b;//a = 10^20 = 不关心
b = a^b;//b = 10^20^20 = 10
a = a^b;//a = 10^20^10^20^20 = 20
```

**4、变换符号** 变换符号其实就是正数变负数，负数变正数。 其实代码中 a = -a就可以实现，这里只是提供另一种可以使用位运算的方式：a = \(~a\) + 1

```java
int a = 10, b = -20;
a = (~a) + 1 = (~10) + 1 = -10
b = (~b) + 1 = (~(-20)) + 1 = 20
```

**5、求绝对值** 思路①：取符号位，判断其正负 a &gt;&gt; 31：取符号位，0为正数，-1位负数\(TODO why\)

```text
abs(a) = (a >> 31) == 0 ? a : (~a + 1)
```

思路②：利用特性：对任意整数a，a^\(-1\) = ~a;

```text
int i = a >> 31;//a为正数，则i=0;a为负数，则i=-1;
abs(a) = (a^i) - i;
//a为正数，则(a^0) - 0 = a;
//a为负数，则(a^(-1)) + 1 =
```

TODO 运算示例以及更多应用

## 参考

[可能是最通俗易懂的 Java 位操作运算讲解](https://juejin.im/entry/58f9b6118d6d8100588060d6)

