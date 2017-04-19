# Contributionガイド

**Ratpack 入門** に興味をもって頂きありがとうございます！

本サイトに貢献頂ける方は、以下の内容を一読の上、お願いします。


## 貢献方法

1. このリポジトリをフォークします。
1. `master` ブランチからトピックブランチを作成します。: `git checkout -b my-new-branch master`
1. 作成したトピックブランチで作業します。
1. 変更をコミットします: `git commit -am "Hogeについて追記。"`
1. フォークした自分のリポジトリに pushします。: `git push origin my-new-branch`
1. プルリクエストを `koji-k/ratpack-tutorial` の `master` ブランチに送ります。

## 文章スタイル

### 文体
語尾は「だ、である」などの硬い表現方法ではなく、「です、ます」という丁寧な表現を心がけるようにしてください。

例：  
OK: Ratpackはとても素敵なAsync & Non-Blockingなweb frameworkです。  
NG: Ratpackはとても素敵なAsync & Non-Blockingなweb frameworkだ。

### サンプルコード
サンプルコードを記載する場合には、可能な限りそのサンプルコードのみで正常に動作するようにしてください。

NGの例：この例示方法だと、date変数が宣言されていないのでエラーになってしまいます。

```groovy
assert "2017/03/29" == date.format("YYYY/MM/dd")
```

OKの例：以下のコードであれば、読者はGroovyConsoleなどでそのまま問題なく動作させることができます。

```groovy
def date = new Date()
assert "2017/03/29" == date.format("YYYY/MM/dd")
```

サンプルコードにそのまま載せられないような情報がある場合は、その旨別途説明を記載するようにしてください。（例：サンプルを実行するためにはhoge.txtを/tmpに配置してください、等々）

## 文章（Markdown）のビルド
`gaiden build`、もしくは`gaiden watch`で行えます。  
最終的なcommitの前には、必ず`gaiden build`を一度実行するようにしてください。  
`gaiden watch`でもHTMLは生成されますが、確認用のWebsocketコードが挿入されてしまいます。  
動作自体に問題はありませんが、不要なHTMLファイルまでcommitされることになりますので、commit前には`gaiden build`で綺麗なHTMLを生成するようにしてください。