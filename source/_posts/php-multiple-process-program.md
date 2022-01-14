---
title: 使用swoole在php中实现多进程编程
date: 2017-08-25 20:57:26
desc: swoole支持php多进程编程
tags:  swoole, multiple process
---

php默认是单进程运行, 这几天php写了一个数据处理脚本, 发现很吃力, 50w的数据, 跑了1个半小时, 好累!
于是想到了swoole的swoole_process来实现提提速度。也发现了一些问题, 记录下。

<!-- more --->

1. swoole创建的进程一定要调用`swoole_process::wait()`来等待回收, 不然就成为了僵死进程了。
2. `swoole_process(callback, false, false)` 中的`callback`中的**`$this`**就是完全fork出来的父对象, 里面
的变量, 在子进程的独立空间中, 不能数据共享. 如果要父子进程数据共享, 可以使用`swoole_table`创建全局共享 内存实现数据共享。
3. 使用多进程操作数据库的时候, 注意, 不能多进程更新数据库中的同一行数据, 否则(innodb)出现死锁, (myisam) 数据会出现混乱。

下面是事例代码

```php

class MultipleProcessDemo 
{

    public $pidList = []; // 保存子进程ID号

    const MAX_PROCESS_NUM = 10; // 默认创建10个进程处理业务

    public $mpid = 0;

    public function run() 
    {
        $this->processStart();
        for($i = 0; $i < self::MAX_PROCESS_NUM; $i++) {
            $this->subprocess();
        }
        $this->processWait();
    }

    /**
     * 设置主进程名称
     */
    protected function processStart()
    {
        swoole_set_process_name(sprintf('hello-multiple-process:%s', 'master'));
        $this->mpid = posix_getpid();
    }

    /**
     * 创建子进程
     */
    public function subprocess()
    {
        $mpid = $this->mpid;
        $subprocess = new \swoole_process(function($work) use ($mpid) {
            $subpid = uniqid();
            swoole_set_process_name(sprintf('hello-multiple-process:%s', $subpid));
            if(!swoole_process::kill($mpid, 0)){ // 校验master进程是否OK, 
                $work->exit();
            }
            echo sprintf("subprocess is run and pid is 【%s】", $subpid);
            echo PHP_EOL;
            sleep(10);
        }, false, false)
        $pid = $subprocess->start();
        $this->pidList[$pid] = $pid;
    }

    /**
     * 等待进程处理完, 回收进程, 不然就成为了僵死进程了
     */
    public function processWait()
    {
        while(true) {
            if(count($this->pidList)) {
                $endprocess = \swoole_process::wait();
                if($endprocess === false) { // 所有进程处理完业务, 跳出
                    break;
                }
                if(isset($endprocess['pid'])) { // 进程结束, 取出pidList中的数量
                    ecoh sprintf("process 【%s】 end ok", $endprocess['pid']);
                    echo PHP_EOL;
                    unset($this->pidList[$endprocess['pid']]);
                }
            } else { // 所有进程处理完业务, 跳出
                break;
            }
        }
    }

}

$multipleTask = new MultipleProcessDemo();
$multipleTask->run();

```

有了多进程编程也不会很难了， PHP5.5.x在处理swoole可能会有BUG：**libgc xxxx php free() xxx**, 若出现次bug 升级php吧, 不好改！
