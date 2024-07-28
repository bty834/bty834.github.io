---
title: BigDecimal使用与存储计算原理
categories: [ 编程, java ]
tags: [ java ]
---

在Java中，金额计算通常使用`java.lang.Long`或`java.math.BigDecimal`类型。

在金额计算简单时，可以将金额单位设置为分，然后用Long表示多少多少分，
这样的金额范围是`[-92233_7203_6854_7758_08, 92233_7203_6854_7758_07]`分，金额范围基本能满足大多数场景了。

但是涉及到复杂金额计算，尤其是除法时（例如:优惠分摊场景），Long则力不从心，而BigDecimal能够保证除法的精度，且范围扩大到 `((-2)^Integer. MAX_VALUE, 2^Integer.MAX_VALUE)`。

此外，金额计算尤其不推荐`java.lang.Double`类型，主要考虑到浮点数在计算机中的存储方式，会导致精度丢失：

```java
double num = 6555555.5555555555;
//     print 6555555.555555555
System.out.println(num); 
```

具体可参考浮点数存储规范:
- [IEEE-754 Floating Point Converter](https://www.h-schmidt.net/FloatConverter/IEEE754.html)
- [How are floating point numbers stored in memory?](https://stackoverflow.com/questions/7644699/how-are-floating-point-numbers-stored-in-memory)


## BigDecimal使用

```java
        BigDecimal bigDecimal = new BigDecimal("1.23"); // 通常使用字符串初始化

        bigDecimal = bigDecimal.setScale(1, RoundingMode.HALF_UP); // 保留一位小数，四舍五入
        System.out.println(bigDecimal);

        BigDecimal bigDecimal2 = new BigDecimal("11E5");
        System.out.println(bigDecimal2.toString()); // 1.1E+6
        System.out.println(bigDecimal2.toPlainString()); // 1100000

        BigDecimal bigDecimal3 = new BigDecimal("1.23000");
        System.out.println(bigDecimal3.toPlainString()); // 1.23000
        System.out.println(bigDecimal3.stripTrailingZeros().toPlainString()); // 1.23

        BigDecimal bigDecimal4 = new BigDecimal("234.4");
        System.out.println(bigDecimal4.longValue());; // 234L
        System.out.println(bigDecimal4.longValueExact()); // java.lang.ArithmeticException
        System.out.println(bigDecimal4.doubleValue()); //  234.4
        System.out.println(bigDecimal4.intValue()); // 234
        System.out.println(bigDecimal4.intValueExact()); // java.lang.ArithmeticException

        BigDecimal add = bigDecimal3.add(bigDecimal2); // +
        BigDecimal subtract = bigDecimal3.subtract(bigDecimal2); // -
        BigDecimal multiply = bigDecimal3.multiply(bigDecimal2); // *
        BigDecimal divide = bigDecimal3.divide(bigDecimal2, 2, RoundingMode.HALF_UP); // /
```

## BigDecimal(BigInteger)存储与计算原理

`BigDecimal`中使用scale表示小数位位数，BigDecimal会将输入的小数放大为整数，最终转化为 `long` 或 `BigInteger` 存储：
如果 在`long`类型的数值范围内则使用`long`，溢出则使用`BigInteger`存储。 
所以，有些业务直接用`long`存储金额也没问题，只是其覆盖场景没有`BigDecimal`全。

`BigDecimal`所有的计算，实际上委托给了`BigInteger`，接下来看下`BigInteger`是如何存储“大数”的。

```java
public class BigInteger extends Number implements Comparable<BigInteger> {
    // 符号位：-1 for negative, 0 for zero, or 1 for positive
    final int signum;

    // 数值：采用int数组存储绝对值防止溢出，big-endian 
    final int[] mag;
```

int最大值：1111111_11111111_11111111_11111111b = 21_4748_3647
此处我们加一个加1个二进制位1来演示：1_11111111_11111111_11111111_11111111b = 85_8993_4591

```java
// 首先添加VM Options: --add-opens java.base/java.math=ALL-UNNAMED

long val = 85_8993_4591L; // 33位： 1_11111111_11111111_11111111_11111111b
BigInteger bigInteger = BigInteger.valueOf(val);

Field magField = BigInteger.class.getDeclaredField("mag");
magField.setAccessible(true);
int[] mag = (int[])magField.get(bigInteger);

System.out.println(Integer.toBinaryString(mag[0])); // 1位： 1
System.out.println(Integer.toBinaryString(mag[1])); // 32位： 11111111_11111111_11111111_11111111
```

### BigInteger 加法
```java
    public BigInteger add(BigInteger val) {
        if (val.signum == 0)
            // 如果val为0，直接返回this
            return this;
        if (signum == 0)
            // 如果当前为0，直接返回val
            return val;
        if (val.signum == signum)
            // 同符号相加，新建实例返回
            return new BigInteger(add(mag, val.mag), signum);
        // 不同符号，则将大绝对值减去小绝对值，符号取大绝对值原数的符号
        
        // 比较两个不同符号数的绝对值谁大
        int cmp = compareMagnitude(val);
        if (cmp == 0)
            // 一样大返回0
            return ZERO;
        // 大绝对值-小绝对值的结果
        int[] resultMag = (cmp > 0 ? subtract(mag, val.mag)
                           : subtract(val.mag, mag));
        // 去掉resultMag里0开头的字节
        resultMag = trustedStripLeadingZeroInts(resultMag);
        // 返回新实例
        return new BigInteger(resultMag, cmp == signum ? 1 : -1);
    }
```

看看同符号的`add`方法:

```java
private static int[] add(int[] x, int[] y) {
        // 设定 x是大数，y是小数
        // If x is shorter, swap the two arrays
        if (x.length < y.length) {
            int[] tmp = x;
            x = y;
            y = tmp;
        }

        int xIndex = x.length;
        int yIndex = y.length;
        // 先用大数x长度做初始化结果
        int result[] = new int[xIndex];
        long sum = 0;
        if (yIndex == 1) {
            // 如果小数y的mag数组长度只有1，那么只要把x的最高索引(也就是十进制里小的位数，这里是big-endian)的int和y[0]相加存在long的sum中
            // 这里不会溢出，因为sum是long类型
            sum = (x[--xIndex] & LONG_MASK) + (y[0] & LONG_MASK) ;
            // 强制转为int，比int大怎么办？result[xIndex]二进制直接存全1，溢出部分后续会处理
            result[xIndex] = (int)sum;
        } else {
            // 如果y的mag数组长度大于1，则对齐int[]索引最高位相加，即从十进制的数位小的往大的加（big-endian就是索引从大到小加）
            // Add common parts of both numbers
            while (yIndex > 0) {
                // 这里sum>>>32就是上一次相加溢出的部分
                sum = (x[--xIndex] & LONG_MASK) + (y[--yIndex] & LONG_MASK) + (sum >>> 32);
                result[xIndex] = (int)sum;
            }
        }
        // Copy remainder of longer number while carry propagation is required
        // carry为true则表示还有溢出
        boolean carry = (sum >>> 32 != 0);
        
        // 如果x数组,y数组长度相同，这里就不会执行
        while (xIndex > 0 && carry)
            // x数组长度比y数组长 且溢出标志carry为true
            // 则剩余的x的都要加1,加的1即是溢出的进位，加完之后如果还有进位，则继续加，没有就break
            carry = ((result[--xIndex] = x[xIndex] + 1) == 0);

        // Copy remainder of longer number
        while (xIndex > 0)
            // 第一个while没有进位了，把x剩余的字节copy到结果
            result[--xIndex] = x[xIndex];

        // Grow result if necessary
        if (carry) {
            // 第一个while还有进位，则需要给result扩容了
            int bigger[] = new int[result.length + 1];
            System.arraycopy(result, 0, bigger, 1, result.length);
            // big-endian
            bigger[0] = 0x01;
            return bigger;
        }
        return result;
    }
```

### BigDecimal & BigInteger乘法：Toom-Cook Algorithm

BigDecimal中会判断乘数和被乘数是否溢出long类型范围，如果没溢出则使用long类型的intCompact做乘（此时如果两个long相乘的结果溢出long范围，则会将其中一个乘数转为BigInteger再乘一次），溢出则使用BigInteger类型的intVal做乘。

当使用BigInteger时，同样乘法都委托给了BigInteger去处理。

BigInteger使用到了大数相乘的算法：[Karatsuba算法](https://en.wikipedia.org/wiki/Karatsuba_algorithm) 和 [Toom-Cook(Took-3)算法](https://en.wikipedia.org/wiki/Toom%E2%80%93Cook_multiplication)

- [Toom-Cook (Toom3) Algorithm Explained with Examples (Generalization of Karatsuba Algo)](https://www.youtube.com/watch?v=1XiSyNzMX6Q)

普通小学学的乘法步骤的时间复杂度为`O(n^2)`，Karatsuba算法时间复杂度为 `O(n^1.585)`，而 Toom-Cook3算法时间复杂度为`O(n^1.465)`

这些算法都是将部分乘法转化为加法，计算机处理加法的速度要快于乘法。

### BigDecimal除法

BigDecimal中divide方法：
```java

    public BigDecimal divide(BigDecimal divisor, int scale, int roundingMode) {
        ...
        if (this.intCompact != INFLATED) {
            if ((divisor.intCompact != INFLATED)) {
                // 除数被除数都没超过long范围，则直接相除
                // 否则借助BigInteger 的divide方法：MutableBigInteger divide(MutableBigInteger b, MutableBigInteger quotient)
                return divide(this.intCompact, this.scale, divisor.intCompact, divisor.scale, scale, roundingMode);
            } else {
        ...
```
再来看看divide的这个重载方法：
```java
// 3/2 这里3是dividend，2是divisor
private static BigDecimal divide(
                                  long dividend, // 被除数
                                  int dividendScale, // dividend的小数位数
                                  long divisor, // 除数
                                  int divisorScale,  // divisor的小数位数
                                  int scale, // 相除结果的小数位数
                                  int roundingMode // 舍入策略
                                  ) {
        // 思路是取max(scale + divisorScale,dividendScale)
        if (checkScale(dividend,(long)scale + divisorScale) > dividendScale) {
            // 除数的小数位数 + 结果的小数位数 > 被除数的小数位数，且和不超过Integer.MAX_VALUE
            
            // 将dividend的scale提升到和newScale一致
            int newScale = scale + divisorScale;
            int raise = newScale - dividendScale; 
            if(raise<LONG_TEN_POWERS_TABLE.length) {
                // xs记录提升scle后的dividend，比如200.1的scale为1，这里raise为2，则变为20010，scale为3
                long xs = dividend;
                if ((xs = longMultiplyPowerTen(xs, raise)) != INFLATED) {
                    // divedend扩大不溢出long范围
                    return divideAndRound(xs, divisor, scale, roundingMode, scale);
                }
                // 溢出long范围则分解dividend，除多次，比如： a = b*c, 则 a / d = c * b/d 
                // 如果商quotient long类型放不下则为null
                BigDecimal q = multiplyDivideAndRound(LONG_TEN_POWERS_TABLE[raise], dividend, divisor, scale, roundingMode, scale);
                if(q!=null) {
                    return q;
                }
            }
            // 转换成BigInteger做除法
            BigInteger scaledDividend = bigMultiplyPowerTen(dividend, raise);
            return divideAndRound(scaledDividend, divisor, scale, roundingMode, scale);
        } else {
            // 操作同上
            int newScale = checkScale(divisor,(long)dividendScale - scale);
            int raise = newScale - divisorScale;
            if(raise<LONG_TEN_POWERS_TABLE.length) {
                long ys = divisor;
                if ((ys = longMultiplyPowerTen(ys, raise)) != INFLATED) {
                    return divideAndRound(dividend, ys, scale, roundingMode, scale);
                }
            }
            BigInteger scaledDivisor = bigMultiplyPowerTen(divisor, raise);
            return divideAndRound(BigInteger.valueOf(dividend), scaledDivisor, scale, roundingMode, scale);
        }
    }
```

先来看看直接用两个long做除的流程：
```java
// 注意这里的ldividend是扩大了raise倍的，且 ldividend的scale = ldivisor的scale + scale
private static BigDecimal divideAndRound(long ldividend, long ldivisor, int scale, int roundingMode,
                                             int preferredScale) { // preferredScale 就是scale

        int qsign; // quotient sign
        // 直接相除存在long中
        long q = ldividend / ldivisor; // store quotient in long
        // 如果是向下取整，直接返回q就行
        if (roundingMode == ROUND_DOWN && scale == preferredScale)
            return valueOf(q, scale);
        long r = ldividend % ldivisor; // 取余数
        qsign = ((ldividend < 0) == (ldivisor < 0)) ? 1 : -1; // 结果判断符号
        if (r != 0) {
            // 如果余数不为0，判断是否要进位，由于dividend扩大了倍数使得其scale等于divisor的scale+预期结果的scale
            // 这里进位也只会进1
            boolean increment = needIncrement(ldivisor, roundingMode, qsign, q, r);
            return valueOf((increment ? q + qsign : q), scale);
        } else {
            // 刚好除尽，没有余数
            if (preferredScale != scale)
                return createAndStripZerosToMatchScale(q, scale, preferredScale);
            else
                return valueOf(q, scale);
        }
    }
```

### BigInteger除法：Knuth & BurnikelZiegler Algorithm

Knuth算法（试商法）时间复杂度为`O(n^2)`：
- [大整数乘除法的实现（二）](https://blog.csdn.net/m0_51303687/article/details/122413575)
- [C++大整数运算（四）：除法](https://blog.kedixa.top/2017/cpp-bigint-div/)

BurnikelZiegler Algorithm 参见：
- [Fast Recursive Division: BurnikelZiegler Algorithm](https://pure.mpg.de/rest/items/item_1819444_4/component/file_2599480/content)

