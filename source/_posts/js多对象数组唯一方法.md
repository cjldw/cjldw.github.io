---
title: js多对象数组唯一方法
date: 2016-09-13 17:33:58
desc:
tags:
---

前些天做项目, 小伙伴遇到一个js多对象数组, 求唯一问题, 思考了很久, 没有找到好的方案. 小伙伴使用了递归, 个人感觉不是最好的方案.
今天有空重写了一个.

<!--more-->

## 原理分析

* 遍历数组中的对象, 第一个对象和接下来所有的对象进行对比, 如果碰到一致的, 将其删除. 自身对象进行处理.

* 问题主要处在删除后, 数组的索引发生变更, 之前没有考虑到. 一直不正确. 今天把代码贴出. 欢迎拍砖.! :)

* es6 code 需要babeljs转换下

    ```javascript
        // es6 code
        const uniq = (source = [], fields = [], callback = ()=>{}) => {
          for(let i = 0; i < source.length; i++){
            for(let j = i+1; j < source.length; j++){
                /*
                 * 对比对象中的字段(传入的参数)是否相等 返回一个和fileds数组相同数量的true, false数组,
                 * 本列子返回[false, true], 对返回的true, false数组reduce计算, 看看对象是否相等, 如果相等
                 * 调用callback, 删除原数组中相同的对象, 原数组相同的对象删除后, for循环需要回退.
                 */
              if(fields.map((field) => {
                return source[i][field] == source[j][field];
              }).reduce((prev, next) => prev&&next)){
                source[i] = callback(source[i]);
                source.splice(j, 1);
                j--; // 回退for
              }
            }
          }
          return source;
        };

        let source = [
          {name: 'luo', age: 3, count:1}, 
          {name: 'v', age: 2, count:1}, 
          {name: 'luo', age: 3, count:1},
          {name: 'luo', age: 3, count:1},
          {name: 'luo', age: 3, count:1},
          {name: 'v', age: 2, count:1}, 
          {name: 'luo', age: 3, count:1}, 
          {name: 'v', age: 3, count:1}, 
          {name: 'v', age: 2, count:1}, 
          {name: 'v', age: 42, count:1}, 
        ];

        let ret = uniq(source, ['name', 'age'], (item) => {
          item.count += 1;
          return item;
        });

        * es5 转换好的code

        // es5 code
        'use strict';

        var uniq = function uniq() {
          var source = arguments.length <= 0 || arguments[0] === undefined ? [] : arguments[0];
          var fields = arguments.length <= 1 || arguments[1] === undefined ? [] : arguments[1];
          var callback = arguments.length <= 2 || arguments[2] === undefined ? function () {} : arguments[2];

          var _loop = function _loop(i) {
            var _loop2 = function _loop2(_j) {
              if (fields.map(function (field) {
                return source[i][field] == source[_j][field];
              }).reduce(function (prev, next) {
                return prev && next;
              })) {
                source[i] = callback(source[i]);
                source.splice(_j, 1);
                _j--;
              }
              j = _j;
            };

            for (var j = i + 1; j < source.length; j++) {
              _loop2(j);
            }
          };

          for (var i = 0; i < source.length; i++) {
            _loop(i);
          }
          return source;
        };

        var source = [{ name: 'luo', age: 3, count: 1 }, { name: 'v', age: 2, count: 1 }, { name: 'luo', age: 3, count: 1 }, { name: 'luo', age: 3, count: 1 }, { name: 'luo', age: 3, count: 1 }, { name: 'v', age: 2, count: 1 }, { name: 'luo', age: 3, count: 1 }, { name: 'v', age: 3, count: 1 }, { name: 'v', age: 2, count: 1 }, { name: 'v', age: 42, count: 1 }];

        var ret = uniq(source, ['name', 'age'], function (item) {
          item.count += 1;
          return item;
        });
        //console.log(ret)
    ```
