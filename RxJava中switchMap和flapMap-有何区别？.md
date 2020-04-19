---
title: RxJava中switchMap和flapMap 有何区别？
date: 2020-03-20 00:40:23
tags: 
- RxJava
categories:
- RxJava
---

flapMap 和 switchMap 都是 RxJava 中的转换操作符，都可以将上游的输入转换为一个数据源（比如 Observable）输出给下游。

<!--more-->

## switchMap 的使用场景

- switchMap 的使用场景：监听用户在输入框输入的内容，定时触发搜索并展示结果

  1. 创建一个 `PublishSubject<String>`

  2. 输入框中的文字变化的时候，调用 onNext 方法发布变更

  3. 使用 debounce 去抖动，避免频繁触发

  4. 使用 switchMap 切换到搜索网络请求

  5. subscribe 处理搜索结果（也可以加一个 map 操作），并更新 UI

     ```java
     
     mExecSearchDisposable = searchObservable
         //防抖
         .debounce(100, TimeUnit.MILLISECONDS, BearSchedulers.mainThread())
         //有变化才往下游发送
         .distinctUntilChanged()
         .toFlowable(BackpressureStrategy.LATEST)
         //对上游的输入（用户输入的搜索关键字）进行查询；如果有新的输入，旧的输入还未拿到查询结果，则旧的查询结果会直接丢弃。
         .switchMap { s ->
     			//如果输入为空串，则展示空页面
           if (s.isEmpty()) {
                 Flowable.just(Notification.createOnError(EmptySearchException()))
             } else {
               //显示 loading
                 view.loading()
                   //构建搜索参数
                 val params = buildSearchParams(s)
                   //有网络，执行网络查询，无网络，执行本地数据库查询
                 val searchFlowable = if (connectionService.networkState.isConnected) {
                     mSearchDataRepository.searchFromNet(params)
                 } else {
                     mSearchDataRepository.searchFromDb(params)
                 }
                 searchFlowable.observeOn(BearSchedulers.computation())
                         .map {
                             val resultList = resolveResponse(it)
                             Notification.createOnNext(resultList)
                         }
                         .onErrorReturn { Notification.createOnError(it) }
             }
         }
         .observeOn(BearSchedulers.mainThread())
         .subscribe { notification ->
             if (notification.isOnNext) {
               //刷新 UI
                 view.refreshUi(notification.value, false)
             } else if (notification.isOnError) {
                 if (notification.error is EmptySearchException) {
                     view.refreshUi(Collections.emptyList(), true)
                 } else {
                     view.onError(notification.error?.message)
                 }
             }
         }
     ```

     

- 如果使用 flatMap，则搜索结果有可能是过时的，因为客户端拿到的搜索结果 顺序有可能是乱序的。（网络请求的时延不可控）

  - 为了解决该问题，可以通过 switchMap，因为switchMap 保证新的 Observable 提供的时候， 旧的 Observable 会被取消掉。


## 大理石图

  - switchMap ；因为在发出第二个绿色方块之前，上游输入一个 深蓝色珠子，因此第二个绿色方块之前被丢弃

    ![img](https://img.mubu.com/document_image/b9064af5-a131-423b-95c8-9e6653d3e42c-4072084.jpg)

  - flatMap，虽然深蓝色的珠子紧随绿色珠子，但是，第二个绿色方块也正常发送了出去，只不过，顺序乱了（这里的 「乱」指的是，第二个绿色方块没有在蓝色的第一个方块之前发出）

    ![img](https://img.mubu.com/document_image/5dccd09c-af63-4a6a-8bda-cfc7d4c30ad0-4072084.jpg)

## 小结：

  - flatMap 适用于各个结果都有用，不管它们的先后时间
  - switchMap就像flatMap，**但它仅保留最新的观察到结果。过时的事件会被丢弃掉。**
    - 怎么样算过期？
      - 假设 Observable1 事件还没发送完，此时有了新的 Observable2 ，则Observable1 未发布的/未转换完的事件就不会发送出去了。

##  参考资料与学习资源推荐

- https://lorentzos.com/improving-ux-with-rxjava-4440a13b157f
- https://www.jianshu.com/p/33c548bce571
- https://stackoverflow.com/questions/28175702/what-is-the-difference-between-flatmap-and-switchmap-in-rxjava

由于本人水平有限，可能出于误解或者笔误难免出错，如果发现有问题或者对文中内容存在疑问请在下面评论区告诉我，谢谢！