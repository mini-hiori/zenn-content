---
title: "TypeScriptで動画・音声形式変換(fluent-ffmpeg)"
emoji: "🎥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [TypeScript,Docker,Node.js]
published: false
---

## はじめに
- 生まれながらにTypeScriptで動画や音声の形式を変換したい欲求があったところ、案外簡単にできたのでその方法の紹介です
- 本当に少しですが環境構築が要るので、Docker上で動くようにしてその手間をできるだけ削減できるようにもしました

## 使うもの
- [ffmpeg](https://www.ffmpeg.org/)
    - 動画や音声ファイルの編集に使えるCLIベースのフリーソフトです
- [fluent-ffmpeg](https://github.com/fluent-ffmpeg/node-fluent-ffmpeg)
    - ffmpegのnode.js用ライブラリです。JavaScript/TypeScriptから動画や音声を変換できるようになります
    - TypeScriptから使うので[@types/fluent-ffmpeg](https://www.npmjs.com/package/@types/fluent-ffmpeg)も導入します
- TypeScriptの実行用に[typescript](https://www.npmjs.com/package/typescript)と[ts-node](https://www.npmjs.com/package/ts-node)も導入します

## 実装
### TypeScript
- 「動画や音声ファイルを読み込んでそのまま複製するコード」が以下のようになります。無事にコピー成功すると`変換完了`と表示されるはずです
```TypeScript
// main.ts
import ffmpeg from 'fluent-ffmpeg';

async function fixMedia(
    input_file: string,
    output_file: string
) {
    // input_fileを読み込んでoutput_fileを生成する
    const converted = await ffmpeg(input_file)
        .on('end', () => {
            console.log(`変換完了`);
        }).save(output_file);
}

fixMedia();
```
- 上記のコードをベースに、`ffmpeg`のメソッドチェーン内で`save`する前に変換用のメソッドを挟むことで動画や音声を希望の形に変換できます
    - メソッドは紹介しきれないほどあるので、適宜[fluent-ffmpegのドキュメント](https://github.com/fluent-ffmpeg/node-fluent-ffmpeg)を参照ください
```TypeScript
// 動画サイズを1980x1080、コーデックをH.264にする例
ffmpeg("input.mp4")
    .size("1980x1080")
    .videoCodec('libx264')
    .on('end', () => {
        console.log(`変換完了`);
    }).save("output.mp4");
```
```TypeScript
// 動画のフレームレートを30fps,音声ビットレートを192kbpsにする例
ffmpeg("input.mp4")
    .fps(30.0)
    .audioBitrate(192)
    .on('end', () => {
        console.log(`変換完了`);
    }).save("output.mp4");
```
```TypeScript
// wav音声ファイルをmp3にする例
ffmpeg("input.wav")
    .toFormat('mp3')
    .on('end', () => {
        console.log(`変換完了`);
    }).save("output.mp4");
```

## 環境構築して動かす
### Node.js関連
- `使うもの`で紹介したライブラリを以下でインストールします
```
npm install fluent-ffmpeg @types/fluent-ffmpeg typescript ts-node
```
- また、前項のmain.tsをコンパイルするために以下を記載したtsconfig.jsonが必要です。詳しくは[こちら](https://numb86-tech.hatenablog.com/entry/2020/07/11/160159)など
```json
{
  "compilerOptions": {
    "esModuleInterop": true
  }
}
```
### Docker関連
- ffmpegはCLIアプリケーションのためインストールが必要です。再現性のためにDockerコンテナ内でインストールするようにします
```Dockerfile
# Dockerfile
FROM node:16-buster

WORKDIR /app

COPY . .

RUN apt update
RUN apt -y upgrade
RUN apt install -y ffmpeg

CMD [ "node_modules/.bin/ts-node","main.ts"]
```
- 変換結果のファイルを得るためにマウント設定を入れた状態でdockerを起動します
```yaml
# docker-compose.yml
version: '3'
services:
  app:
    build: .
    volumes:
      - type: bind
        source: "./"
        target: "/app"
```

- 上記ファイル群+変換したいファイルを同一ディレクトリに配置して以下コマンドを打つと動くはずです
```
npm install
docker-compose build
docker-compose up
```

## 成果物
- ここまでの内容とほぼ同じ内容が以下にあります。再実装する場合など必要に応じて参照ください
    - 同一ディレクトリの動画や音声を全て変換対象にするために[glob](https://www.npmjs.com/package/glob)を利用している点が主な追加点です
https://github.com/mini-hiori/ts-fix-framerate.git


## 参考
- [Node.jsで動画変換処理(fluent-ffmpeg)](https://qiita.com/high-g/items/599836d85ee9e3b166eb)
- [TypeScript の esModuleInterop フラグについて](https://numb86-tech.hatenablog.com/entry/2020/07/11/160159)

## おわりに
- TypeScriptでfluent-ffmpegを利用する日本語記事は軽く検索した感じなかったので、どなたかの参考になれば幸いです
- GWに初めてTypeScript触り始めたくらいの新参なのでマサカリ歓迎です