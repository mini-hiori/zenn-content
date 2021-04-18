---
title: "[python,AWS]FastAPIで作るサーバーレスAPI(Mangum)"
emoji: "🐣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [AWS,Lambda,python,Docker,Serverless Framework]
published: false
---

## はじめに
- [Mangum](https://github.com/jordaneremieff/mangum)を使うとFastAPI製APIを手軽にLambda+APIGateway化できるよ、という紹介です
```
Mangum is an adapter for using ASGI applications with AWS Lambda & API Gateway.
(公式readme.mdより)
```
- Mangumは[ASGI](https://asgi.readthedocs.io/en/latest/)ベースのAPIとLambda+APIGatewayを中継してくれるアダプタになります
    - FastAPIに限らず、[Responder](https://responder.kennethreitz.org/en/latest/#)等でも使えるようです


## 構成図
- 以下のような簡単なサーバーレスAPIを作成します
    - Lambdaはコンテナイメージから起動します
![構成図](https://raw.githubusercontent.com/mini-hiori/mangum-test/main/docs/architecture.png)
- ECRへのデプロイはGithub Actions、APIGateway+Lambdaのデプロイはserverless frameworkでそれぞれ自動化しています
    - serverless frameworkに与えるパラメータはSystems Managerパラメータストアに配置します

## ソースコード
- リポジトリ全体はこちら
    - https://github.com/mini-hiori/mangum-test
- 核になるのは、Lambda用Pythonファイル(src/app.py)、Dockerfile、serverless framework用設定ファイル(serverless.yml)の3つです
- デプロイまでの手順は以下の通りです
    1. app.pyを含むコンテナイメージをECRにアップロードする
    2. Systems Managerパラメータストアにイメージダイジェスト、AWSアカウントIDを登録
    3. serverless.ymlでLambda+APIGatewayをデプロイ

### Python
- パラメータを受け取って返答するだけの単純なAPIを作成します
    - 通常のFastAPIの記述に加えるのは、最終行の`handler = Mangum(app)`のみです。  
    この文により、MangumがFastAPIのインスタンスをAPIGatewayと統合できるLambdaのhandlerに変換してくれます
```python
from fastapi import FastAPI, HTTPException
from mangum import Mangum
from pydantic import BaseModel

app = FastAPI()


class HelloParam(BaseModel):
    name: str


@app.get("/hello")
def get_hello(name: str = None):
    """
    getで返事する
    """
    if name:
        message = f"[GET]hello, {name}!"
    else:
        message = f"[GET]hello, visitor!"

    return {"message": message}


@app.post("/hello_post")
def post_hello(param: HelloParam):
    """
    postで返事する
    """
    if param.name:
        message = f"[POST]hello, {param.name}!"
    else:
        message = f"[POST]hello, visitor!"

    return {"message": message}


handler = Mangum(app)
```


### Dockerfile
- [AWS公式](https://aws.amazon.com/jp/blogs/aws/new-for-aws-lambda-container-image-support/)で紹介されているものをほぼそのまま拝借します
- 以下が満たされていれば、そのまま動くと思います
    - 前項のpythonファイルが`src/app.py`として保存されている
    - ルートディレクトリにrequirements.txtがあり、`fastapi`,`mangum`,`pydantic`が記載されている
```dockerfile
ARG FUNCTION_DIR="/home/app/"
ARG RUNTIME_VERSION="3.8"
ARG DISTRO_VERSION="3.12"

FROM python:${RUNTIME_VERSION}-alpine${DISTRO_VERSION} AS python-alpine
RUN apk add --no-cache \
    libstdc++

FROM python-alpine AS build-image
RUN apk add --no-cache \
    build-base \
    libtool \
    autoconf \
    automake \
    libexecinfo-dev \
    make \
    cmake \
    libcurl
ARG FUNCTION_DIR
ARG RUNTIME_VERSION
RUN mkdir -p ${FUNCTION_DIR}
COPY . ${FUNCTION_DIR}
RUN python${RUNTIME_VERSION} -m pip install awslambdaric --target ${FUNCTION_DIR}

FROM python-alpine
ARG FUNCTION_DIR
WORKDIR ${FUNCTION_DIR}
COPY --from=build-image ${FUNCTION_DIR} ${FUNCTION_DIR}
ADD https://github.com/aws/aws-lambda-runtime-interface-emulator/releases/latest/download/aws-lambda-rie /usr/bin/aws-lambda-rie
COPY entry.sh /
COPY requirements.txt /
RUN chmod 755 /usr/bin/aws-lambda-rie /entry.sh
RUN apk add git
RUN pip install -r requirements.txt
ENTRYPOINT [ "/entry.sh" ]
CMD [ "src/app.handler" ]
```

- このDockerfileをビルドし、ECRに上げておきます

### serverless framework(serverless.yml)
- Lambda+APIGatewayのデプロイは[serverless framework](https://www.serverless.com/)を利用して行います
    - serverless framework自体のインストール・設定は[こちらを参照](https://dev.classmethod.jp/articles/easy-deploy-of-lambda-with-serverless-framework/)
- SystemsManagerのパラメータとして以下を定義すれば動くと思います
    - AccountID: AWSアカウントのID(数字12桁)
    - MangumRepository: 前項のDockerイメージをアップロードしたECRの名称+ダイジェスト
        - リポジトリ名@sha256:(ハッシュ値) の体裁です
```yaml
service: mangum-test # 任意のプロジェクト名に変更しておいてください

provider:
  name: aws
  stage: prod
  region: ap-northeast-1
  deploymentBucket: mini-serverless-framework # デプロイ時に作成されるS3
  logRetentionInDays: 30 # Cloudwatchのログ保存期間

functions:
  index:
    image: "${ssm:AccountID}.dkr.ecr.ap-northeast-1.amazonaws.com/${ssm:MangumRepository}" # Lambdaのソースとなるコンテナイメージ
    events:
      - http:
          integration: lambda-proxy
          path: /{proxy+}
          method: ANY
          cors: true
```

- 末尾のeventsの部分でAPIGatewayが作成されます
    - pathを/{proxy+}、methodをANYにすると、APIGatewayは任意のパスへのアクセスをLambdaに渡します。FastAPI側でエンドポイントやパラメータを追加修正した場合でもAPIGatewayの修正が不要になります
- デプロイ処理(sls deploy)が問題なく成功すれば、出力されたURLからAPI実行できるはずです
```python
>>> import requests
>>> import json
>>> url = "https://xxxx.execute-api.ap-northeast-1.amazonaws.com/prod/hello"
>>> response = requests.get(url)
>>> json.loads(response.text)["message"]
'[GET]hello, visitor!'
>>> response = requests.post(url,json={"name":"mini_hiori"})
>>> json.loads(response.text)["message"]
'[POST]hello, mini_hiori!'
```

## メリット
- FastAPI/Flaskユーザーであれば、Lambda特有の文法を覚えなくてもLambdaが利用できます
- APIの実行環境として、ECS,EKSとLambdaを簡単に行き来できるようになります
    - まずLambdaでスモールスタートして、あまり利用が高頻度だったり処理時間が長すぎる、コールドスタートが気になるなど事情があれば他に移動するのがよさそうです
    - どのサービスもコンテナイメージが使えるので、git→ECRのCIは使い回すことができます

## 参考
- [Serverless Framework+mangum+FastAPIで、より快適なPython API開発環境を作る](https://tech.jxpress.net/entry/2020/03/29/170000)

