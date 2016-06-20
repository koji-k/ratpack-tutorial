# Hello World

まずはやっぱりハローワールドですよね。  
さて、Ratpackなのですが、そのシンプルな構成からいつくかの実行方法を選ぶことが出来ます。  
その中でもお勧めなのが、[Gradle](http://gradle.org/)の機能を利用して実行するか、[Groovy](http://www.groovy-lang.org/)自体の機能を利用して実行するかのどちらかになります。  
まずこのHelloWorldでは、Groovyバージョン、以降の入門、チュートリアル記事ではGradleバージョンを使ってRatpackに入門していきます。  

ということで、まずはGroovyをインストールしておきます。  
Groovyのインストール方法はとても簡単で、Linux、Macの場合は[SDKMAN](http://sdkman.io/)をインストールしてから、`sdk install groovy`を実行するだけです。  
Windowsの場合はGroovyのインストーラがあるので、そちらを利用するのが一番簡単だと思われます。
何れにせよ、Java（8以降）をインストールして、JAVA_HOMEを設定しておいてください。  
この辺りは[Apache Groovyチュートリアルのinstall記事を参照してください。](http://koji-k.github.io/groovy-tutorial/startup/install.html)  

最終的に、コンソール、コマンドプロンプトで以下のコマンドを実行して、インストールされているGroovyのコマンドが表示されればOKです。

```
[koji:ratpack]$ groovy -e "println GroovySystem.version"
2.4.7
[koji:ratpack]$ 
```

さて、準備出来ました！  
既に述べたように、最終的にはGradleをベースにしてRatpackアプリケーションを開発していくのがベターですが、シンプルなAPIサーバを構築するだけなのであれば、Groovyがあれば十分です。  
コレは大げさでも何でも無くて、Groovyがあれば **Ratpackのダウンロード、インストールすら必要ないのです。**  
Ratpack使うのに何故Ratpackが必要ないんだ？と思われるかも知れませんが、Groovyの`Grab`という機能を使えば、Ratpackのダウンロード、設定まで **全てGroovyが自動で面倒を見てくれます。**  
では、HelloWorldプログラムでそのGroovyの威力を見てみましょう！

まず、どこでもいいので以下のコードを `hello-ratpack.groovy`という名前で保存してください。

```
@Grapes([
  @Grab('io.ratpack:ratpack-groovy:1.3.3'),
  @Grab('org.slf4j:slf4j-simple:1.7.12')
])
import static ratpack.groovy.Groovy.ratpack

ratpack {
    handlers {
        path {
            render "Hello World!"
        }
    }
}
```

では、`groovy`コマンドを使って、このコードを実行してみましょう。  

```
groovy hello-ratpack.groovy
```
初めて実行する際は、Groovyが必要なライブラリ（当然Ratpack :))したりするので、少し時間がかかります。  
最終的に、コンソール上には以下のようなメッセージが出力されます。

```
[koji:ratpack]$ groovy hello-ratpack.groovy 
[main] INFO ratpack.server.RatpackServer - Starting server...
[main] INFO ratpack.server.RatpackServer - Building registry...
[main] INFO ratpack.server.RatpackServer - Ratpack started (development) for http://localhost:5050
```

[http://localhost:5050](http://localhost:5050)でRatpackが起動したというメッセージが出ています。  
では、ブラウザで実際にアクセスしてみてください。  
Hello Worldが実際に画面に出力されていますね！  
では、先ほどの`groovy`コマンドを使ってRatpackを起動したコンソール画面に戻って、`ctrl-c`を押して、Ratpackを終了させましょう。  
終了したら、プログラムを少し追加してみます。  

```
@Grapes([
  @Grab('io.ratpack:ratpack-groovy:1.3.3'),
  @Grab('org.slf4j:slf4j-simple:1.7.12')
])
import static ratpack.groovy.Groovy.ratpack

ratpack {
    handlers {
        path {
            render "Hello World!"
        }
        path(":name") {
            render "Hello ${pathTokens.name}!"
        }
    }
}
```

`groovy hello-ratpack.groovy`を実行して、再度Ratpackを起動させます。  
起動したら、[http://localhost:5050/koji](http://localhost:5050/koji)にアクセスしてみてください。  
URLの`koji`の部分が利用されて、`Hello koji!`と表示されています。  
`koji`の部分を`hoge`など任意の値に変奥してアクセスすると、ちゃんと表示内容が変わります。  

## まとめ
いかがでしょうか。Groovyが実行できる環境があれば、その他のライブラリどころか、Ratpack本体すら手動でダウンロードすること無く、 **Groovyを利用してたった1ファイルでアプリケーションサーバを構築することが出来ました。**  
このお手軽さこそがGroovyとRatpackの真骨頂です。  
とは言え、今回プログラムを修正したら一旦Ratpackを再起動しなければならなかったように、この方法だとちょっと面倒くさい面も有ります。  
そこで、今後はプログラムを修正したらその修正内容を自動で再ロードしてくれたり、より複雑なコードを書いたりした際にクラスパスを自動で通してくれたりと何かと便利なGradleを利用する方式でRatpackを利用する方法で説明してきます。