---
title: 身份证校验规则
date: 2017-07-18 17:19:47
desc: 身份证校验规则, 身份证后四位是根据前面生成出来的
tags: idcard, validate
---

身份证的后1位是由前面几位生成出来的!😄😄, 怎么生成的?

<!--more-->

### 身份证校验规则

身份证的校验规则php代码如下, 为什么是这样? 后面补上! 😬

```php
    public static function checkIdCard($idcard){

        if(strlen($idcard)!=18){ return false; } // 只能是18位
        $idcardBase = substr($idcard, 0, 17); // 取出本体码
        $verifyCode = substr($idcard, 17, 1); // 取出校验码
        $factor = [7, 9, 10, 5, 8, 4, 2, 1, 6, 3, 7, 9, 10, 5, 8, 4, 2]; // 加权因子
        $verifyCodeList = ['1', '0', 'X', '9', '8', '7', '6', '5', '4', '3', '2']; // 校验码对应值
        $total = 0; // 根据前17位计算校验码
        for($i=0; $i<17; $i++){ $total += substr($idcardBase, $i, 1) * $factor[$i]; }
        $mod = $total % 11; // 取模
        return $verifyCode == $verifyCodeList[$mod]; // 比较校验码
    }
```
