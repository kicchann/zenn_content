---
title: "Cloud StorageをトリガーにしてCloud Run jobsを実行する"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GCP", "CloudRun", "CloudStorage", "workflows", "eventarc"]
published: true
published_at: 2024-08-23 08:00 # 未来の日時を指定する
---

## 概要

要約アプリ Brewed Summaryは動画や音声を加工して、文字起こししたり要約を作成したりしています。

@[card](https://sites.google.com/view/brewed-summary/ja?authuser=0)

その際、メディアの加工にはffmpeg、文字起こしにはOpenAI Whisper、要約にはGPTなどを使っています。これらの処理をCloud Runで並列に実行しようとすると、Cloud Run jobsで実装するのが妥当な選択肢の１つになります。Cloud Run jobsでは、大きく３つの処理をしています。

1. メディアの転送
   Googleフォームで受け取ったファイルはGoogleドライブに保存されます。そのファイルをCloud Storageに転送します。
2. メディアの加工
   転送されたファイルをffmpegで加工します。
3. 文字起こし・要約作業
   Googleフォームのリクエストに応じて文字起こしや要約を作成します。

これらの処理を、Cloud Storageに保存したタスクファイルをトリガーにしてCloud Run jobsを実行しています。（普通のエンジニアだと、Cloud SQLでタスクを管理するような気もしますが。。。）つまり、それぞれの処理が完了したら、タスクファイルを上書き保存→トリガーが発動して、Cloud Run jobsが起動、といった流れです。この処理を実現するために、EventarcとWorkflowsを使う手順について紹介します。

## 処理の流れ

まずはWorkflowsの「作成」をクリックします。

![](/images/0005_brewed_summary_cloud_workflow/001.jpg =600x)

ワークフロー名はわかりやすく「cloud-run-job-workflow」などにします。利用するサービスアカウントには「ワークフロー起動元」「Eventarc イベント受信者」「Cloud Run 起動元」を付与します。

![](/images/0005_brewed_summary_cloud_workflow/002.jpg =600x)

そして、「新しいトリガーを追加」をクリックして、Eventarc側の設定を行います。

![](/images/0005_brewed_summary_cloud_workflow/003.jpg =600x)

設定はこんな感じです。「google.cloud.storage.object.v1.finalized」はCloud Storageのファイルが作成・上書き保存されたときにトリガーされるイベントです。

![](/images/0005_brewed_summary_cloud_workflow/004.jpg =600x)

「次へ」をクリックすると、workflowの設定を行います。このworkflowの記法がややこしくて苦労しました。ちなみにコードサンプルが用意されていて、以下のリンクにあるCloud Runを実行するサンプルを参考にしました。

https://cloud.google.com/workflows/docs/samples/workflows-cloud-run-jobs?hl=ja

```yaml
main:
    params: [event]
    steps:
        - init:
            assign:
                - project_id: ${sys.get_env("GOOGLE_CLOUD_PROJECT_ID")}
                - event_bucket: ${event.data.bucket}
                - event_file: ${event.data.name}
                - target_bucket: ${"YOUR TARGET BUCKET"}
                - job_name: JOB_NAME
                - job_location: asia-northeast1
        - check_input_file:
            switch:
                # バケット名とファイル名の接頭語で判定
                - condition: ${event_bucket == target_bucket and text.substring(event_file, 0, 5) == "PREFIX"}
                  next: run_job
                - condition: true
                  next: end
        - run_job:
            call: googleapis.run.v1.namespaces.jobs.run
            args:
                name: ${"namespaces/" + project_id + "/jobs/" + job_name}
                location: ${job_location}
                body:
                    # 環境変数でトリガーされたファイルを渡す
                    overrides:
                        containerOverrides:
                            env:
                                - name: INPUT_BUCKET
                                  value: ${event_bucket}
                                - name: INPUT_FILE
                                  value: ${event_file}
            result: job_execution
        - finish:
            return: ${job_execution}
```

```GOOGLE_CLOUD_PROJECT_ID```、```YOUR TARGET BUCKET```、```PREFIX```には適切な値を設定してください。これで準備完了です。