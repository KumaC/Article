##  回调函数与事件（译）

### [原文](http://dean.edwards.name/weblog/2009/03/callbacks-vs-events/)

大多数主流的JavaScript框架都声称以某种形式支持“自定义事件”，比如说，jQuery、YUI和Dojo都支持自定义的“document ready”事件。然而，这些事件却都是以回调函数的机制来实现的。

回调函数的工作原理是这样的，它们先把某个事件的处理函数全部放置在一个数组中，当事件触发时，系统就会依次调用这个数组中的回调函数。所以这里面有什么问题呢？在聊这个问题之前，让我们先来看看下面的代码。

下面是两段使用了DOMContentLoaded事件的初始化代码：

```
document.addEventListener("DOMContentLoaded", function() {
  console.log("Init: 1");
  DOES_NOT_EXIST++; // this will throw an error
}, false);

document.addEventListener("DOMContentLoaded", function() {
  console.log("Init: 2");
}, false);
```

那么在加载文档时控制台会输出些什么呢？

大概会是这个样子：

```
Init: 1

Error: DOES_NOT_EXIST is not defined

Init: 2
```

这里的亮点在于两个函数都被执行了，第一个函数抛出了一个错误，但这没有阻止第二个函数的执行。

#### 问题

再让我们来看一下基于回调函数机制的代码。我选择了最为流行的jQuery：

```
$(document).ready(function() {
  console.log("Init: 1");
  DOES_NOT_EXIST++; // this will throw an error
});

$(document).ready(function() {
  console.log("Init: 2");
});
```

控制台会输出什么呢？

```
Init: 1

Error: DOES_NOT_EXIST is not defined
```

现在问题清楚了，回调函数机制相对来说更脆弱。如果哪个回调函数抛出了一个错误，那么后面的回调函数就不会再执行了。这意味着一个有问题的插件会影响其他插件的初始化。

Dojo有着和jQuery一样的问题，YUi有一点不同。它在回调机制外加上了try/catch块，因此你就看不见函数抛出的错误了，就像下面这样：

```
YAHOO.util.Event.onDOMReady(function() {
  console.log("Init: 1");
  DOES_NOT_EXIST++; // this will throw an error
});

YAHOO.util.Event.onDOMReady(function() {
  console.log("Init: 2");
});
```

输出结果：

```
Init: 1

Init: 2
```

####  结论

结论就是我们得把回调函数和真正的事件处理机制结合在一起才行，我们可以触发一个伪事件，然后通过这个事件来调用回调函数。每个函数都有它自己的执行环境。如果在伪事件中出现了一个错误，它不会影响到回调函数系统。

听起来有一点复杂，可以看一下下面的代码

```
var currentHandler;

if (document.addEventListener) {
  document.addEventListener("fakeEvents", function() {
    // execute the callback
    currentHandler();
  }, false);

  var dispatchFakeEvent = function() {
    var fakeEvent = document.createEvent("UIEvents");
    fakeEvent.initEvent("fakeEvents", false, false);
    document.dispatchEvent(fakeEvent);
  };
} else { // MSIE

  // I'll show this code later
}

var onLoadHandlers = [];
function addOnLoad(handler) {
  onLoadHandlers.push(handler);
};

onload = function() {
  for (var i = 0; i < onLoadHandlers.length; i++) {
    currentHandler = onLoadHandlers[i];
    dispatchFakeEvent();
  }
};
```

让我们用上面的代码来试一试之前的问题函数吧：

```
addOnLoad(function() {
  console.log("Init: 1");
  DOES_NOT_EXIST++; // this will throw an error
});

addOnLoad(function() {
  console.log("Init: 2");
});
```

看一下控制台：

```
Init: 1

Error: DOES_NOT_EXIST is not defined

Init: 2
```

简直完美！两个事件处理函数都得到了执行，我们也得到了第一个函数所抛出的错误，真棒！

嗯？你问在IE里会怎样？IE并不支持标准的事件处理机制。它有它自己的方法：fireEvent，但这只作用于真正的事件（比如click事件）。

直接上代码！

```
var currentHandler;

if (document.addEventListener) {

  // We've seen this code already

} else if (document.attachEvent) { // MSIE

  document.documentElement.fakeEvents = 0; // an expando property

  document.documentElement.attachEvent("onpropertychange", function(event) {
    if (event.propertyName == "fakeEvents") {
      // execute the callback
      currentHandler();
    }
  });

  dispatchFakeEvent = function(handler) {
    // fire the propertychange event
    document.documentElement.fakeEvents++;
  };
}
```

除此之外还有个相似的方法是使用私有属性的性质来进行事件的触发。

#### 总结

我已经演示了如何使用底层的事件机制来触发自定义事件。框架的作者们应该能够明白如何将其拓展为跨浏览器的自定义事件。

#### 更新

有一些评论建议使用setTineout。我的看法是这样的：

> 对于这个例子来说，定时器会很管用。但这只是一个用来演示的小例子。我所讲述的技巧的真正实用性会在其他的自定义事件上体现出来。大多数的框架提供的自定义事件都是建立在回调函数机制上。就像我说的那样，回调函数机制很脆弱。使用定时器来处理事件的确有一定的作用，但它离真正的事件处理还相去甚远。在真正的事件处理机制中，事件应该被连续的处理。还有一些其他的地方，比如取消事件和阻止事件冒泡，这些对定时器来说都是不可能的。

文章的重点在于这个技巧，那就是我把回调函数封装成了一种真正的事件处理机制。这样你就可以在IE 浏览器中使用真正的自定义事件了。如果你正基于事件代理来建立事件机制的话，那你或许会对这一技巧感兴趣。
