---
title: JSON数据重构给HightChart绘图
date: 2018-10-26 20:31:43
desc: 从服务端获取数据, 重构后绘图
tags: javascript, chart
---

从服务端后去统计数据, 日期是间断的, 前端需要拿数据绘图,日期需要补充, 要么服务端补充, 要么客户端补充。

<!-- more -->

优先选着客户端补充会更好。贴码

```javascript

 (function () {
        "use strict";
        var data = [
            {
                "id": 1,
                "date": "2018-01-01",
                "value": 10
            },
            {
                "id": 2,
                "date": "2018-01-02",
                "value": 1000
            },
            {
                "id": 10,
                "date": "2018-01-08",
                "value": 101
            },
            {
                "id": 2,
                "date": "2018-01-20",
                "value": 13
            },
            {
                "id": 3,
                "date": "2018-01-30",
                "value": 20
            }
        ];

        function builder(data, st, et, column = "date", step = 86400000, callback) {
            var index = 0,
                result = {},
                dataLength = data.length,
                start = (new Date(st)).getTime(),
                end = (new Date(et)).getTime();

            if (typeof callback == "undefined") { // 通用格式日期函数
                callback = function (dateObject) {
                    var day = dateObject.getDate(),
                        month = dateObject.getMonth() + 1;
                    day = day < 10 ? "0" + day : day;
                    month = month < 10 ? "0" + month : month;
                    return dateObject.getFullYear() + "-" + month + "-" + day;
                }
            }

            while (start <= end) {
                for (; index < dataLength;) {
                    var item = data[index],
                        current = (new Date(item[column])).getTime();
                    if (current == start) { // 当前遍历的时间和给的数据相等
                        index += 1;
                        if (index >= dataLength) {
                            index = dataLength - 1;
                        }
                        for (var key in item) {
                            result[key] = result[key] || [];
                            if (key == column) {
                                result[key].push(callback(new Date(start)));
                                continue;
                            }
                            result[key].push(item[key])
                        }
                        break;
                    }
                    for (var key in item) {
                        result[key] = result[key] || [];
                        if (key == column) {
                            result[key].push(callback(new Date(start)));
                            continue;
                        }
                        switch (typeof item[key]) {
                            case "string":
                                result[key].push("");
                                break;
                            default:
                                result[key].push(0);
                                break;
                        }
                    }
                    break;
                }
                start += step
            }
            return result;
        }
        var result = builder(data, "2018-01-01", "2018-02-03", 'date', 86400000, function (dateObject) {
            var day = dateObject.getDate(),
                month = dateObject.getMonth() + 1;
            day = day < 10 ? "0" + day : day;
            month = month < 10 ? "0" + month : month;
            return dateObject.getFullYear() + "/" + month + "/" + day;
        });
        console.log(result);

    })();
```

![执行结果](/images/json-buider4chart.jpg)

