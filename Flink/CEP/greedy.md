> 即贪婪匹配

其不能使用在单模式，否模式，组模式上，一般而言，其是用来描述量词，但**greedy仅在多Pattern中有效果**。

```java
"Option not applicable to FollowedByAny pattern"
"Option not applicable to singleton quantifier"
"Option not applicable to NOT pattern"
"Option not applicable to group pattern"
```

# 合法用法

greedy会搭配`oneOrMore`；`times`；`timesOrMore`一起使用。
```java
A.oneOrMore().greedy();
```




# 生效机制

greedy描述为贪婪，但CEP中的循环量词，如`oneOrMore`, `times`, `timesOrMore`，对于单Pattern而言，