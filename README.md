# Wordpress-with-Docker
WordPressをDocker Compose + Cloudflare Tunnel で簡単に構築

## 各種サービスデフォルトポート

各サービスはデフォルトでは以下のポートで起動します  
変更する場合は、`docker-compose.yml`を編集してください

| Service Name      | 外部ポート | 内部ポート |
| ---------------- | -------- | -------- |
| **WordPress**    | `80`     | `80`     |
| **PHP My Admin** | `8080`   | `80`     |

## 環境変数の設定
環境変数は `.env` ファイルに記述し、Docker Compose で読み込めるようにします  
以下のように `.env` ファイルを作成してください

```env
# MySQL Database Settings
MYSQL_ROOT_PASSWORD=wordpress@admin  # MySQLのrootパスワード
MYSQL_DATABASE=wordpress             # 作成するデータベース名
MYSQL_USER=wordpress                 # データベースのユーザー名
MYSQL_PASSWORD=wordpress             # データベースのパスワード

# WordPress Settings
WORDPRESS_DB_HOST=mysql:3306         # WordPressが接続するDBホスト (MySQLのコンテナ名)
WORDPRESS_DB_USER=wordpress          # WordPressのDBユーザー名
WORDPRESS_DB_PASSWORD=wordpress      # WordPressのDBパスワード

# phpMyAdmin Settings
PMA_HOST=mysql:3306                  # phpMyAdminが接続するDBホスト
PMA_USER=wordpress                    # phpMyAdminで使用するDBユーザー名
PMA_PASSWORD=wordpress                # phpMyAdminで使用するDBパスワード

# Cloudflare Tunnel Settings
TUNNEL_TOKEN=your-tunnel-token        # Cloudflare Tunnel のトークン
TUNNEL_TRANSPORT_PROTOCOL=your-protocol # Cloudflareの通信プロトコル（例: "http2"）
```

## 構築手順

### 1. Docker のインストール
Docker がインストールされていない場合は、以下の公式ページからインストール手順を確認してください

- [Docker インストール手順](https://docs.docker.com/engine/install/)

WindowsやMac場合は、Docker Desktopをインストールしましょう
- [DockerDestop インストール手順](https://docs.docker.com/desktop/)

### 2. リポジトリをクローン
```bash
git clone https://github.com/kuwacom/WordPress-with-Docker.git
cd WordPress-with-Docker
```

### 3. `.env` ファイルの作成
プロジェクトルートに `.env` ファイルを作成し、上記の環境変数を設定してください

```bash
cp .env.example .env
vim .env  # 必要に応じて編集
```

### 4. Docker Compose でコンテナを起動
```bash
docker compose up -d
```

### 5. 動作確認
ブラウザで http://localhost にアクセスし、WordPress のセットアップ画面が表示されれば成功です  
phpMyAdmin は http://localhost:8080 で確認できます

## Cloudflare Tunnel で公開する
Cloudflare Tunnel を利用することで、外部ネットワークから WordPress や phpMyAdmin にアクセスできるようにします

**この手順は Cloudflare ダッシュボード側で設定を行うことを前提としています。**

### 1. Cloudflare Tunnel の作成
1. [Cloudflare Zero Trust ダッシュボード](https://one.dash.cloudflare.com/) にアクセス
2. **Network → Tunnels** から **Create a Tunnel** を選択
3. 任意の名前を入力し、トンネルを作成

### 2. 公開するサービスの設定
トンネル作成後、以下のようにエンドポイントを追加してください。

| サービス名      | エンドポイント（例）           | 宛先アドレス             |
| -------------- | -------------------------- | ---------------------- |
| **WordPress**  | `https://wordpress.example.com` | `http://wordpress:80`   |
| **phpMyAdmin** | `https://pma.example.com`      | `http://phpmyadmin:80` |

設定完了後、Cloudflare Tunnel が正しく動作するか確認してください。

## `xmlrpc.php` について
デフォルトの WordPress では、`xmlrpc.php` が有効になっています  
セキュリティ上の理由から、利用しない場合は無効化することを推奨します

### `.htaccess` で `xmlrpc.php` を無効化
以下のルールを `.htaccess` に追加することで、外部からのアクセスをブロックできます

```apache
<Files xmlrpc.php>
    Order Allow,Deny
    Deny from all
</Files>
```

### ファイルを直接削除する
もちろん、`xmlrpc.php` 自体を削除しても構いません

```bash
rm wordpress\xmlrpc.php
```

## コンテナの停止と削除
コンテナを停止する場合は、以下のコマンドを実行します

```bash
docker-compose down
```

すべてのデータを削除する場合は、以下のコマンドを実行してください

```bash
docker-compose down -v
```

## localhost でアクセスする場合
以下のように、localhost経由でhttpで接続する時、ポートを分けるようにしないと、無限リダイレクトされてアクセスできなくなってしまいます

そのため、以下のコードを `wp-config.php` へ記載する必要があります
```php
/**
 * Docker でポートフォワードされたホスト名を使用している場合、WP_HOME と WP_SITEURL を設定
 */
if (isset($_SERVER['HTTP_HOST'])) {
    $host = $_SERVER['HTTP_HOST'];

    if ($host === getenv_docker('WP_LOCALHOST_HOSTNAME', 'localhost')) {
        define('WP_HOME', 'http://' . getenv_docker('WP_LOCALHOST_HOSTNAME', 'localhost'));
        define('WP_SITEURL', 'http://' . getenv_docker('WP_LOCALHOST_HOSTNAME', 'localhost'));
    }
}
```
上記のコードを記載後、`docker-compose.yml`の、wordpressサービス内の以下の環境変数に、アクセスするlocalhostアドレスを記入してください

`WP_LOCALHOST_HOSTNAME`

例) http://localhost:80 でアクセスする場合
```yaml
    ...
    wordpress:
        ...
        environment:
            ...
            WP_LOCALHOST_HOSTNAME: localhost:80 # ローカル接続用のホスト名
```

**ただし、この方法は、動的にWordPressホストネームを書き換えているため、テーマによってはアセットが静的URLで設定されるため、外部から閲覧するときに見れなくなる場合があります**

Cocoonテーマで実際に起きるのですが、サイトアイコンを変更すると、localhostを含んだ静的URLが指定されるため、外部から見るとアイコンが取得できなくなってしまいます

基本的な設定はドメイン越しで行いましょう！

### localhost でサイトを作ってしまったときの移行方法
localhostで設定してしまった場合、`localhost でアクセスする場合`に記載してあるconfigを使わずに起動後、localhost からアクセスして、設定 > 一般 にある WordPressアドレスとサイトアドレスを変更したいグローバルなアドレスへ変えると動作する

---
以上！このドキュメントを参考にして、Docker Compose + Cloudflare Tunnel を活用した WordPress の構築をスムーズに進めてください！