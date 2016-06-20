# 初めてのAPIサーバ

ココではサンプルとして、現在の日時をJSONで返すAPIサーバを構築していきます。  
最終的には、指定されたタイムゾーンで日時を計算して返すようなちょっと便利なAPIにしていきましょう。
前提条件としてGradleをインストールしておいてください。  
Gradleが入っていれば実はGroovyすら必要ありません。  

## 準備

まずは必要なディレクトリを作成します。

```
[koji:ratpack]$ mkdir ratpackapp
[koji:ratpack]$ cd ratpackapp 
[koji:ratpackapp]$ mkdir -p src/ratpack/templates
[koji:ratpackapp]$ mkdir -p src/ratpack/public/images
[koji:ratpackapp]$ mkdir -p src/ratpack/public/scripts
[koji:ratpackapp]$ mkdir -p src/ratpack/public/styles
[koji:ratpackapp]$ mkdir -p src/ratpack/public/lib
[koji:ratpackapp]$ mkdir -p src/main/groovy
[koji:ratpackapp]$ mkdir -p src/src/groovy

```

Gradleの設定ファイルを作成します。

`build.gradle`

```
buildscript {
  repositories {
    jcenter()
  }
  dependencies {
    classpath "io.ratpack:ratpack-gradle:1.3.3"
  }
}

apply plugin: "io.ratpack.ratpack-groovy"
apply plugin: "idea"

repositories {
  jcenter()
}

dependencies {
  runtime "org.slf4j:slf4j-simple:1.7.12"
}
```

以下のファイルを作成します。  
これは、[Hello World](helloworld.html)で作成した`hello-ratpack.groovy`と同じものです。  
先頭から、@Grapes、@Grabが無くなってシンプルになっていますね。

`src/ratpack/Ratpack.groovy`

```
import static ratpack.groovy.Groovy.ratpack

ratpack {
    handlers {
        path {
            render "Hello World!"
        }
        path(":name") {
            render "Hello $pathTokens.name!"
        }
    }
}
```

これで準備は整いました！  
現在、ディレクトリ構成は以下のようになっているはずです。

```
ratpackapp
├── build.gradle
└── src
    ├── main
    │   └── groovy
    ├── ratpack
    │   ├── Ratpack.groovy
    │   ├── public
    │   │   ├── images
    │   │   ├── lib
    │   │   ├── scripts
    │   │   └── styles
    │   └── templates
    └── src
        └── groovy
```

## 実行
`gradle run -t`を実行します。

```
[koji:ratpackapp]$ gradle run -t
Continuous build is an incubating feature.
:compileJava UP-TO-DATE
:compileGroovy UP-TO-DATE
:processResources
:classes
:configureRun
:run
[main] INFO ratpack.server.RatpackServer - Starting server...
[main] INFO ratpack.server.RatpackServer - Building registry...
[main] INFO ratpack.server.RatpackServer - Ratpack started (development) for http://localhost:5050

BUILD SUCCESSFUL

Total time: 9.313 secs

Waiting for changes to input files of tasks... (ctrl-d to exit)

```

これで、[http://localhost:5050](http://localhost:5050)にアクセスするとちゃんとRatpackが起動していることが分かります。  
では、ソースコードの`render "Hello World!"`の部分を`render "こんにちわRatpack！"`に変更して保存しましょう。  
保存すると、Gradleが自動的にファイルの変更を検知してくれて、コンパイル&再ロードしてくれます。  
少し待つ、おｍしくはGradleを実行しているコンソールでGradleが再ビルドした際のメッセージが流れたら、再度[http://localhost:5050](http://localhost:5050)にアクセスしてみましょう。  
自動で変更内容が反映されて、画面には **こんにちわRatpack！** と表示されていますね。  
このように、Gradleを利用するとソースコードの修正が自動で繁栄冴えれるようになり、必要なライブラリ（当然Ratpack本体も）のダウンロード、設定をGradleがGroovyの代わりに受け持ってくれます。  

## 実際のAPIを追加する
さて、では実際に日時を返すAPIを追加しましょう。  

`Ratpack.groovy`を以下のように修正します。


```
import ratpack.jackson.Jackson

import static ratpack.groovy.Groovy.ratpack
import static ratpack.groovy.Groovy.groovyTemplate
import ratpack.groovy.template.TextTemplateModule

ratpack {

    bindings {
        module TextTemplateModule // for groovyTemplate
    }

    handlers {
        path {
            render "こんにちわRatpack!"
        }

        path("date") {
            String date = new Date().format("yyyy/MM/dd HH:mm:ss")
            render Jackson.json(["date":date])
        }
    }
}
```

はい、出来ました！コレでだけで完了です！
実際に[http://localhost:5050/date](http://localhost:5050/date)にアクセスしてみてください。ちゃんとJSONで現在日時が返されているのが解ると思います。  
本当にJSON?renderしてるからテキストで返されてない？と思われる方も居るかもしれません。  
その場合はcurlコマンドで確認してみましょう。

```
[koji:ratpack]$ curl -X GET -i localhost:5050/date        
HTTP/1.1 200 OK
content-type: application/json
content-length: 30
connection: keep-alive

{"date":"2016/06/20 15:00:45"}%                                        
```

ちゃんと`content-type`が`application/json`になっていますね！

## 参考：テンプレートを使う

`src/ratpack/Ratpack.groovy`

```
import static ratpack.groovy.Groovy.ratpack
import static ratpack.groovy.Groovy.groovyTemplate
import ratpack.groovy.template.TextTemplateModule

ratpack {

    bindings {
        module TextTemplateModule // for groovyTemplate
    }

    handlers {
        path {
            render "こんにちわRatpack!"
        }
        path("showTemplate") {
            render groovyTemplate("index.html")
        }
    }
}
```

`src/ratpack/public/scripts/sample.js`

```
$(function(){
    $("#sample").text("This text is filled by jQuery");
});
```

`src/ratpack/public/styles/sample.css`

```
h1 {
    color:#abcdef;
}
```

`src/ratpack/templates/index.html`

```
<!DOCTYPE html>
<html>
<head>
<meta name="robots" content="noindex,nofollow" />
<meta charset="UTF-8">
<title>This is Template</title>
<!--** jQuery **-->
<script src="http://ajax.aspnetcdn.com/ajax/jQuery/jquery-1.11.1.min.js"></script>
<!--** load CSS **-->
<link rel="stylesheet" type="text/css" href="/styles/sample.css">
<script type="text/javascript" src="/scripts/sample.js"></script>
</head>
<body>
    <h1>Hello Ratpack Template</h1>
    <div id="sample"></div>
</body>
</html>

```

上記のRatpack.groovyの修正、html、JavaScript、CSSの追加ができたら、[http://localhost:5050/showTemplate](http://localhost:5050/showTemplate)にアクセスしてみましょう。  
ちゃんとJavaScript、CSSが読み込めていることが確認できます。

なおディレクトリ構成は以下のようになrっています。


```
ratpackapp
├── build.gradle
└── src
    ├── main
    │   └── groovy
    ├── ratpack
    │   ├── Ratpack.groovy
    │   ├── public
    │   │   ├── images
    │   │   ├── lib
    │   │   ├── scripts
    │   │   │   └── sample.js
    │   │   └── styles
    │   │       └── sample.css
    │   └── templates
    │       └── index.html
    └── src
        └── groovy
```