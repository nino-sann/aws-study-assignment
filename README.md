## 1. プロジェクト概要
本リポジトリは、AWSの学習を目的として作成したCloudFormationテンプレートです。

単一のYAMLファイルで、VPCからセキュリティ・監視まで一括でデプロイできる構成になっています。

| 項目 | 内容 |
|:---:|---|
| 目的 | CloudFormationを利用してAWS環境を構築できるようになる|
| テンプレート形式 | AWS CloudFormaiton(単一YAMLファイル) |
| デプロイリージョン | ap-northeast-1(東京) |
| 主な学習テーマ | ネットワーク設計、リソースの作成、セキュリティ、監視 |

## 2. アーキテクチャ
### 2.1 構成図
![構成図](./aws-study.drawio.svg)

### 2.2 各サービスの役割
| サービス | リソース名 | 役割 |
|:---:|:---:|---|
| VPC | aws-study-vpc | 10.0.0.0/16の仮想ネットワーク。パプリック/プライベートに分離 |
| Internet Gateway | aws-study-igw | VPCとインターネットを接続 |
| EC2 | aws-study-ec2 | アプリケーションサーバー |
| RDS | aws-study-rds | データベース |
| ALB | aws-study-alb | HTTP(80)を受け付けEC2(8080)へ転送するロードバランサー |
| SNS | EC2-CPU-Alarm-Topic | CloudWatch Alarm発火時にメール通知 |
| CloudWatch | EC2-CPUUtilization-Alarm | CPU平均使用率70%超過を5分間隔で監視 |
| WAF | aws-study-alb-waf | ALBへの不正アクセス防御 |
| CloudWatch Logs | aws-waf-logs-alb-alc | WAFのリクエストログを1日保持 |

## 3. ネットワーク構成
| サブネット名 | CIDR | AZ | 用途 |
|---|---|---|---|
| Public1-Subnet | 10.0.1.0/24 | ap-northeast-1a | EC2配置・ALB |
| Public2-Subnet | 10.0.3.0/24 | ap-northeast-1c | ALB(冗長化) |
| Private1-Subnet | 10.0.2.0/24 | ap-northeast-1a | RDS配置 |
| Private2-Subnet | 10.0.4.0/24 | ap-northeast-1c | RDSサブネットグループ用 |

## 4. デプロイパラメータ
スタック作成時に、以下のパラメータを入力します。
| パラメータ名 | 型 | 説明 |
|:---:|:---:|---|
| KeyName | KeyPair名 | EC2にSSH接続する際に使用する既存のキーペア名 |
| RDSUserName | 文字列(デフォルト:admin) | RDSマスターユーザー名 |
| RDSMasterUserPassword | 文字列 | RDSマスターユーザーパスワード(非表示) |
| CidrIpFromInternet | 文字列 | EC2にSSH接続を許可するIPアドレス(/32必須) |
| MyEmailAddress | 文字列 | CloudWatch Alarm通知の送信先メールアドレス |

## 5. デプロイ手順
### 5.1 前提条件
- AWSアカウント(デプロイ権限を持つIAMユーザー)
- ap-northeast-1リージョンにEC2 KeyPairが作成済み
- SNSを受け取れるメールアドレス

### 5.2 デプロイ
AWSマネジメントコンソールから実行
1. CloudFormationにアクセス
2. スタックの作成
3. テンプレートファイルのアップロード
4. パラメータを入力してスタックを作成

注）スタック作成後、SNSから確認メールが届くので「Confirm subscription」をクリックしてください

### 5.3 Outputs（デプロイ後に確認できる値）
| キー名 | 内容 |
|:---:|---|
| Public1EC2Instance | EC2インスタンスID |
| Private1RDSInstance | RDSインスタンスID |
| ApplicationLoadBalancer | ALBのDNS名(アクセスURL) |
| SNSTopicEC2 | SNSトピックのARN |
| awstudyVPC | VPCのID |
| WebACL | WAF WebACLのARN |
| WAFLogGroup | WAFログのロググループ名 |

## 6. セキュリティ設計
### 6.1 セキュリティグループ
| SG名 | ポート | 送信元 | 備考 |
|:---:|:---:|:---:|---|
| alb-sg | 80(http) | 0.0.0.0/0 |  |
| ec2-sg | 22(ssh) | 自IP/32 | 特定IPのみ許可(パラメーター指定) |
| ec2-sg | 80,8080 | alb-sg | ALBからのトラフィックのみ許可 |
| rds-sg | 3306(MySQL) | ec2-sg | EC2からのみDB接続を許可 |

### 6.2 WAF設定
- マネージドルール：AWSManagedRulesCommonRuleSet(SQLインジェクション・XSSなどを自動ブロック)
- デフォルトアクション：Allow(ルールにマッチしたリクエストのみをブロック)
- WAFリクエストログをCloudWatch Logへ出力(保持期間1日)

## 7. 監視・通知設計
| 項目 | 設定値 |
|:---:|---|
| 監視メトリクス | EC2CPUUtilization |
| 閾値 | 70%超過 |
| 評価期間 | 1回(5分間の平均) |
| 通知手段 | SNS → Email(デプロイ時に指定したアドレス) |
| アラーム名 | EC2-CPUUtilization-Alarm |

## 8. 削除手段
注）削除前に以下の点を確認してください

- EC2のDisableApiTerminationはfalse(テンプレート通り)なので、削除可能です
- RDSのDelectionProtectionfalse(テンプレート通り)なので、削除可能です
- RDSの最終スナップショットが不要であることを確認してください

AWSマネジメントコンソールから実行
1. CloudFormationにアクセス
2. スタック一覧
3. 作成したスタック名を選択
4. 削除

## 9. 注意事項・学習メモ
### 9.1 コストについて
- EC2、RDS、ALBは稼働中に料金が発生します
- WAFもWebACLとリクエスト数に応じた料金が発生します
- 学習終了後はスタックを削除して、コストを抑えてください

### 9.2 構築時のポイント
- EC2のSSH許可IPは/32形式のみ許可されており、0.0.0.0/0は設定できない仕様にしています
- RDSはプライベートサブネットに配置し、EC2のセキュリティグループからのみ接続を許可することで、外部からのアクセスを遮断しています
- RDSマスターユーザーパスワードは、NoEchoで非表示となるように設定しています
- WAFLogのCloudWatch Logsグループ名は、必ずaws-waf-logs-で始める必要があります

### 9.3 今後の学習候補
- EC2をAuto Scalingでグループ化して、可用性を向上させる
- RDSをマルチAZ構成にする










