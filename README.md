# YouTubeライブ配信で占星術をする

## 概要

- YouTubeライブ配信でコメントを取得し、コメントを元に占星術をするという企画で使用するアプリケーション
- 占い結果のを確認、音声ファイルの再生などの操作を行うためのUIを提供する

## 実行環境

- Python: 3.11
- poetry: 1.8系
- Docker:
  - DB: PostgreSQL
  - Grafana


## 環境構築

1. このリポジトリをクローンする

    ```bash
    git clone https://github.com/Nao-Y1996/soothsayer.git
    ```

2. 依存関係をインストールする
    
    poetry2系を使用している場合は `poetry self add poetry-plugin-shell` を実行して、`poetry shell`を利用できる必要があります
  
    ```bash
    poetry shell
    poetry install --no-root
    ```
3. 必要なファイルの作成

   以下のコマンドで `.env` や `config.py` などが作成される

    ```bash
    poetry run python setup_init.py
    ```
4. 環境変数のセット

    以下の2つのAPIを利用するためのAPIキーを`.env` にAPIキーを設定する
    - YouTube Data API v3
    - Generative Language API
    
    以下の環境変数はそのままでも良い

    ```bash
    POSTGRES_USER=user_for_postgres
    POSTGRES_PASSWORD=password_for_postgres
    POSTGRES_DB=db_for_postgres
    POSTGRES_HOST=localhost
    POSTGRES_PORT=5432
    ```

5. データベースを立ち上げる
    
    Dockerを起動して以下のコマンドを実行する 

    ```bash
    docker compose up -d 
    ```

6. テーブルの初期化

    ```bash
    poetry run alembic upgrade head
    ```

7. アプリケーションの初回起動と音声出力先の設定
  
    以下でアプリケーションを起動する
  
    ```bash
    poetry run python ui_manual.py
    ```
   
    以下のようなエラーと共に、選択可能な音声出力先が表示される。(MacBookの例)

    ```
    ValueError: Configured Device  is not available.
    Please set AUDIO_DEVICE_NAME in config.py from following devices:
    
    Index: 0 | Name: BenQ GW2283 | Input Channels: 0 | Output Channels: 2
    Index: 1 | Name: BenQ GW2283 | Input Channels: 0 | Output Channels: 2
    Index: 2 | Name: MacBook Pro Microphone | Input Channels: 1 | Output Channels: 0
    Index: 3 | Name: MacBook Pro Speakers | Input Channels: 0 | Output Channels: 2
    Index: 4 | Name: VB-Cable | Input Channels: 2 | Output Channels: 2
    ```
   
   この中から、出力先のデバイス名を選択して、`app/config.py`に設定します。  例えば `VB-Cable`を出力先にした場合は、以下のように設定する。
   
   ```python
   # ======= 音声出力先の設定 ========
   AUDIO_DEVICE_NAME = "VB-Cable"
   # ================================
   ```
   
   再度 以下のコマンドでアプリケーションを起動する

   ```bash
   poetry run python ui_manual.py
   ```

   OBSの画面更新と音声再生を自動で行うする場合は以下のコマンドで起動する
   
   ```bash
   poetry run python ui_auto.py
   ```

8. ブラウザで`http://localhost:7860/?__theme=light` にアクセスし、画面が表示されれば環境構築は成功です

## 使用方法

[docs/app_usage.md](docs/app_usage.md)を参照してください

## テーブルの一覧

### youtube_livechat_message

ライブチャットのメッセージを保存するテーブル

| カラム名       | データ型      | 説明                       |
|------------|-----------|--------------------------|
| id         | str       | 主キー                      |
| message    | jsonb     | メッセージの内容やメタデータ全てを含んだjson |
| created_at | timestamp | メッセージの作成日時               |
| updated_at | timestamp | メッセージの更新日時               |

### western_astrology_status

占星術の状態を保存するテーブル

| カラム名              | データ型      | 説明               |
|-------------------|-----------|------------------|
| message_id        | str       | メッセージのID, 主キー    |
| is_target         | bool      | 占い対象かどうか         |
| required_info     | jsonb     | 占いをするために必要な情報    |
| result            | text      | 占いの結果            |
| result_voice_path | text      | 音声ファイルのパス        |
| is_played         | bool      | 音声ファイルが再生されたかどうか |
| created_at        | timestamp | 作成日時             |
| updated_at        | timestamp | 更新日時             |

## 処理の詳細

4スレッドの処理とGradioのUIによって構成されている

### スレッド1: ライブ配信のコメント取得

- youtubeAPIを使って占いをするライブ配信に対するコメントを取得し続ける
- 取得したコメントを`youtube_livechat_message`の`message`列に入れる
- `youtube_livechat_message`テーブルから`western_astrology_status`に`message_id`が存在しないレコードを取得する
- コメントの内容を解析し、占い対象かどうかを判定する（コメントに`占い依頼`キーワードが含まれているかどうか）
- `western_astrology_status`テーブルに対して、`is_target`を設定した上で保存する

### スレッド2: 占い対象のコメントから占いに必要な情報を取得

- 占い対象のコメント一覧のそれぞれに対して、`message`をLLMに投げて`required_info`を抽出して`western_astrology_status`テーブルに保存する
- `western_astrology_status`テーブルから、「`result`がなく、`required_info`がある」レコード一覧を取得する
- レコードのそれぞれに対して`required_info`からLLMのAPIで占い結果を生成する
- 生成され次第、順番に占い結果をテーブルに反映する（`required_info`と`result`を更新する）

### スレッド3: 音声ファイルの生成

- `western_astrology_status`テーブルから、`result`が存在し、`result_voice_path`がない」レコード一覧を取得する
- レコード一覧のそれぞれに対して、text2speechで音声ファイルを作成してディレクトリに保存する（時間がかかる）
- 保存した音声ファイルのパスをテーブルに反映する（`result_voice_path`を更新する）

### スレッド4: 音声ファイルの再生とOBSの画面更新

このスレッドは、ui_auto.pyを実行した場合のみ動作する（ui_manual.pyでは手動で音声ファイルの再生とOBSの画面更新を行う）

- `western_astrology_status`テーブルから、`result_voice_path`が存在し、`is_played`がFalseのレコード一覧を取得する
- 一覧のうち最も古いレコードの音声ファイルともるstatusでOBSの画面を更新する
- 音声ファイルを再生する

### 画面表示

- 更新ボタン: `western_astrology_status`テーブルから、`required_info`が空ではない（占いに必要な情報抽出が完了した）レコード一覧を取得する
- Progress View のリンク: Grafanaのダッシュボードにリンクしており、データの生成状況が可視化されている
- START/STOP ボタン: 各スレッドの開始と停止を行う

### その他

#### 占星術の実施方法

- 取り出したコメントが占星術の対象である場合には、`pyswisseph`とLLMを使って占星術を行う

#### 音声ファイルの生成方法

- 占星術の結果(テキスト)を [style-bert-vit2](https://github.com/litagin02/Style-Bert-VITS2)を用いて音声ファイルに変換する
    - この時style-bert-vit2のAPIサーバーを別途立ち上げておく必要がある

## 開発時に使用するコマンド

1. DBに接続する

  ```
  docker build -t my_postgres_image -f Dockerfile_postgres .
  ```

  ```
  docker-compose exec postgres psql -U uranaishi -d uranai
  ```

2. 独自Dockerfileをcomposeを使ってビルドする場合

  ```bash
  docker-compose up --build -d 
  ```

3. マイグレーションを行う

  ```bash
  alembic revision --autogenerate -m "comment"
  alembic upgrade head
  ```

4. DBを含めてコンテナを作り直す

保存したデータやGrafanaのダッシュボードも消えるので注意

  ```bash
  docker-compose down
  docker-compose up --build -d
  ```

## クエリ

便利なクエリを記載しておく。これらはGrafanaで可視化する際にも使用している

### コメントと占い結果の一覧

```sql
SELECT chats.created_at,
       chats.message -> 'snippet' -> 'displayMessage'    as message,
       chats.message -> 'authorDetails' -> 'displayName' as name,
       status.required_info,
       status.result,
       status.result_voice_path
FROM youtube_livechat_messages as chats
         JOIN western_astrology_statuss as status
              on (chats.id = status.message_id)
```

### コメント数

```sql
SELECT COUNT(1) as コメント数
FROM youtube_livechat_messages
```

### 占い依頼数

```sql
SELECT COUNT(1) as 占い依頼数
FROM western_astrology_statuss
WHERE is_target
```

### 準備完了数

```sql
SELECT COUNT(1) as 準備完了数
FROM western_astrology_statuss
WHERE is_target
  and required_info != '{}' --and result is null and result_voice_path is null
```

### 占い完了数

```sql
SELECT COUNT(1) as 占い完了数
FROM western_astrology_statuss
WHERE is_target
  and required_info != '{}' and result != '' and result_voice_path = ''
```

### 音声ファイル生成数

```sql
SELECT COUNT(1) as 音声生成数
FROM western_astrology_statuss
WHERE is_target
  and required_info != '{}' and result != '' and result_voice_path != ''
```

### 音声再生数

```sql
SELECT COUNT(1) as 再生数
FROM western_astrology_statuss
WHERE is_target
  and required_info != '{}' and result != '' and result_voice_path != '' and is_played
```