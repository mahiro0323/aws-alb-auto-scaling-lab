# 検証内容と確認項目

## 確認した流れ

```text
利用者
  ↓
Application Load Balancer
  ↓
Target Group
  ↓
Auto Scaling Group配下のEC2
```

ALB、Target Group、EC2 Auto Scalingは別々の設定ですが、1つでも設定が合っていないと正常にアクセスできません。

## ヘルスチェック

Target GroupでEC2が`Unhealthy`になる場合は、次の順で確認します。

1. EC2が起動しているか
2. Apacheが起動しているか
3. EC2自身で`curl http://localhost/`が成功するか
4. EC2用Security GroupがALBからのHTTPを許可しているか
5. ヘルスチェックのパスとポートが正しいか
6. サブネットとルートテーブルに問題がないか

確認コマンドの例です。

```bash
sudo systemctl status httpd
curl -I http://localhost/
sudo ss -lntp | grep ':80'
```

## Auto Scaling

負荷を発生させた後は、Auto Scaling Groupのアクティビティ履歴を確認します。

| 確認項目 | 見る場所 |
|---|---|
| スケーリング条件 | Scaling policy |
| EC2起動の開始・完了 | Activity history |
| 起動したEC2 | Instance management |
| ALBへの登録 | Target Group |
| CPUの変化 | EC2メトリクス |

## 切り分けで意識したこと

- ALBへアクセスできない場合と、Targetが`Unhealthy`な場合を分ける
- EC2が起動していてもWebサーバーが停止していれば正常にはならない
- 新しいEC2が起動しても、ヘルスチェックに成功するまでは転送先にならない
- Security GroupはALB用とEC2用を分けて確認する

## 記録の範囲

当時、ALBとAuto Scalingの構成、ヘルスチェック、負荷時の台数変化を学習しました。ただし、次の情報は保存していません。

- 実際に使用したインスタンスタイプ
- CPUしきい値
- スケールアウト・スケールインにかかった時間
- CloudWatchグラフのスクリーンショット
- リソースID

そのため、このリポジトリでは実測値を作らず、確認した仕組みと再現手順を記載しています。

