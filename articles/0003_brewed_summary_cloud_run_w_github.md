---
title: "GitHubからCloud Runに継続的にデプロイする"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "CloudRun", "GCP"]
published: true
published_at: 2024-08-23 08:00 # 未来の日時を指定する
---

## 概要

GitHubを使ってCloud Runにデプロイする方法は以下のリンクに記載されています。

https://cloud.google.com/run/docs/quickstarts/deploy-continuously?hl=ja

ここでは手順を一つずつ確認していきたいと思います。

## 処理の流れ

Cloud Runのサービスの作成から開始することができます。  

![](/images/0003_brewed_summary_cloud_run_w_github/000.jpg =600x)

「リポジトリから継続的にデプロイする」を選択すると、「CLOUD BUILDを設定」が表示されるので、これをクリックします。

![](/images/0003_brewed_summary_cloud_run_w_github/001.jpg =600x)

接続したいリポジトリを選択して、、、  

![](/images/0003_brewed_summary_cloud_run_w_github/002.jpg =600x)

ブランチを選択します。今回は「develop」ブランチを選択しました。
ビルドタイプは「Dockerfile」を選択します。これでリポジトリ内のDockerfileを使ってビルドすることができます。

![](/images/0003_brewed_summary_cloud_run_w_github/003.jpg =600x)

その後、サービス名やリージョンを選択します。「develop」ブランチを選択しましたので、サービス名の語尾に「-dev」を付けました。あとはデフォルト値のままで「作成」をクリックします。このあたりは後で変更することができます。

![](/images/0003_brewed_summary_cloud_run_w_github/004.jpg =600x)

これでCloud Runに継続的にデプロイできるようになりました。「作成」をクリックすると、早速ビルドが開始されます。

![](/images/0003_brewed_summary_cloud_run_w_github/005.jpg =600x)

めちゃくちゃ簡単ですね。