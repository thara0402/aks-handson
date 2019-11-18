# Azure Kubernetes Service ハンズオン

## 事前準備
ハンズオンで利用する PC が Windows の場合、Docker for Windows をインストールしておきましょう。

## ハンズオンで利用するリソースの構築
なにはともあれ、Azure Kubernetes Service（AKS）を作成しておきましょう。
Azure ポータルを利用して、基本的にデフォルトの設定のまま作成するれば OK です。

合わせて、コンテナリポジトリを作成しましょう。  
お手軽なのは、Docker Hub です。GitHub のアカウントでサインインできます。パブリックなリポジトリなら、無料です。  
Azure Container Registry でもいいですが、AKS にコンテナをデプロイする際に資格情報を渡す必要があるので、最初は Docker Hub がお勧めです。

## AKS への接続情報を取得する
Azure CLI 2.0 を使って、AKS から資格情報を取得しましょう。

```shell-session
$ az aks get-credentials -g xxx -n xxx
```

## kubectl をインストールする

公式サイトから kubectl.exe をダウンロードして、任意のフォルダに配置し、環境変数にパスを通せば完了です。  
さっそく、ノードの情報を確認しましょう。
```shell-session
$ kubectl get nodes
```
Kubernetes のダッシュボードも使えます。  
AKS を作成する際に RBAC を有効にしている場合は、権限を付与します。
```shell-session
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
```
ダッシュボードのプロキシを作成します。
```shell-session
$ kubectl proxy
```
下記の URL からダッシュボードを表示できます。  
http://127.0.0.1:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy/#!/overview?namespace=default

## AKS にコンテナをデプロイする
さっそく、AKS にコンテナをデプロイしてみましょう。シンプルな NGINX のコンテナイメージを使います。

```shell-session
$ kubectl run demo-nginx --image nginx
$ kubectl get all -n default
$ kubectl expose deployments demo-nginx --port=80 --type=LoadBalancer
$ kubectl get svc -w
$ kubectl scale deployment demo-nginx --replicas=3
$ kubectl get deploy demo-nginx -o yaml
```

覚えておくと役立つトラブルシュートコマンドです。

```shell-session
$ kubectl describe pod
$ kubectl logs POD_NAME
$ kubectl exec -it POD_NAME /bin/sh
```

## AKS に ASP.NET Core アプリケーションをデプロイする
まずは、ローカル環境でコンテナを動かしてみましょう。  
ASP.NET Core プロジェクトに Dockerfile を追加すれば OK です。

```shell-session
$ docker build -t thara0402/k8sdemo:3.0.0 .
$ docker run --rm -it -p 8000:80 --name k8sdemo thara0402/k8sdemo:3.0.0
$ docker push thara0402/k8sdemo:3.0.0
```

ここでは、YAML ファイルを使って、AKS にデプロイしてみます。

```shell-session
$ kubectl apply -f deployment.yaml
$ kubectl apply -f service.yaml
$ kubectl get deploy -l app=demo-app
$ kubectl get deploy -l app=demo-app,version=v10
$ kubectl delete -f deployment.yaml
```
## AKS に Helm を使ってコンテナをデプロイする
公式サイトから helm.exe をダウンロードして、任意のフォルダに配置し、環境変数にパスを通せば完了です。 

k8s に Helm のサーバーサイドになる tiller という Pod を作成します。  
AKS を作成する際に RBAC を有効にしている場合は、権限を付与します。
```shell-session
$ kubectl apply -f service.yaml
$ helm init --service-account=tiller
$ kubectl get pod --all-namespaces
```
Helm の Server と Client のバージョンが表示できれば成功です。
```shell-session
$ helm version
```

ASP.NET Core プロジェクトに Dockerfile と同じディレクトリに移動します。
```shell-session
$ mkdir chats
$ cd chats
$ helm create k8sdemo
```
k8sdemo ディレクトリに作成された values.yaml を開きます。  
nginx がデプロイされる構成になっているので、下記の項目を編集します。

Parameter | Description | value
--------- | ----------- | -------
`image.repository` | デプロイする Docker image | `thara0402/k8sdemo`
`image.tag` | デプロイする Docker image のタグ | `3.0.0`
`service.type` | サービスのタイプ | `LoadBalancer`

values.yaml の編集が完了したら、デプロイします。
```shell-session
$ helm install -n webapp k8sdemo
```

デプロイしたアプリを更新する場合は、こちらのコマンドを使います。
```shell-session
$ helm upgrade webapp k8sdemo
```

削除する際に、引数で purge を指定することで再度同じ名前でデプロイできます。
```shell-session
$ helm delete webapp --purge
```

## AKS に ACR からコンテナをデプロイする

k8s に ACR へ接続するために資格情報を secret としてデプロイします。

```shell-session
$ kubectl create secret docker-registry acr-gooner \
          --docker-server=gooner.azurecr.io \
          --docker-username=gooner \
          --docker-password=xxxxxxxxxxxxxxxxxxxx \
          --docker-email=test@gmail.com```
```

Deployment.yaml を編集し、imagePullSecrets を設定します。
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: {{ template "fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    metadata:
      labels:
        app: {{ template "name" . }}
    spec:
      imagePullSecrets:
        - name: {{ .Values.image.imagePullSecret }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:  
            - containerPort: {{ .Values.service.internalPort }}
```

## AKS に Helm を使って nginx-ingress をデプロイする

Helm Charts のリポジトリが公開されているので、いろいろなアプリを k8s にデプロイできます。  
https://hub.kubeapps.com/

ここでは、nginx-ingress をデプロイします。

```shell-session
$ helm install -n rp stable/nginx-ingress --set rbac.create=false
```

つぎに、ingress をデプロイして、アプリケーションへのルーティングなどを設定します。

```shell-session
$ kubectl apply -f ingress.yaml
```

デプロイに使った Helm Charts の中身を確認するには、下記のコマンドを使います。

```shell-session
$ helm fetch --untar stable/nginx-ingress
```


