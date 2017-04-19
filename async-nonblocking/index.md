# NonBlocking（暫定メモ）

## 基本的なNonBlockingな処理

```groovy

ratpack {

  // テストのためにサーバが利用するスレッドは1つに限定
  serverConfig {
    threads 1
  }
  
  handlers {
    // ab -n 10 -c 10 http://localhost:5050/test-blocking
    // サーバは1スレッドしか扱えないようにしているので、1スレッド（1アクセス）が終わるまで他のアクセスは待ちになる。
    // なので、上記のabを実行すると、10ユーザが同時に10アクセスしてきて、各アクセスが5秒待機するので、大体50秒ほどかかる。
    // 実行するとコンソールには以下のように出力される
    // tart
    // end
    // start
    // end
    // start
    // end
    // start
    // end
    // start
    // end
    // start
    // end
    // start
    // end
    // start
    // end
    // start
    // end
    // start
    // end

    get ("test-blocking") {
      println "start"
      sleep(5000) // 5秒待機
      println "end"
      render "ok"
    }

    // ab -n 10 -c 10 http://localhost:5050/test-non-blocking
    // サーバは1スレッドしか扱えないという同じルールだが、こちらはsleep処理をBlockingな処理としてRatpackに教えている。
    // こうすることで、Ratpackは一つしかないアクセスを捌くcomputeスレッドとは別の、Blocking処理専用のBlockingスレッドに別出ししてくれる。
    // こうすることで、メインのcomputeすれどは即座の後続の別のアクセスの処理に映ることができる。
    // このBlockingに出した別の処理は、終わった時点で自動的に再度このcomputeスレッドに持っどってきてそy利される。
    // まるですべて同時平行に動いているように見えるので、上記のabコマンドは5秒で全ての10アクセスが処理される。
    // 実行するとコンソールには以下のように出力される
    // start
    // start
    // start
    // start
    // start
    // start
    // start
    // start
    // start
    // start
    // end
    // end
    // end
    // end
    // end
    // end
    // end
    // end
    // end
    // end

    get ("test-non-blocking") {
      Blocking.get {
        println "start"
        sleep(5000) // 5秒待機
        println "end"
      }.then {
        render "ok"
      }
    }
  }
}

```

## PromiseからさらにNonBlockingな処理を続ける
```groovy
    // 以下のようにすれば、parseで帰ってきたPromiseから更にBlockingでblockingスレッドに処理を別出しすることが出来る。
    // ab -n 4 -c 4 -p ./test.json -T "application/json; charset=utf-8" http://localhost:5050/test
    // 以下のような感じになる（computeスレッドを1つに限定）
    // blockingは複数スレッドになっていることが確認できる。
    // start ratpack-compute-13-1
    // end ratpack-compute-13-1
    // start ratpack-compute-13-1
    // end ratpack-compute-13-1
    // start ratpack-compute-13-1
    // end ratpack-compute-13-1
    // main (ratpack-blocking-14-1)
    // main (ratpack-blocking-14-4)
    // main (ratpack-blocking-14-2)
    // start ratpack-compute-13-1
    // end ratpack-compute-13-1
    // main (ratpack-blocking-14-3)
    // ok (ratpack-compute-13-1)
    // ok (ratpack-compute-13-1)
    // ok (ratpack-compute-13-1)
    // ok (ratpack-compute-13-1)

    path("test") {
      println "start ${Thread.currentThread().name}"
      parse(Jackson.jsonNode()).blockingMap {
        println "main (${Thread.currentThread().name})"
        sleep(5000)
      }.then {
        println "ok (${Thread.currentThread().name})"
        render "ok"
      }
      println "end ${Thread.currentThread().name}"
    }

    // 以下のような方法だと、computeスレッド、つまりHTTPを待ち受けて処理するスレッドで、1要求毎に5秒待機してしまうので悪い書き方。
    path("test2") {
      println "start ${Thread.currentThread().name}"
      parse(Jackson.jsonNode()).then {
        println "main ${Thread.currentThread().name}"
        sleep(5000)
        println "ok (${Thread.currentThread().name})"
        render "ok"
      }
      println "end ${Thread.currentThread().name}"
    }
```