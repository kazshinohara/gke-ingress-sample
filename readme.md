# GKE Ingress Sample 

GKE で Ingress(L7 LB) + Managed certificate + Cloud Armor を利用する場合のサンプルコードです。
サンプルアプリは hello-go, sorry-pageは計画時メンテナンスなどで利用するイメージです。
Ingress の Path rule を変更して切り替えます。

# 前提条件
* GCP のプロジェクトは作成済み。
* GCP ユーザーアカウントはオーナー権限など検証に不自由ない IAM ロールを持っていること。
* GKE クラスタを VPC ネイティブクラスタで作成済み。 (NEG を使うため)

# Usage

1. このレポジトリをローカルにクローンする。
```sh
$ git clone https://github.com/kazshinohara/gke-ingress-sample.git
```

2. GCP のプロジェクトID を環境変数にセットする。
```sh
$ export PROJECT_ID=XXXXXXXX
```

3. GCP で外部固定 IP を作成する。
```sh
$ gcloud compute addresses create ingress-sample-ip --global
```

4. GCP で Cloud Armor の Security Policy を作成する。
以下では特定の IP アドレスからのアクセスを deny するルールを設定。
```sh
$ gcloud compute security-policies create ingress-sample-security-policy
$  gcloud compute security-policies rules create 1000 \
    --security-policy ingress-sample-security-policy \
    --description "deny traffic from X.X.X.X/X" \
    --src-ip-ranges "X.X.X.X/X" \
    --action "deny-403"
```

5. コンテナイメージを作成する。
```sh
$ cd gke-ingress-sample/hello-go
$ docker build -t asia.gcr.io/${PROJECT_ID}/hello-go:v1 .
$ cd ../sorry-page
$ docker build -t asia.gcr.io/${PROJECT_ID}/sorry-page:v1 .
```

6. コンテナイメージを GCR にプッシュする。
```sh
$ gcloud auth configure-docker
$ docker push asia.gcr.io/${PROJECT_ID}/hello-go:v1
$ docker push asia.gcr.io/${PROJECT_ID}/sorry-page:v1
```

7. マニフェストを編集する。
コンテナイメージのパスに含まれるプロジェクト ID をお使いの環境のものに変更してください。
```sh
$ vim hello-go-deployment.yaml
$ vim sorry-page-deployment.yaml
```

ドメイン名を実際に利用されるものに変更してください。
またGKEのバージョンが 1.16.5-gke.1 以上をお使いの場合は、API version を v1beta2 に変更してください。
```sh
$ vim certificate.yaml
```
9. お使いの DNS サーバーに今回利用するドメイン名(FQDN)と手順3で作成した IP アドレスの組み合わせの A レコードを設定してください。

10. 各マニフェストを GKE に適用する。
以下の順番で適用することをオススメします。
証明書が作成され、LBがreadyになるまで10 ~ 20 分ほど掛かることがあります。
```sh
$ cd ../manifest
$ kubectl apply -f hello-go-deployment.yaml 
$ kubectl apply -f hello-go-service.yaml 
$ kubectl apply -f sorry-page-deployment.yaml 
$ kubectl apply -f sorry-page-service.yaml 
$ kubectl apply -f backendconfig.yaml 
$ kubectl apply -f certificate.yaml
$ kubectl apply -f ingress.yaml
```

11. アクセス確認を行ってください。
```sh
$ curl https://手順8と9で指定したFQDN/
```

