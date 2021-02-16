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
この問題の影響を受けるリポジトリがどの程度存在するかを調査した所、有名なソフトウェアのリポジトリを含めた複数のリポジトリがこの攻撃に対して脆弱であることがわかった。  

## 調査内容
GitHub Actionsにおける存在しないユーザー名を指定した`uses`構文の使用によるリポジトリ侵害の可能性とその影響範囲を調査した。  

## 調査理由
[GitHub Actionsを使ったDDoSに巻き込まれた](https://blog.utgw.net/entry/2021/02/05/133642)という記事を読み、以前から調査したいと考えていたサプライチェーン攻撃に関連した調査をGitHub Actionsで行うことを思いついたため、調査内容を考え初めた。  
過去に[Repo Jacking: Exploiting the Dependency Supply Chain](https://blog.securityinnovation.com/repo-jacking-exploiting-the-dependency-supply-chain)という記事を読んでおり、追加の検証無しにリポジトリを直接参照している場合、サプライチェーン攻撃が刺さりやすくなる事を知っていたため、GitHub Actionsの[`uses`構文](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepsuses)が同様の挙動をするかどうか確認した。  
その結果、同様の挙動をすることがわかったため、本調査を実施することにした。  

