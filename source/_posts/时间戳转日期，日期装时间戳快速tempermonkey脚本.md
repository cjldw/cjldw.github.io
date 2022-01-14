---
title: 时间戳转日期，日期装时间戳快速tempermonkey脚本
tags: []
date: 2016-04-13 23:52:00
---

使用chrome tempermonkey自定义添加一个时间日期转换插件。
chrome tempermonkey是一个很强大的插件。 可以对页面做任何处理。

<!-- more -->


####　使用　####

１．安装chrome　tempermonkey插件．
２．安装此脚本．
３．连续按**t**键**3**次，　调出时间戳转日期输入框，按回车转换，按`esc`退出．
４．连续按**d**键**3**次，调出日期转时间戳选择框，按回车转化，按`esc`退出．

```javascript
// ==UserScript==
// @name         time-date-convertor
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  press [t/d] key three time then show [timestamp/date] convertor
// @author       arvim.lo <bigpao.luo@gmail.com>
// @require       http://apps.bdimg.com/libs/jquery/2.1.4/jquery.min.js
// @match        http*://*/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';
    var timeTemplateEle = $('<div  style="display: inline; position: fixed; z-index:10000; right: 10px; top: 10px;"><input type="number" id="time-ipt" />
<span id="time-result"></span></div>');
    var dateTemplateEle = $('<div  style=" display: inline; position: fixed; z-index: 10000; right: 10px; top: 10px;"><input type="date" id="date-ipt" />
<span id="date-result"></span></div>');
    var bodyEle = $("body");
    var timeResultEle = $("#time-result", timeTemplateEle);
    var timeIptEle = $("#time-ipt", timeTemplateEle);
    var cssStyle =  {
        "font-size": "12px",
        "padding": " 5px",
        "color": "#555",
        "vertial-align": "middle",
        "border": "1px solid #ccc",
        "border-radius": "3px",
    };
    timeIptEle.css(cssStyle);

    var dateResultEle = $("#date-result", dateTemplateEle);
    var dateIptEle = $("#date-ipt", dateTemplateEle);
    dateIptEle.css(cssStyle);
    var keyCounter = new Map();

    /**
     * 响应ｔ事件，　当ｔ连续三次后，　调出输入框，　输入时间戳，　回车时间计算，　ｅｓｃ键干掉
     */
    bodyEle.on("keyup", "#time-ipt", function (evt) {
        var timeIptEle = $(this);
        if (evt.keyCode == 13) {
            var value = timeIptEle.val();
            if ($.trim(value) === "") {
                alert(" Please Input Value ! ");
            } else {
                var dateObj = new Date(value * 1000);
                timeResultEle.html(dateObj.getFullYear() + '/' + (dateObj.getMonth() + 1) + '/' + dateObj.getDate() + ' ' + dateObj.getHours() + ':' + dateObj.getMinutes() + ':' + dateObj.getSeconds());
            }
        } else if (evt.keyCode == 27) {
            timeTemplateEle.remove();
        }
    });

    /**
     * 响应ｄ事件，　当日期选择好后，　回车转换成时间ＪａｖａＳｃｒｉｐｔ时间戳,ｅｓｃ键干掉
     */
    bodyEle.on("keyup", "#date-ipt", function (evt) {
        var dateIptEle = $(this);
        if (evt.keyCode == 13) {
            var value = dateIptEle.val();
            if ($.trim(value) === "") {
                alert(" Please Input Date ! ");
            } else {
                var dateObj = new Date(value);
                dateResultEle.html(dateObj.getTime());
                return;
            }
        } else if (evt.keyCode == 27) {
            dateTemplateEle.remove();
        }
    });

    /**
     * 响应　ｔ，ｄ事件，　当ｔ，ｄ连续三次，　调出时间，日期转ｉｎｐｕｔ框
     */
    bodyEle.on("keyup", function (evt) {
        var keyCode = evt.keyCode,
        keyFlag = 'key-' + keyCode,
        keyCount = keyCounter.get(keyFlag) || 0;

        if (keyCode == 84) {
            keyCounter.set(keyFlag, ++keyCount);
            if (keyCount == 3) {
                bodyEle.prepend(timeTemplateEle);
                timeIptEle.focus();
                keyCounter.clear();
            }
        } else if (keyCode == 68) {
            keyCounter.set(keyFlag, ++keyCount);
            if (keyCount === 3) {
                bodyEle.prepend(dateTemplateEle);
                dateIptEle.focus();
                keyCounter.clear();
            }
        } else {
            keyCounter.clear();
        }
    });
})();
```
