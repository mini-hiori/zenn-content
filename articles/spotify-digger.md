---
title: "[python,AWS]無限にSpotifyをdigるサーバーレスアプリを作る"
emoji: "⛏️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [AWS,Lambda,python,Docker,DynamoDB,Discord]
published: false
---

## はじめに
- 生まれながらにSpotifyを無限にdigりたい欲求があったので、自動でdigってくれるサーバーレスAPIを作りました
    - digるとは「曲を探す」程度の意味です
    - 良い曲があったら教えて欲しいので、結果はDiscordに通知する体制としています
- 以下を試しました
    - Spotify API
    - DynamoDB
    - API Gateway(Lambda統合)
- [前回](https://zenn.dev/mini_hiori/articles/lambda-rss-reader-bot)と重複する部分に関しては説明を省いている部分があります

## 成果物
- 登録しておいたアーティストを元に、Spotifyから自動で関連アーティストを検索します
- 検索結果のアーティスト名と最新アルバム名、およびそのURLがDiscordに投稿されます

![正直想像していたよりうまくいってうれしい](https://raw.githubusercontent.com/mini-hiori/zenn-content/main/images/spotify-digger/dig_result.png)

## 構成図

![](https://raw.githubusercontent.com/mini-hiori/spotify-digger/master/docs/architecture.png)

- 動作手順は以下のようになります
    1. あらかじめDynamoDBに好きなアーティスト情報を保存しておく
    2. LambdaがDynamoDBからアーティスト情報を取得する
    3. Lambdaが関連アーティスト+最新アルバム情報を検索する
        - Spotify APIを利用して検索
        - 検索結果が3人を超える場合は、ランダムに選んで3人まで採用
        - 検索結果はDynamoDBに登録され、次回の検索の種になります
    4. Lambdaが3.の検索結果をwebhookでDiscordに通知する
    5. 検索結果が気に入らなければ、Lambdaに統合されたAPIGatewayを叩いて2.以降を再実行する
- LambdaはEventbridgeにより定期実行されます
- webhookのURLなどLambdaから利用するパラメータはSystems Managerに保存します

## 実装
- リポジトリ全体はこちら
    - https://github.com/mini-hiori/spotify-digger

### Spotify APIの利用(アーティスト検索)
- Spotifyの検索のためにSpotifyAPI利用権の取得が必要です
    - Spotifyのアカウントを持っていれば非商用なら無償で利用できます([参考](https://qiita.com/shirok/items/ba5c45511498b75aac27))
- PythonからSpotifyAPIを利用する際は[spotipy](https://spotipy.readthedocs.io/en/2.17.1/)を利用します
    - 関連アーティストの検索はspotipyのartist_related_artists関数を利用します。例えば[私](https://open.spotify.com/artist/1nSdmeM9MUMLbhoKdf0JS0?si=s_cG-uWYQ-uQj-jm7lvfOw)の関連アーティストを得る場合は以下のコードで可能です

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
- 検索のキーに利用しているSpotify URI(Spotify側で付与されているアーティストのユニークキー)は、以下画像のようにSpotifyのシェアボタンから入手できます

![](https://raw.githubusercontent.com/mini-hiori/zenn-content/main/images/spotify-digger/get_spotify_uri.png)

### DynamoDB(アーティスト情報の保存)
- 関連アーティスト検索の元データ(好きなアーティスト)の保存先としてDynamoDBを利用します
    - [作成方法等参考](https://qiita.com/blackcat5016/items/e41f7fb8b6b7a0c9b90b)
    - 請求モードはオンデマンドでよいと思います
- テーブル内のattributeは以下のようになります
    - セカンダリインデックスはなしでOKです

|  spotify_uri(PK)  |  name  |  craeted_date  |
| ---- | ---- | ---- | ---- |
|  アーティストのSpotifyURI |  アーティスト名  |  レコード作成日  |

- 最初に検索の種にするアーティスト情報についてはAWSコンソール上から手動で投入します

![](https://raw.githubusercontent.com/mini-hiori/zenn-content/main/images/spotify-digger/dynamodb_create.png)

### Lambda(検索、DynamoDBの操作)
- SpotifyAPIを利用した検索とDynamoDBの操作のためにLambdaを利用します
- 1回のLambda実行で以下の操作を行います
    1. Lambda起動時にDynamoDBの全データを取得
    2. DynamoDB内のアーティストデータを利用してSpotify検索。新しいアーティストの情報を3人まで得る
    3. Spotify検索結果をDynamoDBにput
    4. DynamoDB内のレコード数が一定数を超過していたら、putしたレコード数と同数のレコードをランダムに削除
        - 実行時間やコストが無限に増えることを防止します
    5. Spotify検索結果をwebhook経由でDiscordに投稿
- PythonからDynamoDBを利用する部分のコードに関しては[別記事に記載します](https://zenn.dev/mini_hiori/articles/python-code-for-dynamodb)

- ここまでで、自動digアプリとしての基本的な機能は完成です

### APIGateway+Lambda(アーティスト検索のリロード)
- 定期実行によるdig結果が気に入らなかった場合のために、LambdaをAPIGatewayと統合して簡単に検索を再実行できるようにします
    - 具体的な実行手順は[こちらなどを参考にしてください](https://dev.classmethod.jp/articles/api-gateway-lambda-integration-fabu/)
- APIGatewayとLambdaの統合にはプロキシ統合と非プロキシ統合の2種類がありますが、今回はどちらでもかまいません
    - プロキシ統合の有無による主な違いはLambdaの出力結果のマッピングをAPIGatewayに移譲するかどうかです。Lambdaの出力結果をAPIを介して他ツールで利用したい場合はプロキシ統合が必要になるはず
    - プロキシ統合を利用する場合は、Lambdaの返却値をAPIGatewayが解釈できる形に合わせることに注意してください([参考](https://qiita.com/polarbear08/items/3f5b8584154931f99f43))

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
- IaC化が未了です。[前回](https://zenn.dev/mini_hiori/articles/lambda-rss-reader-bot)と合わせてLambdaのコンテナイメージ実行に汎用性があることがわかってきたので、この部分だけでもワンボタン化したいところです
- DynamoDB内のアーティストのメンテナンス手段がAWSコンソール経由での手動修正/削除以外にありません。レコメンドが好みでない方向に向かった場合の軌道修正が若干手間です
    - フロントエンドアプリから操作できるようにしてみたいです(願望)
- DynamoDBが常に全scanなのはコスト・パフォーマンス両面で不適切です
    - パーティションキーを連番のidとし、Lambda側で抽選したidのアーティストのみを抽出するようにすると改善されそうです