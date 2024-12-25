---
title: SpringBoot + React + MySQL マルチコンテナでサービスを起動する際に、DB起動を待ちきれずエラーになる
tags:
  - Docker
  - React
  - SpringBoot
  - マルチコンテナ
  - テーブルが読めない
private: false
updated_at: '2024-12-14T06:55:54+09:00'
id: 9fcbaab1a50dc618cac5
organization_url_name: null
slide: false
ignorePublish: false
---

こんにちは、hidepon4649 です。
マルチコンテナで、DB が完全に起動する前にバックエンドサービスが起動しテーブルが読めないエラーが発生しハマったのでメモします。

DB は MySQL に限らず、他の DBMS 製品でも同じ理屈だと思います。

# 発生したエラー

- schema.sql で create 文があるのに、一部のテーブルが読めない。
- data.sql で insert 文があるのに、データが読めない。

```エラーログ
SQL Error: 1146, SQLState: 42S02
Table 'データベース名.テーブル名' doesn't exist
```

DB が完全に起動する前(注 1)にバックエンドサービスが起動しているようです。
(注 1)schema.sql, data.sql が完全に実施される前

# 構成

- frontend, React
- backend, SpringBoot
- db, MySQL

デバッグの前後で構成は変えていません。変更したのは docker-compose.yml のみ。

# デバッグ対応

対応内容

- 1 .depends_on の db に「condition: service_healthy」 を付ける。(healthcheck と連動)
- 2 .depends_on の flyway に「condition: service_completed_successfully」 を付ける。(flyway を使っている場合のみ)

パラメータの詳細は下記ドキュメントを参照
<https://docs.docker.jp/compose/compose-file/compose-file-v3.html#depends-on>

- service_healthy: 依存サービスを開始する前に、依存関係が(healthcheck)によって示されるように、"healthy"であることが期待されます。
- service_completed_successfully: 依存サービスを開始する前に、依存関係が正常に終了していることを指定できる

以下がデバッグ対応です。

## デバッグ前

(注：エラーに関係無い箇所は記述を省略しています)

```docker-compose.yml
version: '3'
services:
  backend:
    depends_on:
      - db
      - flyway

  frontend:

  db:
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "db", "-u", "root", "-p$${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      retries: 3

  flyway:
    depends_on:
      - db
```

## デバッグ後

(注：エラーに関係無い箇所は記述を省略しています)

```docker-compose.yml
version: '3'
services:
  backend:
    depends_on:
      # ---- 変化点 ----
      db:
        condition: service_healthy
      # ---- 変化点 ----
      flyway:
        condition: service_completed_successfully

  frontend:

  db:
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "db", "-u", "root", "-p$${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      retries: 3

  flyway:
    depends_on:
      # ---- 変化点 ----
      db:
        condition: service_healthy
```

以上

私が上述のエラーでハマった際に、
condition が肝だという情報がなかなか見つけられなくて無駄に時間を費やしてしまいました。誰かのお役に立てれば幸いです。(私の検索能力が低いだけかも‥？)
