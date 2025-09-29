# 演習3　Kubernetes セキュリティの実践

- [演習3　Kubernetes セキュリティの実践](#演習3kubernetes-セキュリティの実践)
  - [1. Kubernetes の認証認可](#1-kubernetes-の認証認可)
    - [1.1 認証認可の仕組み](#11-認証認可の仕組み)
    - [1.2 Certificate Signing Request (CSR) による新しいユーザーの作成](#12-certificate-signing-request-csr-による新しいユーザーの作成)
    - [1.3 サービスアカウントと RBAC による Pod の権限制御](#13-サービスアカウントと-rbac-による-pod-の権限制御)
    - [1.4 匿名アクセス（Anonymous Access）の検証](#14-匿名アクセスanonymous-accessの検証)
  - [まとめ](#まとめ)
    - [学習した重要なポイント](#学習した重要なポイント)
  - [環境のクリーンアップ](#環境のクリーンアップ)
  - [参考](#参考)

本演習では、Kubernetes におけるセキュリティ対策について、実際の操作を通じて理解を深めます。

演習環境にアクセスし、必要に応じて作業ディレクトリを作成してください。

```bash
mkdir ch03
cd ch03
```

## 1. Kubernetes の認証認可

Kubernetes の認証認可の仕組みを理解し、ユーザー権限の最小化やアプリケーションへの権限付与を体験します。

### 1.1 認証認可の仕組み

Kubernetes への認証情報は kubeconfig ファイルに保存され、`kubectl` コマンドの実行時に自動的に適用されます。<br/>
`kubectl config` コマンドを使い、認証情報を表示してみましょう。


```bash
kubectl config view --minify
```

<details><summary>出力結果</summary>

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.30.1.2:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```

</details>

kubeconfig ファイルは以下の3つの情報で構成されています。

- **clusters**: 接続先のクラスタ情報
- **users**: 接続に使用するユーザー情報
- **contexts**: クラスタとユーザーの組み合わせ

今回はそれぞれ1つずつしか定義されていませんが、接続先やユーザーを切り替える必要がある場合は、複数の設定を用意します。<br/>
現在使用している Context は次のように確認します。

```bash
kubectl config current-context
```

<details><summary>出力結果</summary>

```bash
kubernetes-admin@kubernetes
```

</details>

`kubectl config view` では証明書や鍵データがマスクされていましたが、ファイルを直接表示すればこれらの情報も確認できます。

```bash
cat $HOME/.kube/config
```

<details><summary>出力結果</summary>

```bash
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FFURS0tLS0tCk1JSUR......5NEJReXg5Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://172.30.1.2:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRAVJUSUZJQ0FURS0tLS0StCk1JSURL......S0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQQpNSUlFb2dJQkFBS0NBUUVB......RU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
```

</details>

kubeconfig に記載されいてるユーザー名 `kubernetes-admin` は、設定ファイル上の識別子であり、実際の認証ユーザー名ではありません。

認証ユーザーの確認は `kubectl auth` コマンドで行います。

```bash
kubectl auth whoami
```

<details><summary>出力結果</summary>

```bash
ATTRIBUTE                                           VALUE
Username                                            kubernetes-admin
Groups                                              [kubeadm:cluster-admins system:authenticated]
Extra: authentication.kubernetes.io/credential-id   [X509SHA256=145418d6f3df50......5f15b5f58d335e26186f3a]
```

</details>

ユーザー名は `kubernetes-admin` で、`kubeadm:cluster-admins` `system:authenticated` の2つのグループに所属していることがわかります。<br/>
今回は kubeconfig と同じユーザー名でしたが、kubeconfig に記載されている方のユーザー名は変更しても影響ありません。

このユーザーが、Kubernetes 上でどのような権限を持つか見てみましょう。

```bash
kubectl auth can-i --list
```


初めに、`kubectl auth` コマンドを使用して現在のユーザーや権限を確認しましょう。

```bash
kubectl auth can-i --list
```

<details><summary>出力結果</summary>

```bash
Resources                                       Non-Resource URLs   Resource Names   Verbs
*.*                                             []                  []               [*]
                                                [*]                 []               [*]
selfsubjectreviews.authentication.k8s.io        []                  []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                  []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []               [create]
                                                [/api/*]            []               [get]
                                                [/api]              []               [get]
                                                [/apis/*]           []               [get]
                                                [/apis]             []               [get]
                                                [/healthz]          []               [get]
                                                [/healthz]          []               [get]
                                                [/livez]            []               [get]
                                                [/livez]            []               [get]
                                                [/openapi/*]        []               [get]
                                                [/openapi]          []               [get]
                                                [/readyz]           []               [get]
                                                [/readyz]           []               [get]
                                                [/version/]         []               [get]
                                                [/version/]         []               [get]
                                                [/version]          []               [get]
                                                [/version]          []               [get]
```

</details>

結果の1-2行目から、`kubernetes-admin` がすべてのリソースに対してすべての操作を行えることがわかります。<br/>
この結果は namespaced であるため、現在は default namespace の情報が表示されています。すべての namespace の情報を知りたい場合、namespace の数だけコマンドを実行する必要があります。

特定アクションの権限を持つかどうかだけ知りたい場合は、次のようにコマンド実行してください。

```bash
kubectl auth can-i create pods
kubectl auth can-i delete nodes
kubectl auth can-i get secrets --all-namespaces
```

<details><summary>出力結果</summary>

```bash
yes
Warning: resource 'nodes' is not namespace scoped

yes
yes
```

</details>

現在のユーザーは cluster-admin 権限を持っているため、すべての操作が可能です。<br/>
この権限付与はどのように行われているのでしょうか？

```bash
kubectl get clusterrolebindings kubeadm:cluster-admins -o yaml
```

<details><summary>出力結果</summary>

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: "2025-09-19T19:16:58Z"
  name: kubeadm:cluster-admins
  resourceVersion: "244"
  uid: a30d700f-e31c-4de4-a5d4-c2e741b2f04b
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: kubeadm:cluster-admins
```

</details>

上記 ClusterRoleBinding リソースを見ると、`kubeadm:cluster-admins` グループに cluster-admin 権限の ClusterRole をバインドしています。<br/>
この設定により、`kubeadm:cluster-admins` グループに所属するユーザーに cluster-admin 権限が付与されます。

### 1.2 Certificate Signing Request (CSR) による新しいユーザーの作成

Kubernetes では、Certificate Signing Request（CSR）を使用して新しいユーザーの証明書を作成できます。<br/>
実際に新しいユーザー（developer）を作成し、限定的な権限を付与してみましょう。

まずは、ユーザー用の CSR を作成してクラスタに登録します。

```bash
# 新しいユーザー用の秘密鍵を作成
openssl genrsa -out developer.key 2048

# Certificate Signing Request (CSR) を作成
openssl req -new -key developer.key -out developer.csr -subj "/CN=developer/O=development"

# Kubernetes CSR リソースを作成
cat <<EOF > developer-csr.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: developer-csr
spec:
  request: $(cat developer.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

kubectl apply -f developer-csr.yaml
```

CSR の状態を確認してみましょう。

```bash
# CSR の状態を確認
kubectl get csr
```

<details><summary>出力結果</summary>

```bash
NAME            AGE   SIGNERNAME                                    REQUESTOR                  REQUESTEDDURATION   CONDITION
csr-k2rmb       9d    kubernetes.io/kube-apiserver-client-kubelet   system:node:controlplane   <none>              Approved,Issued
csr-sv96v       9d    kubernetes.io/kube-apiserver-client-kubelet   system:bootstrap:kya3z0    <none>              Approved,Issued
developer-csr   24s   kubernetes.io/kube-apiserver-client           kubernetes-admin           <none>              Pending
```

</details>

作成された `developer-csr` は状態が `Pending` となっています。次のステップに進むには承認操作が必要です。

**重要**: CSRの承認には `certificates.k8s.io/certificatesigningrequests/approval` リソースに対する `update` 権限が必要です。本演習では cluster-admin 権限を持つユーザーで実行しているため承認可能ですが、実際の運用では承認権限を持つ限定されたユーザーのみがこの操作を行うべきです。

`kubectl certificate` コマンドを使用し、CSR を承認します。

```bash
kubectl certificate approve developer-csr
```

<details><summary>出力結果</summary>

```bash
certificatesigningrequest.certificates.k8s.io/developer-csr approved
```

</details>

承認後の CSR の状態を確認してみます。

```bash
kubectl get csr developer-csr
```

<details><summary>出力結果</summary>

```bash
NAME            AGE     SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
developer-csr   4m12s   kubernetes.io/kube-apiserver-client   kubernetes-admin   <none>              Approved,Issued
```

</details>

状態が `Approved,Issued` となりました。これでユーザー作成は完了です。

続いて、作成したユーザー `developer` がどのような権限を持つか見てみます。

```bash
kubectl auth can-i --list --as=developer
```

<details><summary>出力結果</summary>

```bash
Resources                                       Non-Resource URLs   Resource Names   Verbs
selfsubjectreviews.authentication.k8s.io        []                  []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                  []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []               [create]
                                                [/api/*]            []               [get]
                                                [/api]              []               [get]
                                                [/apis/*]           []               [get]
                                                [/apis]             []               [get]
                                                [/healthz]          []               [get]
                                                [/healthz]          []               [get]
                                                [/livez]            []               [get]
                                                [/livez]            []               [get]
                                                [/openapi/*]        []               [get]
                                                [/openapi]          []               [get]
                                                [/readyz]           []               [get]
                                                [/readyz]           []               [get]
                                                [/version/]         []               [get]
                                                [/version/]         []               [get]
                                                [/version]          []               [get]
                                                [/version]          []               [get]
```

</details>

現在、`developer` ユーザーには特別な権限が付与されていないため、基本的な API 情報の取得のみが可能です。

`developer` ユーザーに対して、特定の namespace での Pod の操作権限を付与しましょう。

```bash
# development namespace を作成
kubectl create namespace development

# Role を作成（Pod の読み取り・作成・削除権限）
cat <<EOF > developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-manager
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "delete"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
EOF

kubectl apply -f developer-role.yaml

# RoleBinding を作成（developer ユーザーに Role を紐付け）
cat <<EOF > developer-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-manager-binding
  namespace: development
subjects:
- kind: User
  name: developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f developer-rolebinding.yaml
```

development namespace において、Pod の操作を可能にする Role の作成と、`developer` ユーザーへのロールのバインドを行いました。

権限が正しく付与されているか検証します。

```bash
# development namespace での Pod 操作権限を確認
kubectl auth can-i create pods --namespace=development --as=developer
kubectl auth can-i delete pods --namespace=development --as=developer
kubectl auth can-i get secrets --namespace=development --as=developer

# default namespace での権限を確認
kubectl auth can-i create pods --namespace=default --as=developer
```

<details><summary>出力結果</summary>

```bash
yes
yes
no
no
```

</details>

想定通り、development namespace 内で Pod リソースのみ操作できるようになっています。

このユーザーの権限で Kubernetes の操作を行う場合、kubeconfig へのユーザー情報の追加と、接続先クラスタとの関連付け設定が必要です。

```bash
# 発行された証明書を取得
kubectl get csr developer-csr -o jsonpath='{.status.certificate}' | base64 -d > developer.crt

# kubeconfig にユーザー情報を追加
kubectl config set-credentials developer \
  --client-certificate=developer.crt \
  --client-key=developer.key

# 接続先クラスタとの関連付け
CURRENT_CLUSTER=$(kubectl config view --minify -o jsonpath='{.clusters[0].name}')
kubectl config set-context developer-context \
  --cluster=$CURRENT_CLUSTER \
  --user=developer
```

コンテキストを切り替えれば、`developer` ユーザーの権限でクラスタを操作できるようになります。

```bash
kubectl auth whoami --context developer-context
```

<details><summary>出力結果</summary>

```bash
ATTRIBUTE                                           VALUE
Username                                            developer
Groups                                              [development system:authenticated]
Extra: authentication.kubernetes.io/credential-id   [X509SHA256=a5600d843422......13fbd17f8cf6f1cb53bc9b241698a]
```

</details>

```bash
kubectl run test --image ubuntu -n development --context developer-context -- sleep 3600
```

<details><summary>出力結果</summary>

```bash
pod/test created
```

</details>

```bash
kubectl run test --image ubuntu --context developer-context -- sleep 3600
```

<details><summary>出力結果</summary>

```bash
Error from server (Forbidden): pods is forbidden: User "developer" cannot create resource "pods" in API group "" in the namespace "default"
```

</details>

Kubernetes はID管理機能を持たないため、実践的には外部の IdP と連携して権限管理を行うことになると思います。

IdP を使わず作成した `developer` ユーザーは、証明書と鍵情報のみでクラスタにアクセスできます。<br/>
これはログイン処理を伴わないため攻撃者にも有利な仕組みで、クラスタに侵入した後の永続化の手法として使われることがあるため注意が必要です。

### 1.3 サービスアカウントと RBAC による Pod の権限制御

Pod 内から Kubernetes API にアクセスする際は、サービスアカウント（ServiceAccount）を使用します。<br/>
サービスアカウントに適切な権限を付与し、Pod からの API アクセスを制御してみましょう。

1.2 で作成した development namespace 内に、サービスアカウント `pod-reader` を作成して権限を付与します。

```bash
kubectl create serviceaccount pod-reader -n development

# サービスアカウント用の Role を作成
cat <<EOF > pod-reader-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF

kubectl apply -f pod-reader-role.yaml

# RoleBinding を作成
cat <<EOF > pod-reader-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: development
subjects:
- kind: ServiceAccount
  name: pod-reader
  namespace: development
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f pod-reader-rolebinding.yaml
```

次に、pod-reader サービスアカウントを使用する Pod を作成します。

```bash
cat <<EOF > api-access-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-access-pod
  namespace: development
spec:
  serviceAccountName: pod-reader
  containers:
  - name: alpine
    image: alpine:latest
    command: ["/bin/sh"]
    args: ["-c", "apk add --no-cache curl && sleep 3600"]
EOF

kubectl apply -f api-access-pod.yaml
```

作成した Pod に `exec` を実行し、Pod 内でサービスアカウントの権限を使った Kubernetes へのアクセスを試みます。

```bash
kubectl exec -it -n development api-access-pod -- bash
```

Kubernetes への認証にはサービスアカウントトークンとCA証明書が必要です。<br/>
これらは通常、`/var/run/secrets/kubernetes.io/serviceaccount/` 以下に配置されます。

**注意**: Kubernetes 1.24以降、サービスアカウントトークンは自動的に期限付きのJWTトークンとして作成されます。従来の永続的なトークンとは異なり、定期的に更新される仕組みになっています。

```bash
ls /var/run/secrets/kubernetes.io/serviceaccount/
```

<details><summary>出力結果</summary>

```bash
ca.crt     namespace  token
```

</details>

Pod 内には `kubectl` がインストールされていないため、kube-apiserver に直接リクエストを送ります。

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

echo "=== Pod リストの取得（許可される）==="
curl -s --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc.cluster.local/api/v1/namespaces/$NAMESPACE/pods
```

<details><summary>出力結果</summary>

```bash
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "7538"
  },
  "items": [
    {
      "metadata": {
        "name": "api-access-pod",
        "namespace": "development",
        "uid": "f10c9a30-e92e-494c-a560-744c46a7cbda",
        "resourceVersion": "6814",
        "generation": 1,
        "creationTimestamp": "2025-09-29T16:27:06Z",
        "annotations": {
          "cni.projectcalico.org/containerID": "c7f45724b54cb6863bdfea0a729be0647290f765e6342f8a4136927e8d023fe4",
          "cni.projectcalico.org/podIP": "192.168.1.5/32",
          "cni.projectcalico.org/podIPs": "192.168.1.5/32",
          "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"name\":\"api-access-pod\",\"namespace\":\"development\"},\"spec\":{\"containers\":[{\"args\":[\"-c\",\"apk add --no-cache curl \\u0026\\u0026 sleep 3600\"],\"command\":[\"/bin/sh\"],\"image\":\"alpine:latest\",\"name\":\"alpine\"}],\"serviceAccountName\":\"pod-reader\"}}\n"
        },
        "managedFields": [
          {
            "manager": "kubectl-client-side-apply",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2025-09-29T16:27:06Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {
              "f:metadata": {
                "f:annotations": {
                  ".": {},
                  "f:kubectl.kubernetes.io/last-applied-configuration": {}
                }
              },
              "f:spec": {
                "f:containers": {
                  "k:{\"name\":\"alpine\"}": {
                    ".": {},
                    "f:args": {},
                    "f:command": {},
                    "f:image": {},
                    "f:imagePullPolicy": {},
                    "f:name": {},
                    "f:resources": {},
                    "f:terminationMessagePath": {},
                    "f:terminationMessagePolicy": {}
                  }
                },
                "f:dnsPolicy": {},
                "f:enableServiceLinks": {},
                "f:restartPolicy": {},
                "f:schedulerName": {},
                "f:securityContext": {},
                "f:serviceAccount": {},
                "f:serviceAccountName": {},
                "f:terminationGracePeriodSeconds": {}
              }
            }
          },
          {
            "manager": "calico",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2025-09-29T16:27:07Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {
              "f:metadata": {
                "f:annotations": {
                  "f:cni.projectcalico.org/containerID": {},
                  "f:cni.projectcalico.org/podIP": {},
                  "f:cni.projectcalico.org/podIPs": {}
                }
              }
            },
            "subresource": "status"
          },
          {
            "manager": "kubelet",
            "operation": "Update",
            "apiVersion": "v1",
...
```

</details>

development namespace 内の Pod の取得に成功しました。

許可されていない操作も検証してみます。

```bash
curl -s --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc.cluster.local/api/v1/namespaces/$NAMESPACE/configmaps
```

<details><summary>出力結果</summary>

```bash
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "configmaps is forbidden: User \"system:serviceaccount:development:pod-reader\" cannot list resource \"configmaps\" in API group \"\" in the namespace \"development\"",
  "reason": "Forbidden",
  "details": {
    "kind": "configmaps"
  },
  "code": 403
}
```

</details>

権限不足により、ConfigMap の取得に失敗しました。

さらに権限を追加して、ConfigMap も読み取れるようにしてみましょう。

Pod から `exec` で抜けて、以下のコマンドを実行してください。

```bash
# Role を更新して ConfigMap の読み取り権限を追加
cat <<EOF > pod-reader-role-updated.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list"]
EOF

kubectl apply -f pod-reader-role-updated.yaml

# テスト用の ConfigMap を作成
kubectl create configmap test-config --from-literal=key1=value1 -n development
```

サービスアカウントに紐付けたロールの修正と、ConfigMap の作成が完了したら、再び kube-apiserver にリクエストを送ります。

```bash
kubectl exec -it api-access-pod -n development -- sh -c '
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

echo "=== ConfigMap リストの取得（今度は許可される）==="
curl -s --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc.cluster.local/api/v1/namespaces/$NAMESPACE/configmaps
'
```

<details><summary>出力結果</summary>

```bash
{
  "kind": "ConfigMapList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "4047"
  },
  "items": [
    {
      "metadata": {
        "name": "kube-root-ca.crt",
        "namespace": "development",
        "uid": "c18428af-45e9-49dd-b0fc-daaff82fb0e7",
        "resourceVersion": "3881",
        "creationTimestamp": "2025-09-29T16:47:22Z",
        "annotations": {
          "kubernetes.io/description": "Contains a CA bundle that can be used to verify the kube-apiserver when using internal endpoints such as the internal service IP or kubernetes.default.svc. No other usage is guaranteed across distributions of Kubernetes clusters."
        },
        "managedFields": [
          {
            "manager": "kube-controller-manager",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2025-09-29T16:47:22Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {
              "f:data": {
                ".": {},
                "f:ca.crt": {}
              },
              "f:metadata": {
                "f:annotations": {
                  ".": {},
                  "f:kubernetes.io/description": {}
                }
              }
            }
          }
        ]
      },
      "data": {
        "ca.crt": "-----BEGIN CERTIFICATE-----\nMIIDBTCCAe2gAwIBAgIIUC/iLTNbSfowDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE\nAxMK......Wt4ZP/ZKb38OAA1LnSXxCK5VFyeflC9czmEwZ\nyIoD4y4BQyx9\n-----END CERTIFICATE-----\n"
      }
    },
    {
      "metadata": {
        "name": "test-config",
        "namespace": "development",
        "uid": "e26ce86c-2235-43e0-b0f6-e3b2a1751088",
        "resourceVersion": "3969",
        "creationTimestamp": "2025-09-29T16:48:14Z",
        "managedFields": [
          {
            "manager": "kubectl-create",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2025-09-29T16:48:14Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {
              "f:data": {
                ".": {},
                "f:key1": {}
              }
            }
          }
        ]
      },
      "data": {
        "key1": "value1"
      }
    }
  ]
}
```

</details>

今度は ConfigMap の取得に成功しました。

このように、必要最低限のリソースに絞り、Pod に Kubernetes を操作する権限を付与することができます。

### 1.4 匿名アクセス（Anonymous Access）の検証

Kubernetes では、認証情報なしでAPIサーバーにアクセスした場合、匿名ユーザー（`system:anonymous`）として扱われます。<br/>
匿名アクセスの動作を確認してみましょう。

```bash
kubectl auth can-i --list --as=system:anonymous
```

<details><summary>出力結果</summary>

```bash
Error from server (Forbidden): selfsubjectrulesreviews.authorization.k8s.io is forbidden: User "system:anonymous" cannot create resource "selfsubjectrulesreviews" in API group "authorization.k8s.io" at the cluster scope
```

</details>

実行できる操作の一覧を表示しようとしましたが、権限不足により表示できませんでした。

このままだと不便なので、**演習目的でのみ**匿名ユーザーに権限付与して操作一覧を表示できるようにします。

**⚠️ 警告**: 以下の操作は実際の本番環境では絶対に行わないでください。匿名アクセスに権限を付与することは重大なセキュリティリスクとなります。

```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  namespace: default
  name: bind-basic-user-to-anonymous
subjects:
- kind: User
  name: system:anonymous
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:basic-user
  apiGroup: rbac.authorization.k8s.io
EOF
```

```bash
kubectl auth can-i --list --as=system:anonymous
```

<details><summary>出力結果</summary>

```bash
Resources                                       Non-Resource URLs   Resource Names   Verbs
selfsubjectreviews.authentication.k8s.io        []                  []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                  []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []               [create]
                                                [/healthz]          []               [get]
                                                [/livez]            []               [get]
                                                [/readyz]           []               [get]
                                                [/version/]         []               [get]
                                                [/version]          []               [get]
```

</details>

今度は表示されました。
しかし今付与した権限以外、匿名ユーザーはリソース操作の権限を何も持っていません。

匿名ユーザーはクラスタの基本的な Health とバージョン情報の取得のみが可能です。

```bash
# API サーバーの URL を取得
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
echo "API Server: $APISERVER"

# 匿名アクセスでバージョン情報を取得
curl -k $APISERVER/version
```

<details><summary>出力結果</summary>

```json
API Server: https://172.30.1.2:6443
{
  "major": "1",
  "minor": "33",
  "emulationMajor": "1",
  "emulationMinor": "33",
  "minCompatibilityMajor": "1",
  "minCompatibilityMinor": "32",
  "gitVersion": "v1.33.2",
  "gitCommit": "a57b6f7709f6c2722b92f07b8b4c48210a51fc40",
  "gitTreeState": "clean",
  "buildDate": "2025-06-17T18:31:32Z",
  "goVersion": "go1.24.4",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

</details>

バージョン情報の取得には成功しましたが、Pod 情報の取得は失敗します。

```bash
curl -k $APISERVER/api/v1/namespaces/default/pods
```

<details><summary>出力結果</summary>

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods is forbidden: User \"system:anonymous\" cannot list resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```

</details>

匿名ユーザーに Pod 作成の権限を付与するとどうなるでしょうか？

```bash
# Pod 作成のための Role
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: anonymous-pod-creator
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create"]
EOF

# 匿名ユーザーに Role をバインド
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: default
  name: bind-anonymous-pod-creator
subjects:
- kind: User
  name: system:anonymous
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: anonymous-pod-creator
  apiGroup: rbac.authorization.k8s.io
EOF
```

権限の付与後、再び匿名ユーザーの実行できる操作一覧を表示します。

```bash
kubectl auth can-i --list --as=system:anonymous
```

<details><summary>出力結果</summary>

```bash
Resources                                       Non-Resource URLs   Resource Names   Verbs
pods                                            []                  []               [create]
selfsubjectreviews.authentication.k8s.io        []                  []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                  []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []               [create]
                                                [/healthz]          []               [get]
                                                [/livez]            []               [get]
                                                [/readyz]           []               [get]
                                                [/version/]         []               [get]
                                                [/version]          []               [get]
```

</details>

Pod を作成する権限が付与されたことを確認できました。

この状態で、Pod の作成を認証なしでリクエストしてみます。

```bash
curl -k -X POST \
  -H "Content-Type: application/json" \
  $APISERVER/api/v1/namespaces/default/pods \
  --data @- <<EOF
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "anonymous-test-pod",
    "namespace": "default"
  },
  "spec": {
    "containers": [
      {
        "name": "test-container",
        "image": "nginx",
        "ports": [
          {
            "containerPort": 80
          }
        ]
      }
    ]
  }
}
EOF
```

<details><summary>出力結果</summary>

```bash
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "anonymous-test-pod",
    "namespace": "default",
    "uid": "08fd594b-aa5c-4e7c-ade7-bbf3670f01d3",
    "resourceVersion": "6142",
    "generation": 1,
    "creationTimestamp": "2025-09-29T15:18:11Z",
    "managedFields": [
      {
        "manager": "curl",
        "operation": "Update",
        "apiVersion": "v1",
        "time": "2025-09-29T15:18:11Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {
          "f:spec": {
            "f:containers": {
              "k:{\"name\":\"test-container\"}": {
                ".": {},
                "f:image": {},
                "f:imagePullPolicy": {},
                "f:name": {},
                "f:ports": {
                  ".": {},
                  "k:{\"containerPort\":80,\"protocol\":\"TCP\"}": {
                    ".": {},
                    "f:containerPort": {},
                    "f:protocol": {}
                  }
                },
                "f:resources": {},
                "f:terminationMessagePath": {},
                "f:terminationMessagePolicy": {}
              }
            },
            "f:dnsPolicy": {},
            "f:enableServiceLinks": {},
            "f:restartPolicy": {},
            "f:schedulerName": {},
            "f:securityContext": {},
            "f:terminationGracePeriodSeconds": {}
          }
        }
      }
    ]
  },
  "spec": {
    "volumes": [
      {
        "name": "kube-api-access-ghb9g",
        "projected": {
          "sources": [
            {
              "serviceAccountToken": {
                "expirationSeconds": 3607,
                "path": "token"
              }
            },
            {
              "configMap": {
                "name": "kube-root-ca.crt",
                "items": [
                  {
                    "key": "ca.crt",
                    "path": "ca.crt"
                  }
                ]
              }
            },
            {
              "downwardAPI": {
                "items": [
                  {
                    "path": "namespace",
                    "fieldRef": {
                      "apiVersion": "v1",
                      "fieldPath": "metadata.namespace"
                    }
                  }
                ]
              }
            }
          ],
          "defaultMode": 420
        }
      }
    ],
    "containers": [
      {
        "name": "test-container",
        "image": "nginx",
        "ports": [
          {
            "containerPort": 80,
            "protocol": "TCP"
          }
        ],
        "resources": {},
        "volumeMounts": [
          {
            "name": "kube-api-access-ghb9g",
            "readOnly": true,
            "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
          }
        ],
        "terminationMessagePath": "/dev/termination-log",
        "terminationMessagePolicy": "File",
        "imagePullPolicy": "Always"
      }
    ],
    "restartPolicy": "Always",
    "terminationGracePeriodSeconds": 30,
    "dnsPolicy": "ClusterFirst",
    "serviceAccountName": "default",
    "serviceAccount": "default",
    "securityContext": {},
    "schedulerName": "default-scheduler",
    "tolerations": [
      {
        "key": "node.kubernetes.io/not-ready",
        "operator": "Exists",
        "effect": "NoExecute",
        "tolerationSeconds": 300
      },
      {
        "key": "node.kubernetes.io/unreachable",
        "operator": "Exists",
        "effect": "NoExecute",
        "tolerationSeconds": 300
      }
    ],
    "priority": 0,
    "enableServiceLinks": true,
    "preemptionPolicy": "PreemptLowerPriority"
  },
  "status": {
    "phase": "Pending",
    "qosClass": "BestEffort"
  }
}
```

</details>

リクエストに成功しました。
実際に Pod が作成されているか確認してみます。

```bash
kubectl get po
```

<details><summary>出力結果</summary>

```bash
NAME                 READY   STATUS    RESTARTS   AGE
anonymous-test-pod   1/1     Running   0          61s
```

</details>

Pod が作成されていました。

現在このクラスタは、kube-apiserver のエンドポイントにさえアクセスできれば、誰でも認証なしで Pod を作成できる状態となっています。

これが非常に危険な状態であることは理解できると思います。<br/>
攻撃者は公開されている API を使って、簡単に Kubernetes クラスタに侵入することができてしまいます。<br/>
匿名ユーザー/グループへの権限付与が必要となるケースはないので、この操作は絶対に行うべきではありません。

以下のコマンドで、作成したリソースをすべて削除してください。

```bash
kubectl delete po anonymous-test-pod
kubectl delete rolebinding bind-anonymous-pod-creator
kubectl delete role anonymous-pod-creator
kubectl delete clusterrolebinding bind-basic-user-to-anonymous
```

## まとめ

本演習では、Kubernetes における認証認可の仕組みについて実践的に学習しました。

### 学習した重要なポイント

1. **認証認可の基本構造**
   - kubeconfig ファイルの構成（clusters、users、contexts）
   - 証明書ベースの認証メカニズム
   - `kubectl auth` コマンドによる権限確認手法

2. **ユーザー管理とRBAC**
   - Certificate Signing Request（CSR）を使用した新しいユーザーの作成
   - Role と RoleBinding による名前空間レベルの権限制御
   - 最小権限の原則に基づく段階的な権限付与

3. **サービスアカウントによるPod認証**
   - Pod 内からのKubernetes API アクセス方法
   - サービスアカウントトークンとCA証明書の活用
   - アプリケーションレベルでの権限制御の実装

4. **セキュリティリスクの理解**
   - 匿名アクセスの制限された権限範囲
   - 不適切な権限付与が引き起こすセキュリティリスク
   - 証明書ベース認証の永続化リスク

Kubernetes の認証認可は、セキュリティの要となる重要な機能です。適切に設定・運用することで、安全で効率的なクラスタ運用を実現できます。

## 環境のクリーンアップ

演習で作成したリソースを削除します。

```bash
# Namespace の削除
kubectl delete namespace development --ignore-not-found

# CSR の削除
kubectl delete csr developer-csr --ignore-not-found

# kubeconfig からユーザーとコンテキストを削除
kubectl config delete-user developer 2>/dev/null || true
kubectl config delete-context developer-context 2>/dev/null || true
```

## 参考

- Kubernetes Documentation
  - [Kubernetes Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
  - [Kubernetes Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
  - [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
  - [Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)
  - [Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
  - [Managing Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
