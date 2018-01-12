---
title: "Ionic Project를 Android Native 내에 Embedded 시키는 방법 #2"
subtitle: "Android Activity로 Ionic Project 실행하기"
categories: [Ionic]
tags: [Ionic, Cordova]
date: 2018-01-11 17:57:43
author: 헤노티아
img: /IonicEmbedded_1/9ql8nuci6586ko6r.png
---

> **본 post에서는 Ionic 프로젝트를 Android Activity를 통해 실행하는 방법에 대해 설명합니다**
> **본 post의 내용은 [Ionic Project를 Android Native 내에 Embedded 시키는 방법 #1](/IonicEmbedded_1)에서 이어집니다**

---

# Android Native 코드 작성

전 post에서는 Android Project에서 Ionic Project를 실행하기위해 필요한 설정을 준비하는 작업을 했다.
이번 post에서는 기존의 설정을 바탕으로, Android Activity에서 Ionic Project를 불러오는 방법에 대해 설명한다

# IonicActivity 생성

Ionic을 불러오는 Activity로 `IonicActivity`를 새로 만들어준다.
본 Activity는 CordovaActivity를 상속받아, Cordova에서 처리하는 작업을 override 한다.

``` java IonicActivity.java
import org.apache.cordova.CordovaActivity;

public class IonicActivity extends CordovaActivity {

}
```

IonicActivity에서 override 해서 처리할 메소드는 `onCreate()`, `makeWebView()`, `createViews()` 총 세가지로 다음과 같이 작성해준다

``` java onCreate()
// onCreate
    ...
  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    setContentView(R.layout.cordova_layout);
    super.init();

    loadUrl(launchUrl);
  }
    ...

```
`onCreate()`에서는 임의로 사용하기 위해 만든 layout(`cordova_layout`)을 사용하기 위함이다.


``` java makeWebView()
// makeWebView
    ...
  @Override
  protected CordovaWebView makeWebView() {
    SystemWebView appView = (SystemWebView) findViewById(R.id.cordovaWebView);
    return new CordovaWebViewImpl(new SystemWebViewEngine(appView));
  }
    ...
```
`makeWebView()`는 CordovaWebView를 내가 작성한 레이아웃으로 사용하기 위해 반드시 필요한 작업이다

```java createViews()
// createViews
    ...
  @Override
  protected void createViews() {
    // do nothings
  }
    ...
```
`createViews()`는 CordovaActivity내의 createViews()에서 사용하는 setContentView()를 피하기 위해서 사용한다. Override를 해주지 않는 경우 에러가 발생한다.


위의 세가지를 모두 작성해준 코드는 다음과 같다.
``` java IonicActivity
public class IonicActivity extends CordovaActivity {

  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    setContentView(R.layout.cordova_layout);
    super.init();

    loadUrl(launchUrl);
  }

  @Override
  protected CordovaWebView makeWebView() {
    SystemWebView appView = (SystemWebView) findViewById(R.id.cordovaWebView);
    return new CordovaWebViewImpl(new SystemWebViewEngine(appView));
  }

  @Override
  protected void createViews() {
    // do nothings
  }
}
```

위의 코드를 작성했으면, IonicActivity를 AndroidManifest.xml에 등록한다.

# AndroidManifest.xml
``` xml
    ...
    <application ...>
      <activity android:name=".MainActivity">
        ...
      </activity>
      <activity android:name=".IonicActivity" />         <!-- 새로 추가 -->
    </application>
    ...
```

# cordova_layout 생성
layout은 `res/layout`아래에 작성해주면 된다.
해당 디렉토리에 `cordova_layout.xml`로 layout파일을 생성하고 아래와 같이 작성한다.

```xml cordova_layout.xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout
  xmlns:android="http://schemas.android.com/apk/res/android"
  android:layout_width="match_parent"
  android:layout_height="match_parent">

  <org.apache.cordova.engine.SystemWebView
    android:id="@+id/cordovaWebView"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
  </org.apache.cordova.engine.SystemWebView>

</android.support.constraint.ConstraintLayout>
```

`<org.apache.cordova.engine.SystemWebView/>` 부분이 실제 Ionic Project가 보여질 View이다.



# IonicActivity 실행하기
IonicActivity를 실행해주기 위해 `MainActivity.java`와 `activity_main.xml`을 다음과 같이 수정해준다

``` java MainActivity.java
// MainActivity
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
  }

  @Override
  public void onClick(View v) {
    switch (v.getId()) {
      case R.id.startIonicActivity:
        Intent intent = new Intent(this, IonicActivity.class);
        startActivity(intent);

    }
  }
}
```

``` xml activity_main.xml
<!-- activity_main.xml에 Button 추가 -->
    ...
  <Button
    android:text="Ionic Project 실행"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:id="@+id/startIonicActivity"
    android:onClick="onClick"/>
    ...
```

수정이 끝나면, MainActivity 내의 버튼을 클릭해 IonicActivity를 실행, IonicProject를 불러올 수 있다.