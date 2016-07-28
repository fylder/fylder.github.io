---
published: true
layout: post
title: RxJava在Android上打开方式
category: android
tags: 
  - RxJava
time: 2016.3.28 21:47:02
excerpt: 一个类似于观察者模式的良好体现，扔掉AsyncTask，淡化Handler
---

> 有些事情看上去抽象得不知所以然，经历过了才有恍然如是的感慨

[RxJava](http://reactivex.io){:target="_blank"}从去年开始，就开始流行起来，那时还不太在意，而看到那些技术周刊总是提及后，也去了解一番，在经常使用CallBack很容易就了解RxJava是怎么回事，说太多的原理也是抽象，简单粗暴的弄出结果了再说。

**一: 定义想干什么**

```java
Observable<String> observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
    //业务逻辑，next()触发回调
    
    }).subscribeOn(Schedulers.newThread()).observeOn(AndroidSchedulers.mainThread());
```
**二: 定义怎么处理**

```java
Subscriber<String> subscriber=new Subscriber<String>() {
     @Override
     public void onCompleted() {
     
     }
     @Override
     public void onError(Throwable e) {
     
     }
     @Override
     public void onNext(String s) {
     //想要的结果
     
     };
```
**三: 执行**

```java
observable.subscribe(subscriber);
```
------

***说明***

**ps:**在创建Observable<T>时，必须像建造者模式的方式，否则后续的配置不会生效(不确定)。

给Observable线程上的通常配置Schedulers

> subscribeOn(Schedulers.newThread())

让Call()在一个新的Thread上操作,用来处理会阻塞线程的耗时操作或是复杂的业务逻辑

> observeOn(AndroidSchedulers.mainThread())

让Subscriber在一个UI Thread上操作,也就是在返回值时的回调

<br>

* 多个subscribeOn()，只有第一个subscribeOn()有效，且对observeOn()之前的Observable影响
* observeOn()会对后续所有Observable有效，可以设置多个observeOn()
* observeOn()能控制下面代码的线程，subscribeOn()只能在第一次设置的时候起作用

<br>

##### **具体的操作流程**

在Call()里执行以下方法

* `onNext()`用于传参数,过程调用
* `onCompleted()`和`onError()`都会中途打断跳出,从而结束整个操作,onNext能不能继续执行就看用不用这两个方法,当然在正常的情况下最后一定会调用onCompleted()

```java
subscriber.onNext(s);   //传入参数

subscriber.onError(new Throwable("错了就是错了"));  //异常结束
    
subscriber.onCompleted();   //正常结束
```
    


Subscriber在反馈的处理流程
![subscriber-flow](http://fylder.github.io/img/pic/rxjava-subscriber.png)

RxJava当然也没那么简单就完了，一个Observable还有很多的操作符自由搭配，这里就不多说。

