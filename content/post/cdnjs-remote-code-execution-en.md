---
title: "Cloudflareのcdnjsにおける任意コード実行+α"
date: 2021-07-01T12:53:03+09:00
draft: false
tags: ["cdnjs", "脆弱性", "Go", "Supply Chain", "RCE"]
---

## はじめに

cdnjsの運営元であるCloudflareは、HackerOne上で脆弱性開示制度(Vulnerability Disclosure Program)を設けており、脆弱性の診断行為が許可されています。  
本記事は、当該制度を通して報告された脆弱性をCloudflareセキュリティチームの許可を得た上で公開しているものであり、無許可の脆弱性診断行為を推奨することを意図したものではありません。  
Cloudflareが提供する製品に脆弱性を発見した場合は、[Cloudflareの脆弱性開示制度](https://hackerone.com/cloudflare)へ報告してください。  

## 要約

cdnjsのライブラリ更新用サーバーに任意のコードを実行することが可能な脆弱性が存在し、結果としてcdnjsを完全に侵害することが出来る状態だった。  
これにより、インターネット上のウェブサイトの内12.6%[^1]を改竄することが可能となっていた。

[^1]: [W3Techs](https://w3techs.com/technologies/details/cd-cdnjs)より記事執筆時点の情報を引用。SRIの存在により、即座に改竄可能だったウェブサイトはこの数字よりも少なかった。

## cdnjsとは

[cdnjs](https://cdnjs.com)は、Cloudflareによって運営されている無料のJavaScript/CSSライブラリ用CDNであり、記事執筆時点でインターネット上のウェブサイトの12.6%に使用されている。  
これは、[Google Hosted Libraries](https://developers.google.com/speed/libraries)の12.8%[^2]に次いで2番目に広く使われているライブラリ用CDNであり、現在の使用率の推移を鑑みるに、近いうちに最も使われているJavaScriptライブラリ用CDNになると考えられる。  

<div style="text-align: center;">
  <img src="/img/cdnjs-usage.png" alt="cdnjsのインターネット上での使用率">
  <p>2021年7月2日時点での<a href="https://w3techs.com/technologies/details/cd-cdnjs" target="_blank">W3Techs</a>におけるcdnjsの使用率グラフ</p>
</div>

[^2]: [W3Techs](https://w3techs.com/technologies/details/cd-googlelibraries)より記事執筆時点の情報を引用。

## 調査理由

前回の[Homebrewにおける任意コード実行](/post/homebrew-security-incident/)に関する調査を行う数週間前に、サプライチェーン攻撃に関する調査を行っていた。  
多数のアプリケーションが依存するシステムであり、脆弱性調査を許可しているという条件で絞り込んだ際に、cdnjsが対象に入ったため調査を行うことにした。  

## 初期調査
cdnjsのウェブサイトを眺めている際に、以下のような記述を発見した。

> Couldn't find the library you're looking for?  
You can make a request to have it added on our GitHub repository.

GitHubのリポジトリ上でライブラリの情報を管理している事がわかったため、当該のGitHub Organizationのリポジトリを確認した。  

結果として、以下のような構成になっていることがわかった。  

- [cdnjs/packages](https://github.com/cdnjs/packages): cdnjsに掲載するライブラリの情報を格納
- [cdnjs/cdnjs](https://github.com/cdnjs/cdnjs): ライブラリの実際のファイル群を格納
- [cdnjs/logs](https://github.com/cdnjs/logs): ライブラリの更新ログを格納
- [cdnjs/SRIs](https://github.com/cdnjs/SRIs): 各ライブラリのSRI(Subresource Integrity)を格納
- [cdnjs/static-website](https://github.com/cdnjs/static-website): cdnjs.comのソースコード
- [cdnjs/origin-worker](https://github.com/cdnjs/origin-worker): cdnjs.cloudflare.comのオリジン用Cloudflare Worker
- [cdnjs/tools](https://github.com/cdnjs/tools): cdnjsの管理用ツール
- [cdnjs/bot-ansible](https://github.com/cdnjs/bot-ansible): cdnjsの自動ライブラリ更新用システムのAnsible

これらのリポジトリから分かるように、cdnjsのインフラの大部分はこのGitHub Organizationに集約されている。  
その中で、[cdnjs/bot-ansible](https://github.com/cdnjs/bot-ansible)と[cdnjs/tools](https://github.com/cdnjs/tools)に興味を惹かれた。  
この2つのリポジトリのコードを読んだ所、[cdnjs/bot-ansible](https://github.com/cdnjs/bot-ansible)が定期的に[cdnjs/tools](https://github.com/cdnjs/tools/tree/b6833e08108b2a06b3b3e5f212d604b5951ff924)の`autoupdate`コマンドをライブラリ更新用サーバー内で実行し、[cdnjs/packages](https://github.com/cdnjs/packages)において指定されているGitHubリポジトリ/npmパッケージを用いて更新を確認していることがわかった。  

## 自動更新機能の調査

この自動更新機能は、ユーザーが管理するGitHubリポジトリ/npmリポジトリをダウンロードし、対象のファイルをコピーすることによってライブラリを更新していた。  
npmレジストリは、各ライブラリを`.tgz`ファイルとして圧縮した上でダウンロードができるようにしている。  
この自動更新用のツールがGoで記述されていることから、Goの`compress/gzip`及び`archive/tar`を用いて解凍しているのではないかと推測した。  
Goの`archive/tar`はアーカイブ内に含まれるファイルパスをサニタイズせずに返す[^3]ため、もし仮に`archive/tar`から返されたファイル名を元にしてディスク上に書き込んでいる場合、`../../../../../../../../../tmp/test`のようなファイル名を`.tgz`ファイルに含めることにより、任意のファイルを上書きできる。  
[cdnjs/bot-ansible](https://github.com/cdnjs/bot-ansible)の情報から、いくつかのスクリプトが定期的に実行されており、それらに対して書き込み権限があることがわかっていたため、パストラバーサルを介したファイルの上書きを重点的に確認することにした。

[^3]: https://github.com/golang/go/issues/25849

![パストラバーサルを行うために細工したtgzファイルの画像](/img/cdnjs-tgz-slip.png)

## パストラバーサル
パストラバーサルを探すために、`autoupdate`コマンドの`main`関数を読み始めた。
```go
func main() {
        [...]	
		switch *pckg.Autoupdate.Source {
		case "npm":
			{
				util.Debugf(ctx, "running npm update")
				newVersionsToCommit, allVersions = updateNpm(ctx, pckg)
			}
		case "git":
			{
				util.Debugf(ctx, "running git update")
				newVersionsToCommit, allVersions = updateGit(ctx, pckg)
			}
		[...]
}
```

上記のコードから分かるように、`npm`をベースとした自動更新が指定されていた場合は、`updateNpm`関数へとパッケージ情報を渡している。  

```go
func updateNpm(ctx context.Context, pckg *packages.Package) ([]newVersionToCommit, []version) {
		[...]
		newVersionsToCommit = doUpdateNpm(ctx, pckg, newNpmVersions)
        [...]
}
```

そして、新しいライブラリバージョンの情報と共に`doUpdateNpm`関数を呼び出す。  

```go
func doUpdateNpm(ctx context.Context, pckg *packages.Package, versions []npm.Version) []newVersionToCommit {
    [...]
	for _, version := range versions {
        [...]
		tarballDir := npm.DownloadTar(ctx, version.Tarball)
		filesToCopy := pckg.NpmFilesFrom(tarballDir)
        [...]
}
```
次に、`npm.DownloadTar`関数へ新しいバージョンの`.tgz`ファイルのURLを渡す。  

```go
func DownloadTar(ctx context.Context, url string) string {
	dest, err := ioutil.TempDir("", "npmtarball")
	util.Check(err)

	util.Debugf(ctx, "download %s in %s", url, dest)

	resp, err := http.Get(url)
	util.Check(err)

	defer resp.Body.Close()

	util.Check(Untar(dest, resp.Body))
	return dest
}
```
最後に、`http.Get`を使用して取得した`.tgz`ファイルを、`Untar`関数へと渡す。  

```go
func Untar(dst string, r io.Reader) error {
	gzr, err := gzip.NewReader(r)
	if err != nil {
		return err
	}
	defer gzr.Close()
	tr := tar.NewReader(gzr)
	for {
		header, err := tr.Next()
        [...]
		// the target location where the dir/file should be created
		target := filepath.Join(dst, removePackageDir(header.Name))
        [...]
		// check the file type
		switch header.Typeflag {
        [...]
		// if it's a file create it
		case tar.TypeReg:
			{
                [...]
				f, err := os.OpenFile(target, os.O_CREATE|os.O_RDWR, os.FileMode(header.Mode))
                [...]
				// copy over contents
				if _, err := io.Copy(f, tr); err != nil {
					return err
				}
			}
		}
	}
}
```

予想通り、`Untar`関数内において`compress/gzip`と`archive/tar`を使用して解凍を行っていた。  
最初は`removePackageDir`においてパスのサニタイズを行っているのだと考えていたのだが、関数の内容を確認した所、単純に`package/`という文字列をパスから削除するだけの関数だった。  
これにより、npmへ公開した`.tgz`ファイルからパストラバーサルを行い、サーバー上で定期的に実行されるスクリプトを上書きした上で任意のコードが実行できるということがわかった。  


## 脆弱性のデモンストレーション
CloudflareはHackerOne上に脆弱性開示制度を持っているため、脆弱性が実際に攻撃可能であることを示さなければ、HackerOneのトリアージチームがCloudflare側にレポートを転送しない可能性が高い。  
そのため、脆弱性が実際に攻撃出来ることを示すためにデモンストレーションを行うことにした。  
攻撃手順としては、以下のとおりとなる。
1. cdnjsに登録されたnpmパッケージで細工したファイル名を含む`.tgz`ファイルを使用して新しいバージョンを公開する。
2. cdnjsの自動更新サーバーがファイルを処理するのを待つ。
3. 細工した`.tgz`ファイルの中身が定期実行されているスクリプトファイルへと書き込まれ、任意のコードが実行される。

...と、ここまで考えた所でGitリポジトリをベースとした自動更新機能が気になってきた。  
そのため、脆弱性のデモンストレーションを行う前にコードを流し読みした所、Gitリポジトリからファイルをコピーする際に、シンボリックリンクの存在が考慮されていないように見受けられた。  
Gitはシンボリックリンクを扱うことが可能なため、Gitリポジトリにシンボリックリンクを含め、cdnjsに処理させることによりcdnjsシステム上の任意のファイルを読み取れる可能性がある。  

ファイルを上書きして任意のコードを実行するように書き換えた場合、自動更新機能に障害が発生してしまう可能性があったため、シンボリックリンクによる任意ファイル読み取りを先に検証し、パストラバーサルに関してはそのレポート内でデモンストレーションを行おうと考えた。  
これに伴い、攻撃手順を以下のように変更した。  
1. cdnjsに登録されたGitHubリポジトリに、無害なファイル(ここでは`/proc/self/cmdline`を想定)へとリンクさせたシンボリックリンクを追加する。
2. 当該のGitHubリポジトリ上で、新しいバージョンを公開する。
3. cdnjsの自動更新サーバーがファイルを処理するのを待つ。
4. 指定したファイルが読み取られ、公開される。

この時点で20時頃だったのだが、シンボリックリンクを作成するだけであれば直ぐに済ませられると考え、シンボリックリンクを作成してから夕飯を食べることにした。[^4]  
```bash
ln -s /proc/self/cmdline test.js
```

[^4]: 記憶が定かではないが、この日の夕食は冷凍餃子だったと記憶している。

## インシデント
夕飯を済ませ、PCの前に戻ってきた所、cdnjsがシンボリックリンクを含むバージョンを公開していることが確認できた。  
その後、レポートを送信するためにファイル内容を確認して驚愕した。  
なんと、`GITHUB_REPO_API_KEY`や、`WORKERS_KV_API_TOKEN`といった明らかに機微な情報が表示されていたのである。  
一瞬何が起こったのか理解できず、コマンドのログを確かめた所、誤って`/proc/self/cmdline`ではなく`/proc/self/environ`へのリンクを貼っていたことが確認できた。[^5]  
先述の通り、cdnjsのGitHub Organizationが侵害された場合、cdnjsの大部分を侵害することが可能となる。  
すぐにでも対応する必要があったため、レポートには現在の状況がわかるリンクとトークン類を全て取り消して欲しいという旨のみを記載し、送信した。  

[^5]: 仕事で疲れていたのと、空腹だった事が重なり、ろくに確認もせずに補完されたコマンドを実行してしまっていた。

この時点ではとても焦っており確認していなかったのだが、実はレポートを送信する前にこれらのトークンは無効化されていた。  
これは後からわかったことなのだが、`GITHUB_REPO_API_KEY`(GitHubのAPIキー)が含まれていたことにより、即座にGitHubが自動で通知を行い、それを受け取ったCloudflareはインシデントレスポンスを開始していたらしい。  
cdnjsが細工されたパッケージを処理してから数分も立たない内に各種認証情報の無効化を行っていたらしく、流石Cloudflareという印象を受けた。  

## 事後処理
その後、詳細な影響範囲の調査を行った。  
`GITHUB_REPO_API_KEY`はcdnjsのGitHub Organizationに所属している[robocdnjs](https://github.com/robocdnjs)というアカウントのアクセストークンであり、cdnjsの各リポジトリに対する書き込み権限を持っていた。  
つまり、cdnjs上でホストされている任意のライブラリの改竄や、ウェブサイトの改竄などをこのトークンを用いて行うことができた。  
また、`WORKERS_KV_API_TOKEN`は、cdnjsに使用されているCloudflare WorkersのKVに対する書き込み権限を持っており、KVにキャッシュされたライブラリ情報を改竄することが可能だった。  
これらの権限を組み合わせることにより、cdnjsのオリジンデータからKVキャッシュ、更にはcdnjsのウェブサイトといったcdnjsのコア部分を完全に侵害することができたと考えられる。  

## まとめ
今回の記事では、cdnjsに存在した脆弱性について解説しました。  
脆弱性自体は特殊なスキル無しで悪用可能なものでしたが、それによって潜在的に生じる影響が非常に大きいものでした。    
世の中にはこの脆弱性のような、悪用の容易さに対して影響が見合っていない物が多数あると考えると、非常に恐ろしいものだと感じます。  
本記事に関する質問はTwitter([@ryotkak](https://twitter.com/ryotkak))へメッセージを投げてください。
## タイムライン
|日付 (日本時間)|出来事|
|----|----|
|2021/04/06 19時頃|脆弱性の発見、検証開始|
|2021/04/06 20時頃|デモンストレーション用のファイルを公開|
|2021/04/06 20時半頃|cdnjsがファイルを処理|
|同時刻|GitHubがアラートをCloudflareへ送信|
|同時刻|Cloudflare内部でインシデントレスポンスが始まる|
|数分以内|各種認証情報の取り消しが完了する|
|2021/04/06 20時40分頃|初期レポートを送信|
|2021/04/06 21時頃|詳細なレポートを送信|
|2021/04/07～|二次対応が完了|
|2021/06/03|完全な対応が完了|
|2021/07/XX|本記事の開示|