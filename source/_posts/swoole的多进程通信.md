---
title: swoole的多进程通信
date: 2018-08-04 12:21:35
desc: swoole开启多个进程实现业务
tags: swoole, multiple process.
---

使用瓦力发布平台发布代码, 发现丫的, 越来越慢, 越来越慢，已经到了忍不了了的节奏了. 看了下原理, 它是单进程的把所有的代码`scp`(没有使用ansible)到对应的机器上。
自然想到的就是开启多个进程去出力吧，这样就有个问题，多个进程如何通信呢。

<!-- more -->

测试环境:
1. linux (centos6.5)
2. php(5.6.16)
3. swoole(1.7.22-rc1)

先写简单的测试demo

```php

use swoole_process as Process;

class MultipleProcess
{
    protected $worknum = 2; // 默认2个进程

    protected $processList = [];

    public function __construct($worknum = 2)
    {
        $this->worknum = $worknum;
    }

    public function longtimeTask()
    {
        sleep(3);
        $rand = rand(1, 100);
        if ($rand % 5 == 0) { // 模拟随机一个进程处理超时
            throw new \Exception('timeout error!', $rand);
        }
        return true;
    }

    public function run()
    {
        $self = $this;
        for($i = 0; $i < $this->worknum; $i++) {
            $process = new Process(function(Process $worker) use ($self) {
                try {
                    $result = $self->longtimeTask();
                    $worker->push('{"code": 0, "msg":"success"}'); // 将出力的结果放到队列中
                    $worker->exit(0);
                } catch(\Exception $e) {
                    $worker->push(sprintf('{"code": %d, "msg":"%s"}',$e->getCode(), $e->getMessage()));
                    $worker->exit(1);
                }
            });
            $process->name("php-demo-$i");
            $process->useQueue(); // 使用队列
            $pid = $process->start();
            $this->processList[$pid] = $process;
        }

        foreach($this->processList as $pid => $process) {
            Process::wait(); // 回收进程
            $result = $process->pop();
            var_dump("process: $pid result: $result");
            $resp = json_decode($result);
            if ($resp->code) {
                $process->freeQueue();
                throw new \Exception("000");
            }
        }
    }
}
$multiple = new MultipleProcess(10);
$multiple->run();
```

再来一个例子

```php

use swoole_process as SwooleProcess;

/**
 * Processer interface
 *
 * Interface Processer
 */
interface Processer
{
    public function run(SwooleProcess $woker);
}
class ProcessManager
{

    /**
     * process list maps
     *
     * KEY => [pid => swoole_process]
     * @var array
     */
    protected static $processMap = [];

    /**
     * build success data
     *
     * @param array $data
     * @param string $message
     * @param int $code
     * @return string
     */
    protected static function success($data = [], $message = "success", $code = 0)
    {
        return json_encode(['code' => $code, 'message' => $message, 'data' => $data]);
    }

    /**
     * build error data
     *
     * @param int $code
     * @param string $message
     * @param array $data
     * @return string
     */
    protected static function error($code = 1, $message = "error", $data = [])
    {
        return self::success($data, $message, $code);
    }

    /**
     * submit process task job
     *
     * @param $KEY
     * @param Processer $task
     */
    public static function submitTask($KEY, Processer $task)
    {
        $process = new SwooleProcess(function (SwooleProcess $worker) use ($task) {
            try {
                $respData = $task->run($worker);
                $worker->push(static::success($respData));
                $worker->exit(0);
            } catch (\Exception $e) {
                $worker->push(static::error($e->getCode(), $e->getMessage()));
                $worker->exit($e->getCode());
            }
        });
        $process->useQueue(crc32($KEY));
        $pid = $process->start();
        static::$processMap[$KEY] = array_merge(self::$processMap[$KEY], [
            $pid => $process
        ]);
    }

    /**
     * wait $KEY process list response
     *
     * @param $KEY
     * @param bool $interrupt
     * @return array
     * @throws \Exception
     */
    public static function waitResp($KEY, $interrupt = false)
    {
        $respData = [];
        if (!isset(static::$processMap[$KEY])) {
            throw new \Exception(sprintf('对应进程组: %s, 不存在!', $KEY));
        }
        $processList = self::$processMap[$KEY];
        foreach ($processList as $pid => $process) {
            SwooleProcess::wait();
            $result = json_decode($process->pop(), true);
            if ($result['code'] && $interrupt) {
                $process->freeQueue();
                throw new \Exception($result['message'], $result['code']);
            }
            $respData[] = $result;
        }
        unset(self::$processMap[$KEY]); // 清空 processList
        return $respData;
    }

}

class TestJob implements Processer
{

    public function run(SwooleProcess $woker)
    {
        sleep(5);
        $random = rand(1, 100);
        if ($random % 5 == 0) {
            throw new \Exception('timeout error ! code: ' . $random, $random);
        }
        return $random;
    }
}

ProcessManager::submitTask("a", new TestJob());
ProcessManager::submitTask("a", new TestJob());
ProcessManager::submitTask("a", new TestJob());
ProcessManager::submitTask("a", new TestJob());
ProcessManager::submitTask("a", new TestJob());
ProcessManager::submitTask("a", new TestJob());
ProcessManager::submitTask("a", new TestJob());

$result = ProcessManager::waitResp("a");
var_dump($result);
```

**PS: 亲测是可以的, 😄**

