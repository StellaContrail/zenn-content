---
title: "【AWS】CloudFrontを使用しているアプリケーションでエラーが発生した場合の対処法"
emoji: "☁️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","CloudFront"]
published: true
---

# 概要
CloudFrontを使ったアプリケーションを構築する中で、設定は正しいのに"Failed to load resource: the server responded with a status of 403"や404エラーが返却される場合がある。

# 対処法
CloudFrontのエッジサーバにあるキャッシュが古い可能性がある。
CloudFrontの「キャッシュ削除」から「キャッシュ削除を作成」を選択し、全てのファイル「/*」を対象にてキャッシュ更新を強制させるとエラーが出なくなる。

# その他の可能性
1. CloudFrontからS3のファイルにアクセス出来ていない。
・CloudFront側でOAIないしOACを作成し、S3のバケットポリシーにs3:GetObject権限を追加する。
・S3側のパブリックアクセス許可を有効にしてアクセスをフルオープンにする。
1. CloudFrontのデフォルトルートオブジェクトが設定されていない
・CloudFrontの「一般」にある設定の「編集」を選択し、「index.html」を指定する。
1. React等を使用している場合、適したディレクトリをアップロードしていない。
・(例えばReactの場合) `npm run build`を行った上で、「build」フォルダの中身を全てS3にアップロードする。
