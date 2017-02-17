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
import ratpack.func.Action
import ratpack.handling.Chain
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
                    render "test(GET)"
                }
                post {
                    render "test(POST)"
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


        // このように、context#insertに渡されるHandlerが複数ある場合、それぞれのHandlerの中でcontext#nextを実行しないと、その時点で処理が終了する。
        // insertに渡された順番で、各Handlerがそれぞれ実行されていく（ちゃんとnextを書いていれば。）
        // 例えば、Handler1の中のnextを消すと、Handler1で処理が終了する。（クライアントには404が返る）
        // insertに指定されたHandlerたちは、それぞれ最終的にContext#next()を実行して、自分の後に指定されているHandlerに処理を移譲していく感じ。
        get("handler-chain1") {
            insert(
                    new Handler1(),
                    new Handler2(),
                    new Handler3()
            )
        }

        // 複数のinsertを同じ動作になるっぽいけど、もしかしたらそれぞれ別のスレッドになったりして効率が悪いかも？
        // 基本的には上記のinsertに複数のHandlerを渡す方式のほうが良さそう。
        get("handler-chain2") {
            insert(new Handler4())
            insert(new Handler5())
            insert(new Handler6())
        }

        // Handler側に工夫すれば、それぞれ入れ子にすることも出来る。
        // 実行結果は
        // a
        // a_1
        // a_2
        // b
        // b_1
        // b_1_1
        // b_1_2
        // b_2
        // This is Handler 8
        // となる。（Ratpack側のコンソールで確認できる）
        // この実行結果と、Handler7#handle()を見れば分かるとおり、Context#insertは、次のHandlerを登録しつつ、自動的にContext#nextを読んでいる。
        get("handler-chain3") {
            insert(
                new Handler7("a",
                    new Handler7("a_1"),
                    new Handler7("a_2"),
                ),
                new Handler7("b",
                    new Handler7("b_1",
                        new Handler7("b_1_1"),
                        new Handler7("b_1_2"),
                    ),
                    new Handler7("b_2")
                ),
                new Handler8()
            )
        }


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

// 使い方は同じだけど、上記のGroovyChainActionはこのようにAction<Chain>を実装することで代用できる。
class PiyoChain2 implements Action<Chain> {
    @Override
    void execute(Chain chain) {
        chain.path("chain2-piyo2", PiyoGroovyHandler2)
        chain.path("chain2-piyo4", PiyoGroovyHandler4)
    }
}

class Handler1 extends GroovyHandler {
    void handle(GroovyContext context) {
        println "This is Handler 1"
        context.next()
    }
}

class Handler2 extends GroovyHandler {
    void handle(GroovyContext context) {
        println "This is Handler 2"
        context.next()
    }
}

class Handler3 extends GroovyHandler {
    void handle(GroovyContext context) {
        println "This is Handler 3"
        context.render "This is Handler 3"
    }
}

class Handler4 extends GroovyHandler {
    void handle(GroovyContext context) {
        println "This is Handler 4"
    }
}

class Handler5 extends GroovyHandler {
    void handle(GroovyContext context) {
        println "This is Handler 5"
    }
}

class Handler6 extends GroovyHandler {
    void handle(GroovyContext context) {
        println "This is Handler 6"
        context.render "This is Handler 6"
    }
}

class Handler7 extends GroovyHandler {
    String message
    Handler[] handlers
    Handler7(String message, Handler... handlers) {
        this.message = message
        this.handlers = handlers
    }

    void handle(GroovyContext context) {
        println "${message}"
        if (handlers) {
            context.insert(handlers)
        } else {
            context.next()
        }
    }
}
class Handler8 extends GroovyHandler {
    void handle(GroovyContext context) {
        println "This is Handler 8"
        context.render "This is Handler 8"
    }
}
```