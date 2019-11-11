### 针对的问题
1. 全局变量在改变后通知UI刷新
2. 从UI中剥离业务逻辑

### 引用方法
1. 在pubspec.yaml文件中添加rxdart的依赖
```
dependencies:
    flutter:
        sdk: flutter
    rxdart:     //添加该行
```

2. 执行flutter packages get
3. 在dart文件中进行导入
```dart
import 'package:rxdart/rxdart.dart';
```

### 使用方法
rxdart的具体实现及提供的api可以参考
[PUB](https://pub.dev/packages/rxdart)
、
[Github](https://github.com/ReactiveX/rxdart)

我们在这里针对用户登录、注销、及状态更新这个具体问题使用代码进行演示。

声明一个AuthManager类，用来封装关于用户登录的相关方法和状态；

声明一个UserInfo类，用来封装用户的基本资料

```dart
import 'package:rxdart/rxdart.dart';

class AuthManager{
    static final _userInfoSubject = BehaviorSubject<UserInfo>();
}

class UserInfo{
    String email;       //用户的邮件地址
    String nickName;    //用户昵称
    
    UserInfo({this.email = "", this.nickName = ""});
}
```

BehaviorSubject类是rxdart提供的一个Observable类，即观察者模式中的被观察者。

_userInfoSubject是包裹了UserInfo对象的被观察者对象，稍后我们使用它向外部暴露可供观察的接口，既Stream


```dart
class AuthManager{
    ...
    
    static Stream<UserInfo> get stream => _userInfoSubject.stream;
    
    ...
}
```

该stream是从_userInfoSubject对象获取到的数据流，配合StreamBuilder可以自动获取_userInfoSubject中的数据来构建UI，稍后在构建UI的代码中会进行演示。

下面，我们为AuthManager添加调用方法。


```dart
class AuthManager{
    static Future<void> login(String username, String password) async {
        try{
            //发起网络请求，用户登录并获取用户信息。
            //该方法应根据具体情况来实现，此处不再写实现方法。
            final userInfo = await doHttpRequestToLoginWith(username,password);
        
            //将网络请求到的用户数据添加到被观察者对象中
            _userInfoSubject.add(userInfo);
        } catch (e) {
            //当登录失败时，清空用户数据
            _userInfoSubject.add(null);
        }
    }
    
    static Future<void> logout() async {
        try{
            //发起网络请求，注销登录
            await doHttpRequestToLogout();
        } finally {
            //清空用户数据
            _userInfoSubject.add(null);
        }
    }
}
```

至此，用户数据的全局变量流已经使用rxdart大致实现完成，下面我们使用一个Widget来展示如何在UI构建时使用用户的数据。


```dart
class UserInfoPage extends StatelessWidget{

    Widget build(BuildContext context){
        return StreamBuilder<UserInfo>(
            stream: AuthManager.stream,
            builder: (context, snapshot){
                final userInfo = snapshot.hasData? snapshot.data : null;
                return userInfo == null
                    ? Text("用户尚未登录")
                    : ListTile(
                        title: Text(userInfo.nickName),
                        subtitle: Text(userInfo.email),
                      );
            }
        );
    }
}
```

StreamBuilder有两个重要的属性：

- stream：属性即数据流，这里我们使用的是AuthManager中的stream。
- builder：属性即UI构建器，该builder有两个入参
    - context即BuildContext对象
    - snapshot是数据流中的数据快照，我们可以从中获取到数据流的最新数据

至此，我们已经完成了UI对UserInfo数据的监听。

一旦AuthManager._userInfoSubject中的数据发生改变时，StreamBuilder就会获取流中最新的数据，调用builder提供的方法进行再次构建，无论这个控件现在是否可见。

如此一来，我们可以很轻松地将具体的业务逻辑从UI代码中剥离出来。例如注销按钮的onPressed方法：


```dart
class SomePage extends StatelessWidget{
    ...
    void onLogoutPressed() {
        AuthManager.logout();
    }
    ...
}
```

此外，我们还有某些时候需要直接获取当前的数据或状态，因此在AuthManager中可以添加一些代码来实现这个功能。


```dart
class AuthManager{
    ...
    static bool get isLogin => _userInfoSubject.value != null;
    
    static UserInfo get userInfo => _userInfoSubject.value;
    ...
}
```

这样我们就可以在不能使用StreamBuilder的地方获取到用户的实时状态了，例如：


```dart
void printUserInfo(){
    if(AuthManager.isLogin){
        print(AuthManager.userInfo.email);
    }else{
        print("用户未登录");
    }
}
```
