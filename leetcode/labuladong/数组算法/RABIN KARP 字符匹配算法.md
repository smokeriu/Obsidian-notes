> 为[滑动窗口](滑动窗口.md)的算法延伸。

RK字符匹配算法的核心，就是将字符串转换为数字来进行匹配。简单而言，将字符串的变化转换为数字的变化。

其背后的本质其实就是对符合要求的内容进行**Hash比较**，来提高比较效率。

# 核心

## 数字字符串

我们先把逻辑简化下，就考虑字符串本身也是数字的（10进制）情况，并将其转换为数字：
```java
// 10进制
String str = "8765"
int i = 8 * 1000 + 7 * 100 + 6 * 10 + 5;
```
一般而言，字符串的变化无非是前后添加或移除一位。

### 向后添加一位

```java
int newI = i * 10 + x;
```

### 向后移除一位

```java
int newI = i / 10;
```

### 向前添加一位

```java
int newI = x * Math.pow(10, 4) + i;
```

### 向前移除一位

```java
int newI = i - x * Math.pow(10, 4);
```

所以，只有知道i的位数，就能够将一个字符串的变化转换为数字的变化。而数字的变化在一般语言中是远快于字符串的变化的。

## 非数字字符串

为了使这个算法更加通用，一般的字符串肯定不是纯数字组成的。此时我们就要将字符与数字进行映射，构建自己的进制。

如[187. 重复的DNA序列](187.%20重复的DNA序列.md)此题中，因为只存在4类字符，我们可以很简单的与`0, 1, 2, 3`这几个数字进行关联，并且构建一个4进制的数字计算逻辑。

# 相关陷阱

## 整形溢出

通过`Int`存放`Hash`值，最大的问题就是产生整形溢出，即使通过`Long`来存放也是无法避免的，如果遇到这种情况，我们需要考虑对存放的`Hash`值取模`Q`。根据取模算法，取模后的值一定在`[0, Q-1]`之间。这样就绝对不会产生溢出，但也带来了另一个问题：[哈希冲突](RABIN%20KARP%20字符匹配算法.md#哈希冲突)。

## 哈希冲突

不同的Hash值通过取模后可能产生同样的模值，而我们无法简单的反推。Java的HashMap是通过Hash桶+链表的方式解决Hash冲突的，详见[理解HashMap](理解HashMap.md)。一般而言，不需要如此复杂，可以在评估冲突不多的情况下，进行暴力匹配，来找到实际的Hash值。

## 取余运算

算法会涉及到一些模运算的数学知识：
1. 取模
$$
X\bmod Q == (X + Q) \bmod Q 
$$
2. qu 

> 在Java中，取模的本质其实是取余运算，由于这块儿的数学差异不会对此篇内容产生影响，故可忽略某些描述。


# 参考代码

```java
int rabinKarp(String txt, String pat) {
    // 位数
    int L = pat.length();

    // 进制（只考虑 ASCII 编码）
    int R = 256;

    // 取一个比较大的素数作为求模的除数
    long Q = 1658598167;

    // R^(L - 1) 的结果
    long RL = 1;
    for (int i = 1; i <= L - 1; i++) {
        // 计算过程中不断求模，避免溢出
        RL = (RL * R) % Q;
    }

    // 计算模式串的哈希值，时间 O(L)
    long patHash = 0;
    for (int i = 0; i < pat.length(); i++) {
        patHash = (R * patHash + pat.charAt(i)) % Q;
    }

    // 滑动窗口中子字符串的哈希值
    long windowHash = 0;
    // 滑动窗口代码框架，时间 O(N)
    int left = 0, right = 0;

    while (right < txt.length()) {
        // 扩大窗口，移入字符
        windowHash = ((R * windowHash) % Q + txt.charAt(right)) % Q;
        right++;

        // 当子串的长度达到要求
        if (right - left == L) {
            // 根据哈希值判断是否匹配模式串
            if (windowHash == patHash) {
                // 当前窗口中的子串哈希值等于模式串的哈希值
                // 还需进一步确认窗口子串是否真的和模式串相同，避免哈希冲突
                if (pat.equals(txt.substring(left, right))) {
                    return left;
                }
            }

            // 缩小窗口，移出字符
            windowHash = (windowHash - (txt.charAt(left) * RL) % Q + Q) % Q;
            // X % Q == (X + Q) % Q 是一个模运算法则
            // 因为 windowHash - (txt[left] * RL) % Q 可能是负数
            // 所以额外再加一个 Q，保证 windowHash 不会是负数
            left++;
        }
    }

    // 没有找到模式串
    return -1;
}
```

# 相关题目

1. [187. 重复的DNA序列](187.%20重复的DNA序列.md)