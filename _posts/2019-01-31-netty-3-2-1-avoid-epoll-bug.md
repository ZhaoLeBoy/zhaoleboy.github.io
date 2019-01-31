---
layout:     post
title:      "3-2-1.Netty源码:避免空轮循的bug"
subtitle:   "避免空轮循的bug"
date:        2019-01-31
author:     "ZhaoLe"
header-img: "img/netty/scalable-io/post-bg.jpg"
catalog: true
tags:
    - Netty
---


```java
 //NioEventLoop.java
//重新构造个selector
private void rebuildSelector0() {
  final Selector oldSelector = selector;
  final SelectorTuple newSelectorTuple;
  ...
  try {
    newSelectorTuple = openSelector();
  }
  ...
  
  //获取这个selector上所有注册的channelkey
  for (SelectionKey key : oldSelector.keys()) {
    Object a = key.attachment();
    try {
        if (!key.isValid() || key.channel().keyFor(newSelectorTuple.unwrappedSelector) != null) {
            continue;
        }
        //这个select上所有感兴趣事件
        int interestOps = key.interestOps();
        key.cancel();
        
        SelectionKey newKey = key.channel().register(newSelectorTuple.unwrappedSelector, interestOps, a);
        if (a instanceof AbstractNioChannel) {
            ((AbstractNioChannel) a).selectionKey = newKey;
        }
        nChannels++;
    } catch (Exception e) {
        ...
    }
  }

  selector = newSelectorTuple.selector;
  unwrappedSelector = newSelectorTuple.unwrappedSelector;

  try {
      // time to close the old selector as everything else is registered to the new one
      oldSelector.close();
  } 
  ...
}
```
* 【7】创建 Selector 对象,netty优化了selector创建(解释过长，移步到下面的openSelector()方法说明）
* 【12-30】 获取产生空轮循selecor上的所有的channel，注册到新创建的selector上。
  * 【13】获取每个注册selecor上的SelectoionKey，并且获取这些key的`attachment`
  * 【15-17】判断这些key是否可用。
  * 【20】 将老的channel以及它的参数和感兴趣事件注册到新的selector上
* 【32-33】将`selector`和`unwrappedSelector`指向新的 Selector 对象。
* 【37】关闭旧的selector

---
#### openSelector()方法说明
```java
 //NioEventLoop.java
 private SelectorTuple openSelector() {
    final Selector unwrappedSelector;
    
    try {
        unwrappedSelector = provider.openSelector();
    } catch (IOException e) {
        throw new ChannelException("failed to open a new selector", e);
    }
    if (DISABLE_KEYSET_OPTIMIZATION) {
        return new SelectorTuple(unwrappedSelector);
    }

    Object maybeSelectorImplClass = AccessController.doPrivileged(new PrivilegedAction<Object>() {
        @Override
        public Object run() {
            try {
                return Class.forName(
                        "sun.nio.ch.SelectorImpl",
                        false,
                        PlatformDependent.getSystemClassLoader());
            } catch (Throwable cause) {
                return cause;
            }
        }
    });

    if (!(maybeSelectorImplClass instanceof Class) ||
            !((Class<?>) maybeSelectorImplClass).isAssignableFrom(unwrappedSelector.getClass())) {
        if (maybeSelectorImplClass instanceof Throwable) {
            Throwable t = (Throwable) maybeSelectorImplClass;
            logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, t);
        }
        return new SelectorTuple(unwrappedSelector);
    }

    final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;

    final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

    Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
        @Override
        public Object run() {
            try {
                
                Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
                Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

                if (PlatformDependent.javaVersion() >= 9 && PlatformDependent.hasUnsafe()) {
                    long selectedKeysFieldOffset = PlatformDependent.objectFieldOffset(selectedKeysField);
                    long publicSelectedKeysFieldOffset = PlatformDependent.objectFieldOffset(publicSelectedKeysField);

                    if (selectedKeysFieldOffset != -1 && publicSelectedKeysFieldOffset != -1) {
                        PlatformDependent.putObject(
                                unwrappedSelector, selectedKeysFieldOffset, selectedKeySet);
                        PlatformDependent.putObject(
                                unwrappedSelector, publicSelectedKeysFieldOffset, selectedKeySet);
                        return null;
                    }
                }
                
                Throwable cause = ReflectionUtil.trySetAccessible(selectedKeysField, true);
                if (cause != null) {
                    return cause;
                }
                cause = ReflectionUtil.trySetAccessible(publicSelectedKeysField, true);
                if (cause != null) {
                    return cause;
                }
                
                selectedKeysField.set(unwrappedSelector, selectedKeySet);
                publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
                return null;
            } catch (NoSuchFieldException e) {
                return e;
            } catch (IllegalAccessException e) {
                return e;
            }
        }
    });

    if (maybeException instanceof Exception) {
        selectedKeys = null;
        Exception e = (Exception) maybeException;
        logger.trace("failed to instrument a special java.util.Set into: {}", unwrappedSelector, e);
        return new SelectorTuple(unwrappedSelector);
    }
    selectedKeys = selectedKeySet;
    logger.trace("instrumented a special java.util.Set into: {}", unwrappedSelector);
    return new SelectorTuple(unwrappedSelector,
            new SelectedSelectionKeySetSelector(unwrappedSelector, selectedKeySet));
}
```
* 【4-8】使用原生的方法创建 Selector 对象。
* 【9-12】默认使用selector优化。
* 【13-25反射的方法构建sun.nio.ch.SelectorImpl的类
* 【27-34】获得 SelectorImpl 类失败，则返回`SelectorTuple`对象。
* 【38】创建`SelectedSelectionKeySet`对象 ,是对`Selector` 中的`selectionKeys`重写.用预定义1024大小的列表替换原来HashSet。数组的add方法复杂度比HashSet的add方法复杂度要低.
* 【40-79】将创建好的`SelectedSelectionKeySet`对象利用反射重新设置到 unwrappedSelector 中的`selectedKeys`和 `publicSelectedKeys`。
  * 【45-46】获得`selectorImplClass`中的 "selectedKeys" 和"publicSelectedKeys" 的 Field。
  * 【61-68】设置两个Field的访问权限。
  * 【70-71】设置`SelectedSelectionKeySet`对象到 unwrappedSelector中的 Field 中。
* 【81-90】如果设置失败，返回 SelectorTuple 对象，如果成功设置个`selectedKeys`成员变量，并且返回新创建的`SelectorTuple`对象