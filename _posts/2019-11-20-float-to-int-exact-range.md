---
title: 多大范围内的整数可以用浮点数精确表示
key: t20191120
tags:
  - Mathematics
---

我们知道，IEEE 754中的浮点数是对实数的近似表示，其取值密度在越远离0的地方越稀疏。那么，究竟在多远处浮点数会无法精确地cast到整数呢？本文对此进行了实验和分析。

<!--more-->

实验代码很简单：

```cpp
int main()
{
    for(int i = 0; i < std::numeric_limits<int>::max(); ++i)
    {
        float f = static_cast<float>(i);
        if(i != static_cast<int>(f))
        {
            cout << i << endl;
            break;
        }
    }

    for(int i = 0; i > std::numeric_limits<int>::min(); --i)
    {
        float f = static_cast<float>(i);
        if(i != static_cast<int>(f))
        {
            cout << i << endl;
            break;
        }
    }
}
```

这段代码在`VS2019`中的输出如下：

```
16777217
-16777217
```

也就是说，从+-16777217开始，我们就无法保证一个`int`被转为`float`后，还能完好无损地转回来了。对二进制数比较敏感的人可以注意到：

$$
16777217 = 2^{24} + 1
$$

即浮点数能精确表示整数的范围为$\pm2^{24}$。这数字实在太“整”了，让我们试着从IEEE 754的浮点数格式中一探究竟。

注意到我们讨论的数值落在`float`的规格化表示范围内。`float`包含1个符号位，8个指数位，以及23位的尾数。设符号位为$s$，阶码的无符号整数解释为$e$，尾数的位表示为

$$
f_1f_2\ldots f_{23}
$$

则规格化的浮点数取值为：

$$
(-1)^s \times (1 + 2^{-1}f_1 + 2^{-2}f_2 + \cdots + 2^{-23}f_{23}) \times 2^{e - 127}
$$

当$e - 127 = 23$时，上式变为：

$$
(-1)^s \times (2^{23} + 2^{22}f_1 + 2^{21}f_2 + \cdots + 2f_{22} + f_{23})
$$

此时根据$f$每一位的值，该式可以取遍$[-2^{24} + 1, 2^{24} - 1]$中的每一个整数。而当$e - 127 = 24$且$f$的所有位均为0时，$\pm 2^{24}$也可以被精确地表示。一旦超过了这个范围，就出现了相邻的两个浮点值间的距离超过1的情况，也就无法再精确表示剩余的整数了。
