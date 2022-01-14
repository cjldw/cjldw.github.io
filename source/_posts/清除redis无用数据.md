---
title: 清除 redis 无用数据
date: 2019-05-07 22:06:58
desc: 清除 redis 数据无用数据
tags: bash, redis
---

近期发现redis数据暴增， 检查发现有很多key下入的时候，没有设置过期时间， 而且key的名称是拼接上了时间， 和随机数。这个清理这些key带来了一点麻烦。

<!-- more -->

最基本的原理就是， 把key找出来， 然后再删除。

> 之前有写一片文章说明 直接使用 redis-cli -n {数据库} keys '{glob 匹配, 参考官方文档有惊喜}' | xargs -n 1 redis-cli del 但是， 这个方案再数据量大的时候， 会阻塞线上业务.

### 找出 key 的名称

0. redis-server 执行 bgsave, 把redis数据持久化到硬盘上。
1. 找一个好用的工具**rdb** 一个python写的 dump.rdb 分析工具.
3. 执行 `rdb --command justkeys --key ".*"  dump.rdb`
4. 找出的key需要去重下, rdb这个工具解析出来的key会有重复的


### 构造redis通用数据

0. 发现到处来的keys文件贼大, 需要切割它 `split -a 6 -l 10000 dump.keys`

1. 使用redis的pipleline 来批量处理它, bash 开启多个进程来处理它。也不能开太多。

```bash
#!/bin/bash
dirpath=/path/to/split/dir
num=0
for file in $(ls $dirpath)
do
  #str=$(cat $dirpath$file | sed "s/^/DEL /")
  absfile=$dirpath$file
  echo  "del file ${absfile}"
  (echo -e $(cat $absfile | xargs -n 1 -I {} echo "DEL {}\r\n" | tr -d "\n") | nc redis_host redis_port &)
  num=$num+1
  if [[ $num -eq 100 ]];then
    sleep 100
    num=0
  fi
done
```

2. 发现redis的内存释放飞流直下三千尺。
