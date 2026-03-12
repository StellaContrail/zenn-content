---
title: "DifyをDev Containerでデバッグする"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Dify"]
published: true
---

少し前に [Difyをローカルでデバッグする](https://qiita.com/Stellarium/items/4fe37d1fa4797afd9644) という記事を書いたが、Dev Container で VSCode 上でデバッグするもっと簡単なやり方があったので記事に残す。

# 起動手順
1. VSCode に [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) をインストールする

   ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/27927/5bb8b0d8-05e7-433c-bf74-f736159ea380.png)

1. `Shift + Ctrl + P` でコマンドパレットを開き、`Dev Containers: Open Folder in Container...` を選択する

   ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/27927/8c2ea0b8-4fb8-43fe-8248-2078e6b200ff.png)

1. Dify プロジェクトのルートディレクトリ（`dify`）を `Open` する

   ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/27927/b6871b61-02ba-4562-944c-13791bf7898a.png)

   :::message alert
   構築途中、`.docker/buildx/current` で Permission Denied が発生する場合は
   ```sudo chown -R $USER:$USER ~/.docker/buildx```
   で権限をログインユーザに設定することで解決する
   :::

   :::message alert
   node_modules 等の再インストールでコンテナ構築途中に止まることがあるので、`Enter` を押して継続する

   例：
   ```✔ The modules directory at "/workspaces/dify/web/node_modules" will be removed and reinstalled from scratch. Proceed? (Y/n) · true``` 
   が出てきたら `Enter` キーを押す
   :::
   
1. ターミナルに "Done. Press any key to close the terminal." と出てきたら開発環境の構築が無事完了
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/27927/1827e5ab-44a8-4fe3-a590-7e1ce97ccb6c.png)
1. デバッグに入るため、まずミドルウェアを起動する。ターミナルから `start-containers` コマンドを実行する
   
   ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/27927/e3aa6f83-c1d6-4dfc-a5fc-dda92b47175c.png)

1. VSCode からデバッグできるように、`launch.json` を `launch.json.template` からコピーして作る

   ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/27927/91de1311-9d42-470e-8247-5b6e9a40a22a.png)

1. VSCode の `Run and Debug` から、デバッグに必要なサービスを立ち上げる
    1. backend : `Python: Flask API` 
    1. frontend : `Next.js: debug full stack`
    1. async worker : `Python: Celery Worker (Solo)`

    ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/27927/7c123677-d230-4ea0-841f-0a48ca8ec047.png)

1. 後は通常のデバッグ作業のように、ブレイクポイントなどを設定してデバッグする

   ![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/27927/568c6647-ec51-4573-952e-8b118436e7ac.png)

   :::message alert
   frontend はホットリロードに対応しているので、変更が即時反映される

   一方で、backend と async worker は対応していないので、変更を反映するには該当サービスを再起動する
   :::


# Dev Container を使うメリット
N 番煎じなので簡単にだけ。

Dev Container のメリットはコンテナ内に開発環境を作り出せることであり、環境（OS とか ランタイムとか）に依らず、well-defined な環境で開発ができること。

Microsoft が提唱した概念がゆえに、VSCode との相性が良い。

より詳しくは [公式ウェブサイト](https://containers.dev/) へ。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/27927/924c8f2c-d61a-464f-a511-0d223940bdbd.png)

> This lets VS Code provide a local-quality development experience including full IntelliSense (completions), code navigation, and debugging regardless of where your tools (or code) are located.
> 
> The Dev Containers extension supports two primary operating models:
> 
> - You can use a container as your full-time development environment
> - You can attach to a running container to inspect it.
> 
> https://code.visualstudio.com/docs/devcontainers/containers
