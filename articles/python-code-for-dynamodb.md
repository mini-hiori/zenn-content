---
title: "コピペでできる:PythonからDynamoDB操作"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [AWS,python,DynamoDB]
published: true
---

## はじめに

- [Spotifyを検索するサーバーレスアプリ](https://zenn.dev/mini_hiori/articles/spotify-digger)のためにPythonからDynamoDBを操作するプログラムを組みました
- 「PythonからDynamoDBを操作する」部分については汎用性があると思うので、コードを記事としても公開しておきます
    - 似たような内容の記事は探せばヒットしますが、丸々コピペで使えるものは意外と少ない気がします
- 現在のところは上記記事で利用したDynamoDB操作のみ記載していますが、将来的にはUpdateやクエリ取得等も記載したいと思います(願望)
- エラーハンドリング等は考慮していません。適宜try-catchをお願いします
- コピペで使えない、記載が誤っているなどあれば指摘お願いします

## 動作環境
- OS: Amazon Linux 2
    - Cloud9から動作させています
- Python: 3.8.5
- boto3: 1.17.46
    - Python用AWS SDK。入っていなければインストールお願いします([参考](https://aws.amazon.com/jp/sdk-for-python/))

## Scan
- DynamoDB内のデータをすべて取得する操作です
```
import boto3
from typing import Dict

def scan_dynamodb() -> Dict:
    """
    DynamoDB内のデータを全て取得
    """
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('テーブル名')
    scan_result: Dict = table.scan()
    return scan_result["Items"]
```
- scan_resultには、Items属性以外にも取得できたレコード数やDynamoDBへの接続結果等が入っています
    - DynamoDBのitemに関する情報が入っているのはItemsのみなので、通常はこれだけ取り出してしまってかまわないと思います

## Put
- DynamoDBに1つitemを追加する処理です
```
import boto3
from typing import Dict

def put_dynamodb(put_item: Dict) -> None:
    """
    DynamoDBに1つitem追加
    """
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('テーブル名')
    table.put_item(Item=put_item)
    return None
```
- テーブル作成時に設定したプライマリキーを持たないdictをputしようとするとエラーになります

## Delete
- DynamoDBから1つitemを削除する処理です

```
import boto3

def delete_dynamodb(key: str, value: str) -> None:
    """
    DynamoDBから情報を削除
    """
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('テーブル名')
    table.delete_item(Key={key: value})
    return None
```
- プライマリキーでない列(attribute)をkey,valueに指定してもエラーになり削除できません

## おまけ
- 型ヒントで利用しているtypingライブラリはpython3.9以降では不要になります([参考](https://future-architect.github.io/articles/20201223/))