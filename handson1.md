CNDJP #2 ハンズオンチュートリアル
=================================

これは、cndjp第2回勉強会のハンズオン Par1のチュートリアルです。

このチュートリアルでは、簡単なアプリケーションをKubernetesのコマンドラインインターフェースを利用してデプロイします。このとき、manifestファイルを利用したデプロイ方法を練習します。<br>
また、上記のアプリケーションを利用して、ログを収集する操作を試します。


前提条件
--------
（cndjp第2回勉強会の現地で、運営側がお借しする環境を利用される方は「前提条件（一時借し出し環境向け）」を参照下さい。

このチュートリアルは、[cndjp第1回勉強会のハンズオンチュートリアル](https://github.com/oracle-japan/cndjp1/blob/master/handson/handson1.md)の「2. kubectlのセットアップ」以降までを実施済みであることを前提とします。

以降の手順は、ほとんどの操作をコマンドラインツールから実行します。Mac/Linuxの場合はターミナル、Windowsの場合はWindows PowerShell利用するものとして、手順を記述します。


前提条件（一時借し出し環境向け）
--------------------------------
一時借し出し環境では、クラウド上のLinuxサーバーにVirtualboxの仮想マシンを稼働させ、それらの仮想マシンをノードとしてKubernetesクラスターを稼働させています。

    --------------------------
    |      k8s cluster       |  <-- (管理操作)
    --------------------------          |
    |   vm   |   vm  |   vm  |          |
    --------------------------     -----------
    | Hypervisor(VirtualBox) |     | kubectl |
    ------------------------------------------
    |      OS（クラウド上のLinuxマシン）     |
    ------------------------------------------

上の図で、OSより上位のレイヤーについては、cndjp第1回勉強会のハンズオンチュートリアルで作成するものを変わりません。

この環境を利用される方は、講師から以下の情報について、案内を受けて下さい。

- クラウド上のLinuxマシンのIPアドレス
- SSHアクセス用の鍵ファイル

環境へのアクセスには任意のSSHクライアントを利用した環境へのアクセス手順については、「[SSHによる一時借し出し環境へのアクセス手順](xxxx)」を参照下さい。


0. Kubernetesクラスターへのアクセスの確認
-----------------------------------------
Kubernetesクラスターを停止している方は、ここで起動をしておいて下さい。

まず、Kubernetesのインストールスクリプトが配置されたディレクトリをカレントディレクトリにします。

    > cd [インストールスクリプトが配置されたディレクトリ]

一時借し出し環境の場合、以下のようになります。

    > cd ~/kubernetes-vagrant-coreos-cluster/


クラスターが起動しているかどうか確認するため、`vagrant status`を実行します。<br>
以下のような結果が返る場合、クラスターは停止しています。

    > vagrant status
    Current machine states:

    master                    poweroff (virtualbox)
    node-01                   poweroff (virtualbox)
    node-02                   poweroff (virtualbox)

    This environment represents multiple VMs. The VMs are all listed
    above with their current state. For more information about a specific
    VM, run `vagrant status NAME`.

クラスターを起動するには、以下のコマンドを実行します。

    > vagrant up
    Bringing machine 'master' up with 'virtualbox' provider...
    Bringing machine 'node-01' up with 'virtualbox' provider...
    Bringing machine 'node-02' up with 'virtualbox' provider...
    ==> master: Running triggers before up...
    ==> master: 2017-11-15 01:25:08 +0900: setting up Kubernetes master...
    ...（中略）
        node-02: Running: inline script
    ==> node-02: Running triggers after up...
    ==> node-02: Waiting for Kubernetes minion [node-02] to become ready...
    ==> node-02: 2017-11-15 01:45:39 +0900: successfully deployed node-02

クラスターを起動したら、以下のコマンドを実行してクラスターへの疎通を確認します。

・クラスター情報の取得

    > kubectl cluster-info
    Kubernetes master is running at https://172.17.8.101
    KubeDNS is running at https://172.17.8.101/api/v1/namespaces/kube-system/services/kube-dns/proxy

    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

・ノードの一覧の取得

    > kubectl get nodes
    NAME           STATUS                     AGE       VERSION
    172.17.8.101   Ready,SchedulingDisabled   12h       v1.7.11
    172.17.8.102   Ready                      12h       v1.7.11
    172.17.8.103   Ready                      12h       v1.7.11


1. manifestファイルを用いたPodのデプロイ
----------------------------------------
Kubernetesには様々なオブジェクトがありますが、それらをクラスターにデプロイする手順は基本的には同じです。
はじめにオブジェクトの情報を定義したmanifestファイルを作成し、次にそれを入力として`kubectl create`を実行します。

### 1-1. 最初のmanifestファイルを作成する
まず、適当なフォルダを作成し、それをカレントディレクトリにしておきます。

    > mkdir ~/cndjp2
    > cd cndjp2

counter.yamlというファイルを作成します。

・Mac/Linux

    > touch counter.yaml

・Windows

    > New-Item counter.yaml -itemType File

このファイルに、適当なエディタで以下の内容をコピー/ペーストし、保存します。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args: [/bin/sh, -c, 'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
```

### 1-2. Podをデプロイする
manifestファイルからオブジェクトを作成するには、manifestを入力として、`kubectl create`を実行します。

1-1で作成したmanifestからPodをデプロイするには、以下のコマンドを実行します。

    > kubectl create -f counter.yaml
    pod "counter" created

このPodは、現在の時刻を一定間隔で標準出力に書き出すコンテナを内包しています。標準出力に書き出された内容は所定のログファイルとして記録されており、`kubectl logs`コマンドにてその内容を確認できます。

    > kubectl logs counter
    0: Sun Dec 17 14:31:45 UTC 2017
    1: Sun Dec 17 14:31:46 UTC 2017
    2: Sun Dec 17 14:31:47 UTC 2017
    …

例えば、直近の20件の標準出力を確認するには、以下のように実行します。

    > kubectl logs --tail=20 counter

複数のコンテナを動かすようになると、それらの出力を一括して確認する等が必要になってきます。そのようなときに使えるツールとして、Sternがあります。<br>
Sternについては[第1回勉強会のチュートリアル（後半）](https://github.com/oracle-japan/cndjp1/blob/master/handson/handson2.md)にて紹介していますので、参考として下さい。

現在稼働中のPodの一覧を確認するには、以下のコマンドを実行します。

    > kubectl get pods
    NAME      READY     STATUS    RESTARTS   AGE
    counter   1/1       Running   0          1m

kubectl describeを使うと、個別のオブジェクトの詳細情報を確認することができます。

    > kubectl describe pod/counter

### 1-3. Podを削除する
Podを削除するには、以下の`kubectl delete`を使います。

    > kubectl delete -f counter.yaml
    pod "counter" deleted

Podの一覧を表示すると、counterがなくなっていることが確認できます。

    > kubectl get pods
    No resources found.


2. 2系統のログを出力する場合のログの確認方法
--------------------------------------------
2系統以上のログを出力する場合、系統ごとにcidecarコンテナ用意し、同じPodに含めるようにします。そして、各cidecarコンテナは、アプリケーションが出力したログファイルで自信が担当する系統のものを、標準出力に書き出すようにします。

このようなPodの簡単な例を、実際に動かしてみます。まず、以下の内容を"two-files-loggig-counter.yaml"というファイルを作成してコピー/ペーストします。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-files-logging-counter
spec:
  containers:
  - name: count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-1
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/1.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-2
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/2.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

続いて、このPodをデプロイします。

    > kubectl create -f two-files-loggig-counter.yaml
    pod "two-files-logging-counter" created

Podの情報を見てみると、内部で3つのコンテナ（アプリケーション"counter"と2つのsidecarコンテナ）が稼働していることがわかります。

・Podの一覧

    > kubectl get pods
    NAME                        READY     STATUS              RESTARTS   AGE
    two-files-logging-counter   3/3       Running             0          9s

・Podの詳細情報

    > kubectl describe pod/two-files-logging-counter
    …
    Containers:
      count:
        Container ID:       docker://a6a0da9d3625cfb974e5463401f54960b38a8ad345ddd558cfd1bb8a8166250e
        …
      count-log-1:
        Container ID:       docker://389e52693f44642980538744daf79533885d286b69840d0b2f58f1a0b9711836
        …
      count-log-2:
        Container ID:       docker://02702bb46b05414b9d525fd8eada87ceb9a986634e5e43b1f1f138dcbe4e156b
    …

sidecarコンテナが、2系統のログをそれぞれ標準出力に出力していますので、それぞれのコンテナについて、`kubectl logs`で出力内容を確認できます。

    > kubectl logs two-files-logging-counter count-log-1
    0: Sun Dec 17 15:46:31 UTC 2017
    1: Sun Dec 17 15:46:32 UTC 2017
    2: Sun Dec 17 15:46:33 UTC 2017

    > kubectl logs two-files-logging-counter count-log-2
    Sun Dec 17 15:46:31 UTC 2017 INFO 0
    Sun Dec 17 15:46:32 UTC 2017 INFO 1
    Sun Dec 17 15:46:33 UTC 2017 INFO 2


このサンプルで、sidecarコンテナに当たる部分をfluentdにするなどして、外部のログ収集のバックエンドと連携する仕組みを構築することもできます。<br>
実際の例として、公式のマニュアルから、以下を参考にできます。

- https://kubernetes.io/docs/concepts/cluster-administration/logging/
- https://kubernetes.io/docs/tasks/debug-application-cluster/logging-stackdriver/

---

ハンズオンのPart1は以上です。
