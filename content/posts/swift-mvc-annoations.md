---
title: "SwiftUI中不同属性修饰符的区别"
date: 2021-08-08T17:19:46+08:00
---

SwiftUI提供了State、EnvironmentObject等修饰符，用来实现View和Model之间的关联。

<!--more-->

## State

State一般用于较为简单的视图属性，当属性发生变化时，其相关联的视图也会重新刷新。用State修饰的属性通常不会在其他地方被使用，因此一般会标注为`private`

```swift
struct ContentView: View {
		@State private var count = 0;
    
    var body: some View {
      	Button("Tap count: \(count)") {
        		count += 1
        }
    }
}
```

## EnvironmentObject

EnvironmentObject用于修饰需要在所有View中共享的属性，从而免去了在不同View之间传递大量数据的问题。所以当属性发生改变时，所有与该属性相关联的View都会执行刷新。

