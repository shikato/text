[DroidKaigi2017](https://droidkaigi.github.io/2017/)にて、 @operandoOSさんによる「[コマンドなしでぼくはAndroid開発できない話](https://speakerdeck.com/operando/komantonasitehokuhaandroidkai-fa-tekinaihua-1)」という発表がありました。
その発表内で`adb shell am hang`という、アプリをhangさせるための興味深いコマンドが紹介されており、加えて、発表では各コマンドのコードへのリンクも貼られているのも印象的でした。

これまで一度もadbコマンドのコードを読んだことがなかったため、勉強がてら読んでみることにしました。
読書対象コマンドは、発表を聞いていて、どうやってアプリをhangさせているのか気になった`adb shell am hang`です。

## Am.java
発表で`am`コマンドのコードとしてAm.javaが紹介されていたので、まずはAm.javaから読んでみることにしました。

※ [android/platform_frameworks_base](https://github.com/android/platform_frameworks_base)にて[android-7.1.1_r28](https://github.com/android/platform_frameworks_base/tree/android-7.1.1_r28)としてタグ打ちされているコードで確認していきます。

[Am.java](https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/cmds/am/src/com/android/commands/am/Am.java)

Am.javaは[BaseCommand](https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/core/java/com/android/internal/os/BaseCommand.java)というクラスを継承しており、`adb shell am`を実行すると呼び出されるようです。
`adb shell am`と実行すると出力されるusageも定義されています。
https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/cmds/am/src/com/android/commands/am/Am.java#L125-L368

Am.javaを実行している`am`コマンドは、以下にありました。
https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/cmds/am/am

```bash
#!/system/bin/sh
#
# Script to start "am" on the device, which has a very rudimentary
# shell.
#
base=/system
export CLASSPATH=$base/framework/am.jar
exec app_process $base/bin com.android.commands.am.Am "$@"
```
`app_process`に先程のAm.javaを引数として指定しているようです。
エミュレータを立ち上げて以下のようにコマンドを実行すると、実際に上述した`am`が呼ばれている事が確認できます。

```
adb shell which am
# /system/bin/am
```
```
adb shell cat /system/bin/am
```
`app_process`は、全てのAndroidアプリの親プロセスとなる`zygote`プロセスの実体のようです。
[Android アプリケーションが起動するまでの流れ](http://dsas.blog.klab.org/archives/52003951.html)

Am.javaを見てみると`onRun`メソッドで`am`コマンド実行時の引数をチェックしているようです。
https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/cmds/am/src/com/android/commands/am/Am.java#L371-L464
ãとはこれを追っていけば、実際にhangさせている処理に辿り着けそうです。

## ActivityManagerService.java
Am.javaの`onRun`から、さらに処理を追っていくと、実際にアプリをhangさせていそうな、ActivityManagerService.javaのhangメソッドが見つかります。

https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/services/core/java/com/android/server/am/ActivityManagerService.java#L13018-L13055

まずActivityManagerService.javaについてですが、こちらは名前の通り、全アプリのActivityを管理しているクラスで、上述した`zygote`プロセスをforkして起動される`system_server`プロセス内で実行されるようです。
[Android アプリケーションが起動するまでの流れ](http://dsas.blog.klab.org/archives/52003951.html)

エミュレータを起動して`adb shell ps`で`zygote`プロセスと`system_server`プロセスを確認してみると、上述した記事で説明されているようなプロセスの親子関係を確認できます。

```
adb shell ps | grep -e USER -e init -e zygote -e system_server 
```
<img width="443" alt="スクリーンショット 2017-03-15 9.06.42.png" src="https://qiita-image-store.s3.amazonaws.com/0/47437/c79ec046-fed9-bd4b-f1e7-75fd751baa13.png">


ActivityManagerServiceが提供する機能のいくつかは、アプリからも[ActivityManager](https://developer.android.com/reference/android/app/ActivityManager.html)クラスを使って呼び出すことができます。
例えば、実行中のプロセス情報を返す[ActivityManager#getRunningAppProcesses](https://developer.android.com/reference/android/app/ActivityManager.html#getRunningAppProcesses())の実装を見てみると以下のようになっています。

```java
public List<RunningAppProcessInfo> getRunningAppProcesses() {
    try {
        return ActivityManagerNative.getDefault().getRunningAppProcesses();
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
} 
```
https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/core/java/android/app/ActivityManager.java#L3025-L3031

`ActivityManagerNative.getDefault()`を追ってみると、実行元が同一プロセスの場合は、上述したActivityManagerServiceをそのまま返し、別プロセスから呼ばれた場合にはプロセス間通信をする必要があるため、ActivityManagerProxyを返しています。
ActivityManagerは、アプリ開発者がプロセス間通信を意識しなくても良い作りになっているようです。

```java
static public IActivityManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    }
    IActivityManager in =
        (IActivityManager)obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }

    return new ActivityManagerProxy(obj);
}
```
https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/core/java/android/app/ActivityManagerNative.java#L69-L80

## hang

少し話がそれてしまいましたが、いよいよ`hang`メソッドを見てみます。

```java
@Override
public void hang(final IBinder who, boolean allowRestart) {
    if (checkCallingPermission(android.Manifest.permission.SET_ACTIVITY_WATCHER)
            != PackageManager.PERMISSION_GRANTED) {
        throw new SecurityException("Requires permission "
                + android.Manifest.permission.SET_ACTIVITY_WATCHER);
    }

    final IBinder.DeathRecipient death = new DeathRecipient() {
        @Override
        public void binderDied() {
            synchronized (this) {
                notifyAll();
            }
        }
    };

    try {
        who.linkToDeath(death, 0);
    } catch (RemoteException e) {
        Slog.w(TAG, "hang: given caller IBinder is already dead.");
        return;
    }

    synchronized (this) {
        Watchdog.getInstance().setAllowRestart(allowRestart);
        Slog.i(TAG, "Hanging system process at request of pid " + Binder.getCallingPid());
        synchronized (death) {
            while (who.isBinderAlive()) {
                try {
                    death.wait();
                } catch (InterruptedException e) {
                }
            }
        }
        Watchdog.getInstance().setAllowRestart(true);
    }
}
``` 
https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/services/core/java/com/android/server/am/ActivityManagerService.java#L13018-L13055

メソッドを追っていくと、`death.wait()`という、いかにもhangさせていそうな処理が見つかります。
`wait`はObjectクラスのメソッドです。ドキュメントには以下のように説明されています。

```
別のスレッドがこのオブジェクトのnotify()メソッドまたはnotifyAll()メソッドを呼び出すまで、現在のスレッドを待機させます。
つまり、このメソッドの動作はwait(0)を呼び出した場合と同じです。
現在のスレッドは、このオブジェクãのモニターのオーナーでなければなりません。
スレッドはこのモニターの所有権を解放し、別のスレッドがnotifyメソッドまたはnotifyAllメソッドを呼び出してこのオブジェクトのモニター上で待機するスレッドに通知を出すまで待機します。
そのあと、スレッドはモニターの所有権を再度取得するまで待機してから実行を再開します。

引数が1つのバージョンでは、割り込みやスプリアス・ウェイクアップが発生する可能性があるので、このメソッドは常にループで使用される必要があります。

     synchronized (obj) {
         while (<condition does not hold>)
             obj.wait();
         ... // Perform action appropriate to condition
     }
```
https://docs.oracle.com/javase/jp/8/docs/api/java/lang/Object.html#wait--

`death.wait()`を実行すると、ActivityManagerServiceを実行しているスレッドが待機しそうです。
上でも少し触れていますが、ActivityManagerServiceはアプリのプロセスを管理しているサービスです。

```
Activity Manager 経由で Zygote から fork された アプリ用新規プロセスは、初期処理の段階で Activity Manager とのプロセス間通信を通じて起動対象アプリの情報を取得し、それをもとに所定のアプリをロード～実行する。
また、起動後のアプリケーションプロセスはライフサイクル管理を含め Activity Manager を中心とする文脈の中で一元管理される
```
[Android アプリケーションが起動するまでの流れ](http://dsas.blog.klab.org/archives/52003951.html)

すべてのアプリプロセスの管理者であるActivityManagerServiceが休止すると、各アプリは何もできなくなりそうです。

hang(ANR)が発生すると、端末内の以下ファイルに原因が記録されていくので、実際に確認してみます。

``` 
/data/anr/traces.txt
```

`adb shell am hang`でhangさせた後に`adb shell cat /data/anr/traces.txt`と入力して中身を見てみると、それらしい記載が見つかります。

```
"main" prio=5 tid=1 Blocked                                                                                            
  | group="main" sCount=1 dsCount=0 obj=0x7458e258 self=0xb40b4500                                                     
  | sysTid=1543 nice=-2 cgrp=default sched=0/0 handle=0xb7781c00                                                       
  | state=S schedstat=( 0 0 0 ) utm=50 stm=80 core=1 HZ=100                                                            
  | stack=0xbf3ba000-0xbf3bc000 stackSize=8MB                                                                          
  | held mutexes=                                                                                                      
  at com.android.server.am.ActivityManagerService.broadcastIntent(ActivityManagerService.java:16931)                   
  - waiting to lock <0x0f2e9f09> (a com.android.server.am.ActivityManagerService) held by thread 62                    
  at android.app.ContextImpl.sendBroadcastAsUser(ContextImpl.java:949)                                                 
  at android.app.ContextImpl.sendBroadcastAsUser(ContextImpl.java:938)                                                 
  at com.android.server.DropBoxManagerService$3.handleMessage(DropBoxManagerService.java:162)                          
  at android.os.Handler.dispatchMessage(Handler.java:102)                                                              
  at android.os.Looper.loop(Looper.java:148)                                                                           
  at com.android.server.SystemServer.run(SystemServer.java:283)                                                        
  at com.android.server.SystemServer.main(SystemServer.java:168)                                                       
  at java.lang.reflect.Method.invoke!(Native method)                                                                   
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:726)                                   
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:616)       
```

hangの原因(=ActivityManagerServiceを休止させる)は何となく分かりましたが、もう少しだけ深掘りしてみます。

実際に`wait`メソッドを実行しているのは`death`インスタンスです。
https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/services/core/java/com/android/server/am/ActivityManagerService.java#L13048

`hang`メソッド内で以下のように生成されています。

```java 
final IBinder.DeathRecipient death = new DeathRecipient() {
```
https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/services/core/java/com/android/server/am/ActivityManagerService.java#L13026

DeathRecipientとは、他プロセスの死亡を検知する仕組みとして提供されているクラスのようです。
[Android:DeathRecipientで他プロセスの死亡を検知する](http://yuki312.blogspot.jp/2013/02/androiddeathrecipient.html) 

Binder#linkToDeathで死活監視したい対象のBinderに登録できます。 
(BinderはAndroidでプロセス間通信するために使用されます)
https://developer.android.com/reference/android/os/Binder.html

`hang`メソッド内で`death`を`linkToDeath`している`who`は、Am.javaで生成されているようです。
https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/cmds/am/src/com/android/commands/am/Am.java#L1579

Am.javaは`adb shell am hang`コマンド実行プロセスで動作しているため、hangメソッド内で生成されているDeathRecipientは、`adb shell am hang`コマンド実行プロセスの死活監視をしていると言えそうです。

DeathRecipientは監視対象プロセスの死亡を検知すると、`binderDied`を呼びます。
`hang`メソッドではã`binderDied`が呼ばれると`notifyAll`を呼んでいます。

```java
final IBinder.DeathRecipient death = new DeathRecipient() {
    @Override
    public void binderDied() {
        synchronized (this) {
            notifyAll();
        }
    }
};
```
https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/services/core/java/com/android/server/am/ActivityManagerService.java#L13026-L13033

`notifyAll`は待機中のスレッドを再開します。

```
このオブジェクトのモニターで待機中のすべてのスレッドを再開します。
スレッドは、waitメソッドを呼び出すと、オブジェクトのモニターで待機します。
再開されたスレッドの処理は、現在のスレッドがこのオブジェクトのロックを解除するまでは進むことができません。
再開されたスレッドは、ほかのスレッドと同じように、このオブジェクトと同期するように積極的に競います。
たとえば、ãのオブジェクトをロックする次のスレッドになろうとする場合でも、再開されたスレッドの扱いはほかのスレッドより優勢でも劣勢でもありません。

このメソッドを呼び出すのは、このオブジェクトのモニターを所有するスレッドだけでなければいけません。スレッドがオブジェクトのモニターのオーナーになる方法については、notifyメソッドを参照してください。
```
https://docs.oracle.com/javase/jp/8/docs/api/java/lang/Object.html#notifyAll--

よって`adb shell am hang`コマンド実行プロセスがkillされれば、ActivityManagerServiceが再開するため、hang状態も解消されそうです。

実際に試してみます。
`adb shell am hang`コマンドを実行すると、以下のプロセスが起動します。
<img width="442" alt="スクリーンショット 2017-03-16 9.13.57.png" src="https://qiita-image-store.s3.amazonaws.com/0/47437/5050eb72-1e16-6280-e4fc-143f25c3a0b3.png">


これでPIDが分かったので、以下を実行します。

```
adb shell kill {{PID}}
```
実行すると、再び端末が操作を受け付けるようになりました。
コマンドを実行しているプロセスがkillされれば良いので、`Ctrl-C`でも問題無さそうです。

ちなみに、`hang`メソッド内でも以下のようにログ出力をしているため、そちらからでもコマンド実行プロセスのPIDを確認する事が可能です。

``` java
Slog.i(TAG, "Hanging system process at request of pid " + Binder.getCallingPid());
```
https://github.com/android/platform_frameworks_base/blob/android-7.1.1_r28/services/core/java/com/android/server/am/ActivityManagerService.java#L13044

`Binder.getCallingPid()`はドキュメントで以下のように説明されているため、トランザクション送信元のプロセス(=`adb shell am hang`実行コマンドプロセス)のPIDが出力されそうです。

``` 
Return the ID of the process that sent you the current transaction that is being processed.  
```
https://developer.android.com/reference/android/os/Binder.html#getCallingPid()

これで`adb shell am hang`を実行後、`adb shell reboot`しなくてもhang状態から復帰できる事が分かりました。


## さいごに
今回初めてadbコマンドのコードを読んでみましたが、知らないことが多く勉強になりました。
Android Frameworkの理解を深める意味でも、こういったコードを定期的に読んでいけると良いかもと思いました。

