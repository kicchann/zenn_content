---
title: "TerraformでFastAPI + Cloud Run + Cloud SQL環境を構築した話"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "fastAPI", "GCP", "terraform"]
published: true
published_at: 2025-03-13 08:00 # 未来の日時を指定する
---

## 概要

この記事は、Terraform を用いて以下の構成を自動構築できるようになるまでの経緯のメモです。

- FastAPI
- Cloud Run
- Cloud SQL
- Artifact Registry
- Cloud Build

初めてのGCPアプリなのですが、コンソールやCLIで設定を進めると、後で困ること請け合いです。  
なので、GPT様の力を借りながらなるべくTerraformで管理することを心掛けました。

---

## 最終的に完成したインフラ構成

以下のすべてを Terraform により自動構築しました。

### 構成図

```
GitHub → Cloud Build (CI/CD) → Artifact Registry (コンテナイメージ管理)
                                 ↓
                           Cloud Run (FastAPIアプリケーション)
                                 ↓
                            Cloud SQL (MySQL)
```

### 構築したGCPリソース一覧

| リソース                      | 用途                                     |
|----------------------------|----------------------------------------|
| Artifact Registry           | Dockerイメージ保管                           |
| Cloud Build                 | CI/CD 自動ビルド＆デプロイ                     |
| Cloud Run                   | FastAPI アプリケーション実行                   |
| Cloud SQL (MySQL)            | データベース                                 |
| Cloud Storage               | 画像やファイルの保存                           |
| サービスアカウント (Cloud Run/Build用) | 権限管理                                    |
| IAMポリシー                  | 最小権限のアクセス制御                        |
| GitHub トリガー (手動連携)   | ソースコード更新によるビルド＆デプロイトリガー        |

---

## Terraform の実行前準備

そうはいってもTerraformだけではなく、GCPの設定も必要です。
以下、GCPプロジェクトの作成からTerraform実行までの手順を簡単に紹介します。

### 1. GCP プロジェクトを作成

```bash
gcloud projects create project-name
```

### 2. 有効化

GCPで用いるサービスを有効化します。

```bash
gcloud services enable run.googleapis.com \
sqladmin.googleapis.com \
artifactregistry.googleapis.com \
cloudbuild.googleapis.com \
iam.googleapis.com \
cloudresourcemanager.googleapis.com --project=project-name
```

### 3. サービス アカウントを作成

Terraformを実行するためのサービスアカウントを作成します。

```bash
gcloud iam service-accounts create terraform-admin \
  --display-name "Terraform Admin Service Account"
```

さらに、サービスアカウントに必要な権限を付与します。
実際には最小限の権限に留めるべきですが、今回は簡単のため `roles/owner` を付与します。

```bash
gcloud projects add-iam-policy-binding project-name \
  --member="serviceAccount:terraform-admin@project-name.iam.gserviceaccount.com" \
  --role="roles/owner"
```

### 4. サービス アカウント キーを作成

```bash
gcloud iam service-accounts keys create ~/secrets/project-name-key.json \
  --iam-account=terraform-admin@project-name.iam.gserviceaccount.com
```

### 5. サービス アカウント キーを環境変数に設定

```bash
export GOOGLE_APPLICATION_CREDENTIALS=~/secrets/app-key.json
```

これで、Terraformの実行環境が整いました。  
こういうのも忘れそうなので、私はTerraformディレクトリに'before_terraform.md'として保存しておくことにしました。

---

## Terraform モジュール（主要部）

結論からですが、`main.tf`には以下のようなリソースが定義されています。
初学者の自分のために、なるべくコメントを入れていますが、ご容赦ください。

```hcl:main.tf
# ========================================
# Terraform のプロバイダー設定
# ========================================
# GCP (Google Cloud Platform) を使用するためのプロバイダー
# Staging や Production など、環境ごとに gcp_project を切り替えることで複数環境に対応
provider "google" {
  project = var.gcp_project  # 環境ごとの GCP プロジェクトID
  region  = var.region      # デフォルトリージョン
}

# ========================================
# Artifact Registry (Docker レポジトリ)
# ========================================
# コンテナイメージを管理するためのリポジトリ
resource "google_artifact_registry_repository" "app_repo" {
  location      = var.region
  repository_id = var.artifact_repo  # 変数化
  description   = "Artifact Registry for app api"
  format        = "DOCKER"
}

# ========================================
# Cloud Storage バケット
# ========================================
# 画像やドキュメントの保管先
# プロジェクトごとに異なる名前を付与
resource "google_storage_bucket" "app_bucket" {
  name     = "app-storage-${var.gcp_project}"
  location = var.region
  uniform_bucket_level_access = true
}

# ========================================
# Cloud SQL (MySQL) データベースインスタンス
# ========================================
# 小規模プラン (db-f1-micro) を使用し、削除保護も環境ごとに切替可能
resource "google_sql_database_instance" "app_db" {
  name             = "app-db"
  database_version = "MYSQL_8_0"
  region           = var.region
  deletion_protection = var.deletion_protection
  settings {
    tier = "db-f1-micro"
    backup_configuration {
      enabled = true
    }
  }
}

# Cloud SQL 内のデータベース（スキーマ）
resource "google_sql_database" "app_database" {
  name     = "app-db"  # ← これがアプリで接続するDB名
  instance = google_sql_database_instance.app_db.name
}

# ========================================
# Cloud SQL ユーザー
# ========================================
resource "google_sql_user" "app_user" {
  name     = "app_admin"
  instance = google_sql_database_instance.app_db.name
  password = var.db_password
}

# ========================================
# Cloud Run & Cloud Build 用 サービスアカウント
# ========================================
resource "google_service_account" "cloud_run_sa" {
  account_id   = "app-cloud-run-sa"
  display_name = "Service Account for Cloud Run"
}

resource "google_service_account" "cloud_build_sa" {
  account_id   = "app-cloud-build-sa"
  display_name = "Service Account for Cloud Build"
}

# ========================================
# IAM権限付与
# ========================================
# Cloud Run サービス用権限
resource "google_project_iam_member" "artifact_reader" {
  project = var.gcp_project
  role    = "roles/artifactregistry.reader"
  member  = "serviceAccount:${google_service_account.cloud_run_sa.email}"
}

resource "google_project_iam_member" "run_cloud_sql_client" {
  project = var.gcp_project
  role    = "roles/cloudsql.client"
  member  = "serviceAccount:${google_service_account.cloud_run_sa.email}"
}

# Cloud Build 用権限
resource "google_project_iam_member" "artifact_writer" {
  project = var.gcp_project
  role    = "roles/artifactregistry.writer"
  member  = "serviceAccount:${google_service_account.cloud_build_sa.email}"
}

resource "google_project_iam_member" "run_admin" {
  project = var.gcp_project
  role    = "roles/run.admin"
  member  = "serviceAccount:${google_service_account.cloud_build_sa.email}"
}

resource "google_project_iam_member" "cloud_build_storage_admin" {
  project = var.gcp_project
  role    = "roles/storage.admin"
  member  = "serviceAccount:${google_service_account.cloud_build_sa.email}"
}

resource "google_project_iam_member" "cloud_build_log_writer" {
  project = var.gcp_project
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.cloud_build_sa.email}"
}

resource "google_service_account_iam_member" "cloud_build_actas_cloud_run" {
  service_account_id = google_service_account.cloud_run_sa.name
  role               = "roles/iam.serviceAccountUser"
  member             = "serviceAccount:${google_service_account.cloud_build_sa.email}"
}

# ========================================
# Cloud Run サービス (API サーバー)
# ========================================
# Cloud SQL 接続には annotations を使用し、Unix ソケット経由で接続
resource "google_cloud_run_service" "app_api" {
  name     = "app-api"
  location = var.region
  template {
    metadata {
      annotations = {
        "run.googleapis.com/cloudsql-instances" = google_sql_database_instance.app_db.connection_name
      }
    }
    spec {
      service_account_name = google_service_account.cloud_run_sa.email
      containers {
        image = "${var.region}-docker.pkg.dev/${var.gcp_project}/${var.artifact_repo}/app-api:latest"
        env {
          name  = "DATABASE_URL"
          value = "mysql+pymysql://app_admin:${var.db_password}@/app-db?unix_socket=/cloudsql/${google_sql_database_instance.app_db.connection_name}"
          # username:password@/dbname?unix_socket=/cloudsql/project:region:instance
        }
        env {
          name  = "ALEMBIC_CONFIG_PATH"
          value = "alembic.ini"
        }
        env {
          name = "ENV"
          value = var.env
        }
        env {
          name  = "FIREBASE_CERT"
          value = var.firebase_cert
        }
        env {
          name  = "GCS_BUCKET_NAME"
          value = google_storage_bucket.app_bucket.name
        }
        env {
          name  = "GOOGLE_CLIENT_ID"
          value = var.google_client_id
        }
      }
    }
  }
  autogenerate_revision_name = true
}

# ========================================
# Cloud Build トリガー (GitHub 連携)
# ========================================
# GitHub のブランチ push を契機に自動デプロイ
resource "google_cloudbuild_trigger" "app_build" {
  name = "app-build-trigger"
  location = "global"
  filename = "cloudbuild.yaml"
  project  = var.gcp_project
  github {
    owner = var.github_owner
    name  = var.github_repo
    push {
      branch = "^${var.github_branch}$"  # 指定ブランチでの push に反応
    }
  }
  # これがないと400エラーになる
  # 仕様変更: https://blog.g-gen.co.jp/entry/cloud-build-service-account-changes
  service_account = google_service_account.cloud_build_sa.id
}

# ========================================
# 変数は variables.tf に分離
# - gcp_project (環境ごとの GCP プロジェクト)
# - region (リージョン)
# - db_password (DBパスワード)
# - deletion_protection (削除保護: 本番 true, 開発 false)
# - github_owner (GitHub オーナー)
# - github_repo (GitHub リポジトリ名)
# - github_branch (対象ブランチ)
# - artifact_repo (Artifact Registry リポジトリ名)
# - env (環境名)
# - firebase_cert (Firebase 証明書)
# ========================================
```

---

## ハマったポイント・注意点

### ① Cloud Build がログを書き込めない（Storage権限）
#### ❌ エラー内容
```
service account ... does not have access to bucket "gs://24329002846-global-cloudbuild-logs". Please grant the service account roles/storage.admin.
```

#### ✅ 解決策
ログをCloud Storageに書き込んでいるようで、Cloud Build用サービスアカウントに `roles/storage.admin` を付与する必要があります。

```hcl
resource "google_project_iam_member" "cloud_build_storage" {
  project = var.gcp_project
  role    = "roles/storage.admin"
  member  = "serviceAccount:${google_service_account.cloud_build_sa.email}"
}
```

---

### ② Cloud Run デプロイ時の権限不足
#### ❌ エラー内容
```
Permission 'iam.serviceaccounts.actAs' denied on service account .... This command is authenticated as ... which is the active account specified by the [core/account] property.
```

#### ✅ 解決策
これもよくわからないエラーですが、要はCloud Build が Cloud Run 用サービスアカウントとしてアクセスする権限が不足しているということです。  
Cloud Build用サービスアカウントに `roles/iam.serviceAccountUser` を付与する必要があります。

```hcl
resource "google_project_iam_member" "run_act_as" {
  project = var.gcp_project
  role    = "roles/iam.serviceAccountUser"
  member  = "serviceAccount:${google_service_account.cloud_build_sa.email}"
}
```

---

### ③ Cloud SQL 接続時の接続文字列ミス

#### ✅ 正しい例
```bash
mysql+pymysql://username:password@/dbname?unix_socket=/cloudsql/project:region:instance
```

#### Terraform 環境変数渡し例
```hcl
env {
  name  = "DATABASE_URL"
  value = "mysql+pymysql://app_admin:${var.db_password}@/app-db?unix_socket=/cloudsql/${google_sql_database_instance.app_db.connection_name}"
}
```

---

### ④ GitHub 連携トリガーのTerraform化

#### ❌ 第1世代トリガーでの問題
GitHub 連携トリガーには第1世代と第2世代があります。  
第2世代は完全にTerraformで管理できるようになっています（とはいえ、Github Appを用意する必要があり、こっちはこっちでややこしそうですが）。  
しかし、第1世代はすべてをTerraformで管理することができず、GitHubリポジトリの接続は **Terraform非対応**です。

#### ✅ 解決策
GitHub接続はGUI (Cloud Console) で設定して、トリガー自体はTerraformで管理。
ただし、Terraformで記述できるのはリポジトリやブランチの指定程度で、cloudbuild.yaml はリポジトリに配置しておく必要があります。

```hcl
resource "google_cloudbuild_trigger" "app_build" {
  name = "app-build-trigger"
  location = "global"
  filename = "cloudbuild.yaml" ← cloud build の設定ファイル
  project  = var.gcp_project
  github {
    owner = var.github_owner
    name  = var.github_repo
    push {
      branch = "^${var.github_branch}$"  # 指定ブランチでの push に反応
    }
  }
  ...
}
```

### ⑤ Cloud Buildの仕様変更

#### ❌ エラー内容
```
Error: Error creating Trigger: googleapi: Error 400: Request contains an invalid argument.
```

#### ✅ 解決策

GPTに聞くだけでは解決できず、これが一番困りました。  
この記事によると仕様変更があったのは2024/4/29のことらしいです。  
やはり、「公式ドキュメントを見ろ」の原理原則は大事と痛感しましたが、私の実力では読み解けないので助かります。。。

https://blog.g-gen.co.jp/entry/cloud-build-service-account-changes

```hcl
resource "google_cloudbuild_trigger" "app_build" {
  ...
  # これがないと400エラーになる
  service_account = google_service_account.cloud_build_sa.id
}
```

---

## 最終的な運用ルールと工夫

| 項目                    | 工夫したポイント                                  |
|----------------------|-------------------------------------------|
| CI/CD                 | GitHub→Cloud Build→Artifact Registry→Cloud Run   |
| DB接続               | Cloud SQL Auth Proxyを使わず、unixソケット直指定  |
| 権限 (IAM)             | Cloud Run/Buildごとに専用サービスアカウントと権限最小限付与 |
| GitHubトリガー         | 第一世代はGUI連携、Terraformでビルド/デプロイ定義  |
| コンテナイメージ管理     | Artifact Registry (Docker形式)                  |
| ストレージ             | Cloud Storageでファイル保管                    |

---

## まとめ

いろいろ苦労はしましたが、Terraformでインフラを管理することで、再現性が高まり、修正コストも下がりました。
また、コメントをしつこく残すことで、後で見返しても理解しやすい資料になりました。  
コードの書き込みに集中できるようになったので、その点でも満足しています。  
  
なにより、**コンソールで設定を進めてしまうとGPTで相談しづらい**のです。。。  
Terraformであれば単なるテキストなので、どこにどんな問題が潜んでいるのか相談ができます。  
でなければ、初学者でこんなことはできません。  
  
参考になれば幸いです。
