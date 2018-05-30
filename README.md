一句话介绍PASSIVE EVENT LISTENERS

passive event listeners是一种新的DOM事件处理方案，改进了addEventListener，由2016年Google I/O 大会上提出，目的是用于提升页面流畅度。
ADDEVENTLISTENER的前世今生

在《JavaScript高级程序设计（第3版）》中，定义了两个“DOM2”级事件：
addEventListener()
removeEventListener()
这两个方法用于对添加和删除DOM事件的处理函数。

```
```

在这之后的很长一段时间内addEventListener()的基本语法为：

target.addEventListener(type, listener[, useCapture]);
即该方法接受三个参数：

要监听的事件类型
被监听事件发生时的的处理函数
（选填）处理函数是在捕获阶段执行（true）还是在冒泡阶段执行（false），默认值为false（冒泡阶段执行）
老旧策略的弊端

在添加事件监听方法时监听事件事件处理函数时，并不会去解析处理函数listener，所以当事件发生时，浏览器才会去执行listener
由于我们在listener函数体内经常调用两个方法：

stopPropagation() ——阻止事件传播
preventDefault()——阻止事件的默认行为
此时有一种特殊的场景，就是为移动设备的touch事件绑定preventDefault()阻止目标DOM结构的滑动：

target.addEventListener('touchmove',function(e){
	e.preventDefault()
})

在添加touch事件处理函数listener时，浏览器并不知道listener内部到底有没有调用preventDefault()方法阻止滑动行为，所以浏览器会首先执行listener函数，等listener执行完成之后，如果listener中调用了preventDefault()则页面保持停止不动，如果没有调用，则立马进行滚动行为，所以在listener函数执行期间页面无论如何是不会滚动的。

即如果listener函数执行耗费了200ms，而listener函数内部又没有调用preventDefault()方法，页面会停滞200ms然后滑动，这就造成了页面滑动卡顿。

优化方案的诞生

数据支持
Google的数据统计结果显示，在 Android 版 Chrome 浏览器的 touch 事件监听器的页面中，80% 的页面都不会调用 preventDefault 函数来阻止事件的默认行为。
在滑动流畅度上，有 10% 的页面增加至少 100ms 的延迟，1% 的页面甚至增加 500ms 以上的延迟。

也就是说，其实大部分touch事件的处理函数listener没有使用preventDefault函数阻止页面滑动，但是浏览器在执行滑动前都先停下来耗费时间去执行了listener以确认到底要不要阻止滑动，从而造成了页面滑动的卡顿。

优化策略

为了提升页面滑动的流畅性，passive event listeners便应运而生，其核心思想是：
在绑定事件处理函数时设置一个变量passive，该变量的值声明了事件处理函数内部有没有调用preventDefault，如果没有（passive=true），则在touch事件发生时就不必等待listener的执行，而直接滑动页面；否则（passive=false），在touch事件触发时直接禁用页面滑动行为。

passive event listeners的语法如下：

target.addEventListener(type, listener ,{ passive: Boolean});
绕不开的兼容性问题
浏览器的支持程度

可以在Can I use中查看passive event listeners的支持情况：


由于旧版浏览器仍然使用

target.addEventListener(type, listener[, useCapture])
的语法将第三个参数作为布尔值来处理，遇到了passive event listeners的语句，则第三个参数无论是{ passive: false}还是{ passive: false}都会被认为是true，从而非但不能起到提前获知listener内部是否调用preventDefault方法，而且还将listener
的执行顺序设置为捕获方式（大多数情况下都是冒泡方式）。
可以使用以下示例代码进行实验：

<body>
  <div id="father">
    <div id="child" style="height: 200px;"></div>
  </div>
  <script type="text/javascript">
    document.addEventListener('touchmove', function(e){ alert(1) },{passive:false})
    document.querySelector('#father').addEventListener('touchmove', function(e){ alert(2) },{passive:false});
    document.querySelector('#child').addEventListener('touchmove', function(e){ alert(3) },{passive:false});
  </script>
</body>
当发生touch事件时，以上代码在支持passive event listeners的环境中会依次弹框3、2、1；在不支持passive event listeners的环境中会依次弹框1、2、3。

passive event listeners引发的新问题

在一些比较新的机型上，使用

target.addEventListener('touch', function(e){
  e. preventDefault()
} ,false)
的方式来阻止滑动事件不能够起作用，因为新的浏览器采取passive event listeners的策略，默认会设置{passive:true}，从而导致touch事件发生时，页面仍然可以滑动，listener内部的preventDefault不能够生效。

推荐的解决方案

为了避免passive event listeners写法在老旧浏览器中引起的事件捕获时机的问题，以及老式addEventListener写法在新浏览器中无法阻止滑动事件的问题，我们通常使用以下方式来实现一种兼容的方法，使得addEventListener方法在新老浏览器中都能够较好的执行：

// 表示当前环境是否支持passive event listeners的变量

var passiveSupported = false;

try {
    // 设置对象属性的get方法，get方法内将passiveSupported赋值为true
  var options = Object.defineProperty({}, "passive", {
    get: function() {
      passiveSupported = true;
    }
  });

// 调用addEventListener

window.addEventListener("test", null, options);
} catch(err) {}

这段代码在定义 passive 属性时创建了一个带有getter函数的 options 对象；get设定了一个标识， passiveSupported，get方法执行后passiveSupported会被赋值为true。

在浏览器检查第三个参数options时，如果是支持passive event listeners的浏览器，会将其作为一个对象，而去读取其passive属性的值，此时就会调用get方法，从而将passiveSupported赋值为true；
对于老旧浏览器而言，会将参数options直接转为布尔值处理，get方法没有得到调用，passiveSupported的值仍然为false

所以在之后使用如下形式的事件处理代码，就可以很好的兼容新老浏览器：

target.addEventListener("touchmove", listener, passiveSupported? { passive: true } : false);
