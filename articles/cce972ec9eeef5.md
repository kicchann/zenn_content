---
title: "[vue.js]Docker内でvueとfirebase emulatorを使って開発"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [vue.js, docker, firebaseemulator, firebase]
published: false
---

## 概要
vueの勉強をしている際に，firebaseのemulatorを使ってみたいと思ったのですが，苦労したので備忘録として残します．

## 環境

- WSL2 + Ubuntu 22.04
- Docker Desktop for Windows
- vue.js

## docker構成

フォルダ構成は以下の通りです．

```yaml
version: '3'

services:
  vue-app:
    build:
      context: "./frontend"
      dockerfile: "Dockerfile"
    ports:
      - 8080:8080
    volumes:
      - ./frontend/app:/app
    tty: true
    environment:
      - NODE_ENV=development
  firebase:
    build:
      context: "./firebase"
      dockerfile: "Dockerfile"
    volumes:
      - ./.firebaserc:/opt/firebase/.firebaserc
      - ./firebase.json:/opt/firebase/firebase.json
      - ./firestore.indexes.json:/opt/firebase/firestore.indexes.json
      - ./firestore.rules:/opt/firebase/firestore.rules
      - ./storage.rules:/opt/firebase/storage.rules
      - ./firebase/bin/:/root/.cache:cached
      - ./firebase/config/:/root/.config:cached
      - ./functions:/opt/firebase/functions
      - ./extensions:/opt/firebase/extensions
    ports:
      - 4000:4000 # Emulator Suite UI
      - 5000:5000 # Firebase Hosting
      - 5001:5001 # Clound Functions
      - 9000:9000 # Realtime Database
      - 9099:9099 # Firebase Authentication
      - 8000:8000 # Cloud Firestore
      - 8085:8085 # Cloud Pub/Sub
      - 9199:9199 # Cloud Storage
    working_dir: /opt/firebase
    # firebase init してからemulators:startするまでのコマンドを実行
    command: bash
    tty: true
```