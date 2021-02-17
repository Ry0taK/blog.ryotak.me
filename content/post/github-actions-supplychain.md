---
title: "GitHub Actionsにおけるサプライチェーン攻撃を介したリポジトリ侵害"
date: 2021-02-16T11:28:40+09:00
draft: false
---

## はじめに
GitHubはBug Bountyプログラムを実施しており、その一環として脆弱性の診断行為を[セーフハーバー](https://docs.github.com/en/github/site-policy/github-bug-bounty-program-legal-safe-harbor)により許可しています。  
本記事は、そのセーフハーバーの基準を遵守した上で調査を行い、結果をまとめたものであり、無許可の脆弱性診断行為を推奨することを意図したものではありません。  
GitHub上で脆弱性を発見した場合は、[GitHub Security Bug Bounty](https://hackerone.com/github?type=team)へ報告してください。  

## 要約
GitHub Actionsの仕様上、デフォルトではサプライチェーン攻撃に対する適切な保護が行われないため、特定の条件を満たしたリポジトリを侵害することが出来る。  
この問題の影響を受けるリポジトリがどの程度存在するかを調査した所、[yay](https://github.com/Jguer/yay)等の広く使われているソフトウェアのリポジトリを含めた複数のリポジトリがこの攻撃に対して脆弱であることがわかった。  

## 調査理由
[GitHub Actionsを使ったDDoSに巻き込まれた](https://blog.utgw.net/entry/2021/02/05/133642)という記事を読み、以前から調査したいと考えていたサプライチェーン攻撃に関連した調査をGitHub Actionsで行うことを思いついたため、調査内容を考え初めた。  
過去に[Repo Jacking: Exploiting the Dependency Supply Chain](https://blog.securityinnovation.com/repo-jacking-exploiting-the-dependency-supply-chain)という記事を読んでおり、追加の検証無しにリポジトリを直接参照している場合、サプライチェーン攻撃が刺さりやすくなる事を知っていたため、GitHub Actionsの[`uses`構文](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepsuses)が同様の挙動をするかどうか確認した。  
その結果、同様の挙動をすることがわかったため、本調査を実施することにした。  

## 脆弱なサプライチェーン
GitHub Actionsの`uses`構文には、他のユーザーが作成したActionを指定し、ワークフロー内に組み込むことが出来る機能が存在する。  
この機能では、デフォルトで整合性チェックが行われておらず、結果として参照先のActionが改竄されていたとしても正常に処理が実行されてしまう。  
これは、サプライチェーンの保護という観点においては推奨されるものではない。 (例として、[Goでは依存関係の整合性チェックを行っている。](https://www.usenix.org/sites/default/files/conference/protected-files/enigma2020_slides_valsorda.pdf))

![GitHub Actionsのワークフロー内で参照したActionが悪意あるものだった時の画像](/img/github-actions-simple.png)

## GitHubのRepository Redirects
また、この問題を深刻化させている機能に、[Repository Redirects](https://github.blog/2013-05-16-repository-redirects-are-here/)という物がある。  
これは、リポジトリのオーナーがリポジトリ名やユーザー名を変更した際に、元のユーザー名とリポジトリ名を使用した参照を行う際に自動でリダイレクトする機能で、これによりActionを提供している開発者がGitHubのユーザー名を変更した後も、そのActionを使用しているワークフローは動作し続ける。  

![GitHub ActionsにおけるRepository Redirectsの動作方法の画像](/img/github-actions-redir.png)

しかしながら、このリダイレクトは攻撃者が簡単に制御することが可能となっている。  
攻撃者は、リポジトリオーナーの元々のIDを取得し、同名のリポジトリを作成することによりこのリダイレクトを止め、攻撃者が制御するリポジトリを参照するように変更することができる。  
![GitHub Actionsにおいて、攻撃者がRepository Redirectsを制御している画像](/img/github-actions-compromised.png)

これにより、Actionを公開している開発者がユーザー名を変更した後、ワークフローが参照しているActionが移動したことに気がつけず、長期間に渡ってリポジトリが脆弱な状況に置かれる可能性が高まる。  

## 調査
前述の通り、Actionを公開している開発者がユーザー名をした後、それが検出されずに放置されている可能性がある程度存在する。  
その状態になっているリポジトリを検出するため、以下の検索クエリにマッチするリポジトリの取得を[コード検索API](https://docs.github.com/en/rest/reference/search#search-code)から試みた。  

```
path:.github/workflows language:yaml uses
```

調査時点で919,249件のファイルが上記の条件にマッチしたが、GitHubは検索結果の最初の1000件のみを返すため、全件取得ができない。  
そのため、[`size`クエリパラメータ](https://docs.github.com/en/github/searching-for-information-on-github/searching-code#search-by-file-size)を用い、検索APIから取得できるデータ数を最大化した。  
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

それぞれのワークフローを簡単なスクリプトで解析した結果、以下のような内訳になりました。  

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

これにより118個のユーザー名が、一つ以上のワークフローにより使用されているにも関わらず、登録可能な状態になっている事がわかった。 (これらのユーザー名は現在GitHubによって所有されている。)  
これらのユーザーが公開しているActionを使用しているリポジトリを調べた所、AUR Helperの[yay](https://github.com/Jguer/yay)等、有名なリポジトリを含む多くのリポジトリに影響を及ぼしていた。  
また、プライベートリポジトリが普及している現状を考えると、この問題は今回判明した以上の影響を持っていると考えられる。  

## 軽減策
この問題は、ユーザーによりある程度軽減することができる。  
以下に軽減策を効果が高いと思われる順番に並べた。

1. Actionのリポジトリを使用するリポジトリに含める: ユーザー名の変更などにより影響を受けず、Actionを書き換えることが可能なユーザーであれば直接ワークフローを変更できるため、これは最も効果的な軽減策だと思われる。
2. Actionをフォークし、フォークした物をワークフローから参照する: これは、他のユーザーの名前変更による影響を受けなくなる軽減策だが、自分がユーザー名を変更した際にワークフローの書き換えを忘れると依然として脆弱になる。
3. Actionの指定をコミットハッシュで行う: これは、[GoogleがSHA-1の衝突に成功した](https://security.googleblog.com/2017/02/announcing-first-sha1-collision.html)ことを考えると、完全に安全な軽減策ではない。しかしながら、依然として攻撃を困難にすることが可能であると思われる。

## まとめ