
# Flutter Key总结

Widget、Element、SemanticsNode的标识。
如果两个widget的key相同，新的widget仅用于更新旧的widget。
Key在具有相同父级的Elements中必须唯一。

## 0. how widget use key?

1. 控制树中的widget替换另一个widget。

2. 如果两个widget中的runtimeType和key相等，则新的widget通过更新底层元素替换旧的widget，不相等的话，旧的element被移除，用新的widget构建新的element并插入树中。

3. 使用GlobalKey作为widget的key，允许元素在树上移动而不丢失state。当发现一个新的widget（它的key和type跟之前相同位置的widget不同），但在前一帧中有global key相同的widget，那么这个widget会被移动到新位置。

4. 通常，如果一个widget是另一个widget唯一的child，就不需要key。

## 1. GlobalKey

1. 整个应用中唯一的键，当Widgets在树结构中从一个地方移动到另一个地方时，拥有global key的Widget会重新设置其子树的父级。

2. 使用global key重置父级相当‘昂贵’，因为操作将会触发该Widgets相关联的State和所有子孙的State.deactivate方法并强制rebuild。

3. 加入不需要上述的特性，可以设置Key，ValueKey，ObjectKey，UniqueKey代替。

4. Pitfalls:
- GlobalKey不能重复创建，应该在State中long-lived
- 不要每次创建新GlobalKey，每次build创建新的GlobalKey，旧key相关联的子树的state被清除，并为新key创建新的子树。这样做不但会带来性能损耗，还会给子widgets造成未知的行为。
- 好的做法是在State中定义GlobalKey，在build方法外初始化，如State.initState中。

## 2. LocalKey

1. Key在具有相同父级的Elements中必须唯一。
2. GlobalKey在整个App中必须是唯一的。

### 1. ValueKey：

1. 继承LocalKey。
2. 用特定类型的值创建Key，比较的是值。

```dart
@override
  bool operator ==(Object other) {
    if (other.runtimeType != runtimeType)
      return false;
    return other is ValueKey<T>
        && other.value == value;
  }
```

### 2. ObjectKey

1. 继承LocalKey。
2. 用特定类型的值创建Key，比较的是Object。

```dart
@override
  bool operator ==(Object other) {
    if (other.runtimeType != runtimeType)
      return false;
    return other is ObjectKey
/// Check whether two references are to the same object.
        && identical(other.value, value);
  }
```

### 3. UniqueKey
1. 继承LocalKey。
2. 每次生成不同的值。
