---
title: "GPU Utilization vs Saturation: どちらを測定すべきか？"
emoji: "📝"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["GPU","NVIDIA","grafana","prometheus"]
published: true
---

# GPU Utilization vs Saturation: どちらを測定すべきか？

GPU のパフォーマンスを監視する際、主に **"Utilization（使用率）"** と **"Saturation（飽和度）"** という 2 つの指標が使われます。
一見、どちらも GPU の稼働状況を示しているように見えますが、それぞれ異なる情報を提供します。

本記事では、これらの指標の違いを解説し、Utilization を測定するだけでは不十分な理由を説明します。

また、`dcgm-exporter` を用いて GPU の Saturation を確認する方法や、ストリーミングプロセッサ（SM）の役割、GPU アーキテクチャの制約についても触れながら、どのタイミングでどちらの指標を使用すべきかをご紹介します。

## tl;dr
- GPU リソースの監視には主に Utilization と Saturation の２つが使われる
- Utilization は GPU が稼働している時間の割合、Saturation はリソースの割合を示す
- このため、計算ユニットのスケーリング指標や、コスト効率を算出する指標には Utilization は不向きであり、Saturation を使うことが推奨される
- Saturation は dcgm-exporter を使うことで監視できる


## GPU Utilization と Saturation の定義

### GPU Utilization
GPU がカーネルを実行している **時間の割合（time in use）** を示す指標です。

例えば、GPU が 60 秒間に 30 秒稼働したとき、GPU Utilization は 50% となります。
この指標はシンプルで、GPU の基本的な動作状況を把握するのに便利です。

### GPU Saturation
GPU の計算パイプラインやメモリパイプラインが **どれだけ活用されているか（resource in use）** を示す指標です。

Saturation を確認することで、GPU が本当にフル稼働しているのか、
あるいは一部のリソースがアイドル状態（余剰がある状態）であるのかを把握できます。


## Utilization だけでは正確にリソースを測定できない理由

GPU Utilization は直感的で分かりやすい指標ですが、この指標だけでは GPU のリソースが十分に活用されていることを判断できません。

具体的には以下の点が問題となります。

- **見誤った 100% の認識（Misleading understanding of Utilization）**  
  たった 1 つのストリーミングプロセッサだけが稼働している状態でも、GPU Utilization が 100% になってしまう場合があります。
  しかし、実際には GPU 全体の多くのプロセッサがアイドル状態であり、余剰リソースが過分に存在している状態かもしれません

- **スケーリングのための洞察不足（Inappropriate scaling metric）**
  計算ユニットのスケーリングには、リソースがどれだけ「飽和」しているかを知ることが重要です。
  Utilization のみでは追加のワークロードを処理できる余裕があるか判断しにくいです

- **コスト効率の低下（Cost inefficiency）**
  Utilization は GPU が動作している時間の割合のみを示します。
  そのため、実際にいくつのストリーミングプロセッサが稼働しているか、メモリ帯域幅が十分に活用されているかは分かりません。
  本当は１つの計算ユニットで複数のプロセスを処理できるのに、新たな計算ユニットを用意してしまっており、システムのコスト効率を悪化させている可能性があります

このように、100% の Utilization が GPU の性能を最大限に引き出していることの確かなエビデンスとはならないことに注意しなければいけません。


## ストリーミングプロセッサの理解

GPU のパフォーマンスを正しく理解するには、**ストリーミングプロセッサ（SMs）** の役割を知ることが重要です。

- **ストリーミングプロセッサとは？**  
  ストリーミングプロセッサは、GPU 内で算術演算や論理演算を行う基本単位です。
  これらは **ストリーミングマルチプロセッサ（SM）** にまとめられ、各 SM は複数の CUDA コアを持ちます

- **並列処理の役割**  
  GPU は何百、何千ものストリーミングプロセッサを搭載し、大規模な並列処理が可能です。
  これにより、行列乗算などの計算を高速に実行できます

- **Saturation との関連**  
  Saturation は GPU の計算パイプライン（ワープ）がどれだけ活用されているかを示します。
  もし一部のプロセッサがアイドル状態であれば、Saturation の値が低くなり、ワークロードの最適化が必要であることを示します

![Typical NVIDIA GPU architecture](https://raw.githubusercontent.com/StellaContrail/zenn-content/main/images/img_16.png)

> Typical NVIDIA GPU architecture
> 
> ref. Moisés Hernández, et al. 2013. "Accelerating Fibre Orientation Estimation from Diffusion Weighted Magnetic Resonance Imaging Using GPUs". https://doi.org/10.1371/journal.pone.0061892

より詳細な情報は [Achieved Occupancy](https://docs.nvidia.com/gameworks/content/developertools/desktop/analysis/report/cudaexperiments/kernellevel/achievedoccupancy.htm) を参照。

## dcgm-exporter を使用した GPU Saturation の監視

LLM の推論処理などの GPU に依存するワークロードでは、Utilization だけでなく Saturation を確認することで、より正確な GPU のパフォーマンス状況を把握できます。

![dcgm-exporter + Prometheus + Grafana](https://raw.githubusercontent.com/StellaContrail/zenn-content/main/images/img_17.png)


> This diagram shows where each service is deployed. NVIDIA GPUs are attached to machines. The
machines run Kubernetes. NVIDIA DCGM and Node Exporter run inside of Kubernetes. Prometheus
pulls data from the Kubernetes node, NVIDIA DCGM, and Node Exporter to store in a database.
Grafana pulls data from Prometheus for use in analysis.
> 
> ref. Timothy Bargo. 2022. "Monitoring COMPaaS" (https://www.evl.uic.edu/documents/bargo_675370482_sp-22-cs-398-final-research-report.pdf)

本記事では Grafana を使っていますが、Prometheus 形式でメトリクスデータを吐き出せるので、CWAgent や AMA と連携することで CloudWatch や Azure Monitor にメトリクスを反映することも可能です。

### 手順

1. `nvidia-toolkit` を導入する
   - [NVIDIA Toolkit インストール手順](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
2. `Docker` を導入する
   - [Docker インストール手順](https://docs.docker.com/engine/install/ubuntu/)
3. プロジェクトをクローンする
   ```bash
   git clone git@github.com:teppeisasaki/dcgm-metrics.git
   ```
4. Docker コンテナを起動する
   ```bash
   cd dcgm-metrics
   sudo docker compose up -d
   ```
5. `Grafana` ダッシュボードにアクセスする
   - `http://localhost:3000/` にアクセスしてログインする。（初期状態の認証情報は admin:admin）
6. `NVIDIA DCGM Exporter Dashboard` をインポートする
   - [NVIDIA DCGM Exporter Dashboar](https://grafana.com/grafana/dashboards/12239-nvidia-dcgm-exporter-dashboard/)
7. ダッシュボードに `DCGM_FI_PROF_SM_OCCUPANCY` メトリクスを追加します。これが Saturation のメトリクスになります
8. 完了

## GPU Utilization と Saturation の使い分け

### Utilization メトリクスが有用な場面

- **簡易な状態確認**  
  GPU が動作しているか確認するために利用します

- **基本的なトラブルシューティング**  
  適切にワークロードが GPU にロードされているかを検証する際に役立ちます

- **一般的な監視**  
  詳細な情報が不要な場合に適しています

### Saturation メトリクスが有用な場面

- **パフォーマンスチューニング**  
  GPU の最適化や飽和状態の確認に利用します

- **スケーリングの判断**  
  追加の GPU が必要か、ワークロードの調整が必要かを判断します

- **ボトルネックの特定**  
  GPU 内の計算ユニットやメモリコントローラの制約を特定するのに役立ちます


## まとめ

GPU Utilization は稼働状況を簡単に把握するのに適していますが、GPU の真の能力を評価するには不十分です。

一方、Saturation は GPU の計算リソースやメモリパイプラインがどれだけ活用されているかを示し、パフォーマンスチューニングに役立ちます。

**両方の指標を適切に使い分けることで、GPU リソースを最大限に活用し、深層学習アプリケーションの最適化が可能になります。**

## 参考文献
- Understanding NVIDIA GPU Performance: Utilization vs. Saturation (2023)
  - https://arthurchiao.art/blog/understanding-gpu-performance/
- おうちディープラーニングPCにモニタリング環境を整える
  - https://qiita.com/dcm_miura-h/items/1e545f6cd486f27ecfaa
