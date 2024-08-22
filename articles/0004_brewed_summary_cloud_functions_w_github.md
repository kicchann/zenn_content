---
title: "GitHubからCloud Functionsに継続的にデプロイする"
emoji: "✍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "CloudFunctions", "GCP"]
published: true
published_at: 2024-08-23 08:00 # 未来の日時を指定する
---

## 概要

GitHubを使ってCloud Functionsにデプロイする方法は以下のリンクに記載されています。

https://cloud.google.com/build/docs/deploying-builds/deploy-functions?hl=ja

ここでは手順を一つずつ確認していきたいと思います。

## 処理の流れ

Cloud Functions APIの有効化や、Cloud BuildのサービスアカウントにCloud Functionsデベロッパーのロールを付与したあと、Cloud Buildの設定ページを開きます。

![](/images/0004_brewed_summary_cloud_functions_w_github/001.jpg =600x)

「トリガーの作成」をクリックします。

![](/images/0004_brewed_summary_cloud_functions_w_github/002.jpg =600x)

トリガーの設定を行います。トリガーの名前や、リージョン、を設定します。イベントは「ブランチにpushする」にします。

![](/images/0004_brewed_summary_cloud_functions_w_github/003.jpg =600x)

ソースリポジトリを選択すると、これまで利用したリポジトリとともに、「新しいリポジトリに接続」が表示されますので、必要に応じて選択します。

![](/images/0004_brewed_summary_cloud_functions_w_github/004.jpg =600x)

「新しいリポジトリに接続」を選択すると、GitHubの認証とリポジトリの選択ができるようになります。

![](/images/0004_brewed_summary_cloud_functions_w_github/005.jpg =600x)

構成は```cloudbuild.yaml```を選択します。```cloudfunctions.yaml```には、以下のように記述します。

```yaml
steps:
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
  args:
  - gcloud
  - functions
  - deploy
  - FUNCTION_NAME # 関数名
  - --region=FUNCTION_REGION # リージョン
  - --source=.
  - --trigger-http # トリガー
  - --runtime=RUNTIME # ランタイム 例えばpython312
```

ここで、以下を記述すると、GitHubにプッシュされると同時にCloud Functionsにデプロイされるようになります。

```yaml
  - gcloud
  - functions
  - deploy
```

他には、以下のようなものを指定したりします。

```yaml
  - --gen2 # 第2世代のランタイムを使用する場合
  - --entry-point=hoge # エントリーポイントを指定する場合
  - --timeout=60s # タイムアウトを指定する場合
  - --memory=256MB # メモリを指定する場合
  - --project=$PROJECT_ID # プロジェクトIDを指定する場合
  - --env-vars-file=.env.yaml # 環墩変数を.env.yamlで指定する場合
```

環境変数を指定する方法は、こちらを参照してください。

https://cloud.google.com/functions/docs/configuring/env-var?hl=ja

例えば、以下のように記述します。

```yaml
 FOO: bar
 BAZ: boo
```

最後に「作成」をクリックすると、トリガーが作成されます。これで、GitHubにプッシュされるとCloud Functionsにデプロイされるようになります。