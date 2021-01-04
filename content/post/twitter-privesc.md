---
title: "Twitterのフリート機能に対する権限昇格"
date: 2020-12-18T12:58:27+09:00
draft: false
---

## はじめに
TwitterはBug Bountyプログラム(脆弱性報奨金制度とも呼ばれる)を実施しており、脆弱性の診断行為を行うことが認められています。  
本記事は、そのプログラムを通して報告された脆弱性についてを解説したものであり、Twitterが認知していない未修正の脆弱性を公開する事を意図したものではありません。  
また、Twitter上で脆弱性を発見した場合は[TwitterのBug Bountyプログラム](https://hackerone.com/twitter)より報告してください。  
  
(This article is written in Japanese. If you'd like to read this article in English, please visit [HackerOne report](https://hackerone.com/reports/1032468).)

## TL;DR
Twitterが公開したフリート機能が使用しているAPIに脆弱性が存在し、`READ`権限しか持っていないサードパーティアプリケーションがフリートの作成や削除などを行えた。  

## 調査したきっかけ
Twitterは2020年11月10日に、[フリート](https://blog.twitter.com/ja_jp/topics/product/2020/ntroducing-fleets-new-way-to-join-the-conversation-jp.html)と呼ばれる機能を日本に対して公開した。  
当初はiOS版のクライアントのみに実装されたが、翌日の11日には手元のAndroid端末にフリート機能用の更新が降ってきていたため、APIを解析してみることにした。  

## 解析してみる
Twitter for Androidは通常のAndroidアプリと同じくJavaで書かれており、難読化はされているものの解析はそこまで難しくない。  
そのため、[`apktool`](https://ibotpeaches.github.io/Apktool/)と[`dex2jar`](https://github.com/pxb1988/dex2jar)、[`CFR`](https://www.benf.org/other/cfr/)を用いてデコンパイルし、ある程度可読性が高い状態に戻した。(詳細なデコンパイル方法に関してはここでは触れないが、ググれば出てくるのでそちらを参照してほしい。)  
エンドポイント名がわからなければAPIを解析できないため、`grep`を用いて`fleet`という文字列が含まれる`.java`ファイルを検索し、`/fleet/v1/user_fleets`という文字列が含まれるファイルを発見した。  
そのファイルと同じ階層にあるファイルを調べた所、他のエンドポイントと思わしき文字列が見つかったため、一旦それらを解析し、Gistにまとめた。  

## 検証中に...
その後、Gistの内容を精査している際に、サードパーティのアプリケーションとして認証している際にフリート関連のAPIを叩くと、問題なく動作することがわかった。  
この事をドキュメントにまとめて公開すれば非公式クライアントの製作者の方が喜ぶのでは？と思い[詳細なAPIドキュメント](https://gist.github.com/Ry0taK/005b79eccb4297469a09696dae9fa3c6)を書いた。  
公開する前にこのGistの内容が間違っていないか検証していた所、`POST /fleets/v1/create`に対して読み取り権限しか持たないアプリケーションとしてリクエストを送信した際に、フリートが作成されてしまっていることがわかった。  
![APIから作ったフリートの画像](/img/fleet_from_api.png)

## これは脆弱性なのか？
最初は検証用アプリケーションに誤って書き込み権限を与えてしまったのだと思い、権限を確認したが、明らかに読み取り権限しか与えられていなかった。  
この時点で、これは脆弱性なのでは？と思い始めたが、確証が得られなかったのでもう少し深く調査することにした。  
その結果、通常のTwitterのAPI(`POST /1.1/statuses/update.json`等)では、APIの処理が走る前に権限チェックをしていたが(当たり前だが)、どうやらフリート関連のエンドポイントは通常のAPIでは行われる権限チェックが行われていないことが判明した。  
```bash
$ twurl /1.1/statuses/update.json --header 'Content-Type: application/json' -d '{"status":"Test"}'
{"request":"\/1.1\/statuses\/update.json","error":"Read-only application cannot POST."}

$ twurl /fleets/v1/create -X POST --header 'Content-Type: application/json' -d '{"text":"Hey yo"}'
{"fleet":{"created_at":"2020-11-12T12:29:16.180000000Z","deleted_at":null,"expiration":"2020-11-13T12:29:16.189235445Z","fleet_id":"F1-328253875041691174","fleet_thread_id":"T1-328253875041625638","mentions":null,"mentions_str":null,"read":false,"text":"Hey yo","user_id":1195137762027962368},"fleet_thread_id":"T1-328253875041625638","fleet_id":"F1-328253875041691174","users":null}
```
## 報告しよう
脆弱性であることがわかったため、一旦フリート関連のAPIドキュメントの公開を見送り、Twitterに報告することにした。  
報告は11月12日に行ったのだが、11月18日(実際にはもう数日前だったのだと思う)には修正されており、非常に印象的な修正速度だった。  
しかしながら、残念なことにフリート関連のAPIがサードパーティのアプリケーションによって使用できていた事自体が問題だったようで、フリート関連のAPIをサードパーティのアプリケーションに叩かせないように変更することで修正されてしまった。  

## その後
Twitterが脆弱性を修正した3日後、同じくフリートのAPIを解析してフリートの画像が24時間経った後もCDNから削除されない事を[ツイート](https://twitter.com/donk_enby/status/1330078983350837248)している人がおり、フリート機能の実装に使えた期間はとても短かったのかな、と少しTwitter内部の開発者がかわいそうになってしまった。  
{{< tweet 1330078983350837248 >}}
この脆弱性を発見する理由となったGistに関しては、[ここで公開](https://gist.github.com/Ry0taK/005b79eccb4297469a09696dae9fa3c6)しているので何かの役にたててほしい。

## まとめ
この一連の流れで、新機能を解析することの重要さと、どんな大企業でもミスはするという教訓を得ることが出来た。  
Twitterに報告した際の実際のレポートは[ここ](https://hackerone.com/reports/1032468)から見れるので是非読んでみてほしい。  
  
このブログ記事に関して、なにか質問等がある場合は[Twitter @ryotkak](https://twitter.com/ryotkak)へDMを飛ばしてください。  

## タイムライン
 日付   | 出来事
---------------|----------
  2020/11/10 | Twitterがフリート機能を日本向けにリリース
  2020/11/11 | 手元の環境でフリート機能が使えるようになった
  2020/11/12 | 脆弱性を発見、報告
  2020/11/13 | Twitter: 現在確認中
  2020/11/14 | Twitter: 脆弱性として認定
  2020/11/18 | Twitter: 脆弱性を修正
  2020/12/15 | Twitter: 報奨金額の決定
  2020/12/TODO | 脆弱性の開示
  
