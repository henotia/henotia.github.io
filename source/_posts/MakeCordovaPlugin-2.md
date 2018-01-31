---
title: "Cordova Plugin 만들기 #2"
categories: [Ionic]
tags: [Cordova, Android, Plugin]
date: 2018-01-31 11:07:05
author: 헤노티아
img: /MakeCordovaPlugin-1/cordova_bot.png

---

> **본 post에서는 Cordova Plugin 작성에 대해 설명합니다**
> **본 post의 내용은 [Cordova Plugin 만들기 #1](/MakeCordovaPlugin-1)에서 이어집니다**

---

## Plugin Structure

Cordova Plugin을 이루고 있는 골자는 크게 다음과 같다.

1. 사용하고자 하는 Native 기능이 담긴 `Native Interface`
2. `cordova.exec`를 통해 Native platform과 통신하는 `Javascript Interface`
3. 플러그인의 명세가 담긴 `config`파일

![0.zdjr5962sa](0.zdjr5962sa.png)

이후 위의 내용을 바탕으로 사용자에게 텍스트를 입력받아 그대로 출력하는 `Echo` 플러그인을 작성해본다.


## Native Interface
Cordova Project에서 사용하는 Native 기능을 작성한다.

본 파일은 CordovaPlugin을 extend하는 자바 클래스로서, CordovaPlugin으로부터 메소드를 Override 받아 필요한 기능을 작성한다.

``` java
import org.apache.cordova.CordovaPlugin;

public class myPlugin extends CordovaPlugin {}
```

### 생명주기
`myPlugin` 플러그인 객체는 Cordova Webview의 수명을 따른다.
기본적으로 플러그인 객체는 JavaScript의 호출 시점에 인스턴스화 되며, `plugin.xml`에서 별도의 옵션을 주면 시작과 동시에 인스턴스화 시킬 수 있다.

### 플러그인 작성
`JavaScript Interface`에서 플러그인 호출이 일어나면 `Native Interface`의 `execute` 메소드로 전달이 된다.
execute 메소드는 CordovaPlugin에서 Override를 받아 작성하며, 다음과 같이 작성할 수 있다.

``` java
package com.test.myplugin;

import org.apache.cordova.CallbackContext;
import org.apache.cordova.CordovaPlugin;
import org.json.JSONArray;
import org.json.JSONException;

public class myPlugin extends CordovaPlugin {
  @Override
  public boolean execute(String action, JSONArray args, CallbackContext callbackContext) throws JSONException {

    // plugin.xml에 맵핑되어있는 action과 JavaScript Interface에서 전달받는 action이 동일할 때 수행한다
    if (action.equals("echo")) {
      // JavaScript Interface에서 전달받은 매개변수를 받아 저장한다
      String message = args.getString(0);

      // JavaScript Interface로 보낼 Callback 함수
      if (message != null && message.length() > 0) {
        callbackContext.success(message);
      } else {
        callbackContext.error("Echo Fail");
      }

      return true;
    }

    return false;
  }
}
```

플러그인 코드 작성이 끝났으면 이를 `plugin.xml`에 `Native Interface`에 대한 명세를 작성한다

## Plugin 명세 작성
Plugin에 대한 명세는 Plugin 폴더 상단에 위치한 `plugin.xml`(파일 명이 다를 수 있음)에 작성한다.

``` xml
<?xml version='1.0' encoding='utf-8'?>
<plugin id="com.test.myplugin" version="0.0.1" xmlns="http://apache.org/cordova/ns/plugins/1.0">
  <name>myPlugin</name>

  <!-- src에는 JavaScript Interface 파일이 위치한 곳을 기입한다 -->
  <js-module name="myPlugin" src="www/myPlugin.js">
    <!-- clobbers에 target으로 작성된 부분이 추후 JavaScript로 호출시 사용된다 -->
    <!-- target은 window에 객체로 등록되며, 아래의 경우는 window.cordova.plugins.myPlugin으로 호출 할 수 있다-->
    <clobbers target="cordova.plugins.myPlugin"/>
  </js-module>

  <!-- platform 별 명세를 작성한다 -->
  <platform name="android">
    <config-file parent="/*" target="res/xml/config.xml">
      <feature name="myPlugin">
        <!-- value에는 작성한 플러그인이 위치할 곳의 package와 class명을 적어준다 -->
        <param name="android-package" value="com.test.myplugin.myPlugin"/>
      </feature>
    </config-file>
    <config-file parent="/*" target="AndroidManifest.xml"></config-file>
    <!-- source-file의 src에 위치한 파일을 build시 target-dir의 위치에 저장한다 -->
    <source-file src="src/android/myPlugin.java" target-dir="src/com/test/myplugin"/>
  </platform>
</plugin>
```

명세를 작성한 다음에는 `JavaScript Interface`를 수정해

## JavaScript Interface

JavaScript로 Native Interface를 호출 할 수 있도록 JavaScript Interface를 수정해준다.

``` javascript
var exec = require('cordova/exec');

exports.echo = function(args, success, error) {
    exec(success, error, "myPlugin", "echo", [args]);
}

```

`exec`는 platform 추가시 `platform_www/cordova.js`에 위치한다.
JavaScript Interface를 좀 더 전문적으로 사용하기 위해서는 확인이 필요하지만 본 포스트에서는 넘어가도록 한다.

`exports.echo` 를 통해 등록한 함수를 JavaScript로 호출할 수 있다.
함수 내부의 exec는 총 다섯가지의 매개변수를 갖는다.

``` javascript
function(Arguments, SuccessCallback, ErrorCallback) {
  exec(SuccessCallback, ErrorCallback, "FeatureName", "ActionName", [Arguments])
}
```
* SuccessCallback: `Native Interface`에서 `callbackContext.success`를 통해 전달받는 콜백함수
* ErrorCallback: `Native Interface`에서 `callbackContext.error`를 통해 전달받는 콜백함수
* "FeatureName": `config`파일 내에 `<feature name="" >`에서 사용되는 name을 기입한다
* "ActionName": `Native Interface`에서 `execute`메소드에서 전달받아 사용할 Action 명을 기입한다
* \[Arguments]: `Native Interface`로 보낼 매개변수 (반드시 배열로 보내야 함)

위의 매개변수를 바탕으로 `JavaScript Interface`에서 `Native Interface`로 매개변수와 호출할 함수를 전달한다.


## package.json 작성
플러그인 작성이 끝났으면 plugin에서 사용할 package.json을 작성해 주어야 한다
이전 포스트에서 사용했던 `plugman` 모듈을 사용해 package.json을 작성해 줄 수 있다.

`myPlugin` 디렉토리에서 아래와 같은 명령어를 사용한다

``` bash
$ plugman createpackagejson . # .을 반드시 써주어야 한다
```

이후 나오는 내용을 본인에게 맞게 수정해주면 된다.

![0.wxaoyebsm3c](0.wxaoyebsm3c.png)
예시에서는 별다른 수정없이 엔터만을 쳐서 넘겼다.

## 플러그인 설치

위의 내용을 잘 따라왔다면 프로젝트 디렉토리는 다음과 같을 것이다.

![0.i8yaeifavc9](0.i8yaeifavc9.png)

이후 Android Platform을 설치해준다.

``` bash
$ cordova platform add android
```

platform이 추가되었으면 다음의 명령어를 통해 `myPlugin`을 설치해준다

``` bash
$ cordova plugin add myPlugin --link
```
명령어 실행 후 `plugins`디렉토리 아래에 `com.test.myplugin`으로 디렉토리가 하나 생기고 그 안에 우리가 작성한 `myPlugin`의 내용이 담겨있다
`plugin.xml`의 `<plugin id="">`의 값이 디렉토리의 이름으로 지정이 된다.
따라서 이름을 바꾸고 싶으면 위의 부분을 바꾸어 주면 된다.

![0.5tqssi4ezat](0.5tqssi4ezat.png)

마찬가지로 아래의 사진에서 `plugin.xml`의 `<source-file src="" target-dir="">`에서 target-dir의 값이 platform 내에 들어간 것을 확인 할 수 있다
![0.6enw08uhwaj](0.6enw08uhwaj.png)


## 플러그인 실행

플러그인 설치까지 완료 되었으면, 실제 Cordova App에서 플러그인이 정상적으로 작동하는지 테스트 해보자

### 플러그인 호출 코드 작성

프로젝트 루트 내에 있는 `www/`아래의 `index.html`을 다음과 같이 수정해준다.

``` html
~~~
<div class="app">
  <h1>Apache Cordova</h1>
  <div id="deviceready" class="blink">
    <p class="event listening">Connecting to Device</p>
    <p class="event received">Device is Ready</p>
  </div>
  <div>
    <input type="text" id="echo" placeholder="Echo">
    <button id="send">전송</button>
  </div>
</div>
~~~
```

이어서 `www/`아래의 `js/index.js`에 아래의 내용을 추가해준다
`button#send`를 클릭하면 input의 값을 plugin으로 보내고, callback받은 값을 alert으로 띄우는 코드이다
``` js
~~~

app.initialize();

document.getElementById('send').addEventListener('click', sendInput);

function sendInput() {
    var input = document.getElementById('echo').value,
        success = function(v) { alert(v); },
        error = function(v) { alert(v); };

    cordova.plugins.myPlugin.echo(input, success, error);
}
```

### 테스트
작성이 모두 끝났으면 device를 pc에 연결하고 cordova project를 실행한다

``` bash
$ cordova run android
```

정상적으로 실행이 되면 기기에서 수정된 html을 확인 할 수 있다
<img src="0.upajtrhcqp.png" width="300"/>


Input창에 임의의 텍스르를 입력하고 전송을 누르면 Native Interface를 거쳐 callback으로 받아와 화면에 보여준다
<img src="0.c0rkuz5eovu.png" width="300"/>

정상적으로 플러그인이 연결된 것을 확인 할 수 있다.

---

## 플러그인 삭제

> 본 섹션에서 설명하는 내용은 여지껏 작업한 `myPlugin` 프로젝트를 바탕으로 설명합니다

플러그인 개발을 하다보면 설치, 수정, 삭제 등 다양한 시도를 해보게 된다.
하지만 설치된 플러그인을 제대로 지우지 않는 경우 복잡하게 꼬여 문제가 될 수 있다.
이 번외 섹션에서는 플러그인을 삭제할 경우 어느부분을 어떻게 지워야 하는지에 대해 설명한다.

### Cordova 명령어 사용
Cordova-CLI에서는 plugin을 지우는 `cordova plugin rm <plugin_id>` 명령어를 제공한다.
해당 명령어로 플러그인을 지우기 위해서는 우선 플러그인의 ID가 필요하다.
플러그인의 ID는 `cordova plugin ls`로 확인이 가능하다.

![0.ery5nr8fyrm](0.ery5nr8fyrm.png)

위의 내용을 바탕으로 `com.test.myplugin` 플러그인을 지우면 다음과 같은 문제가 생긴다

![0.t0xml3cttyj](0.t0xml3cttyj.png)

`com.test.myplugin`의 `myPlugin`버전을 찾을수 없다는 내용인데 package.json을 확인해보면
![0.zfp86y9rfbf](0.zfp86y9rfbf.png)

`com.test.myplugin`의 버전이 `myPlugin`의 형태로 되어있는 것을 확인할 수 있다.
위의 명령어 실행이후로, 추가적인 삭제가 필요한 부분을 하나씩 찾아나가자

### 프로젝트 내부

프로젝트 내부에서 설치된 플러그인은 아래의 위치에 각각 저장이 된다
* node_module
![0.p9qaih3axc](0.p9qaih3axc.png)
* plugins/fetch.json
![0.fhptplthk28](0.fhptplthk28.png)
* config.xml
![0.ii7j6ka82l](0.ii7j6ka82l.png)
* package.json
![0.yctaadvv1l](0.yctaadvv1l.png)

위의 내용들을 전부 지워주면 된다.

* node_module
단순히 삭제만 해주면 된다
* plugins/fetch.json
``` json
{}
```
* config.xml
`<plugin name="com.test.myplugin" spec="myPlugin" />`을 지워준다
* package.json
cordova-plugins와 dependencies에 작성된 부분을 지워준다

### 프로젝트 외부

`cordova plugin add --link`를 통해 설치한 플러그인은 npm 전역에도 설치가 된다

윈도우 기준으로
`%appdata%/npm/node_modules`로 들어가서
![0.c8p1gwvmmxw](0.c8p1gwvmmxw.png)
본인이 설치한 플러그인에 해당 되는 부분을 삭제해준다.


---

Cordova Plugin 만드는 기본적인 방법에 대해 설명했다.

본 포스트에서는 수정을 최소화 하고 동작에 중점을 두고 작업했기에 이름이나 ID 등이 마음에 들지 않을 수 있다.

그 부분을 수정하고 적용해 보는건 본인의 몫으로 남긴다.


