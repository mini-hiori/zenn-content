---
title: "[python,AWS]RSSをサーバーレスに自動取得してDiscordに送ってみる"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [AWS,Lambda,python,Docker,VS Code]
published: false
---

## はじめに
- 生まれながらにRSSリーダーを自作したい欲求があったので、自作しました
- 以下を試しました
    - VSCode Remote Containerで開発
    - EventbridgeによるLambdaの定期実行
    - Lambdaをコンテナで動かす

## 成果物
- 登録しておいたサイトのRSSが1時間おきに自動取得され、リンクとタイトルがDiscordに投稿されます
![](https://raw.githubusercontent.com/mini-hiori/zenn-content/main/images/lambda-rss-reader-bot/discord-webhook-example.png)

## 構成図
![](https://raw.githubusercontent.com/mini-hiori/zenn-content/main/images/lambda-rss-reader-bot/architecture.png)

- LambdaがRSS情報を取得し、webhookを利用してDiscordに通知します
    - LambdaはEventbridgeにより1時間おきに自動実行されます
- RSS取得先URLはSystems Managerパラメータストアに保存します
    - URLの追加変更はデプロイを介さず可能です
- Lambdaはコンテナイメージを利用して動きます、言語はpythonです
    - デプロイはGithub Actionsを利用して自動化しています

## 実装
- リポジトリ全体はこちら
    - https://github.com/mini-hiori/lambda-rss-reader-bot
### RSSリーダー本体(Lambda)
- RSSの取得&解析はfeedparserというライブラリを利用して行います
    - [参考](https://note.nkmk.me/python-feedparser-tutorial/)
    - 最小構成は以下です、get_rssにRSSのURLを渡すと記事情報が得られます
```
import feedparser
import time
from dataclasses import dataclass
from typing import List


@dataclass
class RssContent():
    title: str
    url: str


def get_rss(endpoint: str) -> List[RssContent]:
    feed = feedparser.parse(endpoint)
    rss_list: List[RssContent] = []
    for entry in feed.entries:
        if not entry.get("link"):
            continue
        rss_content = RssContent(
            title=entry.title,
            url=entry.link
        )
        rss_list.append(rss_content)
    return rss_list
``` 
- Lambdaはコンテナイメージで動かします
    - Dockerfileは[こちら](https://github.com/mini-hiori/lambda-rss-reader-bot/blob/master/Dockerfile)
    - VSCode Remote Containerを利用すると、このDockerfileで作られるコンテナの中で開発ができます
        - [参考](https://qiita.com/d0ne1s/items/d2649801c6f804019db7)
    - コンテナを利用する場合でも、handler関数を作成してその中のコードを動かす点は同じです
        - 詳しくは[公式ドキュメント](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/python-image.html)へ
    - コンテナを利用するLambdaの作成自体はGUIから可能です。  
    作成時にコンテナイメージを保存したURIを求められるので、次項のデプロイを済ませてから作成します  
    ![](https://raw.githubusercontent.com/mini-hiori/zenn-content/main/images/lambda-rss-reader-bot/lambda-config.png)
### デプロイ(Github Actions)
- [参考](https://dev.classmethod.jp/articles/github-action-ecr-push/)
- 上記記事を参考に以下を行うことでデプロイの自動化が可能です
    - ECRの作成
    - IAMユーザーの作成
    - Github Actionsのyamlファイル作成、.github/workflows/main.ymlに配置
    - GitHubのSecretsにIAMログイン情報等を記載
        - AWS_ECR_REPO_NAMEにはリポジトリ名を記載することに注意。URIではありません(1敗)
### webhookURL・RSS取得先URLの管理(Systems Manager)
- URL類はSystems Managerパラメータストアに保存します
    - RSS取得先URLについては改行区切りで単一パラメータに保存してしまっています  
![](https://raw.githubusercontent.com/mini-hiori/zenn-content/main/images/lambda-rss-reader-bot/ssm-params.png)
- LambdaからSystems Managerを参照し、パラメータを改行で分割することでRSS取得先URLをLambdaに与えます
```
from typing import List
import boto3

ssm = boto3.client('ssm')

def get_target_url() -> List[str]:
    """
    取得対象にするrssのURLを返却する
    """
    url_param: str = ssm.get_parameter(
        Name='RSSURLList'
    )['Parameter']['Value']
    # url_paramには改行区切りでURLが保存されているので、改行で分割して出力
    url_list: List[str] = url_param.split("\n")
    return url_list
```
- DiscordのwebhookURLについても同様にSystems Managerに保存しています
    - webhookURL自体の取得については[こちら](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks)

- ここまでで、Lambdaを手動実行すればRSSがDiscordに送られる状態が実現します

### 定期実行(Eventbridge)
- [参考](https://dev.startialab.blog/etc/a105)
- EventbridgeをLambdaのトリガーに設定すると定期実行が可能です
- 今回は1時間おきに実行するので「rate(1 hour)」で設定しました
    - rateは開始時間を設定できないようです。  
    試した限りではEventbridgeの設定完了直後〜数分後までに1回目が起動して、以降はrateの周期にしたがって動きます

- RSS情報が1時間おきにDiscordに送られれば完成です

## 改善点
- LambdaやECR、Eventbridgeは今回手動作成しましたが、本当はSAWやterraform等でコード化した方が良いと思います
- pythonのDockerfileはalpineでない方がパフォーマンスが良いらしく、改善の余地があります
    - [参考](https://pythonspeed.com/articles/alpine-docker-python/)
    - AWS公式もalpineで紹介していたので今回は流用しましたが、次回はbuster系で試す予定です
- スクレイピング系の処理をLambdaで実行するのは実はコスパがよくない説があります
    - [参考](https://blog.yuuk.io/entry/2017/lambda-disadvantages-from-a-cost-viewpoint)
    - [FargateのScheduled task](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/scheduled_tasks.html)でも同様の挙動は実現できるはず