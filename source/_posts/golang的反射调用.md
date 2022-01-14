---
title: golang反射调用
date: 2019-08-30 10:18:54
desc: golang 反射调用方法实现
tags: golang, reflect
---

通过反射， 简单的调用实例的方法。虽然这个反射性能不是很好， 但有时候用起来往往还挺爽的。

<!-- more -->

暴力直接上代码

```golang

package main

import (
	"log"
	"reflect"
)

type Logic struct{}
type Query struct {
	Id int
}

type Person struct {
	Id   int
	Name string
}

func (l *Logic) Echo(q *Query) (*Person, error) {
	return &Person{
		Id:   1,
		Name: "luowen",
	}, nil
}

func main() {

	l := &Logic{}
	_, ok := reflect.TypeOf(l).MethodByName("Echo")
	if !ok { // check method 'Echo' exists
		log.Fatalf("instance method %s not exists!", "Echo")
	}
	// sm.Type.NumOut() // number of method return.
    // sm.Type.NumIn() // number of method arguments need.
	resp := reflect.ValueOf(l).MethodByName("Echo").Call([]reflect.Value{ // reflect invoke it
		reflect.ValueOf(&Query{
			Id: 10,
		}),
	})
	if !resp[0].IsNil() { // get result
		p := resp[0].Interface().(*Person)
		log.Printf("result[0]: %d, %s\n", p.Id, p.Name)
	}
	if !resp[1].IsNil() { // get result
		e := resp[1].Interface().(error)
		log.Printf("result[1]: %v\n", e)
	}
}

```
