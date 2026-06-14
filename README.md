# cleartradewiki
Cleartrade Communityで検討しているwikiのリポジトリ  

# はじめに
## 目的
目的はCommunity内での知識の共有。

## 開発の経緯
[2026.5.23 質疑応答会](https://www.chatwork.com/#!rid175176676-2110216131632693248)で話が示された。
* 単発チャットでは情報が流れてしまう。
* 知識の持続的蓄積が難しい課題に対して、メンバー共同編集が可能な「Wikipedia的」ナレッジベースを構築する。

## 利用ルール面 (仮)
[ツール設定・操作](https://www.chatwork.com/#!rid379516146-2110276849862180864)の草案
* wikiに載せる情報に関しては、企業概要などの一般公開情報が前提になると思っています。
* 外部LLMへ投入する前提の情報かどうかは別途考える必要有り。
* 一義的には投稿ルールで縛る.
* もし掲出してはいけないと思われる情報が出ていた場合は、モデレーターが適宜チェックして削除する。  
という運用がベースになるかなと思っています。
* 完全な防止は難しいと思いますが、実際には内容を確認・巡回して必要に応じて削除する役割は必要になりそうです。  
この点はコミュニティ内でご協力をお願いする形になると思います。

# Wiki OSSの選定
* WikiのOSSを探したところ、 `Wiki.js` が良さそうなのでこれを選んで試す。  
→ 記事の投稿をしやすいく、権限管理しやすいOSSに見える。
* ライセンス自体はAGPLの様だが、ソースコードの開示に関してはこのリポジトリでできる。

# システム面
## システム構成
最初の候補である `wiki.js` を構築する。  
メンテナンス性を考慮してansible-playbookで構築出来る仕組みにする。  
[ツール設定・操作](https://www.chatwork.com/#!rid379516146-2110272628421033984)
### 想定環境
```
[Wikiの投稿、閲覧ユーザ]
ブラウザ
    │ HTTPS
    ▼
[VPS]
Nginx
 └ Wiki.js
      └ PostgreSQL

必要になったら
[ChatGPT / Claude / AIツール]
    │ HTTPS(JSON)
    ▼
https://example.com/api/...
    │
    ▼
[VPS]
Python API(仮)
 (Wiki連携用ラッパAPI)
    │ Wiki.js API(仮)
    ▼
Wiki.js
    │
    ▼
PostgreSQL
```
### 検証環境
dev2 (Docker)  
自宅LANの中にdockerを置いたので、外からアクセスできる様にする.  
WAN側がCGNATのIPアドレスのためトンネルで繋ぐ.  
```
[インターネット] --> [Cloudflare] - tunnel -> [ASUS ExpertCenter PN42 (Ubuntu26.04 + docker-compose)]
※ docker-compose
 - cloudflared
 - docker-compose
 - wikijs
 - postgresql
 - caddy (Reverse Proxy)
```

## 構築手順
1. VMを用意する。
2. ansible-playbookを用意する。
3. 以下の様に実行する。
```
(1) Dev: VM 開発環境
% /opt/homebrew/bin/ansible-playbook -i dev.ini initialize.yml --user=rocky -bK --ask-vault-pass --list-hosts
→ 対象ホストが合っているか
% /opt/homebrew/bin/ansible-playbook -i dev.ini initialize.yml --user=rocky -bK --ask-vault-pass
→ 成功するか

(2) Dev2 Dockerのみ
% /opt/homebrew/bin/ansible-playbook -i dev2.ini initialize_only_dev2.yml -bK --ask-vault-pass --list-hosts
% /opt/homebrew/bin/ansible-playbook -i dev2.ini initialize_only_dev2.yml -bK --ask-vault-pass
```

構築後の状況
```
----
$ docker ps
CONTAINER ID   IMAGE                           COMMAND                  CREATED          STATUS          PORTS                                                                NAMES
2245b648e916   cloudflare/cloudflared:latest   "cloudflared --no-au…"   17 minutes ago   Up 17 minutes                                                                        wikijs-cloudflare-tunnel-1
75e86d238502   ghcr.io/requarks/wiki:2         "docker-entrypoint.s…"   17 minutes ago   Up 17 minutes   3000/tcp, 3443/tcp                                                   wikijs-wikijs-1
7a52054bece7   postgres:15-alpine              "docker-entrypoint.s…"   17 minutes ago   Up 17 minutes   5432/tcp                                                             wikijs-db-1
40305f27471b   caddy:2-alpine                  "caddy run --config …"   17 minutes ago   Up 17 minutes   80/tcp, 2019/tcp, 0.0.0.0:443->443/tcp, [::]:443->443/tcp, 443/udp   wikijs-reverse-proxy-1
----
```

4. アクセス確認
ブラウザアクセスして成功すること。

|Key|Value|
| --- | --- |
|Dev| [Devサイト](https://wikidev.ctcommunity.f5.si/)|
|Dev2| [コミュニティお試し環境](https://wikidev.lushspknet.com/)|
|Prod| TBD|

5. 管理者設定
初回の管理者設定をする。（wiki.jsの最初の管理者登録)

6. 環境設定
以下の環境設定をする。

(1) 全般設定
|Key|Value|
| --- | --- |
|Locale| Language → (画面右) Download Locale → 日本語 をダウンロードする|
|Locale| Language → Locale Settings → Japanese へ変更する|
|タイトル| 全体設定 → サイト情報 → サイトのタイトル → サイト名を入れる|
|ロゴ| 全般設定 → サイト情報 → ロゴ → ロゴのURLを削除する|
|Copyright| 全般設定 → サイト情報 → 「会社/組織名」へ組織名を入れる|
|コメント|全般設定 → Features → Comments を OFFにする|
|編集ボタン| 全般設定 → 編集ショートカット → 「編集メニューバーの表示」をON, 「外部編集ボタンを表示」をOFF|

(2) ナビゲーション  
画面左のページツリーを見やすくする。  

|Key|Value|
| --- | --- |
|ナビベーションモード|サイトツリー|

(3) テーマ
|Key|Value|
| --- | --- |
|サイドメニュー（サブ）| テーマ → テーマオプション → 「Right」へ変更する|

(4) グループ
ロールを管理者、投稿者、ゲストの3種類にする。
|Key|Value|
| --- | --- |
|Developグループ|ユーザ → グループ → (右上) NewGroup → 「Development」で作成する|

(5) ユーザ
作成したグループへ書き込み権限を付与する。  
* Developユーザ → グループ → Development → Edit Group

|Key|Value|
| --- | --- |
|PERMISSIONS|+write:page, +manage:pages, +delete:pages, +write:assets, manage:assets|
|PAGE RULES|+Create+ Edit Pages, +Rename / Move Pages, +Delete Pages, +View Pages Source, +View Page History, +Upload Assets, +Edit+Delete Assets|

* ★Guestユーザの削除★
検討中  
Guestユーザ＝ログインしていないアカウント(Annonymous)扱いの模様.  
権限を落とす場合は、Guestグループの権限を落とす。

(6) 登録ユーザの初期グループ  
★ ここは運用時検討 Default Guestにして閲覧不可にしても良いとも思っている ★  
手間を減らすため一時的に自己登録有りにする。  
* 認証 → Local → 登録

|Key|Value|
| --- | --- |
|自己登録を許可する| On |
|割り当てるグループ| Developer |

(7) タイムゾーン変更  
wiki.jsのデフォルト設定が「America/New_York」である。日本時間へ変更する。
```
$ docker exec -t wikijs-db-1 psql -U wikijs -d wikijs -c "ALTER TABLE users ALTER COLUMN timezone SET DEFAULT 'Asia/Tokyo';"
$ docker exec -t wikijs-db-1 psql -U wikijs -d wikijs -c "UPDATE users SET \"timezone\" = 'Asia/Tokyo';"
$ docker exec -t wikijs-db-1 psql -U wikijs -d wikijs -c "ALTER TABLE users ALTER COLUMN \"localeCode\" SET DEFAULT 'ja';"
$ docker exec -t wikijs-db-1 psql -U wikijs -d wikijs -c "UPDATE users SET \"localeCode\" = 'ja';"
```

(8) ストレージ設定  
wikiへアップロードされたファイルを永続化するためLocalDisk保存にする.  

* モジュール → ストレージ

|Key|Value|
| --- | --- |
|Local File System|チェック|
|Local File System| `/wiki/data/uploads` |
|Create Daily Backups| チェック (とりあえず1ヶ月分取ってくれる) |


(9) メール登録  
★ 送信メールアドレスを検討する必要がある. ★  
自己登録を入れるため、メール送信サーバの設定を入れる。  

* システム → メール

|Key|Value|
| --- | --- |
|送信者名| よしなに |
|送信者アドレス| よしなに |
|ホスト| SMTPサーバ名|
|ポート| 587 |
|安全な通信(TLS)| Off (メールプロバイダ次第)|
|SSL証明書の検証(StartTLS)| On (メールプロバイダ次第)|
|ユーザ名|よしなに|
|パスワード|よしなに|

* テスト用のメールを送信する
テストメールを送付する先を入力して送信。届けば設定OK。


## 初期ページの作成
1. Wiki.jsのトップを開く
2. 「ホームページの作成」を開く
3. 「トップページ」を作成する
→ サイトの概要と、注意項目を載せる。
4. 「日本株」, 「米国株」の階層を作る。
5. 「銘柄コード」のページを作る。
→ この辺は作り方相談という感じ。

以上
