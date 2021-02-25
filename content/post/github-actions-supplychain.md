---
title: "GitHub Actionsにおけるサプライチェーン攻撃を介したリポジトリ侵害"
date: 2021-02-15T12:30:00+09:00
draft: false
tags: ["GitHub", "脆弱性", "GitHub Actions", "Supply Chain"]
---

## はじめに
GitHubはBug Bountyプログラムを実施しており、その一環として脆弱性の診断行為を[セーフハーバー](https://docs.github.com/en/github/site-policy/github-bug-bounty-program-legal-safe-harbor)により許可しています。  
本記事は、そのセーフハーバーの基準を遵守した上で調査を行い、結果をまとめたものであり、無許可の脆弱性診断行為を推奨することを意図したものではありません。  
GitHub上で脆弱性を発見した場合は、[GitHub Security Bug Bounty](https://hackerone.com/github?type=team)へ報告してください。  

## 要約
GitHub Actionsの仕様上、デフォルトではサプライチェーン攻撃に対する適切な保護が行われないため、特定の条件を満たしたリポジトリを侵害することが出来る。  
この問題の影響を受けるリポジトリがどの程度存在するかを調査した所、[yay](https://github.com/Jguer/yay)等の広く使われているソフトウェアのリポジトリを含めた多数のリポジトリがこの攻撃に対して脆弱であることがわかった。  

## 調査理由
[GitHub Actionsを使ったDDoSに巻き込まれた](https://blog.utgw.net/entry/2021/02/05/133642)という記事を読み、以前から調査したいと考えていたサプライチェーン攻撃に関連した調査をGitHub Actionsで行うことを思いついたため、調査内容を考え初めた。  
過去に[Repo Jacking: Exploiting the Dependency Supply Chain](https://blog.securityinnovation.com/repo-jacking-exploiting-the-dependency-supply-chain)という記事を読んでおり、追加の検証無しにリポジトリを直接参照している場合、サプライチェーン攻撃が刺さりやすくなる事を知っていたため、GitHub Actionsの[`uses`構文](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepsuses)が同様の挙動をするかどうか確認した。  
その結果、同様の挙動をすることがわかったため、本調査を実施することにした。  

## 脆弱なサプライチェーン
GitHub Actionsには、`uses`という他のユーザーが作成したActionのリポジトリを指定し、ワークフロー内に組み込むことが出来る構文が存在する。  
この機能では、デフォルトで整合性チェックが行われておらず、結果として参照先のActionが改竄されていたとしても正常に処理が実行されてしまう。  
参照先のActionが書き換えられた場合、ワークフローの発火元のイベントが`pull_request`以外であれば[^1]、Action側が参照するシークレットを指定できるため[^2]、書き込み権限を持つ`GITHUB_TOKEN`の窃取に繋がり、結果としてリポジトリを侵害することが可能となる。  
これは、サプライチェーンの保護という観点においては推奨されるものではない。 (例として、[Goでは依存関係の整合性チェックを行っている。](https://www.usenix.org/sites/default/files/conference/protected-files/enigma2020_slides_valsorda.pdf))

[^1]: https://docs.github.com/en/actions/reference/authentication-in-a-workflow#permissions-for-the-github_token
[^2]: https://github.com/actions/checkout/blob/61b9e3751b92087fd0b06925ba6dd6314e06f089/action.yml#L12-L24


![GitHub Actionsのワークフロー内で参照したActionが悪意あるものだった時の画像](/img/github-actions-simple.png)

## GitHubのRepository Redirects
また、この問題を深刻化させている機能に、[Repository Redirects](https://github.blog/2013-05-16-repository-redirects-are-here/)という物がある。  
これは、リポジトリのオーナーがリポジトリ名やユーザー名を変更した後、元のユーザー名とリポジトリ名を使用した参照を行う際に自動でリダイレクトする機能で、これによりActionを提供している開発者がGitHubのユーザー名を変更した後も、そのActionを使用しているワークフローは動作し続ける。  

![GitHub ActionsにおけるRepository Redirectsの動作方法の画像](/img/github-actions-redir.png)

しかしながら、このリダイレクトは攻撃者が簡単に制御することが可能となっている。  
攻撃者は、リポジトリオーナーの元々のIDを取得し、同名のリポジトリを作成することによりこのリダイレクトを止め、攻撃者が制御するリポジトリを参照するように変更することができる。  
![GitHub Actionsにおいて、攻撃者がRepository Redirectsを制御している画像](/img/github-actions-compromised.png)

これにより、Actionを公開している開発者がユーザー名を変更した後、ワークフローが参照しているActionが移動したことに気がつけず、長期間に渡ってリポジトリが脆弱な状況に置かれる可能性が高まる。  

## 調査
前述の通り、Actionを公開している開発者がユーザー名を変更した後、それが検出されずに放置されている可能性がある程度存在する。  
その状態になっているリポジトリを検出するため、以下の検索クエリにマッチするワークフローファイルの取得を[コード検索API](https://docs.github.com/en/rest/reference/search#search-code)から試みた。  

```
path:.github/workflows language:yaml uses
```

調査時点で919,249件のファイルが上記の条件にマッチしたが、GitHubは検索結果の最初の1000件のみを返すため、全件取得ができない。  
そのため、[`size`クエリパラメータ](https://docs.github.com/en/github/searching-for-information-on-github/searching-code#search-by-file-size)を用い、検索APIから取得できるデータ数を可能な限り増やした。  
これにより、810,177件のワークフローを取得することができた。  

また、このエンドポイントはファイルの内容をレスポンスに含めないため、ワークフローを解析するためには別のエンドポイントを叩く必要があった。  
しかしながら、810,177件のファイルを普通に取得しようとすると、APIのレートリミットによりかなりの時間がかかり、現実的な時間内に終わらない可能性がある。  
そこで、[GraphQL API](https://docs.github.com/en/graphql)を用いて、一回のリクエストで200個ほどのクエリを実行することによりこの問題を解決した。  

```
query{
	query1: repository(owner:"owner_name",name:"repository_name"){
		content: object(expression:"ref:filepath"){
			... on Blob{
				text
			}
		}
	}
	query2: repository(owner:"owner_name",name:"repository_name"){
		content: object(expression:"ref:filepath"){
			... on Blob{
				text
			}
		}
	}
	...
}
```

それぞれのワークフローを簡単なスクリプトで解析した結果、以下のような内訳になった。  

|件数|概要|
|---|---|
|699170|`actions/`のActionを除いたユニークな`uses`|
|5462|無効なYAML形式|

この結果を元に、Action所有者のユーザー名が存在するかどうかを確認した。  
ユーザーの存在確認は、以下のGraphQLクエリを実行し、結果がnullであれば存在しないといった形で確認することができる。  
```
query{
	repositoryOwner(login:"owner_id"){
		id
	}
}
```

結果として、118個のユーザー名が一つ以上のワークフローにより使用されているにも関わらず、登録可能な状態になっている事がわかった。 (これらのユーザー名はGitHubによって保護される予定となっている。[^3])  
これらのユーザーが公開しているActionを使用しているリポジトリを調べた所、AUR Helperの[yay](https://github.com/Jguer/yay)[^4]等、有名なリポジトリを含む多くのリポジトリに影響を及ぼしていた。  
また、プライベートリポジトリが普及している現状を考えると、この問題は今回判明した以上の影響を持っていると考えられる。  
[^3]: 記事公開より前にGitHubがこれらのユーザー名の保護を行うはずだったが、なぜか現時点で保護されていないため、ユーザー名一覧並びにリポジトリ一覧の公開ができなかった。
[^4]: ここにIssueのURL

## 軽減策
この問題は、ユーザーによりある程度軽減することができる。  
以下に軽減策を効果が高いと思われる順番に並べた。

1. Actionをワークフローと同じリポジトリに含める: ユーザー名の変更などにより影響を受けず、Actionを書き換えることが可能なユーザーであれば直接ワークフローを変更できるため、これは最も効果的な軽減策だと思われる。
2. Actionをフォークし、フォークした物をワークフローから参照する: 他のユーザーの名前変更による影響を受けなくなる軽減策だが、自分がユーザー名を変更した際にワークフローの書き換えを忘れると依然として脆弱になる。
3. Actionの指定をコミットハッシュと共に行う: これは、[GoogleがSHA-1の衝突に成功した](https://security.googleblog.com/2017/02/announcing-first-sha1-collision.html)ことを考えると、完全に安全な軽減策ではないと考えられる。しかしながら、依然として攻撃を困難にすることが可能となる。  

## まとめ
今回の記事では、GitHub Actionsの仕様上の問題がどのようにしてリポジトリの侵害につながるかと、その影響範囲の調査結果を紹介しました。  
この問題に関しては、GitHubにも報告しましたが、「将来的に挙動を変更する計画はあるが、現時点でそれ以上の情報はない」という返信が来たため、現在の所ユーザーが適切な設定を行うことが重要となっています。  
もしこの記事を読んでいる方がGitHub Actionsを使用しており、かつ`uses`構文を用いて他人のActionを使用している場合は、今一度設定を見直して欲しいと思います。  

本記事に関する質問はTwitter([@ryotkak](https://twitter.com/ryotkak))へメッセージを投げてください。
## タイムライン
|日付|出来事|
|----|----|
|2021/02/05 22:48|ファイル一覧取得開始|
|2021/02/06 07:05|ファイル一覧取得完了|
|2021/02/07 10:08|ファイル内容の取得並びに解析開始|
|2021/02/07 18:24|ファイル内容の取得並びに解析完了|
|2021/02/07 19:11|ユーザーの存在状況確認開始|
|2021/02/07 19:58|ユーザーの存在状況確認完了|
|2021/02/08 18:43|影響を受けるリポジトリの特定|
|2021/02/08 22:21|GitHubに問題の報告と開示許可の要請|
|2021/02/12 02:24|GitHubからの返信並びに開示許可が出る|
|2021/02/25 12:30|本記事の公開|
