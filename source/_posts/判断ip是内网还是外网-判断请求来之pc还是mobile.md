---
title: '判断ip是内网还是外网, 判断请求来之pc还是mobile'
tags: []
date: 2015-09-10 15:01:00
---

如何是php判断是内网访问, 还是外网访问呢? 废话不多说, 贴码。

<!-- more -->

### 判断内网ip ###

代码如下

```php
    function is_internal_id($ip_address)
    {

        $get_ip_number = function($ip)
        {
            $ip_segment = explode('.', $ip);
            if(!is_array($ip_segment) || count($ip_segment) != 4)
                return -1;

            $ip_num = $ip_segment[0] * 256 * 256 * 256 + $ip_segment[1] * 256 * 256 + $ip_segment[2] * 256 + $ip_segment[3];
            return $ip_num;
        };

        $process_ip = $get_ip_number($ip_address);

        /**
         * 私有IP：A类  10.0.0.0    -10.255.255.255
         *       B类  172.16.0.0  -172.31.255.255
         *       C类  192.168.0.0 -192.168.255.255
         *       D类   127.0.0.0   -127.255.255.255(环回地址)
         */
        $a_begin = $get_ip_number("10.0.0.0");
        $a_end = $get_ip_number("10.255.255.255");
        if($process_ip >= $a_begin && $process_ip <= $a_end)
            return true;

        $b_begin = $get_ip_number("172.16.0.0");
        $b_end = $get_ip_number("172.31.255.255");
        if($process_ip >= $b_begin && $process_ip <= $b_end)
            return true;

        $c_begin = $get_ip_number("192.168.0.0");
        $c_end = $get_ip_number("192.168.255.255");
        if($process_ip >= $c_begin && $process_ip <= $c_end)
            return true;

        $d_begin = $get_ip_number("127.0.0.0");
        $d_end = $get_ip_number("127.255.255.255");
        if($process_ip >= $d_begin && $process_ip <= $d_end)
            return true;

        return false;
    }
    ```
### PHP 自带判断私有ip 方法 ###

代码如下

```php
function is_private_ip($ip) { 
return !filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE); 
} 
```
### 判断Mobile,还是pc ###

代码如下

```php
function ismobile() {
$is_mobile = '0';

if(preg_match('/(android|up.browser|up.link|mmp|symbian|smartphone|midp|wap|phone)/i', strtolower($_SERVER['HTTP_USER_AGENT']))) {
    $is_mobile=1;
}

if((strpos(strtolower($_SERVER['HTTP_ACCEPT']),'application/vnd.wap.xhtml+xml')>0) or ((isset($_SERVER['HTTP_X_WAP_PROFILE']) or isset($_SERVER['HTTP_PROFILE'])))) {
    $is_mobile=1;
}

$mobile_ua = strtolower(substr($_SERVER['HTTP_USER_AGENT'],0,4));
$mobile_agents = array('w3c ','acs-','alav','alca','amoi','andr','audi','avan','benq','bird','blac','blaz','brew','cell','cldc','cmd-','dang','doco','eric','hipt','inno','ipaq','java','jigs','kddi','keji','leno','lg-c','lg-d','lg-g','lge-','maui','maxo','midp','mits','mmef','mobi','mot-','moto','mwbp','nec-','newt','noki','oper','palm','pana','pant','phil','play','port','prox','qwap','sage','sams','sany','sch-','sec-','send','seri','sgh-','shar','sie-','siem','smal','smar','sony','sph-','symb','t-mo','teli','tim-','tosh','tsm-','upg1','upsi','vk-v','voda','wap-','wapa','wapi','wapp','wapr','webc','winw','winw','xda','xda-');

if(in_array($mobile_ua,$mobile_agents)) {
    $is_mobile=1;
}

if (isset($_SERVER['ALL_HTTP'])) {
    if (strpos(strtolower($_SERVER['ALL_HTTP']),'OperaMini')>0) {
        $is_mobile=1;
    }
}

if (strpos(strtolower($_SERVER['HTTP_USER_AGENT']),'windows')>0) {
    $is_mobile=0;
}

return $is_mobile;
}

var_dump(ismobile());
```
