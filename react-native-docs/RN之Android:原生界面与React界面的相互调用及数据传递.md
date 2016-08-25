这里原生界面是指用布局文件实现或java代码实现view的Activity，React界面是指用ReactJS实现的界面的Activity。

从某种角度看，React只是充当了Android里的view层，因此原生界面与React界面的相互调用及数据传递同原生界面之间的互动基本是一致的。

下面是我对两种界面的相互调用和数据传递的一种实现尝试，不一定是最有效率或最佳的，纯当练习和探索而已。   
###原生界面调用React界面

1.只是启动React界面的话很简单，同原生界面间的启动一样，直接用startActivity即可。

2.在启动React界面时传递数据给React界面。

先来捋下思路，原生界面启动是没问题的，同原生一样；关键是React界面如何获取到传递过来的值：从前面的学习中我们知道原生模块是有很大自由度的，只要能得到React界面所在activity就可以顺着得到传递的来的intent，而JS端想得到数据就要利用回调函数了。

接下来是实现思路，从后往前来：首先先要按之前构建原生模块的步骤一步步来构建我们这次需要的功能
```
public class MyIntentModule extends ReactContextBaseJavaModule
@ReactMethod
public void getDataFromIntent(Callback successBack,Callback erroBack){
    try{
        Activity currentActivity = getCurrentActivity();
        String result = currentActivity.getIntent().getStringExtra("result");//会有对应数据放入
        if (TextUtils.isEmpty(result)){
            result = "No Data";
        }
        successBack.invoke(result);
    }catch (Exception e){
        erroBack.invoke(e.getMessage());
    }
}
public class MyReactPackage implements ReactPackage
@Override
public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
    return Arrays.<NativeModule>asList(
            new MyIntentModule(reactContext)
    );
}
public class MyReactActivity  extends ReactActivity
@Override
    protected List<ReactPackage> getPackages() {
        return Arrays.<ReactPackage>asList(
                new MainReactPackage(),
                new MyReactPackage()
        );
    }
```

在JS端配置相应代码

```
.......
class myreactactivity extends React.Component {
constructor(props) {
    super(props);   //这一句不能省略，照抄即可
    this.state = {
        TEXT: 'Input Text',//这里放你自己定义的state变量及初始值
    };
  }
  render() {
    return (
     <View style={styles.container}>
             <TextInput
                 style={{height: 40, borderColor: 'gray', borderWidth: 1,}}
                 onChangeText={(text) => this.setState({text})}
                 value={this.state.TEXT} />
     </View>
    )
  }
     componentDidMount(){   //这是React的生命周期函数，会在界面加载完成后执行一次
        React.NativeModules.MyIntentModule.getDataFromIntent(
            (successMsg) =>{
            this.setState({TEXT: successMsg,}); //状态改变的话重新绘制界面
             },
             (erroMsg) => {alert(erroMsg)}
        );
      }
}
.......
```

最后在原生界面将数据存放在intent里，然后用startActivity启动；

```
Intent intent =new Intent(this,MyReactActivity.class);
 intent.putExtra("result","haha");
 startActivity(intent,10);
```

3.原生界面启动React界面后等待回调数据。

在原生应用里是通过intent来传递数据，有 startActivityForResult和onAcitvityResult来满足启动和回调，回传数据的用setResult设置intent。同上面一样，由于可以得到当前activity，这些问题也迎刃而解了。

先在MyIntentModule里添加方法

```
@ReactMethod
    public void finishActivity(String result){
        Activity currentActivity = getCurrentActivity();
        Intent intent = new Intent();
        intent.putExtra("result",result);
        currentActivity.setResult(11,intent);
        currentActivity.finish();
    }
```

在JS端配置相应代码

`React.NativeModules.MyIntentModule.finishActivity(this.state.TEXT);`

至于原生界面里代码就略去不写了，会安卓的人都可以写出来了。

###React界面调用原生界面

1.只是启动原生界面

安卓中启动activity有隐式和显式，这里只考虑显示启动。显式启动需要指定目的activity的类，而我们虽然可以定制原生模块供JS调用，但并不能直接指定类或字节码为参数。一些特殊的情况下时可以用swtich的思路来解决，适用性更好的方法有没有呢，如何用允许的参数来得到目的类呢。这里就要要到反射了。

先在MyIntentModule里添加方法

```
@ReactMethod
    public void startActivityByString(String activityName){
        try {
            Activity currentActivity = getCurrentActivity();
            if (null != currentActivity) {

                       Class aimActivity = Class.forName(activityName);
                Intent intent = new Intent(currentActivity,aimActivity);
                currentActivity.startActivity(intent);
            }
        } catch (Exception e) {
            throw new JSApplicationIllegalArgumentException(
                    "Could not open the activity : " + e.getMessage());
        }
    }
```

在JS里调用

`React.NativeModules.MyIntentModule.startActivityByString("com.example.SecondActivity")`

2.启动原生界面并传递进数据

这和上面相比就是多了几个参数的事，没有啥好讲的，只是只能传递基本数据类型的参数。要想做到和原生一样在intent里任意传递基本数据和类还做不到，而且要想自己用的方便一点的话还需要再考虑封装模块方法，在此不赘述。

3.React界面启动原生界面后等待回调数据

启动原生界面是同2一样的，问题在于原生界面返回的数据是在React界面所在Activity的onActvityResult里接受的，而JS端接受原生模块传回的数据是通过调用原生模块方法时设置的回调函数接受到的，如何将onActivityResult里的数据实时传递到原生模块里方法设置的回调函数里呢？如果不考虑这些回调只是把数据存在变量或者文件里，用一个类似messageQueue的思路应该可以解决，不过那样需要可能效率上不太高，我们自己实现起来也不太方便。幸运的是在Java里已经有BlockingQueue了。

先在MyReactActivity里添加代码

```
public static ArrayBlockingQueue<String> myBlockingQueue = new ArrayBlockingQueue<String>(1);

@Override
    public void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if(resultCode == RESULT_OK && requestCode == 100){
            String result = data.getStringExtra("result");
            if (！TextUtils.isEmpty(result)){
                    MyConstants.myBlockingQueue.add(result);
                } else {
                    MyConstants.myBlockingQueue.add("无数据传回");
                }
            }
        }else{
            MyConstants.myBlockingQueue.add("没有");
        }
    }
```

在MyIntentModule里添加方法

```
@ReactMethod
    public void startActivityForResult(String activityName,int requestCode,Callback successCallback,Callback erroCallback){
        try {
            Activity currentActivity = getCurrentActivity();
            if ( null!= currentActivity) {
                Class aimActivity = Class.forName(activityName);
                Intent intent = new Intent(currentActivity,aimActivity);
                currentActivity.startActivityForResult(intent,requestCode);
                String result = MyConstants.myBlockingQueue.take();
                successCallback.invoke(result);
            }
        } catch (Exception e) {
            erroCallback.invoke(e.getMessage());
            throw new JSApplicationIllegalArgumentException(
                    "Could not open the activity : " + e.getMessage());
        }
    }
```

在JS端调用

```
React.NativeModules.MyIntentModule.startActivityForResult(
      "com.example.SecondActivity",100,
      (successMsg) => {
            this.setState({TEXT: successMsg, });
      },
      (erroMsg) => {alert(erroMsg)}
      );
```

原生界面的代码也要记得写，不过也没啥好说的了，按原生安卓一样就可以了。

###React界面与Activity
  之前一直有个思维误区，就是认为在index.android.bundle里
  `AppRegistry.registerComponent('RN0825', () => RN0825);`
  这个注册只能用一次。之所以有这样的错误认识，可能是因为之前觉得JSBundle只有一个，入口文件只有一个（index.android.js）,因此注册的组件只能有一个。
  然而，图样图森破。
  下面是index.android.js
  ```
  import React, { Component } from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View,
    TouchableOpacity
} from 'react-native';
import MyModule from './MyModule';
class RN0825 extends Component {

  goToNext(){
    alert('hehe');
    MyModule.startActivityByString("com.rn0825.ThirdActivity");

  }

  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native!
        </Text>
        <Text style={styles.instructions}>
          To get started, edit index.android.js
        </Text>
        <TouchableOpacity onPress={()=>this.goToNext()}>
          <Text>
            goToNext
          </Text>
        </TouchableOpacity>
      </View>
    );
  }
}
class Hello extends Component {
  render() {
    return (
        <View style={styles.container}>
          <Text style={styles.welcome}>
            Hello!
          </Text>
        </View>
    );
  }
}
const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  welcome: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  },
  instructions: {
    textAlign: 'center',
    color: '#333333',
    marginBottom: 5,
  },
});

AppRegistry.registerComponent('RN0825', () => RN0825);
AppRegistry.registerComponent('hello', () => Hello);
```
在Java端里MainActivity.java
```
public class MainActivity extends ReactActivity {
    @Override
    protected String getMainComponentName() {
        return "RN0825";
    }
}
```
SecondActivity.java
```
public class SecondActivity extends Activity implements DefaultHardwareBackBtnHandler {
    private ReactRootView mReactRootView;
    private ReactInstanceManager mReactInstanceManager;
    @Override
    public void invokeDefaultOnBackPressed() {
        super.onBackPressed();
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mReactRootView=new ReactRootView(this);
        mReactInstanceManager=ReactInstanceManager.builder()
                .setApplication(getApplication())
                .setBundleAssetName("index.android.bundle")
                .setJSMainModuleName("index.android")
                .addPackage(new MainReactPackage())
                .setUseDeveloperSupport(BuildConfig.DEBUG)
                .setInitialLifecycleState(LifecycleState.RESUMED)
                .build();
        mReactRootView.startReactApplication(mReactInstanceManager,"hello",null);
        setContentView(mReactRootView);
    }
    @Override
    protected void onPause() {
        super.onPause();

        if (mReactInstanceManager != null) {
            mReactInstanceManager.onHostPause();
        }
    }

    @Override
    protected void onResume() {
        super.onResume();

        if (mReactInstanceManager != null) {
            mReactInstanceManager.onHostResume(this, this);
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();

        if (mReactInstanceManager != null) {
            mReactInstanceManager.onHostDestroy();
        }
    }
    @Override
    public void onBackPressed() {
        if (mReactInstanceManager != null) {
            mReactInstanceManager.onBackPressed();
        } else {
            super.onBackPressed();
        }
    }
    @Override
    public boolean onKeyUp(int keyCode, KeyEvent event) {
        if (keyCode == KeyEvent.KEYCODE_MENU && mReactInstanceManager != null) {
            mReactInstanceManager.showDevOptionsDialog();
            return true;
        }
        return super.onKeyUp(keyCode, event);
    }
}
```
至于中间两个activity调用跳转的代码就不贴了，按正常的来就行，记得在清单文件注册。
弄完以后，run一下你会发现，这样也是可以的。
也就是说是可以注册不同的入口组件的。自由度又高了一层有没有。
要注意一点是如果偷懒将SecondActivity同MainActivity一样简单继承ReactActivity的话，问题就来了，你会发现第一次跳转是没问题的，但再次回到MainActivity时点击界面会不响应。
这是因为ReactInstanceManager没有重新创建，两个activity共用一个。这样当关闭一个时，ReactInstanceManager就会被destroy，另外一个自然就不会有界面响应了。

###总结那点事
  通过上面这些React界面与原生界面、activity与react view之间的这些破事还是可以看出他们之间的关系还是很自由的，供自定义的范围和选择还是很宽泛的：多个原生界面与多个React界面是可以有的，彼此之间的相互调用、数据传递也是可以有的。
  也可以看出如果fb在Android和iOS兼容性和运行性能上做的更好时，前景的确非常好。
  还是要多读源码多练习，多晒被子多思考啊！







