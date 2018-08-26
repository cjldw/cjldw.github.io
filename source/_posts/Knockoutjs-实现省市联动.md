---
title: Knockoutjs 实现省市联动
tags: []
date: 2015-08-25 00:22:00
---

## Knockoutjs 实现省市联动 ##

html code

```html

<!doctype>
<html>
    <head>
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <meta charset="utf-8">
        <link rel="stylesheet" type="text/css" href="bower_components/bootstrap/dist/css/bootstrap.css">
        <link rel="stylesheet" type="text/css" href="bower_components/bootstrap/dist/css/bootstrap-theme.css">
        <script src="bower_components/jquery/dist/jquery.js"></script>
        <script src="bower_components/bootstrap/dist/js/bootstrap.js"></script>
        <script src="bower_components/knockoutjs/dist/knockout.debug.js"></script>
    </head>
    <body>
        <div class="container">
            <div class="row">
                <div class="col-md-3" id="country">
                    <select class="form-control" data-bind="options: countryArray, optionsText: 'countryName', optionsValue: 'countryId', value: countryValue, optionsCaption: 'please select', event:{'change': getCity}"></select>
                </div>
                <div class="col-md-2" id="village">
                    <select class="form-control" data-bind="visible: countryValue, options: cityArray, optionsText: 'cityName', optionsCaption: 'Please select', optionsValue: 'cityId', value: 'cityValue'"></select>
                </div>
            </div>
        </div>
    </body>
    <script>
        var countryModel = function(){
            var that = this;

            that.countryArray = ko.observableArray();

            that.countryValue = ko.observable();

            $.getJSON('http://localhost/demo.php', function(response) {//动态获取国家
                that.countryArray(response);
            })

            that.getCity = function() {
                var id = that.countryValue();
                $.getJSON('http://localhost/demo.php', {id: id}, function(response){//动态获取城市
                    that.cityArray(response);
                })
            }

            that.cityArray = ko.observableArray();
        }

        ko.applyBindings(new countryModel());
    </script>
</html>

```

javascript code

```php
<?php

$country = array(
    array(
        'countryName' => 'china',
        'countryId' => 1,
    ),
    array(
        'countryName' => 'japan',
        'countryId' => 2,
    ),
    array(
        'countryName' => 'korean',
        'countryId' => 3,
    ),
);

if(isset($_GET['id']) && $_GET['id'] == '1')
{
    $city = array(
        array(
            'cityName' => 'beijing',
            'cityId' => 1,
        ),
        array(
            'cityName' => 'shanghai',
            'cityId' => 2,
        ),
    );

    echo json_encode($city);
}

else if(isset($_GET['id']) && $_GET['id'] == 2)
{
    $city = array(
        array(
            'cityName' => 'dongjing',
            'cityId' => 1,
        ),
        array(
            'cityName' => 'daban',
            'cityId' => 2,
        ),
    );

    echo json_encode($city);
}

else if(isset($_GET['id']) && $_GET['id'] == 3)
{
    $city = array(
        array(
            'cityName' => '首尔',
            'cityId' => 1,
        ),
        array(
            'cityName' => 'ahha',
            'cityId' => 2,
        ),
    );

    echo json_encode($city);
}

else
{
    echo json_encode($country);
}

```