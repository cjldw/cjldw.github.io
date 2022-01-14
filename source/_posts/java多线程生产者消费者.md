---
title: java多线程生产者消费者
date: 2016-11-19 21:50:30
desc: java多线程之间的通信
tags:
---

最近看java, 感觉多java多线程通信有点意思.
引入: 在生活中, 我们很多这样的场景, 早餐店, 早上卖包子, 当外面包子卖完了, 会通知厨房做包子, 厨房做好包子, 通知外面人出来拿包子.

<!-- more -->

### 代码说明

1. 生产者 `Produce.java`

2. 消费者 `Consume.java`

3. 包子 `Item.java`

4. 包子摊 `DataSource.java`

5. 代码测试入口 `TestEntrance.java`


#### 代码详情

1. 生产者, 实现`Runnable`接口, 提供多线程调用支持

    ```java
    package thread.test;


    /**
     * @description:
     * @author: luowen<bigpao.luo@gmail.com>
     * @time: 11/19/2016.
     */
    public class Produce implements Runnable{

        DataSouce dataSouce;

        public Produce(DataSouce dataSouce) {
            this.dataSouce = dataSouce;
        }

        @Override
        public void run() {
            while (true) {
                dataSouce.produceItem();
            }
        }
    }

    ```
2. 消费者, 实现`Runnable`接口, 提供多线程调用支持

    ```java
    package thread.test;

    /**
     * @description:
     * @author: luowen<bigpao.luo@gmail.com>
     * @time: 11/19/2016.
     */
    public class Comsume implements Runnable{

        protected DataSouce dataSouce = null;

        public Comsume(DataSouce dataSouce) {
            this.dataSouce = dataSouce;
        }

        @Override
        public void run() {

            while (true) {
                dataSouce.comsumeItem();
            }

        }
    }
        
    ```

3. 包子实体

    ```java
    package thread.test;

    /**
     * @description:
     * @author: luowen<bigpao.luo@gmail.com>
     * @time: 11/19/2016.
     */
    public class Item {

        private String name;
        private float price;

        private String color;

        public Item() {
        }

        public Item(String name, float price, String color) {
            this.name = name;
            this.price = price;
            this.color = color;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public float getPrice() {
            return price;
        }

        public void setPrice(float price) {
            this.price = price;
        }

        public String getColor() {
            return color;
        }

        public void setColor(String color) {
            this.color = color;
        }
    }
            
    ```

4. 包子摊, 主要暴露获取, 和消费包子方法

    ```java
    package thread.test;

    import java.util.ArrayList;

    /**
     * @description:
     * @author: luowen<bigpao.luo@gmail.com>
     * @time: 11/19/2016.
     */
    public class DataSouce {

        protected ArrayList<Item> items = new ArrayList<Item>();


        public synchronized void comsumeItem() {
            if (items.isEmpty()) {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } else {

                System.out.println(Thread.currentThread().getName() + "--------------------------------consume-----------------------------------");
                items.clear();
                this.notifyAll();
            }

        }

        public synchronized void produceItem() {

            if(!items.isEmpty()) {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } else  {
                System.out.println(Thread.currentThread().getName() + "--------------------------------produce-----------------------------------");
                for (int i = 0; i < 10; i++) {
                    items.add(new Item("luowen"+i, 0.003f + i, "red" + i));
                }
                this.notifyAll();
            }
        }

    }
            
    ```
5. java主入口文件, 开启了3个消费者, 3个生产者工作

    ```java
    package thread.test;

    /**
     * @description:
     * @author: luowen<bigpao.luo@gmail.com>
     * @time: 11/19/2016.
     */
    public class TestEntrance {

        public static void main(String[] args)
        {
            DataSouce dataSouce = new DataSouce();
            Comsume comsume1 = new Comsume(dataSouce);
            Comsume comsume2 = new Comsume(dataSouce);
            Comsume comsume3 = new Comsume(dataSouce);
            Produce produce1 = new Produce(dataSouce);
            Produce produce2 = new Produce(dataSouce);
            Produce produce3 = new Produce(dataSouce);

            new Thread(comsume1).start();
            new Thread(comsume2).start();
            new Thread(comsume3).start();
            new Thread(produce1).start();
            new Thread(produce2).start();
            new Thread(produce3).start();

        }

    }
    ```

6. 执行结果

   ![result](/images/java/multiple-thread.png)


