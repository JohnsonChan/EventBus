EventBus
========
EventBus是一个事件发布/订阅总线，有效适用于Android系统平台.<br/>
<img src="EventBus-Publish-Subscribe.png" width="500" height="187"/>

简介
 * 组件之间的通信更加简单 
    * 针对在事件的发送者和订阅者之间进行解耦 
    * 非常好的运用在Activitys、Fragments和后台线程 
    * 避开了联系紧密易出错的依赖关系和容易出错生命周期 
 * 使你的代码更加简洁 
 * 快 
 * 小(小于50K的jar包) 
 * 有100,00,000+的app使用了EventBus
 * 有比较好的特征如多支持多线程，订阅者优先级等
 [![Build Status](https://travis-ci.org/greenrobot/EventBus.svg?branch=master)](https://travis-ci.org/greenrobot/EventBus)

使用EventBus 只需要 3 步
-------------------
1. 自定义事件events:

    ```java  
    public static class MessageEvent { /* Additional fields if needed */ }
    ```

2. 在订阅者类里 subscribers:
    声明带有@Subscribe注释及用自定义事件做参数的函数, 另外支持多种线程模式 [thread mode](http://greenrobot.org/eventbus/documentation/delivery-threads-threadmode/):  

    ```java
    @Subscribe(threadMode = ThreadMode.MAIN)  
    public void onMessageEvent(MessageEvent event) {/* Do something */};
    ```
    在订阅者类里注册和反注册. 如Android可以根据 activities and fragments声明周期做注册和反注册:

   ```java
    @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }
 
    @Override
    public void onStop() {
        super.onStop();
        EventBus.getDefault().unregister(this);
    }
    ```

3. 广播事件:

   ```java
    EventBus.getDefault().post(new MessageEvent());
    ```

Read the full [getting started guide](http://greenrobot.org/eventbus/documentation/how-to-get-started/).

添加 EventBus 到你的项目
----------------------------
<a href="https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.greenrobot%22%20AND%20a%3A%22eventbus%22"><img src="https://img.shields.io/maven-central/v/org.greenrobot/eventbus.svg"></a>

通过 Gradle:
```gradle
compile 'org.greenrobot:eventbus:3.1.1'
```

通过 Maven:
```xml
<dependency>
    <groupId>org.greenrobot</groupId>
    <artifactId>eventbus</artifactId>
    <version>3.1.1</version>
</dependency>
```

或者下载从Maven 中心库 [最新 JAR](https://search.maven.org/remote_content?g=org.greenrobot&a=eventbus&v=LATEST) .

EventBus实现流程
------------------------------
注册订阅者
```java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

反射方式从订阅者里找出有`@Subscribe`注释的函数 
```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }

    if (ignoreGeneratedIndex) {
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```


用类`Subscription`关联订阅者和订阅函数（一种事件会对应一个Subscription（订阅者-订阅函数）列表）
这些列表存储在subscriptionsByEventType里
```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    // 一种事件类型对应一个Subscription（订阅者-订阅函数）列表
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType); // 获取事件对应的订阅者表（类和订阅方法的对应表）
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            // 存在重复注册
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }

    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        // 加在末尾或按优先级插入到队列里
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }

    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);

    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

广播事件post每次从当前线程获取列表添加事件，并从列表栈顶取事件广播
```aidl
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);

    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

`postSingleEvent`获取`eventClass`的父类，遍历着找到对应的 订阅者-订阅方法的列表
```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    if (eventInheritance) {
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass); // 获取eventClass的父类和接口的calss对象
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}

private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    // 通过事件类，找出所有订阅者和方法的关联类列表，
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```

`postToSubscription`根据线程模式，调用对应的订阅者方法
4种模式
默认情况`POSITING`模式，post的线程里直接触发订阅者方法
`MAIN`，`MAIN_ORDERED`模式，通过`mainThreadPoster`，使用`Hander.sendMessage`方式触发订阅者方法
`BACKGROUND`模式，通过`backgroundPoster`新Runable线程触发订阅者方法，`eventBus.getExecutorService().execute()`
`ASYNC`异步的方式，也是用通过`asyncPoster`新Runable线程触发订阅者方法,`eventBus.getExecutorService().execute()`

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                // 当前post为主线程，直接调用
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

最后`invokeSubscriber`通过反射，传入类实例和参数触发订阅者方法，完成整个流程
```java
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```

EventBus关键类：
`FindState`
`PendingPost`
`Subscription`
`SubscriberMethod` 等

首页, 文档, 连接
------------------------------
更多细节可以到网站 [EventBus website](http://greenrobot.org/eventbus)查看. 下面是可能对你有用的连接:

[EventBus的特性](http://greenrobot.org/eventbus/features/)

[EventBus的文档](http://greenrobot.org/eventbus/documentation/)

[混淆](http://greenrobot.org/eventbus/documentation/proguard)

[版本更新记录](http://greenrobot.org/eventbus/changelog/)

[常见问题](http://greenrobot.org/eventbus/documentation/faq/)

EventBus对比其他解决方案, 像Square的Otto? 可以看看这个 [对比结果](COMPARISON.md).


声明
-------
Copyright (C) 2012-2017 Markus Junginger, greenrobot (http://greenrobot.org)

EventBus binaries and source code can be used according to the [Apache License, Version 2.0](LICENSE).

greenrobot的更多开源库
==============================
[__ObjectBox__](http://objectbox.io/) ([GitHub](https://github.com/objectbox/objectbox-java)) is a new superfast object-oriented database for mobile.

[__Essentials__](http://greenrobot.org/essentials/) ([GitHub](https://github.com/greenrobot/essentials)) is a set of utility classes and hash functions for Android & Java projects.

[__greenDAO__](http://greenrobot.org/greendao/) ([GitHub](https://github.com/greenrobot/greenDAO)) is an ORM optimized for Android: it maps database tables to Java objects and uses code generation for optimal speed.

[Follow us on Google+](https://plus.google.com/b/114381455741141514652/+GreenrobotDe/posts) or check our [homepage](http://greenrobot.org/) to stay up to date.
