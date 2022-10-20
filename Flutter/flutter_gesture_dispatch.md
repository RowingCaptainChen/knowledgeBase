# Flutter 手势事件分发

## 1. 事件收集

1. 单击事件是由Embedder生成，通过Engine下面的window.onPointDataPacket转换成统一的数据交给Flutter处理。

```dart
mixin GestureBinding on BindingBase implements HitTestable, HitTestDispatcher,         HitTestTarget {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    //_handlePointerDataPacket 是点击事件PointerDataPacketCallback?回调
    platformDispatcher.onPointerDataPacket = _handlePointerDataPacket;
  }
 //用于存储触控点的队列
  final Queue<PointerEvent> _pendingPointerEvents = Queue<PointerEvent>();
    //将触控点物理像素转换为逻辑像素，隔离设备差异
  void _handlePointerDataPacket(ui.PointerDataPacket packet) {
    _pendingPointerEvents.addAll(PointerEventConverter.expand(packet.data, window.devicePixelRatio));
    if (!locked) _flushPointerEventQueue();
 }
  //依次处理点击事件
  void _flushPointerEventQueue() {
    while (_pendingPointerEvents.isNotEmpty)
      handlePointerEvent(_pendingPointerEvents.removeFirst());
  }
```

## 2. 事件处理

1. Flutter根据不同的事件类型采取不同的策略：

2. PointerDownEvent、PointerDownEvent、PointerHoverEvent事件，初次的手势事件，会收集一个单击测试列表hitTestResult，记录当前哪些对象相应本次的单击。

3. PointerUpEvent、PointerCancelEvent事件，手势取消事件，将通过event.pointer取出第一种事件获取的hitTestResult，并将其移除。

4. event.down事件，介于以上两种事件之间，多为滑动、拖拽事件，将直接取出单击测试结果并使用。

```dart
void _handlePointerEventImmediately(PointerEvent event) {
    HitTestResult? hitTestResult;
    if (event is PointerDownEvent || event is PointerDownEvent || event is PointerHoverEvent) {
    //第一类事件，开始形成一个手势事件类型
      hitTestResult = HitTestResult();
      //单击测试，收集相应本次点击的实例
      hitTest(hitTestResult, event.position);
      //PointerDownEvent事件表示手势开始
      if (event is PointerDownEvent) { 
      //存储单击测试结果，用于后续使用
        _hitTests[event.pointer] = hitTestResult;
      }
      //PointerUpEvent、PointerCancelEvent表示手势结束
    } else if (event is PointerUpEvent || event is PointerCancelEvent) {
    //第二类事件，根据event.pointer取出第一类事件的HitTestResult并移除
    //接收到手势结束事件，移除本次结果
      hitTestResult = _hitTests.remove(event.pointer);
    } else if (event.down) { // 其他类型的手势如滑动、拖拽等
    //取出形成手势时存储的单击测试结果继续使用
      hitTestResult = _hitTests[event.pointer];
    }
    if (hitTestResult != null ||
        event is PointerAddedEvent ||
        event is PointerRemovedEvent) {
        //向可相应手势集合分发事件
      dispatchEvent(event, hitTestResult);
    }
  }
```

## 3. hitTest实现

1. RendererBinding的hitTest首先会执行renderView的hitTest方法，

2. renderView作为Render Tree的根节点，会遍历每个节点的单击测试，如RenderBox的hitTest方法将递归的对每个子节点和自身进行单击测试，然后一次加入队列。

3. GestureBinding会将自己加入队列的最后。

```dart
  //GestureBinding
  @override
  void hitTest(HitTestResult result, Offset position) {
    result.add(HitTestEntry(this));
  }
  
  //RendererBinding
  @override
  void hitTest(HitTestResult result, Offset position) {
  //触发Render Tree的根节点
    renderView.hitTest(result, position: position);
    //执行GestureBinding的hitTest方法
    super.hitTest(result, position);
  }
  
  //RenderView
  bool hitTest(HitTestResult result, { required Offset position }) {
  //Render Tree的根节点将触发子节点的hitTest方法
    if (child != null)
      child!.hitTest(BoxHitTestResult.wrap(result), position: position);
      //最后将自己加入单击测试结果
    result.add(HitTestEntry(this));
    return true;
  }
  
  //RenderBox
  bool hitTest(BoxHitTestResult result, { required Offset position }) {
    if (_size!.contains(position)) { //首先判断点击位置是否在Layout范围内
      if (hitTestChildren(result, position: position) || hitTestSelf(position)) {//再判断子节点或者自身是否通过单击测试
        result.add(BoxHitTestEntry(this, position));
        return true;
      }
    }
    return false;
  }

```

## 4. dispatchEvent实现
1. 所有的可响应的单节对象都存储在HitTestResult后，GestureBinding触发dispatchEvent方法完成本次事件分发

2. 对于hitTestResult不为null的情况，会一次调用每个HitTestTarget对象的handleEvent方法，需要处理手势的HitTestTarget子类，如RenderPointerListener，通过实现handleEvent方法参与手势竞技场，并在赢得竞争后处理手势。

```dart
void dispatchEvent(PointerEvent event, HitTestResult? hitTestResult) {
//PointerAddedEvent、PointerRemovedEvent使用路由统一分发，其他情况则通过handleEvent方法处理
if (hitTestResult == null) {
    pointerRouter.route(event);
     return;
}
    for (final HitTestEntry entry in hitTestResult.path) {
      entry.target.handleEvent(event.transformed(entry.transform), entry);
    }
  }
```

## 5. 总结

1.  在点击的时候，hitTest根据点击的position通过rendview获取一个可以响应事件的object集合，并且在集合最后 GestureBinding将自己添加到队尾

2. 通过dispatchEvent 事件进行分发，但并不是所有的控件的 RenderObject 子类都会处理 handleEvent ，大部分时候，RenderPointerListener 处理 handleEvent 事件，这个控件被嵌套在RawGestureDetector中，handleEvent会根据不同的事件类型，回调到RawGestureDetector的相关手势处理。

## 6. 补充
1. 竞技场: https://juejin.cn/post/6874570159768633357
2. 事件处理: https://juejin.cn/post/6844903919563309064
3. 滑动事件: https://blog.csdn.net/china_2014/article/details/115123672
4. 屏蔽widget多点触控行为：https://www.jianshu.com/p/690c28823ea1
5. https://docs.flutter.dev/development/ui/advanced/gestures#gesture-disambiguation
6. https://medium.com/flutter-community/flutter-deep-dive-gestures-c16203b3434f
