---
title: PHP5.4 Trait 使用
tags: []
date: 2015-12-25 15:36:00
---

今天接触了laravel框架, 发现里面使用了很多的**trait**, 引发了我写这博文的冲动。

<!-- more -->

### Trait 是个什么鬼 ###

* PHP Trait 是一个抽象类, 不能实例化, 简单理解就是将trait的代码复制, 粘贴到你使用trait的类中. 和接口对比, 还是有区别的有这么个问题。

    ```php
        <?php
        class DbReader extends Mysqli
        {
        }

        class FileReader extends SplFileObject
        {
        }
    ```

* DbReader, FileReader 都需要些公共方法, 假设都设计为单列模式, 两个类都需要getInstance方法, php不支持多继承, 怎么办?  

    ```php
        <?php
        interface Singleton
        {
            public static function getInstance();
        }

        class DbReader extends Mysqli implements Singleton
        {
            public static function getInstance(){ return "instanceOfDbReader";}
        }

        class FileReader extends SplFileObject implements Singleton
        {
            public static function getInstance(){ return "instanceOfFileReader";}
        }
    ```

    是你想要的吗? 显然是NO  

* SO PHP特性 Trait

    ```php
        <?php
        trait Singleton
        {
            private static $instance;

            public static function getInstance() {
                if (!(self::$instance instanceof self)) {
                    self::$instance = new self;
                }
                return self::$instance;
            }
        }

        class DbReader extends ArrayObject
        {
            use Singleton;
        }

        class  FileReader
        {
            use Singleton;
        }
        $a = DbReader::getInstance();
        $b = FileReader::getInstance();
        var_dump($a);  //object(DbReader)
        var_dump($b);  //object(FileReader)
    ```

    是不是爽了!!!

### 使用多个Trait ###

* PHP Trait 是可以多个使用的

    ```php
        <?php
        trait Hello
        {
            function sayHello() {
                echo "Hello";
            }
        }

        trait World
        {
            function sayWorld() {
                echo "World";
            }
        }

        class MyWorld
        {
            use Hello, World;
        }

        $world = new MyWorld();
        echo $world->sayHello() . " " . $world->sayWorld(); //Hello World
    ```

    是不是很方便!!!  

### Trait 组装 Trait ###

* 多个Trait可以组装成一个Trait  

    ```php
        <?php
        trait Hello
        {
            function sayHello() {
                echo "Hello";
            }
        }

        trait World
        {
            function sayWorld() {
                echo "World";
            }
        }
        trait HelloWorld
        {
            use Hello, World;
        }

        class MyWorld
        {
            use HelloWorld;
        }

        $world = new MyWorld();
        echo $world->sayHello() . " " . $world->sayWorld(); //Hello World
    ```

### Trait 优先级 ###

* Trait中的方法重写继承父类的方法。

* 当前类中方法复写trait中的方法。

    ```php
        <?php
        trait Hello
        {
            function sayHello() {
                return "Hello";
            }

            function sayWorld() {
                return "Trait World";
            }

            function sayHelloWorld() {
                echo $this->sayHello() . " " . $this->sayWorld();
            }

            function sayBaseWorld() {
                echo $this->sayHello() . " " . parent::sayWorld();
            }
        }

        class Base
        {
            function sayWorld(){
                return "Base World";
            }
        }

        class HelloWorld extends Base
        {
            use Hello;
            function sayWorld() {
                return "World";
            }
        }

        $h =  new HelloWorld();
        $h->sayHelloWorld(); // Hello World
        $h->sayBaseWorld(); // Hello Base World
    ```

* 解决Trait冲突

    ```php
        <?php
        trait Game
        {
            function play() {
                echo "Playing a game";
            }
        }

        trait Music
        {
            function play() {
                echo "Playing music";
            }
        }

        class Player
        {
            use Game, Music;
        }

        $player = new Player();
        $player->play(); // Fatal Error

        // resolve one
        class Player
        {
            use Game, Music {
                Music::play insteadof Game;
            }
        }

        $player = new Player();
        $player->play(); //Playing music

        // resolve two
        class Player
        {
            use Game, Music {
                Game::play as gamePlay;
                Music::play insteadof Game;
            }
        }

        $player = new Player();
        $player->play(); //Playing music
        $player->gamePlay(); //Playing a game
    ```

    使用PHP Reflection类 `ReflectionClass::getTraits()` 获取类使用的traits. `ReflectionClass::getTraitNames()` 获取类中trait名称 `ReflectionClass::isTrait()` 此类是否是trait

### 其他特性 ###

* trait中的 `protected, private` 可以随便访问。

    ```php
        <?php
        trait Message
        {
            private $message;

            function alert() {
                $this->define();
                echo $this->message;
            }
            abstract function define();
        }

        class Messenger
        {
            use Message;
            function define() {
                $this->message = "Custom Message";
            }
        }

        $messenger = new Messenger;
        $messenger->alert(); //Custom Message
    ```

PS: [参考Shameer C's Blog](http://www.sitepoint.com/using-traits-in-php-5-4/)
