---
title: "GPT-4oとGoogle Forms、GCPを使って動画要約アプリを開発しました"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GPT4o", "whisper", "GCP", "video"]
published: true
published_at: 2024-08-02 12:00 # 未来の日時を指定する
---

## 概要

最近のオンライン会議ツールは、プレミアム版であれば要約機能がついていますし、要約アプリも多くリリースされています。  
しかしながら大抵の場合は単発利用はできず、サブスク契約やアカウント登録が必要です。  
そこで、単発利用で手軽に試すことができる動画要約ツールをリリースしました。  
  
https://sites.google.com/view/brewed-summary/ja  

手軽ではありますが、OpenAIのwhisperとGPT-4oを使って要約生成しているので、それなりの品質ではあるかと思います。  
また、動画キャプチャを参照情報として使うことで、専門分野の内容にもある程度理解した上で要約できるようになっています。  

UIを作るのは得意ではありませんので、Google sitesとForms、要約結果の送付にはGmailを使って簡単に作っています。  

使い方はこちらに整理しましたので、こちらを参照してください。

https://note.com/kicchann/n/n5a9cab7fa5e8

以下では、処理の内容について説明したいと思います。

## 処理の流れ

大まかには、以下のような流れで動作しています。

1. Google Formsで動画ファイルおよび必要な情報を入力
   - 動画ファイルはGoogle Driveにアップロード
   - 回答内容はスプレッドシートに保存
2. Cloud RunでデータをCloud Storageに転送
   - Cloud SchedulerとEventarcを使って定期的に監視・処理
3. 文字起こしと動画ファイルの場合はキャプチャ画像の抽出
   - 文字起こしにはWhisperを使用
   - キャプチャ画像の抽出にはffmpegを使用。画像の類似度で適度にフィルタリング
4. 要約生成
   - GPT-4oを使用
   - キャプチャ画像を参照情報として使用
5. 要約結果をメール送信

![](/images/0002_brewed_summary/flow.png)


特段難しいことはしていないのですが、Google Cloudの使い方で苦労したのと、あとはプロンプトやFunction Calling、画像のフィルタリングについて、自分の備忘として詳細記事を残していきたいと思います。