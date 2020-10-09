---
title: "Twitterの非公開リストが見れた話"
date: 2020-10-09T18:00:00+09:00
draft: false
---

## はじめに
TwitterはBug Bountyプログラム(脆弱性報奨金制度とも呼ばれる)を実施しており、脆弱性診断を行うことが認められています。  
本記事は、そのプログラムを通して報告された脆弱性についてを解説したものであり、インターネット上のサービスに無差別に攻撃する事を推奨するものではありません。  
また、Twitter上で脆弱性を発見した場合は無闇に公開せず、[TwitterのBug Bountyプログラム](https://hackerone.com/twitter)より報告してください。  

## 要約
不適切なアクセス制御とレートリミットの欠如、タイミング攻撃を組み合わせることによりTwitterの非公開リストの中身を閲覧できた。  

## 不適切なアクセス制御

TwitterのGraphQLエンドポイントに対して以下のようなリクエストを送信すると、リストが非公開であったとしてもリストのメンバーリストを取得することが出来た。  
```http
GET /graphql/iUmNRKLdkKVH4WyBNw9x2A/ListMembers?variables=%7B%22listId%22%3A%22[ここにリストID]%22%2C%22count%22%3A20%2C%22withTweetResult%22%3Afalse%2C%22withUserResult%22%3Afalse%7D HTTP/1.1
Host: api.twitter.com
User-Agent: [Redacted]
Accept: */*
Accept-Language: ja,en-US;q=0.7,en;q=0.3
Accept-Encoding: gzip, deflate
content-type: application/json
x-twitter-auth-type: OAuth2Session
x-twitter-client-language: ja
x-twitter-active-user: yes
x-csrf-token: [Redacted]
Origin: https://twitter.com
authorization: Bearer [Redacted]
Connection: close
Cookie: [Redacted]


```

リストのIDが取得できれば非公開リストの内容が読み取れる状態だったので報告したが、Twitter曰く非公開リストのIDを取得するのは困難らしく、それを回避しない限り潜在的な脆弱性でしか無いと拒否されそうになった。[^1]

[^1]: 後のレポートでIDを取得することは容易であるとTwitterチームが認めたが、直近のレポートでもトリアージチームはIDを取得する方法が無いという理由で拒否しようとしてくる。

![Twitterチームの返信画像](/img/twitter-needmoreinfo-listid.png)
しかしながら、TwitterのIDは`ミリ秒単位のタイムスタンプ`、`Worker ID`、`データセンターID`、`ID衝突回避用の値`で構成されており、そのうち`ID衝突回避用の値`は0始まりで衝突するたびに加算されていく物で、かつ`データセンターID`は基本的に10であるため、`ミリ秒単位のタイムスタンプ`と`Worker ID`を推測することができれば実質的にIDがわかる。  
攻撃は現実的であるということを証明する必要があったため、更に調査することにした。  

## レートリミットの欠如
非公開リストのIDを取得する方法を探していた際に、Twitter APIの一部のレートリミットが、ドキュメント上には存在すると書いてあるにも関わらず、実際には存在しないことがわかった。  
それらのエンドポイントの一つに[`POST /1.1/lists/destroy.json`](https://developer.twitter.com/en/docs/twitter-api/v1/accounts-and-users/create-manage-lists/api-reference/post-lists-destroy)が存在した。  
このエンドポイントは、リストを削除するためのエンドポイントなのだが、当然他人の非公開リストを削除しようとしたとしても404が帰ってくるだけだった。  

```http
POST /1.1/lists/destroy.json
Content-Type: application/x-www-form-urlencoded
Authorization: OAuth [Redacted]

list_id=[ここにリストID]
```

## タイミング攻撃
前述のエンドポイントを詳しく調べている際に、TwitterのAPIにリクエストを送った際に帰ってくるレスポンスのほぼ全てに`x-response-time`というヘッダが存在していることがわかった。  
確認した所、このヘッダはTwitter API内部で処理にかかった時間を正確に返しているようだった。  
ここまで露骨な前振りをされたらタイミング攻撃しか無いだろうということで試してみた所、他人の非公開リストの削除を試みた場合と存在しないリストの削除を試みた場合とで`x-response-time`の値に～20程の差異が存在した。  
これと前述のレートリミットの欠如を組み合わせることにより、非公開リストのIDを現実的な時間内で総当りすることが可能となり、それを報告した所無事にTwitterのセキュリティチームによりトリアージされた。

## まとめ
Twitterのトリアージチームがこの脆弱性を調査している間、非公開リストのメンバーリストを取得できるのは「Bug Bountyプログラムで扱うほどのリスクではない」という返信が帰ってくるなど、拒否されそうになる場面が何度かあったが、根気良く説明した事により脆弱性として認められた。  
一度拒否することが確定した物[^2]でも、セキュリティ上の問題が明確なものに関してはなぜ問題なのかを説明することが重要で、それによって一転して脆弱性であると認めることがあるため、今後Twitterに脆弱性を報告する人はその点を留意したほうが良いかもしれない。  

[^2]: 例として、限定公開のコンテンツのURLが推測出来るということを報告したらそれは脆弱性ではなく仕様であるとされたことや、非公開ツイートの一部データが取得できる事を報告したらそれはBug Bountyプログラムで扱うほどのリスクではないとされたこと等がある。

## タイムライン
 日付   | 出来事
---------------|----------
  2020/5/29 | 脆弱性を報告
  2020/5/30 | Twitter: 更に情報が必要
  2020/5/30 | 追加の情報を送信
  2020/6/1 | Twitter: 更に情報が必要
  2020/6/2 | 追加の情報を送信
  2020/6/2 | Twitter: 内部調査中
  2020/6/6 | Twitter: 脆弱性として認定
  2020/6/24 | Twitter: 報奨金額の決定
  2020/8/1 | Twitter: 脆弱性を修正
  2020/8/4 | 脆弱性の開示
  
