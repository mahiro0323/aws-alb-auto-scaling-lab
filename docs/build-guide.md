# 構築・再現手順

## 1. 検証の目的

ALBでWebアクセスを複数のEC2へ振り分け、EC2 Auto Scalingによってインスタンス台数が変化する流れを確認するための構成です。

当時のリソースIDや負荷値は残っていないため、このドキュメントでは確認した構成を再現できる順番に整理しています。

## 2. Launch Template

EC2 Auto Scalingから同じ設定のEC2を起動できるよう、Launch Templateへ次の内容を設定します。

- Amazon Machine Image
- インスタンスタイプ
- EC2用Security Group
- キーペアまたは管理方法
- Webサーバーを起動するUser Data

User Dataの例です。

```bash
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
echo "Web server: $(hostname)" > /var/www/html/index.html
```

ホスト名を表示することで、ALBの振り分け先が変化することを確認できます。

## 3. Target Group

次の条件でTarget Groupを作成します。

```text
ターゲット: Instance
プロトコル: HTTP
ポート: 80
ヘルスチェックパス: /
```

確認項目は次のとおりです。

- EC2上でApacheが起動していること
- EC2用Security GroupがALB用Security GroupからのHTTPを許可していること
- ヘルスチェックパスがHTTP 200を返すこと

## 4. Application Load Balancer

ALBを複数のPublic Subnetへ配置し、HTTPリスナーからTarget Groupへ転送します。

Security Groupは役割ごとに分けます。

| Security Group | 許可する通信 |
|---|---|
| ALB用 | 利用者からのHTTPまたはHTTPS |
| EC2用 | ALB用Security GroupからのHTTP |

## 5. Auto Scaling Group

Launch Templateを指定し、Auto Scaling Groupを作成します。

```text
最小台数: 検証用の最小値
希望台数: 通常時の台数
最大台数: 負荷時の上限
```

実際の設定値は記録していないため、このリポジトリでは固定値として断定していません。

## 6. 負荷試験

検証用EC2にCPU負荷を発生させる場合の例です。

```bash
yes > /dev/null &
```

停止するときは、プロセスを確認して終了します。

```bash
jobs
kill %1
```

負荷試験中は次を確認します。

1. EC2のメトリクス
2. Auto Scalingのアクティビティ履歴
3. 新しいEC2の起動
4. Target Groupへの登録
5. ヘルスチェックが`Healthy`になること

検証環境で実行し、不要なインスタンスが残らないよう終了後に削除します。

