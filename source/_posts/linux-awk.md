---
title: 理解linux-awk命令
date: 2017-11-30 21:36:33
desc: awk常用案例
tags: linux, awk
---

AWK是一门古老的 linux shell 语言。每个unix-like系统都默认安装, linux下主要是 gawk这个版本。awk有着其独特的魅力。来简单的学习下它。

<!--more-->

awk的语法结构简单如下

```bash
    BEGIN { print "start"}
    { awk_expression }
    END {print "end"}
```

awk表达式支持算数运算

|操作符|类型|意义|
|:--|:---|:--|
|+|integer|加|
|-|integer|减|
|*|integer|乘|
|/|integer|除|
|%|integer|取模|
|空格|string|字符串连接|

简单的栗子:

```bash
    $ echo ""| awk "{print 1+2}" // output 3
```

awk支持的判断表达式

|操作符|意义|
|:--|:--|
|==| 相等 |
|!=| 不等 |
|>|大于|
|>=| 大等于 |
|<|小于|
|<=|小等于|

```bash
    $ echo "" | awk '{print 2 == 3}' // output 0
```

awk正则匹配

|操作符|意义|
|:--|:--|
|~| 匹配 |
|!~| 不匹配 |


```bash
    $ echo "luowen" | awk '{print $0 ~ "uow"}' // output 1
```

awk命令语法

```bash
if ( conditional ) statement [ else statement ]
while ( conditional ) statement
for ( expression ; conditional ; expression ) statement
for ( variable in array ) statement
break
continue
{ [ statement ] ...}
variable=expression
print [ expression-list ] [ > expression ]
printf format [ , expression-list ] [ > expression ]
next 
exit
```

循环栗子

```bash

BEGIN {
    i=1;
    while (i <= 10) {
        printf "The square of ", i, " is ", i*i;
        i = i+1;
    }

    for (i=1; i <= 10; i++) {
        printf "The square of ", i, " is ", i*i;
    }
exit;
}
```

awk使用`-F`指定分隔符定义或在`BEGIN{FS=":"}`制定, 默认awk使用空格分隔输入的字段

```bash
    $ awk -F: '{if ($2 == "") print $1 ": no password!"}' </etc/passwd
    $ 或使用 FS 指定
    $ awk 'BEGIN{ FS=":"} {if ($2 == "") {print $1 "passoword not set yet!"}}'
```

awk在`BEGIN{OFS="#"}`制定输出分隔符

```bash
    $ awk 'BEGIN{OFS="#"} {print $1, $2}' < /etc/passwd // output root#x ..
```

awk使用`NF`常量判断输入的字段数量

```bash
    $ awk -F: '{ print "字段数量" NF; if (NF == 8) { print $0 }}' < /etc/passwd // output NONE
```

awk使用`NR`常量来计算行数

```bash
    $ awk -F: '{print "当前字段数量: " NR; if (NR > 4) {exit;}}' < /etc/passwd // 之处里前4个字段数据
```

awk生成随机数, 会根据硬件设备生成一个不变化的随机数, 严格来说, 这个假的随机数。

```bash
    $ echo "" | awk '{printf("生成随机数: %d", int(rand() * 10))}'
```

**awk的字符串函数**

|名称| 解释|
|:--|:--|
|index(string,search)|搜索字符的索引值|
|length(string)|判断字符串长度|
|split(string,array,separator)|切割字符串到数组array中, 返回数组的长度|
|substr(string,position)|前切字符串 position表示从那个字符开始, 默认索引1开始算|
|substr(string,position,max)|剪切字符串, position表示从那个字符串开始, max表示剪切几个字符|
|sub(regex,replacement)|替换, 类似sed -n 's/old/new/p'|
|sub(regex,replacement,string)||
|match(string,regex)|正则匹配, 返回1/0|
|tolower(string)|字符串小写|
|toupper(string)|字符串大写|

awk调用系统函数

```bash
    $ echo ""| awk "{if(system('/bin/rm /tmp/test.log') != 0) { print 'delete /tmp/test.log not OK'}}"
```

awk数组的使用, (计算网站nginx访问日志的ip请求量)

```bash
    $ awk '{ipList[$1]++} END{ for (ipAddr in ipList) {printf("ip地址:%s 访问数量:%d\n", ipAddr, ipList[ipAddr])}}' < /var/log/nginx/access.log
```



[参考原帖](http://www.grymoire.com/Unix/Awk.html#uh-26)
