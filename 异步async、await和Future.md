本文仅做应用演示，详细原理可参看以下文章

- [Dart asynchronous programming: Isolates and event loops](https://medium.com/dartlang/dart-asynchronous-programming-isolates-and-event-loops-bffc3e296a6a)
- [Flutter 事件机制 - Future 和 MicroTask 全解析](https://juejin.im/post/5cadacdb6fb9a068a03ae550)

### 相关知识
1. Dart是单线程语言
2. Dart中的异步其实是通过调度任务的优先级实现的
3. Dart中没有线程的概念，只有Isolate（隔离），各个Isolate之间不共享内存，使用Port进行通信。
4. 我们的代码一般运行在MainIsolate中（除非单开Isolate执行特别耗时的计算）。
5. 一个Isolate中有一个Looper和两个Queue
    - Microtask Queue
    - Event Queue
6. 执行优先级：Main方法 > Microtask > Event

### Microtask Queue
微任务队列，用来执行非常快速的任务。

Looper会在执行完Main方法后取出Microtask Queue内的任务进行执行，当该队列为空时才去取出Event Queue中的任务进行执行。

可以执行如下代码安排微任务：

```
scheduleMicrotask(()=>print("Execute Microtask"));
```
一般在我们的代码中不推荐使用该队列。

### Future
每创建一个Future对象，系统会向Event Queue中添加一个任务。

Future的链式调用方法包括then，catchError，whenComplete：

- then：任务执行完毕返回结果时，会调用then方法，并将任务结果传入
- catchError：任务执行中抛出异常时，会调用catchError方法，并将异常传入
- whenComplete：无论任务是否成功，都会调用whenComplete方法

```dart
main() {
  Future.value("abc")
  .then((value) {
    print(value); //print 'abc'
  });

  Future.error("throw an error")
  .catchError((error) {
    print(error); //print 'throw an error'
  });

  Future(()=>print("Event Executed"))
  .whenComplete((){
    print("Event completed");
  });
}
```
直接使用Future的链式调用可以快速进行异步操作，但是一旦业务逻辑比较复杂，尤其是牵扯到异常处理时，链式调用就会非常不直观。

因此，我们在代码中一般使用async和await的方式，将异步任务用同步代码的写法来实现。

### async、await
在dart中，我们可以使用async关键字将一个方法声明为异步方法。

```dart
doSthAsync() async {
    //执行耗时计算
}
```

标记为async的方法，会自动将返回值包裹为一个Future对象。上面的方法没有声明返回值，则返回Future<dynamic>对象。

```dart
main(){
    var result = doSthAsync();
    print(result.runtimeType);  //print 'Future<dynamic>'
}
```

如果方法需要返回数据，则使用Future<T>的返回值，其中泛型T是方法返回的数据类型。如果方法不需要返回数据，声明为Future<void>即可。

```dart
Future<String> doSthReturnValue() async {
    //执行耗时计算
    return Future.value("计算结果");
}

Future<void> doSthReturnVoid() async {
    //执行耗时计算
}
```
上面这个方法的返回值可以简化，async方法会自动将返回值包裹为Future对象，因此直接返回String即可。

```dart
Future<String> doSthReturnValue() async {
    //执行耗时计算
    return "计算结果";
}
```
当在async标记的方法中调用其他async方法时，我们可以使用await关键字将异步方法转为同步。
```dart
Future<int> childTask1() async {
    return 100;
}

Future<int> childTask2() async {
    return 200;
}

Future<int> sum() async {
    final result1 = await childTask1();
    final result2 = await childTask2();
    return result1 + result2;
}
```
当业务中包含异常处理时，使用await关键字调用的async方法，也会像同步方法一样将异常抛出，可以使用try-catch将异常捕获并中断流程。
```dart
onLoginPressed() async {
    try{
        final userId = await doLogin(); //该方法抛出了异常，表示登录时发生了异常
        await getUserInfo(userId);  //流程被异常中断，该方法不会执行
        print("登录成功");
    } catch (e) {
      print("登录失败：${e.toString()}");  //print "登录失败：用户不存在"
    }
}

Future<String> doLogin() async {
    throw "用户不存在";
}

Future<void> getUserInfo(String uid) async {
    //获取用户信息
}

```
这种处理异常的写法会让代码的可读性提高很多，或者你可以使用Future的链式调用试试实现上面这个登录流程。
