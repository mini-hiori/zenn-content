---
title: "[python,AWS]無限にSpotifyをdigるサーバーレスアプリを作る"
emoji: "⛏️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [AWS,Lambda,python,Docker,DynamoDB,Discord]
published: false
---

## はじめに
- 生まれながらにSpotifyを無限にdigりたい欲求があったので、自動でdigってくれるサーバーレスAPIを作りました
    - ちなみにdigるとは「曲を探す」程度の意味です
    - 良い曲があったら教えて欲しい欲求もあったので、dig結果はDiscordに通知する体制としました
- 以下を試しました
    - Spotify API
    - DynamoDB
    - API Gateway(Lambdaプロキシ統合)

## 成果物
- 登録しておいた好きなアーティストを元に、Spotifyから自動で関連アーティストを検索します
- 検索結果のアーティスト名と最新アルバム名、およびそのURLがDiscordに投稿されます

![正直想像していたよりうまくいってうれしい](https://raw.githubusercontent.com/mini-hiori/zenn-content/main/images/lambda-rss-reader-bot/discord-webhook-example.png)

## 構成図

![](https://raw.githubusercontent.com/mini-hiori/spotify-digger/master/docs/architecture.png)

- 動作手順は以下のようになります
    1. あらかじめDynamoDBに好きなアーティスト情報を保存しておく
    2. LambdaがDynamoDBから好きなアーティスト情報を取得する
    3. LambdaがSpotifyAPIを利用してDynamoDB内アーティストの関連アーティスト+最新アルバム情報を検索する
        - 検索結果が3人を超える場合は、ランダムに選んで3人のみ残します
        - 検索結果はDynamoDBに登録され、次回の検索の種になります
    4. Lambdaが3の検索結果をwebhookでDiscordに通知する
    5. 3の検索結果が気に入らなければ、Lambdaに統合されたAPIGatewayを叩いてリロードする
- LambdaはEventbridgeにより定期実行されます
- webhookのURLなどLambdaから利用するパラメータはSystems Managerに保存します
    - 構成図にないので書き足しておく

## 実装
- リポジトリ全体はこちら
    - https://github.com/mini-hiori/spotify-digger

### Lambda①(SpotifyAPIの利用)
- Spotifyの検索のためにSpotifyAPI利用権の取得が必要です
    - Spotifyのアカウントを持っていれば非商用なら無償で利用できます([参考](https://qiita.com/shirok/items/ba5c45511498b75aac27))
- PythonからSpotifyAPIを利用する際は[spotipy](https://spotipy.readthedocs.io/en/2.17.1/)を利用します
    - 関連アーティストの検索はartist_related_artists()を利用します。例えば私の関連アーティストを得る場合は以下のコードで可能です

```
import spotipy
from spotipy.oauth2 import SpotifyClientCredentials

# SpotifyAPI有効化時に提示されるclient_idとclient_secretを渡す
client_id = 'xxxx'
client_secret = 'xxxx'

# 認証
client_credentials_manager = spotipy.oauth2.SpotifyClientCredentials(client_id, client_secret)
spotify = spotipy.Spotify(client_credentials_manager=client_credentials_manager)

# 関連アーティストの検索
mini_hiori_uri = "spotify:artist:1nSdmeM9MUMLbhoKdf0JS0"
spotify.artist_related_artists(mini_hiori_uri)
```
- 検索のキーにはSpotify URI(Spotify側で付与されているアーティストのユニークキー)が使えます。以下画像のようにSpotifyのシェアボタンから入手できます

![](https://raw.githubusercontent.com/mini-hiori/spotify-digger/master/docs/architecture.png)

- artist_related_artists()で関連アーティストのURIが得られるので、さらにartist_albums()を実行して最新アルバム情報を手に入れます
    - この検索結果をDiscordにwebhookで投稿すれば、検索部分に関しては完成です
    - レコメンド対象にするアーティストはLambda1回の実行あたり3人までとしています

### Lambda②(DynamoDBとの連携)
- SpotifyAPIでの関連アーティスト検索の種となるアーティスト情報を保存する先が必要です。今回はDynamoDBを利用します
    - [作成方法等参考](https://qiita.com/blackcat5016/items/e41f7fb8b6b7a0c9b90b)
    - 請求モードはオンデマンドでよいと思います
- テーブル定義は以下になります
    - セカンダリインデックスはなしでOKです

|  spotify_uri(PK)  |  name  |  craeted_date  |
| ---- | ---- | ---- | ---- |
|  アーティストのSpotifyURI |  アーティスト名  |  レコード作成日  |

- 最初に検索の種にするアーティスト情報についてはAWSコンソール上から手動で投入してしまってかまいません

![](https://raw.githubusercontent.com/mini-hiori/spotify-digger/master/docs/architecture.png)

- Lambdaでは以下操作を行います
    - Lambda起動時にDynamoDBの全データを取得(scan)
    - SpotifyAPIの検索結果をDynamoDBにput
    - DynamoDB内のレコード数が一定数(300件)を超過していたら、putしたレコード数と同数のレコードをランダムに削除
        - 実行時間やコストが無限に増えることを防止します
- PythonからDynamoDBを利用する部分のコードに関しては[別記事に記載します]()

- ここまでで、自動digアプリとしての基本的な機能は完成です

### APIGateway(Lambdaプロキシ統合)
- 

### そのほか
- 以下サービス・機能を利用していますが、これらは[前回記事](https://zenn.dev/mini_hiori/articles/lambda-rss-reader-bot#%E5%AE%9F%E8%A3%85)とほぼ設定が同じのため割愛します
    - EventbridgeによるLambdaの定期実行
        - 今回は半日に1回の実行なので間隔はrate(12 hours)
    - Systems ManagerによるwebhookURLなどの保存
    - Lambdaのコンテナイメージによる起動
    - VSCode Remote Continerによる開発
    - Github ActionsによるCI/CD
        - 今回はActionsのトリガーをmasterへのpush時に変更しています。[yamlはこちら](https://github.com/mini-hiori/spotify-digger/blob/master/.github/workflows/main.yml)

## 改善点
- IaC化が未了です。Lambdaのコンテナイメージ実行に汎用性があることがわかってきたので、この部分だけでもワンボタン化したいところです
- DynamoDB内のアーティストのメンテナンス手段に関して、AWSコンソール経由で手動修正/削除する以外にありません。レコメンドが好みでない方向に向かった場合の軌道修正が若干手間です
    - フロントエンドアプリを作れば解決しそうです
- DynamoDBが常に全scanなのはコスト・パフォーマンス両面で不適切です
    - パーティションキーを連番のidとし、Lambda側で抽選したidのアーティストのみを抽出するようにすると改善されそうです