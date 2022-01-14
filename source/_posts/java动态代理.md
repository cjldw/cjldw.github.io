---
title: java动态代理
date: 2016-11-28 23:13:20
desc: java常见的代理实现
tags:
---

代理模式, 可以只提供方法, 实际操作交给被代理对象去完成, 好处是可以在不修改原来代码基础上新增功能, 也可以优化, 当然还有等等...
简单说下java的三中代理实现

<!-- more -->


### 静态代理

1. JDK8的新特性, 接口中可以有方法体, 主要实现一个print方法

    ```java
       package main.staticproxy;
        public interface JDK8Tools {

            public default void print(Object msg) {
                System.out.println(msg);
            }
        }
    ```

2. 学习接口, 定义了三个方法

    ```java
        package main.staticproxy;
        public interface Study {

            public abstract void studyJava();

            public abstract void studyPhp();

            public abstract void studyJavaScript();

        }
    ```


3. "我"实现了学习接口

    ```java
        package main.staticproxy;
        public class StudentOfLuowen implements JDK8Tools, Study {
            @Override
            public void studyJava() {
                print("I love Java");
            }

            @Override
            public void studyPhp() {
                print("I'am paper");
            }

            @Override
            public void studyJavaScript() {
                print("JavaScript is very Hot!!!");
            }
        }
    ```


4. 通过代理类, 代理我学习, 外面看是"代理类"在学习, 其实是"我"在学习

    ```java
        package main.staticproxy;
        public class StudentProxy implements JDK8Tools, Study{
            private StudentOfLuowen studentOfLuowen = null;

            public StudentProxy(StudentOfLuowen studentOfLuowen) {
                this.studentOfLuowen = studentOfLuowen;
            }

            @Override
            public void studyJava() {
                print("proxy luowen Study JavaStart");
                studentOfLuowen.studyJava();
                print("proxy luowen Study JavaEnd");

            }

            @Override
            public void studyPhp() {
                print("proxy luowen Study PHP start");
                studentOfLuowen.studyPhp();
                print("proxy luowen Study PHP end");
            }

            @Override
            public void studyJavaScript() {
                print("proxy luowen Study JavaScript start");
                studentOfLuowen.studyJavaScript();
                print("proxy luowen Study JavaScript end");
            }
        }
        
    ```


5. 测试类

    ```java
       package main.staticproxy;
        public class StudentProxyTest {
            public static void main(String[] args) {
                StudentProxy studentProxy = new StudentProxy(new StudentOfLuowen());

                studentProxy.studyJavaScript();
                studentProxy.studyJava();
                studentProxy.studyPhp();
            }
        } 
    ```

### JDK5 反射包中的Proxy代理对象(被代理对象一定要有接口实现, 这是硬伤)

1. 同上JDK8Tools接口和Study接口

2. 同上"我(StudentOfLuowen)"对象实现了 JDK8Tools, Study接口

3. JDK5 Proxy实现

    ```java
        package main.jdkproxy;

        import java.lang.reflect.InvocationHandler;
        import java.lang.reflect.Method;
        import java.lang.reflect.Proxy;

        public class Jdk5Proxy<T> implements InvocationHandler{

            private T target = null;

            public T bind (T target) {
                this.target = target;
                return (T) Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
            }


            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println(method.getName() + ": invoke start!!");
                Object result = method.invoke(target, args);
                System.out.println(method.getName() + ": invoke end!!");
                return result;
            }

        }
        
    ```

4. JDK5 Proxy 测试

    ```java
        package main.jdkproxy;

        import main.StudentOfLuowen;
        import main.Study;

        public class Jdk5ProxyTest {
            public static void main(String[] args) {
                Study proxy = new Jdk5Proxy<Study>().bind(new StudentOfLuowen());

                proxy.studyJava();
                proxy.studyJavaScript();
                proxy.studyPhp();
            }
        }
        
    ```

### cglib动态代理 (完美解决JDK5Proxy代理的问题)

1. 独立的学生, 没有实现任何接口

    ```java
        package main.cglibproxy;
        public class IndependentStudent {
            public void studyMath() {
                System.out.println("学习数学");
            }

            public void studyEnglish() {
                System.out.println("学习英语");
            }
        }
    ```

2. CGlib代理实现

    ```java
        package main.cglibproxy;

        import net.sf.cglib.proxy.Enhancer;
        import net.sf.cglib.proxy.MethodInterceptor;
        import net.sf.cglib.proxy.MethodProxy;
        import java.lang.reflect.Method;
        public class CGlibProxy<T> implements MethodInterceptor{

            private T target = null;

            public T bind(T target) {
                this.target = target;
                return (T) Enhancer.create(target.getClass(), this);

                // another implement method
                // Enhancer enhancer = new Enhancer();//该类用于生成代理对象
                // enhancer.setSuperclass(this.target.getClass());//设置父类
                // enhancer.setCallback(this);//设置回调用对象为本身
                // return enhancer.create();
            }

            @Override
            public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                /* invokeSuper() */
                return methodProxy.invokeSuper(o, objects);
            }
        }
        
    ```

3. 测试代码

    ```java
        package main.cglibproxy;

        public class CGlibProxyTest {

            public static void main(String[] args) {
                IndependentStudent independentStudent = new CGlibProxy<IndependentStudent>().bind(new IndependentStudent());
                independentStudent.studyEnglish();
                independentStudent.studyMath();
            }
        }
        
    ```



PS: 欢迎拍砖

