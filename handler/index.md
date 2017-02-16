# Handlerのメモ（暫定）

適当なファイルに保存して、通常のgroovyコマンドで実行するだけ。

```groovy
@Grapes([
  @Grab('io.ratpack:ratpack-groovy:1.4.5'),
  @Grab('org.slf4j:slf4j-simple:1.7.20')
])

import ratpack.groovy.handling.GroovyChainAction
import ratpack.groovy.handling.GroovyContext
import ratpack.groovy.handling.GroovyHandler
import ratpack.handling.ByMethodSpec
import ratpack.handling.Context
import ratpack.handling.Handler
import ratpack.handling.InjectionHandler
import static ratpack.groovy.Groovy.ratpack

ratpack {

    serverConfig {
        threads 1
    }

    bindings {
        // bindingInstanceで何かのインスタンスを生成して渡すことで、Ratpackが自動的にそのインスタンスを管理してDIしてくれるようになる。（つまりSingleton）
        bindInstance(new PiyoGroovyHandler2())
        bindInstance(new PiyoGroovyHandler4())
        bindInstance(new Date()) // Java標準のクラスでも当然OK
        bindInstance(new PiyoChain())
    }

    // handlersの中では、GroovyChainのメソッドが利用できる。（delegateされてる）
    handlers {
        // 同じURLで、異なるHTTPメソッドで異なる処理する場合は、pathでURLを指定して、byMethodの中でそれぞれ指定する。
        path("test") {
            byMethod {
                get {
                    render "test-verbose(GET)"
                }
                post {
                    render "test-verbose(POST)"
                }
            }
        }
        // 上記の処理を無駄に冗長に書くとこうなる。
        path("test-verbose") {
            byMethod { ByMethodSpec byMethodSpec -> byMethodSpec
                .get {
                    render "test-verbose(GET)"
                }
                .post {
                    render "test-verbose(POST)"
                }
            }
        }

        // 当然、上記のように書いていくとratpack.groovyがドンドン肥大化していく。
        // そのため、実際の処理を別のクラス（Handler関連のクラスをimplements、継承したクラス）に追い出し、以下のようにして呼び出すことが出来る。
        path("piyo") {
            byMethod {
                get {context.insert(new PiyoGroovyHandler())}
                post {context.insert(new PiyoGroovyHandler())}
            }
        }


        // Handler系クラスには、その中でHTTPメソッドを指定できるので、そのようなHandlerを利用するのであれば、このように普通にpathで指定すればOK
        path("piyo2", new PiyoGroovyHandler2())
        path("piyo3", new PiyoGroovyHandler3())

        // bindInstance(new PiyoGroovyHandler2())をbindingsの中に指定しているためにアクセス毎にPiyoGroovyHandlerをnewする必要は無い！
        // Classオブジェクトを渡すことで、Ratpacが自動的にインスタンスを探して渡してくれる。
        // Ratpackは大量のアクセスをNonBlockingな処理で大量にこなせる、ということを謳っているので、そのもそも大量のアクセスをこなしたいのにアクセス毎にインスタンスをnewしているのは無駄。
        // なので基本的にはHandlerなどはbindInstanceなどでSingletonとして扱うようにするのがベター。
        path("piyo-x", PiyoGroovyHandler2)


        // この呼び出し自体は一見何の変哲もないもだけど、PiyoGroovyHandlerのhandleはDateクラスを要求している。
        // それはRatpackによって自動的にbindInstanceされたインスタンス群から自動的に合致する型の値が検索されて渡される（DI）
        path("piyo4", new PiyoGroovyHandler4())

        // 今までのやり方で分かるように、処理をHandlerに切り出すことでratpack.groovyが肥大化するのを防ぐことが出来る。
        // ただそれでもURLの指定を全てratpack.groovyの中にしなければならない、という状況に変わりはない。
        // そこで、URLの指定自体も「ChainAction」に切り出すことが出来る。
        // そのChainActionの中で、このratpack.groovyの用にURLの指定が自由に出来る。
        insert(PiyoChain)
        //insert(new PiyoChain()) // メリットは無いけど、コレもnewしてインスタンスを渡してもOK


        // 注意
        // Handlerとは直接関係ないが、「同じURLで別々のHTTPメソッドを受け取る」場合に、以下のようにbyMethodを利用せずに記述すると、ratpack.groovyの中で最初に定義されている物のみ有効になる。
        // つまりgetは405が返る
        // ---------------------------------------------
        // [koji:~]$ curl localhost:5050/piyo-error -X POST
        // piyop%
        // [koji:~]$ curl localhost:5050/piyo-error -X GET
        // Client error 405%
        // [koji:~]$
        // ---------------------------------------------
        post("piyo-error") {render "piyop"}
        get("piyo-error")  {render "piyog"}

        // これで、静的なファイルにアクセスできるようになる。
        // src/ratpack/publicがトップディレクトリに鳴って、http://localhost:5050/data/sample.txtみたいな感じでアクセエスできる。
        files { dir "public" }
    }

}

// -----------------------------------------------------------------------------
// ココ以降は利用されるクラスの定義
// -----------------------------------------------------------------------------

class PiyoGroovyHandler extends GroovyHandler {
    void handle(GroovyContext context) {
        context.render("/piyo")
    }
}

class PiyoGroovyHandler2 extends GroovyHandler {
    void handle(GroovyContext context) {
        context.byMethod {
            get {
                context.render("/piyo2(GET)")
            }
            post {
                context.render("piyo2(POST)")
            }
        }
    }
}

// GroovyHandlerじゃなくても、ratpack.handling.Handlerを実装することも出来る。
// ただし、よく見れば分かるけど、getとpostにドットを付けてちゃんとJavaライクにチェインしないといけないので若干不便。多分これはJava用。
class PiyoGroovyHandler3 implements Handler {
    void handle(Context context) {
        context.byMethod {
            it.get {
                context.render("/piyo3(GET)")
            }
            .post {
                context.render("piyo3(POST)")
            }
        }
    }
}

// handleメソッドの第1引数（Context）以降の引数は、RatpackによってbindInstance()されたものの中から自動で探されてDIされる。
// そのためには、InjectionHandlerをextendsしないと駄目
// GroovyHandlerじゃないので少し記述が冗長になる。（Javaライクになる）
// Ratpack起動時にbindInstanceしたDateがDIされるので、何度実行しても同じ日付が表示されることが確認できる。（開発時の再コンパイルが走るとその時点でリフレッシュされる）
class PiyoGroovyHandler4 extends InjectionHandler {

    void handle(Context context, Date startup) {

        // GroovyHandlerじゃないので記述が少し煩雑になるけど、もしHTTPメソッドは何でもOKなのであれば、下記の処理は、
        // context.render xxx
        // に置き換えることは可能。

        context.byMethod {
            it.post {
                context.render startup.format("yyyy/MM/dd HH:mm:ss") + "(POST)"
            }
            .get {
                context.render startup.format("yyyy/MM/dd HH:mm:ss") + "(GET)"
            }
        }
    }
}

// GroovyChainActionを継承したクラスを用意することで、URLの指定もratpack.groovyから追い出すことが出来るようになる。
// execute()の中に、ratpack.groovyのhandlers{...}に記述していた内容をそのまま書ける。
class PiyoChain extends GroovyChainAction {
    @Override
    void execute() {
        path("chain-piyo2", PiyoGroovyHandler2)
        path("chain-piyo4", PiyoGroovyHandler4)
    }
}
```