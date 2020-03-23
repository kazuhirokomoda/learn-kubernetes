# learn-kubernetes

Kubernetes学習用の個人的なメモ

## References

- [チュートリアル](https://kubernetes.io/ja/docs/tutorials/)
- [用語集](https://kubernetes.io/docs/reference/glossary/?fundamental=true)

## [Kubernetesの基本を学ぶ](https://kubernetes.io/ja/docs/tutorials/kubernetes-basics/)

- Katacodaを使用して、Webブラウザ上の仮想ターミナルでMinikubeを実行

### クラスターの作成

- Kubernetesクラスター
  - マスター：クラスターの管理
  - ノード：Kubernetesクラスターのワーカーマシンとして機能するVMまたは物理マシン
    - プロダクションのトラフィックを処理するKubernetesクラスターには、最低3つのノードが必要
- Minikube
  - ローカルマシン上にVMを作成し、1つのノードのみを含む単純なクラスターをデプロイする軽量なKubernetes実装

```
minikube version
minikube start
kubectl version
kubectl cluster-info
kubectl get nodes
```

### kubectlを使ったDeploymentの作成

kubectlを使用して、Deploymentを作成、管理。Kubernetes APIを使用してクラスターと対話

```
# syntax: kubectl action resource
# kubectl create deployment
kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
kubectl get deployments

kubectl proxy
```

### Podとノードについて

- Podは、1つ以上のアプリケーションコンテナ(Dockerやrktなど)のグループとそれらのコンテナの共有リソースを表すKubernetesの抽象概念。下記を含みうる。
  - 共有ストレージ(ボリューム)
  - ネットワーキング(クラスターに固有のIPアドレス)
  - コンテナのイメージバージョンや使用するポートなどの、各コンテナをどう動かすかに関する情報
- 例：Podには、Node.jsアプリケーションを含むコンテナと、Node.js Webサーバによって公開されるデータを供給する別のコンテナの両方を含めることができます
- Kubernetes上にDeploymentを作成すると、そのDeploymentはその中にコンテナを持つPodを作成します(**コンテナを直接作成するのではなく**)。
- ノード
  - Kubelet
  - レジストリからコンテナイメージを取得し、コンテナを解凍し、アプリケーションを実行することを担当する、Docker、rktのようなコンテナランタイム。
- ```kubectl get``` - リソースの一覧を表示
- ```kubectl describe``` - 単一リソースに関する詳細情報を表示
- ```kubectl logs``` - 単一Pod上の単一コンテナ内のログを表示
- ```kubectl exec``` - 単一Pod上の単一コンテナ内でコマンドを実行

```
kubectl get pods
kubectl describe pods

kubectl proxy

export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME

kubectl logs $POD_NAME
kubectl exec $POD_NAME env
kubectl exec -ti $POD_NAME bash
-----
cat server.js
curl localhost:8080
exit
-----
```

### Serviceを使ったアプリケーションの公開

- Podにはライフサイクルがあります。ワーカーのノードが停止すると、そのノードで実行されているPodも失われます。
- KubernetesのServiceは、Podの論理セットと、それらにアクセスするためのポリシーを定義する抽象概念です。
- Serviceは、すべてのKubernetesオブジェクトのように、YAML(推奨)またはJSONを使って定義されます。
- 各Podには固有のIPアドレスがありますが、それらのIPは、Serviceなしではクラスターの外部に公開されません。
  - Serviceによって、アプリケーションはトラフィックを受信できるようになります。
- ServiceSpecのtype
  - ClusterIP: クラスター内の内部IPでServiceを公開
  - NodePort: ```NodeIP:NodePort```を使用してクラスターの外部からServiceにアクセスできる
  - LoadBalancer: 現在のクラウドに外部ロードバランサを作成し(サポートされている場合)、Serviceに固定の外部IPを割り当てる
  - ExternalName: SpecのexternalNameで指定した名前のCNAMEレコードを返すことによって、任意の名前を使ってServiceを公開
  - ref. https://kubernetes.io/docs/tutorials/services/source-ip/
- Serviceは、一連のPodにトラフィックをルーティングします。
  - Serviceは、アプリケーションに影響を与えることなく、KubernetesでPodが死んだり複製したりすることを可能にする抽象概念です。

Serviceを使用してアプリケーションを公開し、いくつかのラベルを適用してみましょう。

```
kubectl get pods
kubectl get services

kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
kubectl describe services/kubernetes-bootcamp

export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT

curl $(minikube ip):$NODE_PORT

kubectl describe deployment
kubectl get pods -l run=kubernetes-bootcamp
kubectl get services -l run=kubernetes-bootcamp
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME

kubectl label pod $POD_NAME app=v1
kubectl describe pods $POD_NAME
kubectl get pods -l app=v1

kubectl delete service -l run=kubernetes-bootcamp
kubectl get services

# Expected: not reachable from outside of the cluster.
# curl: (7) Failed to connect to 172.17.0.29 port 32245: Connection refused
curl $(minikube ip):$NODE_PORT

# however accessible from inside of the cluster
kubectl exec -ti $POD_NAME curl localhost:8080
```

### アプリケーションの複数インスタンスを実行

- スケーリングは、Deploymentのレプリカの数を変更することによって実現可能です。

```
kubectl get deployments

# To see the ReplicaSet created by the Deployment
kubectl get rs

# Scale
kubectl scale deployments/kubernetes-bootcamp --replicas=4

kubectl get pods -o wide
kubectl describe deployments/kubernetes-bootcamp

# To find out the exposed IP and Port
kubectl describe services/kubernetes-bootcamp

export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT

curl $(minikube ip):$NODE_PORT

# Scale down
kubectl scale deployments/kubernetes-bootcamp --replicas=2

kubectl get deployments
kubectl get pods -o wide
```

### ローリングアップデートの実行

- Deploymentがパブリックに公開されている場合、Serviceはアップデート中に利用可能なPodにのみトラフィックを負荷分散します。

```
kubectl get deployments
kubectl get pods
kubectl describe pods

kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
kubectl get pods

kubectl describe services/kubernetes-bootcamp
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT

curl $(minikube ip):$NODE_PORT
...

kubectl rollout status deployments/kubernetes-bootcamp
kubectl describe pods

kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10
kubectl get deployments
kubectl get pods
kubectl describe pods

kubectl rollout undo deployments/kubernetes-bootcamp
kubectl get pods
kubectl describe pods
```


## [Hello Minikube](https://kubernetes.io/ja/docs/tutorials/hello-minikube/)

### Deploymentの作成

```
kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
kubectl get deployments
kubectl get pods
kubectl get events
kubectl config view
```

### Serviceの作成

- コンテナをKubernetesの仮想ネットワークの外部からアクセスするためには、KubernetesのServiceとしてポッドを公開する必要があります

```
kubectl expose deployment hello-node --type=LoadBalancer --port=8080
kubectl get services

minikube service hello-node
```

### アドオンの有効化

```
minikube addons list
minikube addons enable logviewer
kubectl get pod,svc -n kube-system
minikube addons disable logviewer
```

### クリーンアップ

```
kubectl delete service hello-node
kubectl delete deployment hello-node
```
