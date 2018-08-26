---
title: 合并多个promise对象
date: 2016-11-13 11:15:47
desc: JavaScript 异步处理多个Promise后, 需要计算多个Promise返回结果集合
tags: JavaScript
---

引入: 当多个异步操作后, 如何拿到所有异步结果做聚合运算后返回.
真实场景: 百度地图获取距离和时间, 由于百度地图api距离计算api最多支持10个途径点, 因此当坐标多于10个点是, 就需要做切割调用, 这样就出现多个异步操作, 而需要的结果是所有异步操作的结果做聚合运算的结果. 怎么搞?


<!-- more -->

### 步操作数量已知

1. 让每个异步操作返回一个promise对象

2. 通过jQuery的`$.when(promise1, promise2, ...).then(resultSet1, resultSet2, ...)`简单解决.


### 异步操作未知 (百度地图为例)

1. 将需要规划的坐标集合切分成多个数组.

2. 创建一个Promise容器, 循环遍历上步切分的数组, 多次调用百度规划api(**规划接口为异步操作**), 将每次异步调用的返回值(**Promise对象***)添加到Promise容器中

3. 通过`$.when.apply($, promisePool, callback)`, `promisePool`为上步返回的promise对象容器池, `callback`为回调函数, 我们在回调函数拿到结果.


4. 具体代码贴出

    ```html
        <!DOCTYPE html>
        <html>
        <head>
          <meta charset="utf-8">
          <meta name="viewport" content="width=device-width">
          <title>JS Bin</title>
          <script src="http://api.map.baidu.com/api?v=2.0&ak=Zyep8gikHp8WokQ2qD1vtyONsfZXxEHR" type="text/javascript"></script>
          <script src="https://code.jquery.com/jquery-2.2.4.js"></script>
        </head>
        <body>
        <script>
          var distancdDuration = function(startPoint, endPoint, wayPoints, dfd){
            var route = new BMap.DrivingRoute(startPoint);
            route.search(startPoint, endPoint, {waypoints: wayPoints});
            route.setSearchCompleteCallback(function(result) {
                return dfd.resolve(result.getPlan(0));
            });
            return dfd.promise();
          };

        var doSearch = function(dataSource, callback) {

          var promises = []; // 多个promise容器
          for(var i = 0, len = dataSource.length; i < len; i++) {
              var item = dataSource[i];
              var dfd = distancdDuration(item.startPoint, item.endPoint, item.wayPoints, $.Deferred());
              promises.push(dfd)
          }

          $.when.apply($, promises).then(function(){ // 处理容器中的promise
              var resultSet = {distance: 0, duration: 0};
              var resultList = arguments;
              for (var i = 0, len = resultList.length; i < len; i++) {
                  var item = resultList[i];
                  if(typeof item !== "undefined") {
                      resultSet.distance += parseFloat(item.getDistance(false));
                      resultSet.duration += parseFloat(item.getDuration(false));
                  }
              }
              callback(resultSet);
          });
        }

        var dataSource = [{
          startPoint: new BMap.Point(116.470959,39.879951),
          endPoint: new BMap.Point(116.423241,39.990599),
          wayPoints: [
            new BMap.Point(116.570994,39.805494),
            new BMap.Point(116.251341,39.967599),
            new BMap.Point(116.491081,39.908294)
          ]
        },{
          startPoint: new BMap.Point(116.353676,39.983523),
          endPoint: new BMap.Point(116.312282,39.895452),
          wayPoints: [
            new BMap.Point(116.442788,39.993695),
            new BMap.Point(116.39277,39.930871),
            new BMap.Point(116.386446,40.011823),
            new BMap.Point(116.492231,39.917591)
          ]
        }];

        var resultSet = doSearch(dataSource, function(resultSet) {
            console.log(resultSet)
        });
        </script>

        </body>
        </html>

    ```

5. 亲测OK, 有问题欢迎拍砖.
