# クラウドネイティブセキュリティ入門

[セキュリティ・キャンプ2025ミニ（香川）](https://www.security-camp.or.jp/minicamp/kagawa2025.html) の講義資料です。

※ 内容は今後アップデートされる可能性があります。

## 概要

本講義では、クラウドネイティブ環境におけるセキュリティの基礎を学びます。クラウドネイティブの特徴や固有のセキュリティリスクと対策について、コンテナやKubernetesといった技術に焦点を当てて解説します。さらに、実際にハンズオンを通してこれらの環境に触れ、クラウドネイティブセキュリティの実践的な視点を養います。

## 対象者・前提知識

- Linuxの基本的なコマンド操作ができる方（ls, cd, cat, vi など）
- コンテナ、Kubernetesに興味がある方（事前知識は不要）
- クラウドネイティブ環境のセキュリティに興味がある方

## 学習目標

受講生は本講義を通じて以下を習得します。

- クラウドネイティブおよびコンテナ、Kubernetesセキュリティの基本概念
- Kubernetesを通して見る、クラウドネイティブセキュリティの考え方

## 章構成

### [1章 クラウドネイティブとKubernetes](./01_cloud_native_intro)　(60分)
クラウドネイティブとは何か、Kubernetesにおける実践を通して学習します。

### [2章 クラウドネイティブセキュリティ入門](./02_cloud_native_sec) (90分)
クラウドネイティブセキュリティの全体像から、コンテナ・Kubernetesの具体的なセキュリティ対策まで学習します。

## 演習環境

本講義の演習には Kubernetes クラスタを使用します。クラスタ内に事前にツールをインストールする必要はありません。

ローカルPC上に構築する場合は、以下のようなツールを使ってください。

- https://github.com/kubernetes/minikube
- https://github.com/kubernetes-sigs/kind
- https://www.docker.com/products/docker-desktop/
- https://podman-desktop.io/

PC上に構築するのが難しい場合は、Killercoda などの Playgroundを利用することができます。

- https://killercoda.com

Kubernetes クラスタはシングルノード構成でも構いませんが、できればマルチノード構成（Control Plane 1台 ＋ Worker Node 1台以上）が望ましいです。

- https://killercoda.com/playgrounds/course/kubernetes-playgrounds/two-node

また、コマンド実行には Mac または Linux のターミナル環境を使用してください。<br/>
Windows の場合は、WSL や Git Bash を使用することで Linux コマンドを実行できます。

確認用コマンド：

```bash
kubectl version
```

## 事前準備

### ローカル環境の場合

- kubectl コマンドのインストール
- Docker Desktop または Podman のインストール
- テキストエディタの用意（VS Code推奨）

### Playground 利用の場合

- KillerCodaのアカウント作成

## 注意事項

- この講義資料は教育目的で作成されています
- 演習環境は講義用に簡略化されています。実際の本番環境での運用時は、各組織のセキュリティポリシーに従って適切に調整してください

---

## License

Apache License Version 2.0
