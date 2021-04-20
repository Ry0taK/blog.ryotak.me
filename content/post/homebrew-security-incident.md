---
title: "HomebrewのCaskリポジトリを介した任意コード実行"
date: 2021-04-19T20:18:29+09:00
tags: ["Homebrew", "脆弱性", "Ruby", "Supply Chain"]
---

English version is available here: https://blog.ryotak.me/post/homebrew-security-incident-en/
## はじめに
HomebrewプロジェクトはHackerOne上で脆弱性開示制度(Vulnerability Disclosure Program)を設けており、脆弱性の診断行為が許可されています。  
本記事は、当該制度に参加し、Homebrewプロジェクトのスタッフから許可を得た上で実施した脆弱性診断行為について解説したものであり、無許可の脆弱性診断行為を推奨することを意図したものではありません。  
Homebrewに脆弱性を発見した場合は、[Homebrewプロジェクトの脆弱性開示制度](https://hackerone.com/homebrew)へ報告してください。  

## 要約
HomebrewのCaskを管理しているリポジトリにおいて、プルリクエストの自動レビュー機能に使用されているライブラリにバグが存在し、結果として自動レビュー機能に悪意のないプルリクエストだと誤判定させることが可能だった。  
これにより、悪意のあるコードを[Homebrew/homebrew-cask](https://github.com/Homebrew/homebrew-cask)へ自動的にマージし、Homebrewを利用しているユーザーのコンピューター上で任意のRubyコードを実行することができた。  

## 調査理由
とある日の午後、次の予定までの時間が微妙に空いていた[^1]ため、HackerOne上で面白そうなプログラムを探すことにした。  
自分が利用しているサービス/ソフトウェアの脆弱性を優先して探したいと思ったため、手元のPCの中を漁っていると、`brew`コマンドに目が行った。  
以前、HackerOne上のプログラム一覧を眺めているときに、Homebrewというプログラムを見かけたことを思い出し、せっかくなので脆弱性を探すことにした。  

![HackerOne上のHomebrewプログラムの画像](/img/homebrew-hackerone-program.png)

[^1]: 具体的に言うと、改善前の[Dead by Daylight](https://ja.wikipedia.org/wiki/Dead_by_Daylight)のマッチ待ち時間ぐらい
## 調査対象の選定
調査する対象を選定するために、脆弱性開示制度のポリシーページを確認すると、`brew`コマンド本体の他に`Homebrew/homebrew-*`にマッチするGitHubリポジトリに関する脆弱性報告を受け付けていることに気がついた。  
複雑なRubyのコードを読むのが苦手なので、ひとまず`Homebrew/homebrew-*`における脆弱性を探すことにした。  

![HackerOne上のHomebrewプログラムにおけるスコープセクション](/img/homebrew-hackerone-scope.png)

## 初期調査
GitHubリポジトリにおける脆弱性は、主に以下の2つが多いように思われる。
1. 当該のリポジトリに対する書き込み権限を持ったトークンの漏洩
2. 当該のリポジトリが利用しているCIスクリプトにおける脆弱性

そのため、これら2つの脆弱性に絞り、脆弱性を確認していくことにした。  
1つ目に関して確認するために、[Homebrew](https://github.com/Homebrew)に所属しているユーザーが所有しているリポジトリをすべて取得し、それらの中からトークンと思わしき文字列を抽出していった。  
しかしながら、この方法での脆弱性が見つかるのは非常に稀であり、当然有効なトークンは見つからなかった。[^2]  

そのため、2つ目に関連した脆弱性が無いかどうか確認することにした。  

[^2]: また、Homebrewでは2018年に[GitHub API Tokenの漏洩に関連したインシデント](https://brew.sh/2018/08/05/security-incident-disclosure/)が発生していたため、所属メンバーの意識も高かったものと思われる。

## CIスクリプトの調査
Homebrewは、CIスクリプトを実行するために[GitHub Actions](https://github.com/features/actions)を使用している。[^3]  
そのため、各リポジトリの`.github/workflows/`配下のファイルを調べていくことにした。  

いくつかのリポジトリを確認した後、`Homebrew/homebrew-cask`に置かれていた[`review.yml`](https://github.com/Homebrew/homebrew-cask/blob/aa89774e95530994ae95a9e6aad7eca1bde41033/.github/workflows/review.yml)と、[`automerge.yml`](https://github.com/Homebrew/homebrew-cask/blob/4986c6b25133c8e536a4f39136c8dafe20ff2a38/.github/workflows/automerge.yml)に非常に強い関心を持った。  
どうやら、`review.yml`がユーザーが送信したプルリクエストの内容を確認し、そのプルリクエストが非常に単純な変更(例: バージョンアップなど)のみを加える場合、自動でプルリクエストを承認し、その後`automerge.yml`が承認されたプルリクエストを自動でマージしているようだった。  

![review.ymlの抜粋画像](/img/homebrew-review-workflow.png)
![automerge.ymlの抜粋画像](/img/homebrew-automerge-workflow.png)

[^3]: 直近3つの記事が全てGitHub Actionsに関連したものであることに関しては非常に申し訳ないと思っている。 (が、GitHub Actionsは攻撃対象領域として非常に優秀なのだ。)
## review.ymlの調査

`review.yml`が利用しているRubyスクリプト[^4]は変更点をdiffファイルとして取得し、それを[`git_diff`](https://github.com/anolson/git_diff)というGemを用いてパースする。  
その後、以下の条件を満たしている場合にのみプルリクエストを承認していた。  
1. 変更中のファイルが1つのみ
2. ファイルを移動/新規作成/削除していない
3. 変更中のファイルパスが`\ACasks/[^/]+\.rb\Z`にマッチする
4. 削除した行数と追加した行数が同数
5. 削除/追加した行の全てが`/\A[+-]\s*version "([^"]+)"\Z/`か`\A[+-]\s*sha256 "[0-9a-f]{64}"\Z`のどちらかにマッチする
6. バージョンが変更される場合、フォーマットが以前と同様 (例: `1.2.3` => `2.3.4`)

...等[^5]

条件を精査したが、非常に厳密な条件だったため脆弱性は見つからず、この条件下で任意のコードを挿入するのは不可能であると結論付けた。  
その後、しばらく他のスクリプトの確認を行っていたのだが、どういうわけかこのスクリプトの事が頭から離れなかった。  
そのため、このスクリプトを徹底的に調べることにし、diffファイルのパースを行っているライブラリを調べ始めた。  

[^4]: https://github.com/Homebrew/actions/blob/bac0cf0eef64950c5fa7b60134da80f5f52d87ab/.github/workflows/review-cask-pr.yml
[^5]: 発見した脆弱性に重要でない条件は省略したが、他にも多数の条件が存在した。
詳しくは https://github.com/Homebrew/actions/blob/bac0cf0eef64950c5fa7b60134da80f5f52d87ab/.github/workflows/review-cask-pr.yml を確認してほしい。

## git_diffの調査

[`git_diff`](https://github.com/anolson/git_diff)のリポジトリを眺めている際に、[変更された行数のパースが間違っているという旨のIssue](https://github.com/anolson/git_diff/issues/12)を発見した。  
このIssueを見て、どうにかして`git_diff`を混乱させ、上記の条件を満たすように偽装できないかと考え始めた。  

詳しくコードを読んでいくと、どうやら`git_diff`は以下のような処理を行いdiffファイルをパースしているようだった。
1. ファイルの内容を改行区切りで分割する 
2. それぞれの行に対して、`^diff --git(?: a/(\S+))?(?: b/(\S+))?`がマッチするかどうかを確認し、マッチすればファイル情報として判定し、現在処理中のファイルを正規表現にマッチしたものに置き換える。
3. 2でマッチしなければ、以下の正規表現にマッチするかを確認、マッチしていれば変更元/変更先のファイルパスを内容に応じて書き換える。
![diffファイルの移動元/移動先ファイルパスにマッチする正規表現](/img/homebrew-path-regex.png)
4. 3でマッチしなければ、ファイル自体の変更として扱い、`+`で始まっていれば追加、`-`で始まっていれば削除、それ以外は元々のファイル内容として扱う。
5. 上記を繰り返し、全ての行を処理し次第終了する。

一見問題ないように見えるこの処理だが、3の移動元/移動先ファイルパスにマッチするかどうかの判定を複数回走らせることが可能となっていた。  

GitHubが返すdiffファイルは、以下の構造の変更データをファイルごとに作成し、それを単一ファイルに纏めたものとなっている。
```diff
diff --git a/変更前のファイルパス b/変更後のファイルパス
index 親コミットハッシュ..現コミットハッシュ ファイルモード
--- a/変更前のファイルパス
+++ b/変更後のファイルパス
@@ 行情報 @@
以下変更詳細 (例: `+asdf`、`-zxcv`等)
```

行追加に関しては、追加された行の前に`+`を追加することにより表現される。
つまり、`++ "?b/(.*)`にマッチする行追加であれば、ファイル自体の変更**ではなく**変更中のファイルパス情報として認識されてしまう状態だった。  
ここで、先程の条件を確認すると、変更中のファイルパスに対する条件は`\ACasks/[^/]+\.rb\Z`だけとなっている。  
先述の通り変更中のファイルパス情報変更は複数回行えるため、以下のような変更を加えることにより上記の条件を回避し、変更された行が0行の無害なプルリクエストと判定させることができる可能性が高かった。[^6]  
```ruby
++ "b/#{ここに悪意のあるコード}"
++ b/Casks/cask.rb
```

[^6]: 二行目が文字列リテラルになっていないのは意図的であり、`git_diff`のバグにより文字列リテラルを閉じた際のダブルクオーテーションもファイルパスとして扱われてしまうためである。

## デモンストレーションの準備
HackerOne等のバグバウンティプラットフォームは、基本的にPoCや脆弱性のデモンストレーションをレポート内で送信する必要がある。  
そのため、この脆弱性が実際に悪用可能かどうかを確かめることにした。　 

実際に使用されているCaskを許可なく変更するのはあまりよろしくないため、`homebrew-cask`内において、テスト用のCaskを探すことにした。  
しかしながら、テスト用のCaskは見つからなかったため、仕方なくHomebrewの脆弱性開示制度を担当している方にメールを送信した。  

その後、担当している方から`これだけのためにテスト用のCaskを追加する無理なので、代わりに既存のCaskに無害な変更を加えてくれ`という旨の返信が帰ってきた。  
そのため、適当なCaskを選び、無害な変更を加えることにした。[^7]  

[^7]: この時点では、各Caskファイルは`brew install`を実行した際にしか実行されないと思っていたため、メンテナと連絡が取れている状況を鑑み、すぐに取り消せば実際に変更したコードを実行するユーザーは発生しないと考えていた。

## 脆弱性のデモンストレーション
この日の前日に[GitHubのAPI Tokenをうっかり載せてしまったプルリクエスト](https://github.com/Homebrew/homebrew-cask/pull/104167)を見かけていたため、このプルリクエストが変更しようとしていた`iterm2.rb`に対して変更を加えることにした。  

変更を加える際に気がついたのだが、`++ "b/"`はRubyの文法として間違っていないが、`++ b/Casks/iterm2.rb`は変数を定義しないとエラーになってしまう。  
そのため、`Homebrew/homebrew-cask`をフォークし、以下の2行を`Casks/iterm2.rb`へ追記した。  
```ruby
++ "b/#{puts 'Going to report it - RyotaK (https://hackeorne.com/ryotak)';b = 1;Casks = 1;iterm2 = {};iterm2.define_singleton_method(:rb) do 1 end}"
++ b/Casks/iterm2.rb
```
一行目で`b`、`Casks`、`iterm2`、`iterm2.rb`を定義することにより、二行目でエラーが発生しなくなるため、有効なRubyスクリプトとして実行することができるようになる。  
また、この変更を加えることにより、以下のようなDiffをGitHubが返すようになる。
```diff
diff --git a/Casks/iterm2.rb b/Casks/iterm2.rb
index 3c376126bb1cf9..ba6f4299c1824e 100644
--- a/Casks/iterm2.rb
+++ b/Casks/iterm2.rb
@@ -8,6 +8,8 @@
     sha256 "e7403dcc5b08956a1483b5defea3b75fb81c3de4345da6000e3ad4a6188b47df"
   end
 
+++ "b/#{puts 'Going to report it - RyotaK (https://hackeorne.com/ryotak)';b = 1;Casks = 1;iterm2 = {};iterm2.define_singleton_method(:rb) do 1 end}"
+++ b/Casks/iterm2.rb
   url "https://iterm2.com/downloads/stable/iTerm2-#{version.dots_to_underscores}.zip"
   name "iTerm2"
   desc "Terminal emulator as alternative to Apple's Terminal app"
```

前述の通り、`git_diff`は`+++ "?b/(.*)`にマッチする変更を行追加ではなく変更中のファイル情報として扱うため、このDiffは`Casks/iterm2.rb`に対して0行の変更を加えるものとして扱われてしまう。  

この変更を加えた上でプルリクエストを作成[^8]し、脆弱性のデモンストレーションを開始した。  

[^8]: https://github.com/Homebrew/homebrew-cask/pull/104191

## 問題発生
その後、しばらく待ってもプルリクエストがマージされず、なぜかと思いCIの実行ログを確認すると、`Required status checks for pull request 104191 are not successful.`というログが出力されていた。  
失敗しているチェックを洗い出した所、`brew syntax`を実行してコードの構文チェックを行っており、この部分で実行される`Rubocop`がコードが汚いと怒っていることがわかった。  

![GitHub Actions上でRubocopが失敗している画像](/img/homebrew-rubocop-fail.png)

`Rubocop`は`# rubocop:disable all`というコメントを行末尾に追加することにより、特定の行に対しての警告を無効化することができる。  
一行目に関してはこれを追記するだけで済んだのだが、二行目は`+++ "?b/(.*)`のグルーピングされている部分にマッチする文字列が`Casks/iterm2.rb`である必要があったため、このコメントを追加することによる解決はできなかった。　　

いくつかの試行錯誤の後、以下のような変更を加えることにより最後の行を変更せずにRubocopを黙らせることが可能であることがわかった。
```ruby
++ "b/#{puts 'Going to report it - RyotaK (https://hackerone.com/ryotak)';b = 1;Casks = 1;iterm2 = {};iterm2.define_singleton_method(:rb) do 1 end; }" # rubocop:disable all
++ "b/" if # rubocop:disable all
++ b/Casks/iterm2.rb
```
二行目で`if`を追加し、次の行を`if`式として評価させることにより`Operator / used in void context.`という警告が出ないように調整している。  

これにより、プルリクエストに対して実行されているチェックが全て成功するようになり、無事に`BrewTestBot`によってプルリクエストがマージされた。  
![BrewTestBotが自動的にマージした際の画像](/img/homebrew-merge-success.png)

## 問題発生、再び
無事にプルリクエストがマージされたため、`brew install iterm2 --cask`を実行し、`Going to report it - RyotaK (https://hackerone.com/ryotak)`が出力されることを確認し、PoCとして画像を送信した。  

![brew install iterm2 --caskが変更後のコードを参照している事を示す画像](/img/homebrew-install-poc.png)

その後、送信したレポートに対する返信を待っている間に、以下のようなリプライがTwitter上で送られてきた。  
{{< tweet 1383730296609144847 >}}
一瞬理解できなかったのだが、よく見ると`brew cleanup`を実行した際にも`Going to report it - RyotaK (https://hackerone.com/ryotak)`が表示されてしまっているようだった。  
慌てて手元で試すと、確かに`brew cleanup`実行時にも`Going to report it - RyotaK (https://hackerone.com/ryotak)`が表示されてしまっていた。  

この時は非常に慌てていたため気が付かなかったのだが、後から調査した結果、`brew cleanup`以外にも`brew search`等で変更したCaskが実行されてしまうことがわかった。  
どうやら、一部のコマンドを実行した際に全てのCaskを評価する設計になっていたため、対象のCaskをインストールしていなかったとしても、変更後のコードが実行されてしまう状態になっていたようだった。  

加えた変更がログを出力するのみだったこと、及びすぐに変更が巻き戻されたことにより大した影響はなかったが、このようなことが起きることを想定していなかったため、非常に焦った。  

## まとめ
今回の記事では、HomebrewのOfficial Tapに存在した脆弱性について解説しました。  
この脆弱性が悪用されていた場合、発覚するまでの間に`brew`で特定の操作を行ったユーザー全員のコンピュータが侵害されることに繋がったと考えると、エコシステムに関する監査は必要不可欠だと感じました。  
PyPI等に対する脆弱性診断も行いたいとは思っているのですが、残念ながら脆弱性開示制度等が設けられていないため、現在の所行うことができません。  

本記事に関する質問はTwitter([@ryotkak](https://twitter.com/ryotkak))へメッセージを投げてください。

## タイムライン
|日付 (日本時間)|出来事|
|----|----|
|2021/04/17|脆弱性の発見|
|2021/04/17|メンテナに対して連絡|
|2021/04/18|メンテナからの返信を受信|
|2021/04/18 17時頃|デモンストレーションを開始|
|2021/04/18 17時頃|レポートを送信|
|2021/04/18 18時頃|プルリクエストをマージさせることに成功|
|2021/04/18 19時頃|プルリクエストが巻き戻される|
|2021/04/18 20時頃|一次対応が完了|
|2021/04/19|二次対応が完了|
|2021/04/XX|インシデントの開示|
