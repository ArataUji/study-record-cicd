# 『Kubernetes CI/CDパイプラインの実装』学習記録
[『Kubernetes CI/CDパイプラインの実装』](https://www.amazon.co.jp/dp/4295012750)は2021年10月に出版されたものであり、情報が少し古いものになっている。紹介されているコマンドなどをそのまま実行するとエラーに直面することが多かったので、ここで独自に内容の補完を行う。

- [『Kubernetes CI/CDパイプラインの実装』学習記録](#kubernetes-cicdパイプラインの実装学習記録)
  - [第1章 開発プロセスの運用変化](#第1章-開発プロセスの運用変化)
  - [第2章 クラウドネイティブ開発に向けた環境準備](#第2章-クラウドネイティブ開発に向けた環境準備)
    - [2-2 Kuberenetesクラスタの構築](#2-2-kuberenetesクラスタの構築)
    - [2-3 Gitリポジトリの準備](#2-3-gitリポジトリの準備)
    - [2-4 マイクロサービスのデプロイ](#2-4-マイクロサービスのデプロイ)
  - [第3章 Tekton Pipelines の概要](#第3章-tekton-pipelines-の概要)
  - [第4章 継続的インテグレーションのパイプライン](#第4章-継続的インテグレーションのパイプライン)
    - [4-2 ソースコードの取得](#4-2-ソースコードの取得)
    - [4-5 コンテナビルド](#4-5-コンテナビルド)
    - [4-7 コンテナデプロイ](#4-7-コンテナデプロイ)
    - [4-8 パイプラインの実装](#4-8-パイプラインの実装)
  - [第5章 イベント駆動のパイプライン実行](#第5章-イベント駆動のパイプライン実行)
    - [5-2 Tekton Triggers の導入](#5-2-tekton-triggers-の導入)
    - [5-3 アプリケーションソースコードの更新](#5-3-アプリケーションソースコードの更新)
    - [5-4 マニフェストリポジトリの更新](#5-4-マニフェストリポジトリの更新)
  - [第6章 Argo CD の概要](#第6章-argo-cd-の概要)
  - [第7章 継続的デリバリのデプロイメント](#第7章-継続的デリバリのデプロイメント)
    - [7-3 機密情報の管理](#7-3-機密情報の管理)
    - [7-4 マニフェストのデプロイ](#7-4-マニフェストのデプロイ)
  - [8章 継続的デリバリのリリース](#8章-継続的デリバリのリリース)
    - [8-3 Argo Rollouts の導入](#8-3-argo-rollouts-の導入)

## 第1章 開発プロセスの運用変化
この章では特筆すべき点はなかった。

## 第2章 クラウドネイティブ開発に向けた環境準備
### 2-2 Kuberenetesクラスタの構築
本書と同様にGCPでk8sクラスタを構築した。GCPでは初回登録時に、90日間有効な47000円程度のFreeTrialクレジットがあるからだ。既に自分のGoogleアカウントで初回クレジットを使い切ってしまっている場合は、適当にGoogleアカウントを新規作成して、それでGCPに初回登録すればよいだろう。
<br>

本書ではコマンドで全てのGCPの操作をしていたが、わかりやすさのためGUIで操作した所もあった。GCPプロジェクトの作成はヘッダのプロジェクトを選択できる箇所をクリックし、出てきた画面の「新しいプロジェクト」から作成した。k8sクラスタの作成は、GKEのダッシュボードから「作成」をクリック、クラスタモードはStandardモードを選択する。Kubernetesのバージョンはv1.28である。
<br>

ここからは、できるだけ安価にGKEを使えるような設定を紹介する。ロケーションタイプは「ゾーン」を選択し、ゾーンは一番安価（2024年5月時点）な「us-central」のいずれかから選択する。「ノードプール」の「default-pool」のタブを押下し、「ノード数」を最小構成の2とする。「ノード」タブを押下し、「ブートディスクのサイズ」も変更する。最小は10GBだが、これだと4章で実行するコンテナイメージスキャンのTaskで、ディスクサイズが足りずエラーとなる。50GBほどあると安心である。以上を変更したら「作成」をクリックする。
<br>

GKEクラスタは常時課金され続けるので、学習終了時には「ノードプールの詳細」画面から「編集」をクリックし、ノード数を0にして課金を抑える。なお、ノード数が0でもGKEクラスタがある限りは、その分課金され続ける。GKEクラスタを削除してしまうと各種インストール作業を一からやり直す必要があり手間であるため、GKEクラスタ単体分の課金は許容した。GKEクラスタ単体だと1日あたり420円程度の費用が掛かるので、90日フル稼働させたら費用は37800円程度になるだろう。これにノードとネットワーキング代を加算しても、初回クレジットの47000円以内に収まるはずである。
<br>

学習開始時には、同じく「ノードプールの詳細」画面から「サイズ変更」をクリックし、ノード数を2に変更する。しばらくするとノードがプロビジョニングされ、行き場を失っていたPodなどがノードに配置される。
<br>

2章ではではkubectxとkubensをインストールする手順が紹介されているが、現在のGKEにはデフォルトでインストールされているため、この手順は不要である。
<br>

### 2-3 Gitリポジトリの準備
プッシュやクローン時に使用するGitLabのURLについて、本書で記述されている形式と、実際の形式が異なっている。実際の形式は以下である。
```
https://${GITLAB_USER}:${GITLAB_TOKEN}@gitlab.com/${GITLAB_GROUP}/bookinfo.git
```
`gitlab.com`以下の最初のパス部分はユーザ名ではなく、自身で作成したグループ名が来る。そのため、URLを指定する場合は、URLにグループ名を直接書き込むか、`GITLAB_GROUP`などの環境変数を作成して、それを呼び出すとよいだろう。2章以降もGitLabのURLは何回も登場するので、都度読み替え（書き換え）る必要がある。

### 2-4 マイクロサービスのデプロイ

## 第3章 Tekton Pipelines の概要
Tekton PipelinesとTekton CLIは本書と同様のバージョンをインストールすること。Tekton PipelinesとTekton CLIは、両方最新バージョンをインストールすると、相性の問題でtknコマンドが使えなくなる。

## 第4章 継続的インテグレーションのパイプライン
### 4-2 ソースコードの取得
本書ではsedコマンドを使ってマニフェストを生成していたが、わかりやすさのためVimで直接マニフェストを編集し、それをデプロイした。
```
vim bookinfo-manifests/tekton/auth/git-lab-token.yaml
```
```
vim bookinfo-manifests/tekton/taskruns/git-clone-bookinfo-run.yaml
```
```
vim bookinfo-manifests/tekton/taskruns/git-clone-bookinfo-manifests-run.yaml
```
以後マニフェストの編集にはすべてVimを使った。
<br>

### 4-5 コンテナビルド
tknコマンドで取得したKanikoのTaskをそのまま使用すると、GitLabの認証に失敗し、コンテナレジストリにビルドしたイメージをプッシュできない。TaskRunにサービスアカウントをマッピングすると、step毎にコンテナ内に`$HOME/.docker/config.json`が作成され、認証情報が書き込まれる。しかしKanikoのイメージはデフォルトでは`/kaniko/.docker/config.json`を見に行こうとするため認証エラーとなる。そのため、`$HOME/.docker/config.json`を見るように、環境変数の`DOCKER_CONFIG`を変更する必要がある。（[参考](https://zenn.dev/koduki/articles/5fc5e70d056fd2)）。
<br>

以上を踏まえ、新しく`kaniko-modified.yaml`というマニフェストファイルを作成し、[Tekton HubからKanikoのYAML](https://hub.tekton.dev/tekton/task/kaniko)をコピペし、環境変数設定を追記してからTaskをデプロイする。
```
vim kaniko-modified.yaml
```
```yaml
# ～省略～

  steps:
    - name: build-and-push
      env:                     # 追記箇所
        - name: DOCKER_CONFIG  # 追記箇所
          value: /root/.docker # 追記箇所
      workingDir: $(workspaces.source.path)
      image: $(params.BUILDER_IMAGE)
      args:
        - $(params.EXTRA_ARGS)
        - --dockerfile=$(params.DOCKERFILE)
        - --context=$(workspaces.source.path)/$(params.CONTEXT)
        - --destination=$(params.IMAGE)
        - --digest-file=$(results.IMAGE_DIGEST.path)

# ～省略～
```
```
kubectl apply -f kaniko-modified.yaml
```
<br>

TaskRunもデフォルトのままだとエラーになるので編集が必要である。本書だと`BUILDER_IMAGE`にv1.3.0のKanikoを指定しているが、このイメージでTaskRunを実行すると以下のようなエラーが出て失敗する。
```
[build-and-push] kaniko should only be run inside of a container, run with the --force flag if you are sure you want to continue
```
これはDocker EngineとKanikoのバージョン相性の問題で発生するエラーである（[参考](https://gitlab-docs.creationline.com/ee/ci/docker/using_kaniko.html#%E3%82%A8%E3%83%A9%E3%83%BCkaniko-should-only-be-run-inside-of-a-container-run-with-the---force-flag-if-you-are-sure-you-want-to-continue)）。TaskのデフォルトのKanikoバージョン、v1.5.1を使用したことでエラーは解消した。
<br>

以上を踏まえると、KanikoのTaskRunはこうなる。
```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: kaniko-reviews-run-
spec:
  serviceAccountName: tekton-admin
  taskRef:
    name: kaniko
  params:
    - name: IMAGE
      value: registry.gitlab.com/kubernetes-exercises/bookinfo/reviews:$(context.taskRun.name)
    - name: CONTEXT
      value: ./reviews-wlpcfg
  workspaces:
  - name: source
    persistentVolumeClaim:
      claimName: bookinfo
    subPath: reviews
```
`BUILDER_IMAGE`をTaskRunで指定しないことで、デフォルトのv1.5.1が使用される。また、`EXTRA_ARGS`の記述は不要だと思ったので削除した。

### 4-7 コンテナデプロイ
「ServiceAccountを作成するとそれと連携した『トークン』と『証明書』が自動的にSecretとして発行されます」と本書では書かれているが、Kubernetes v1.24からはSecretの自動発行はされないようになっている（[参考](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.24.md)）。
<br>

また、「ブートディスクのサイズ」を最小の10GBにしていたために、KubectlのTaskRunデプロイ時にリソースが足りないためエラーとなった。新しいノードプールを作成し、「ブートディスクのサイズ」を50GBにしたノードをプロビジョニングし、再度TaskRunをデプロイしたところ、成功した。

### 4-8 パイプラインの実装
[4-5 コンテナビルド](#4-5-コンテナビルド)の時と同様、Docker EngineとKanikoイメージのバージョン相性の問題でPipelineRunが失敗するので、`build-deploy-reviews.yaml`内の`build-container`Taskのparam、`BUILDER_IMAGE`を削除し、デフォルトのバージョンを適用する。
<br>

また、`test-container`と`deploy-container`の`IMAGE_DIGEST`のvalue部分が、`$(tasks.build-container.results.IMAGE-DIGEST)`となっているが、これだとイメージダイジェストを拾えない。正しくは、最後の部分が`IMAGE_DIGEST`（単語の区切りがハイフンではなくアンダースコア）となる。

## 第5章 イベント駆動のパイプライン実行
### 5-2 Tekton Triggers の導入
Kubernetes v1.25以降では、PodSecurityPolicy (PSP) が削除された（[参考](https://kubernetes.io/docs/concepts/security/pod-security-policy/)）。本書では、Tekton Triggersのバージョンはv0.15.0を指定しているが、このバージョンはPSPを使用しているため、Kubernetes v1.25以上でインストールしようとするとエラーが発生する。
```
error: resource mapping not found for name: "tekton-triggers" namespace: "" from "https://storage.googleapis.com/tekton-releases/triggers/previous/v0.15.0/release.yaml": no matches for kind "PodSecurityPolicy" in version "policy/v1beta1"
ensure CRDs are installed first
```
最新バージョンのTekton Triggersをインストールすることにした。
```
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml
```
<br>

また、v0.48.1のNginx Ingress Controllerをインストールしたところ、Ingress ControllerのPodでエラーが起きており、Ingress Controllerが機能しなかった。
```
NAME                                        READY   STATUS             RESTARTS       AGE
ingress-nginx-admission-create-jcxjj        0/1     Completed          0              93m
ingress-nginx-controller-678566c5d9-2pkf9   0/1     CrashLoopBackOff   29 (86s ago)   93m
```
v0.48.1が古いので、Kubernetes v1.28と相性が悪い。Ingress ControllerがKubernetes v1.22で廃止されたAPI`networking.k8s.io/v1beta1/ingresses`を呼び出していることが影響している（[参考](https://kubernetes.io/docs/reference/using-api/deprecation-guide/#ingress-v122)）。最新バージョンのNginx Ingress Controllerをインストールすることにした。
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```
これにより、Ingressが機能した。なお、IngressへのIPアドレス払い出しは、しばらく時間がかかることに注意すること。また、PCのhostsファイルにIPアドレスとドメインの対応付けをせず、直接IPアドレスからWebページにアクセスしようとすると、404エラーになることに注意すること。

### 5-3 アプリケーションソースコードの更新
ソースコードの更新後にプッシュし、パイプラインが成功したが、Reviews画面に星マークは追加されていなかった。ソースコード変更を契機とするパイプライン自体は成功したので、一旦見逃すことにする。

### 5-4 マニフェストリポジトリの更新
[4-8 パイプラインの実装](#4-8-パイプラインの実装)と同様に、Pipelineをそのままデプロイするとエラーとなるので、修正を施す。それに加えて、`update-manifests`と`push-manifests`のスクリプトのvalue部分も`IMAGE-DIGEST`となっているので、`IMAGE_DIGEST`に直す。

## 第6章 Argo CD の概要
この章では特筆すべき点はなかった。

## 第7章 継続的デリバリのデプロイメント
### 7-3 機密情報の管理
本書で紹介されているExternalSecretはKubernetes External Secrets（KES）だが、出版された翌月の2021年11月には、KESは非推奨であると宣言された（[参考](https://github.com/external-secrets/kubernetes-external-secrets/issues/864)）。よって、KESではなくExternal Secrets Operator（ESO）をインストールし、ExternalSecretを実現する。KESとESOの違いについては、[ZOZOの技術ブログ](https://techblog.zozo.com/entry/kubernetes-external-secrets-to-external-secrets-operator)がわかりやすく解説してくれている。


[ESOの導入方法を紹介しているブログ](https://www.creationline.com/tech-blog/66988)をベースに、ExternalSecret導入のためのコマンドを実行していった。
<br>

まず、HelmでESOをリポジトリ登録し、インストールする。
```
helm repo add external-secrets https://charts.external-secrets.io
```
```
helm install external-secrets external-secrets/external-secrets \
-n external-secrets \
--create-namespace
```
`stg`に移動し、ServiceAccountの`bookinfo-reviews`に以下の通りアノテーションを追加する。これがGSAに対応するKSAとなる。
```
kubens stg

kubectl annotate sa bookinfo-reviews "iam.gke.io/gcp-service-account"="argo-serviceaccount@${PROJECT_ID}.iam.gserviceaccount.com"
```
GSAの`argo-serviceaccount`を作成する。
```
gcloud iam service-accounts create argo-serviceaccount --project=${PROJECT_ID}
```
GSAに`roles/secretmanager.secretAccessor`を付与する。Secret Managerのシークレットへのアクセスに必要なロールである。
```
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member "serviceAccount:argo-serviceaccount@${PROJECT_ID}.iam.gserviceaccount.com" \
--role "roles/secretmanager.secretAccessor"
```
GSAに`roles/iam.serviceAccountTokenCreator`を付与する。GCPアクセストークンの生成に必要なロールである。
```
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member "serviceAccount:argo-serviceaccount@${PROJECT_ID}.iam.gserviceaccount.com" \
--role "roles/iam.serviceAccountTokenCreator"
```
権限`iam.serviceAccounts.getIamPolicy`だけを含む、カスタムロール`getIamPolicy`を作成する。この権限はgcloudコマンドでWorkloadIdentity設定時に必要となる。
```
gcloud iam roles create getIamPolicy --project=${PROJECT_ID} \
--title=getIamPolicy \
--permissions="iam.serviceAccounts.getIamPolicy" \
--stage=GA
```
GSAに`getIamPolicy`を付与する。
```
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member "serviceAccount:argo-serviceaccount@${PROJECT_ID}.iam.gserviceaccount.com" \
--role "projects/${PROJECT_ID}/roles/getIamPolicy"
```
GSA（argo-serviceaccount）とKSA（bookinfo-reviews）のIAMポリシーバインディングを行う。
```
gcloud iam service-accounts add-iam-policy-binding argo-serviceaccount@${PROJECT_ID}.iam.gserviceaccount.com \
--role "roles/iam.workloadIdentityUser" \
--member "serviceAccount:${PROJECT_ID}.svc.id.goog[stg/bookinfo-reviews]" \
--project ${PROJECT_ID}
```
最後に、NameSpace:`external-secrets`にあるServiceAccount:`external-secrets`に、アノテーションを追加し、GSAと紐づける。
```
kubectl annotate sa external-secrets -n external-secrets \
"iam.gke.io/gcp-service-account"="argo-serviceaccount@${PROJECT_ID}.iam.gserviceaccount.com"
```
<br>

ESOのインストールと、GSAとKSAの紐づけが完了したら、カスタムリソースのSecretStoreとExternalSecretをデプロイする。NameSpaceは`stg`で行う。
```
kubens stg
```
以下の通りSecretStoreを作成し、デプロイする。
```
vi secret-store.yaml
```
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gitlab-auth-secretstore
spec:
  provider:
    gcpsm:
      projectID: cloudnativecicd            # 自分で設定したプロジェクトIDにする
      auth:
        workloadIdentity:
          clusterLocation: us-central1-c    # クラスタが配置されているゾーンにする
          clusterName: cluster-1            # 自分で設定したクラスタ名にする
          clusterProjectID: cloudnativecicd # 自分で設定したプロジェクトIDにする
          serviceAccountRef:
            name: bookinfo-reviews
```
```
kubectl apply -f secret-store.yaml
```
以下の通りExternalSecretを作成し、デプロイする。
```
vi external-secret.yaml
```
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: gitlab-auth-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gitlab-auth-secretstore
    kind: SecretStore
  target:
    name: gitlab-auth-secret
    creationPolicy: Owner
    template:
      type: kubernetes.io/dockerconfigjson
  data:
    - secretKey: .dockerconfigjson
      remoteRef:
        key: gitlab-auth
```
```
kubectl apply -f external-secret.yaml
```
<br>

以上によりExternalSecretが動作し、Secret ManagerをもとにSecretが作成された。
```
kubectl get secret
~~~

NAME                 TYPE                             DATA   AGE
gitlab-auth-secret   kubernetes.io/dockerconfigjson   1      3d11h
```
また、認証情報が有効になったことで、reviewsのPodも展開された。
```
kubectl delete pod -l app=reviews
kubectl get pod 
~~~

NAME                          READY   STATUS    RESTARTS   AGE
details-7f79767d5d-dsh8d      1/1     Running   0          22h
productpage-8bd45b586-cdmw5   1/1     Running   0          22h
ratings-76d4f59686-69x9g      1/1     Running   0          22h
reviews-67bbdd6b9d-9w9zd      1/1     Running   0          22h
reviews-67bbdd6b9d-pc8dn      1/1     Running   0          22h
reviews-67bbdd6b9d-q4f8x      1/1     Running   0          22h
```

### 7-4 マニフェストのデプロイ
ExternalSecret登録とApplicationの登録の順番を逆にした。Application登録後にデプロイされるServiceAccount:`bookinfo-reviews`がKSAとなるため、先にこちらにアノテーションを付与する。
```
kubens prod

kubectl annotate sa bookinfo-reviews "iam.gke.io/gcp-service-account"="argo-serviceaccount@${PROJECT_ID}.iam.gserviceaccount.com"
```
GSAとKSAを紐づける。
```
gcloud iam service-accounts add-iam-policy-binding argo-serviceaccount@${PROJECT_ID}.iam.gserviceaccount.com \
--role "roles/iam.workloadIdentityUser" \
--member "serviceAccount:${PROJECT_ID}.svc.id.goog[prod/bookinfo-reviews]" \
--project ${PROJECT_ID}
```
`stg`と同じExternalSecretとSecretStoreを`prod`にデプロイする。
```
kubectl apply -f secret-store.yaml
kubectl apply -f external-secret.yaml
```

## 8章 継続的デリバリのリリース
### 8-3 Argo Rollouts の導入
「Argo Rolloutsは本書執筆時点においてはまだまだ導入期のプロジェクトであるため」という説明があるため、最新バージョンのArgo Rolloutsをインストールした方が良いと判断した。
```
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```
kubectl用プラグインも最新のものをインストールする。
```
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
sudo install kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

kubectl argo rollouts version
```