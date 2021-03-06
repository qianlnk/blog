# int类型在内存中的存储方式

本文中的int类型的相关数据都以32位操作系统下，int类型表示带有符号的整型，而unsigned int类型为无符号的整型。

|类型名称	|占字节数|	取值范围|
|----|----|----|
|int	|4B	|-2^31~2^31-1|
|unsigned int	|4B|	0 ~ 2^32|

## 原码

原码，是计算机中一种对数字的二进制定点表示方法。原码表示法在数值前面前面有一位符号位（即最高位为符号位），正数该位为0，负数该位为1（0有两种表示：+0和-0），其余位表示数值的大小。

## 反码

正数的反码与其原码相同；负数的反码是对其原码逐位取反，但符号位除外。

## 补码
补码正是基于反码的“-0”问题诞生的，可以解决这个问题。

补码的计算方法是：正数和+0的补码是其本身，负数则先计算其反码，然后反码加上1，得到补码。

补码换算为原码的过程中，如果补码是正数或者+0的补码，则其原码就是补码本身；如果补码是负数或者-0的补码，则其原码的计算方法是，先将补码减掉1，得到反码，再将反码取反，得到原码。

int类型在内存中存储的方式，即int类型在内存中，以补码的形式存储。而且我们还知道了为何int类型的取值范围中负数的最小值的绝对值比正数的最大值大1的原因，即-2^32的补码是10000000 00000000 00000000，原本-0的位置被-2^32取代了。