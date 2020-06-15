# EGG ハンズオン #3

## Google Cloud Platform（GCP）プロジェクトの選択

ハンズオンを行う GCP プロジェクトを作成し、 GCP プロジェクトを選択して **Start/開始** をクリックしてください。

**なるべく新しいプロジェクトを作成してください。**

<walkthrough-project-setup>
</walkthrough-project-setup>

## ハンズオンの内容

### 内容と目的

本ハンズオンでは、Cloud Spanner を触ったことない方向けに、インスタンスの作成から始め、Cloud Spanner に接続し API を使ってクエリする簡易アプリのビルドや、 SQL でクエリをする方法などを行います。

本ハンズオンを通じて、 Cloud Spanner を使ったアプリケーション開発における、最初の 1 歩目のイメージを掴んでもらうことが目的です。


### 前提条件

本ハンズオンははじめて Cloud Spanner を触れれる方を想定しておりますが、Cloud Spanner の基本的なコンセプトや、主キーによって格納データが分散される仕組みなどは、ハンズオン中では説明しません。
事前知識がなくとも本ハンズオンの進行には影響ありませんが、Cloud Spanner の基本コンセプトやデータ構造については、Coursera などの教材を使い学んでいただくことをお勧めします。 


## ハンズオンで使用するスキーマの説明

今回のハンズオンでは以下のように、3 つのテーブルを利用します。これは、あるゲームの開発において、バックエンド データベースとして Cloud Spanner を使ったことを想定しており、ゲームのプレイヤー情報や、アイテム情報を管理するテーブルに相当するものを表現しています。

![スキーマ](https://storage.googleapis.com/egg-resources/egg3/public/1-1.png "今回利用するスキーマ")

このテーブルの DDL は以下のとおりです、実際にテーブルを CREATE する際に、この DDL は再度掲載します。

```sql
CREATE TABLE players (
player_id STRING(36) NOT NULL,
name STRING(MAX) NOT NULL,
level INT64 NOT NULL,
money INT64 NOT NULL,
) PRIMARY KEY(player_id);
```

```sql
CREATE TABLE items (
item_id INT64 NOT NULL,
name STRING(MAX) NOT NULL,
price INT64 NOT NULL,
) PRIMARY KEY(item_id);
```

```sql
CREATE TABLE player_items (
player_id STRING(36) NOT NULL,
item_id INT64 NOT NULL,
quantity INT64 NOT NULL,
FOREIGN KEY(item_id) REFERENCES items(item_id)
) PRIMARY KEY(player_id, item_id),
INTERLEAVE IN PARENT players ON DELETE CASCADE;
```

## Cloud Spanner インスタンスの作成

### Cloud Spanner インスタンスの作成

![](https://storage.googleapis.com/egg-resources/egg3/public/2-1.png)

1. ナビゲーションメニューから「Spanner」を選択

![](https://storage.googleapis.com/egg-resources/egg3/public/2-2.png)

2. 「インスタンスを作成」を選択

### 情報の入力

![](https://storage.googleapis.com/egg-resources/egg3/public/2-3.png)

以下の内容で設定して「作成」を選択します。
1. インスタンス名：dev-instance
2. インスタンスID：dev-instance
3. 「リージョン」を選択
4. 「asia-northeast1 (Tokyo) 」を選択
5. ノードの割り当て：1
6. 「作成」を選択

### インスタンスの作成完了
以下の画面に遷移し、作成完了です。
どのような情報が見られるか確認してみましょう。

![](https://storage.googleapis.com/egg-resources/egg3/public/2-4.png)

### スケールアウトとスケールインについて

Cloud Spanner インスタンスノード数を変更したい場合、編集画面を開いてノードの割り当て数を変更することで、かんたんに行われます
ノード追加であってもノード削減であっても、一切のダウンタイムなく実施することができます。

![](https://storage.googleapis.com/egg-resources/egg3/public/2-5.png)

## 接続用テスト環境作成 Cloud Shell 上で構築

続いて、作成した Cloud Spanner に対して負荷掛け等を行うコマンドを実行するために Cloud Shell を実行します。

Cloud Shell によるターミナルを開いていない人は、以下の手順に従って、Cloud Shell を開いてください。

本チュートリアルの開始時点で、Cloud Shell は自動的に起動していますので、そのまま利用いただいて問題ありません。

![](https://storage.googleapis.com/egg-resources/egg3/public/3-1.png)

1. ナビゲーションメニューから「Cloud Shell をアクティブにする」を選択
2. 初回実行時は画面下部に説明文が表示されるので、「続行」を選択すると、ターミナルが起動する

続いて、環境変数 `PROJECT_ID` に、各自で利用しているプロジェクトのIDを格納しておきます。以下のコマンドを、Cloud Shell のターミナルで実行してください。

```bash
export PROJECT_ID=$(gcloud config list project --format "value(core.project)")
```

以下のコマンドで、正しく格納されているか確認してください。

```bash
echo $PROJECT_ID
```

## Cloud Spanner 接続クライアントの準備

### Cloud Spanner に対するデータの読み書きの方法

Cloud Spanner へのデータの読み書きには、様々な方法があります。

クライアント ライブラリ を使用しアプリケーションを作成し読み書きする方法が代表的なものであり、ゲームサーバー側のアプリケーション内では、C++, C#, Go, Java, Node.js, PHP, Python, Ruby といった各種言語用のクライアント ライブラリを用いて、Cloud Spanner をデータベースとして利用します。クライアント ライブラリ内では以下の方法で、Cloud Spanner のデータを読み書きすることができます。
- アプリケーションのコード内で API を用いて読み書きする
- アプリケーションのコード内で SQL を用いて読み書きする

またトランザクションも実行することが可能で、リードライト トランザクションはシリアライザブルの分離レベルで実行でき、強い整合性を持っています。またリードオンリー トランザクションを実行することも可能で、トランザクション間の競合を減らし、ロックやそれに伴うトランザクションの abort を減らすことができます。

Cloud Console の GUI または gcloud コマンドを利用する方法もあります。こちらはデータベース管理者が、直接 SQL を実行したり、特定のデータを直接書き換える場合などに便利です。
 
これは Cloud Spanner が直接提供するツールではありませんが、 `spanner-cli` と呼ばれる、対話的に SQL を発行できるツールがあります。これは Cloud Spanner Ecosystem と呼ばれる、Cloud Spanner のユーザーコミュニティによって開発メンテナスが行われているツールです。MySQL の mysql コマンドや、PostgreSQL の psql コマンドの様に使うことのできる、非常に便利なツールです。

本ハンズオンでは、主に上記の方法で読み書きを試します。


## テーブルの作成

## データの書き込み


