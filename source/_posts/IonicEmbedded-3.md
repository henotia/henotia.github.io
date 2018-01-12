---
title: "Ionic Project를 Android Native 내에 Embedded 시키는 방법 #3"
subtitle: "Android Fragment로 Ionic Project 실행하기"
categories: [Ionic]
tags: [Ionic, Cordova]
date: 2018-01-12 16:37:08
author: 헤노티아
img: /IonicEmbedded-1/9ql8nuci6586ko6r.png

---

> **본 post에서는 Ionic 프로젝트를 Android Fragment를 통해 실행하는 방법에 대해 설명합니다**
> **본 post의 내용은 [Ionic Project를 Android Native 내에 Embedded 시키는 방법 #1](/IonicEmbedded-1)과 [Ionic Project를 Android Native 내에 Embedded 시키는 방법 #2](/IonicEmbedded-2)에서 이어집니다**

---

# Android Native 코드 작성

전 post에서는 Activity를 통해 Ionic Project를 실행하는 방법에 대해 설명했다.
이번 post에서는 Fragment를 통해 Ionic Project를 실행하는 방법에 대해 설명한다.


# IonicFragment 생성

Ionic을 불러오는 Fragment로 `IonicFragment`를 새로 만들어준다.

``` java IonicFragment.java
public class IonicFragment extends Fragment {

}
```

IonicFragment에서는 CordovaActivity에서 Ionic Project를 불러오기까지의 과정을 직접 처리해 주어야 한다.

``` java IonigFragment.java
public class IonicFragment extends Fragment {

  private CordovaWebView appView;
  protected CordovaPreferences preferences;
  protected String launchUrl;
  protected ArrayList<PluginEntry> pluginEntries;
  protected CordovaInterfaceImpl cordovaInterface;

  public static IonicFragment newInstance() {
    return new IonicFragment();
  }

  @Nullable
  @Override
  public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, Bundle savedInstanceState) {
    View rootView = inflater.inflate(R.layout.fragment_layout, container, false);

    cordovaInterface = new CordovaInterfaceImpl(getActivity());

    if (savedInstanceState != null) {
      cordovaInterface.restoreInstanceState(savedInstanceState);
    }

    loadConfig();

    appView = makeWebView();

    createViews(rootView);

    if (!appView.isInitialized()) {
      appView.init(cordovaInterface, pluginEntries, preferences);
    }
    cordovaInterface.onCordovaInit(appView.getPluginManager());

    appView.loadUrl(launchUrl);
    return rootView;
  }

  private void createViews(View rootView) {
    // Cordova SystemWebView에 대한 Layout Parameter 설정
    RelativeLayout.LayoutParams params = new RelativeLayout.LayoutParams(
      RelativeLayout.LayoutParams.MATCH_PARENT,
      RelativeLayout.LayoutParams.MATCH_PARENT);

    appView.getView().setLayoutParams(params);

    // rootView에 SystemWebView 추가
    ((RelativeLayout) rootView).addView(appView.getView());
  }

  private CordovaWebView makeWebView() {
    return new CordovaWebViewImpl(makeWebViewEngine());
  }

  private CordovaWebViewEngine makeWebViewEngine() {
    return CordovaWebViewImpl.createEngine(getActivity(), preferences);
  }

  private void loadConfig() {
    ConfigXmlParser parser = new ConfigXmlParser();
    parser.parse(getActivity());
    preferences = parser.getPreferences();
    preferences.setPreferencesBundle(getActivity().getIntent().getExtras());
    pluginEntries = parser.getPluginEntries();
    launchUrl = parser.getLaunchUrl();
  }
}
```

IonicFragment에서 사용하는, CordovaWebView를 삽입시킬 `fragment_layout.xml`을 함께 작성한다
해당 layout 아래에 **자바코드를 사용해 직접** `SystemWebView`를 삽입시킨다.

```xml fragment_layout.xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
                android:layout_width="match_parent"
                android:layout_height="match_parent">

</RelativeLayout>
```

위의 코드는 이전 IonicActivity를 Fragment 형태로 변경한 내용이다.


# Plugin 통신을 위한 코드 추가 (MainActivity.java)
Plugin은 `onActivityResult()`를 통해 Plugin 결과값을 Activity로 보낸다.
Fragment의 경우 Activity로 보내진 onActivityResult()의 값을 직접 받지 못하고 Fragment를 감싸고 있는 Activity에서 가져와야 한다.

따라서 Fragment를 감싸고 있는 MainActivity에서 onActivityResult의 Override를 통한 추가 구현이 필요하다

``` java MainActivity.java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

  // 위에서 작성한 IonicFragment의 인스턴스를 만들어준다.
  IonicFragment ionicFragment = IonicFragment.newInstance();

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    ...
  }

  @Override
  public void onClick(View v) {
    ...
  }

  // onActivityResult 구현 부분
  @Override
  protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    ionicFragment.onActivityResult(requestCode, resultCode, data);
  }
}

```

이후 ionicFragment에서도 onActivityResult()의 결과값을 받아 처리할 수 있도록 코드를 작성한다.

# Plugin 통신을 위한 코드 추가 (IonicFragment.java)

MainActivity에서 넘겨받는 onActivityResult()의 값을 IonicFragment의 cordovaInterface가 받아서 처리할 수 있도록 해주어야 한다.

Plugin쪽과 관련된 부분은 `loadConfig()`메소드 아래에 기입한다

``` java IonicFragment.java
public class IonicFragment extends Fragment {
  ... // end of loadConfig()

  @Override
  public void startActivityForResult(Intent intent, int requestCode) {
    cordovaInterface.setActivityResultRequestCode(requestCode);
    super.startActivityForResult(intent, requestCode);
  }

  @Override
  public void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    cordovaInterface.onActivityResult(requestCode, resultCode, data);
  }

  @Override
  public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
    try {
      cordovaInterface.onRequestPermissionResult(requestCode, permissions, grantResults);
    } catch (JSONException e) {
      e.printStackTrace();
    }
  }

  public void onSaveInstanceState(Bundle outState) {
    cordovaInterface.onSaveInstanceState(outState);
    super.onSaveInstanceState(outState);
  }

  @Override
  public void onConfigurationChanged(Configuration newConfig) {
    super.onConfigurationChanged(newConfig);
    if (appView == null) return;
    PluginManager pm = appView.getPluginManager();
    if (pm != null) {
      pm.onConfigurationChanged(newConfig);
    }
  }
}

```

# IonicFragment 실행하기
IonicFragment를 실행하기 위해 `MainActivity.java`와 `activity_main.xml`을 다음과 같이 수정해준다.

``` java MainActivity.java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

  // 위에서 작성한 IonicFragment의 인스턴스를 만들어준다.
  IonicFragment ionicFragment = IonicFragment.newInstance();

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    ...
  }

  @Override
  public void onClick(View v) {
    switch (v.getId()) {
      case R.id.startIonicActivity:
        Intent intent = new Intent(this, IonicActivity.class);
        startActivity(intent);
        break;
      case R.id.startIonicFragment:
        Button btns[] = {
          findViewById(R.id.startIonicActivity),
          findViewById(R.id.startIonicFragment)
        };

        // 버튼들이 있으면 Fragment 화면 상단에 보여지므로 전부 상태를 View.GONE으로 변경
        for (Button btn : btns) {
          btn.setVisibility(View.GONE);
        }

        FragmentManager fm = getFragmentManager();
        FragmentTransaction tx = fm.beginTransaction();
        tx.add(R.id.cordovaWebViewFrag, ionicFragment);
        tx.commitAllowingStateLoss();
        break;
    }
  }

  // onActivityResult 구현 부분
  @Override
  protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    ...
  }
}
```

``` xml activity_main.xml
<!-- activity_main.xml에 Button과 RelativeLayout 추가 -->
    ...

  <!-- Fragment 호출을 위한 Button -->
  <Button
    android:text="Ionic Project 실행(Fragment)"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:id="@+id/startIonicFragment"
    android:onClick="onClick"
    app:layout_constraintTop_toBottomOf="@+id/startIonicActivity"/>

  <!-- Fragment가 들어갈 Layout -->
  <RelativeLayout
    android:id="@+id/cordovaWebViewFrag"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:layout_constraintTop_toBottomOf="@+id/startIonicFragment"
    android:layout_alignParentStart="true"/>

```

이후 MainActivity 내의 Fragment 실행 버튼을 누르면 Fragment의 형태로 IonicProject가 동작한다.























