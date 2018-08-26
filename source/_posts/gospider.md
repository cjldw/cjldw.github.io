---
title: Golang 协程下载网站图片资源
date: 2017-05-12 22:18:38
desc:
tags:
---

开始之前, 今天5.12 为我们汶川大地震逝去的人默哀, 愿他们在天堂安好.
go语言的协程实战, 爬去网页后, 匹配对应网页上的图片, 对应每个图片开启一个协程去下载. 之前有做对比下载`http://www.zhuangbi.info/page=1~72`页所有图片, 用php费时2分50多秒, 使用goroutine实现, 费时间7s多点, 这个差距还是挺可观的.

<!-- more -->

## 项目地址

* `https://github.com/vvotm/gospider`

## 使用

1. 可克隆代码go build 安装
2. 或直接下载window版的main.exe直接使用

3. 直接运行`main.exe`查看帮助

    ```bash
        Uses: main.exe {url} [savepath] [totalPage]

        url: (must) etc. http://www.zhuangbi.info
        savepath: (option) the path where you want save images
        totalPage: (options) only url has page placeholder. for example:

            main.exe http://www.zhuangbi.info/?page={page} ./tmp 10

            command will download all images from http://www.zhuangbi.info/?page=1 to http://www.zhuangbi.info/?page=10
    ```

## 实现原理

* 通过命令行的url, 匹配下载对应的网页内容, 做正则匹配到网页的img标签, 获取到图片地址, 在对应开启协程去下载图片保存.

* 主要代码实现

    ```bash
            var wg sync.WaitGroup

            func Run(url string, path string, page string) {
                regex := regexp.MustCompile("\\{page\\}")
                newPage, err := strconv.Atoi(page)
                except.ErrorHandler(err)
                if newPage > 0 && regex.FindAllString(url, -1) != nil {
                    for i := 1; i <= newPage ; i++  {
                        wg.Add(1)
                        pageIndex := strconv.Itoa(i)
                        newUrl := strings.Replace(url, "{page}", pageIndex, -1)
                        go fetch(newUrl, path)
                    }
                } else {
                    wg.Add(1)
                    go fetch(url, path)
                }
                wg.Wait()
            }

            func fetch(url, path string)  {

                imgname := strconv.Itoa(rand.Int())

                lastSlashIndex := strings.LastIndex(url, "/")
                if lastSlashIndex != -1 {
                    imgname = url[lastSlashIndex+1:]
                }

                lastQutesIndex := strings.LastIndex(imgname, "?")
                if lastQutesIndex != -1 {
                    imgname = imgname[:lastQutesIndex]
                }
                _, err := os.Stat(path)
                if err != nil && os.IsNotExist(err) {
                    os.Mkdir(path, 755)
                }

                content := utils.FetchUrl(url)
                types := []string{"jpg", "png", "gif"}
                resultSet := typeMatch.GetImgSlice(content, -1, types)

                for _, url := range(resultSet)  {
                    wg.Add(1)
                    go downloadImg(url, path)
                }
                wg.Done()
            }

            func downloadImg(url, path string)  {
                fmt.Println("download image [" + url + "]")
                imgStr := utils.FetchUrl(url)
                lastSlashIndex := strings.LastIndex(url, "/")
                filename := strings.Trim(path, "/") + "/" + url[lastSlashIndex+1:]
                fmt.Println("file:" + filename)
                utils.WriteFile(imgStr, filename)
                wg.Done()
            }
    ```

### 后期优化

* 参数优化
* 正则匹配
* panic优化处理
* 使用channel限制协程数, 目前没有限制
