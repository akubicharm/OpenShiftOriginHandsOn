# はじめに
このハンズオンでは、仮想OSに OpenShift Origin のオールインワンの環境を構築し、アプリケーションのデプロイを実施します。

## 概要

1. 準備
- OpenShift Origin v0.5.1 を利用してオールインワンの環境を作成します。
2.  



#### 利用するツール
* vagrant
* virtualbox
* git


# 準備

* VirtualBox
** https://www.virtualbox.org/ からダウンロードし、インストールします。

* Vagrant
https://www.vagrantup.com/  からダウンロードし、インストールします。

* git
https://git-scm.com/ からダウンロードし、インストールします。

## OpenShift Origin の入手
OpenShift Orign は GitHub <http://github.com/openshift/origin> で公開されています。
安定版のソースコードどビルド済みのバイナリは releases <https://github.com/openshift/origin/releases> から取得可能です。

OpenShift Origin v0.5.1(beta3)のソースコードのアーカイブを GitHub から入手します。
```
wget https://github.com/openshift/origin/archive/v0.5.1.zip
unzip v0.5.1.zip
```
origin-0.5.1 というディレクトリが作成されていることを確認します。


最新版を取得する場合は、以下のコマンドで入手します。
```
git clone https://github.com/openshift/origin.git
```


# 仮想環境の起動
## 環境
Vagrant で起動する仮想環境は、次のような構成となっています。
* ユーザとパスワード

|ユーザ|パスワード|権限|
|------|----------|----|
|vagrant|vagrant|sudoでrootユーザ権限でのコマンド実行が可能|

* ポートフォワーディング
Vagrant ファイルの config.vm.define のセクションで設定

|guest|host|
|----|----|
|80|1080|
|443|1443|
|8080|8080|
|8443|8443|

MacOSX Yosemite の場合は、config.vm.define "openshiftdev" の直後に、次の記述を追加します。
```
    config.trigger.after [:provision, :up, :reload] do
      system('echo "
        rdr pass on lo0 inet proto tcp from any to 127.0.0.1 port 80 -> 127.0.0.1 port 1080  
        rdr pass on lo0 inet proto tcp from any to 127.0.0.1 port 443 -> 127.0.0.1 port 1443
        rdr pass on lo0 inet proto tcp from any to 127.0.0.1 port 8080 -> 127.0.0.1 port 8080  
        rdr pass on lo0 inet proto tcp from any to 127.0.0.1 port 8443 -> 127.0.0.1 port 8443
  " | sudo pfctl -ef - > /dev/null 2>&1; echo "==> Fowarding Ports: 80 -> 1080, 443 -> 1443, 8080 -> 8080, 8443 -> 8443 & Enabling pf"')  
    end

    config.trigger.after [:halt, :destroy] do
      system("sudo pfctl -df /etc/pf.conf > /dev/null 2>&1; echo '==> Removing Port Forwarding & Disabling pf'")
    end
```

参考: https://www.danpurdy.co.uk/web-development/osx-yosemite-port-forwarding-for-vagrant/


## 仮想OSの起動とログイン
### 起動
```
vagrant up
```
仮想OSのイメージのダウンロードをするので、時間がかかります。

### ログイン
```
vagrant ssh
```
vagrant ユーザで仮想環境へログインされていることを確認してください。
ログイン後、/data/src/github.com/openshift/origin ディレクトリに展開したソースコードがマウントされていることを確認してください。
```
cd /data/src/github.com/openshift/origin
```


### OpenShift のインストール
```
cd ~/bin
wget https://github.com/openshift/origin/releases/download/v0.5/openshift-origin-v0.5-cecef65-linux-amd64.tar.gz
tar zxvf openshift-origin-v0.5-cecef65-linux-amd64.tar.gz
```


### 環境確認
#### Docker
* 設定ファイルの確認
```
OPTIONS='--insecure-registry=172.30.0.0/16'
```

* デーモンの再起動
```
sudo systemctl restart docker
```

#### firewalld
* 停止
```
sudo systemctl stop firewalld
```


### Docker Image の取得
アプリケーションやビルド作業に利用する Docker Image を入手します。
```
cd /data/src/github.com/openshift/origin/examples/sample-app
./pullimages.sh
```


__ここまでで準備完了です。__

----



# OpenShift の実行

## 動作確認
### OpenShift の起動
```
[vagrant@openshiftdev ~]$ cd ~
[vagrant@openshiftdev ~]$ mkdir logs
[vagrant@openshiftdev ~]$ sudo ./bin/openshift start --public-master=localhost &> logs/openshift.log &
```
※ kubernetes proxy が iptables のルールを操作するので sudo は必須

### クライアント設定
```
[vagrant@openshiftdev ~]$ cd ~ 
[vagrant@openshiftdev ~]$ export CURL_CA_BUNDLE=~/openshift.local.config/master/ca.crt
[vagrant@openshiftdev ~]$ sudo chmod a+rwX /openshift.local.config/master/admin.kubeconfig
[vagrant@openshiftdev ~]$ sudo chmod a+rwX openshift.local.config/master/admin.kubeconfig
```

### test-admin ユーザへの権限割り当て
```
[vagrant@openshiftdev ~]$ cd ~ 
[vagrant@openshiftdev ~]$ osadm policy add-role-to-user view test-admin --config=openshift.local.config/master/admin.kubeconfig
```

### 管理コンソールへのアクセス（コマンドライン）
```
[vagrant@openshiftdev ~]$ osc login --certificate-authority=openshift.local.config/master/ca.crt
OpenShift server [https://localhost:8443]:  ←ここでリターンを入力
Username: test-admin
Authentication required for https://localhost:8443 (openshift)
Password:  ← なんでもOK
Login successful.

Using project "default".
[vagrant@openshiftdev ~]$ osc get services
NAME            LABELS                                    SELECTOR   IP           PORT(S)
kubernetes      component=apiserver,provider=kubernetes   <none>     172.30.0.2   443/TCP
kubernetes-ro   component=apiserver,provider=kubernetes   <none>     172.30.0.1   80/TCP
```
「default」プロジェクトで動作しているサービスが確認できます。
OpenShift Origin(OpenShift v3 のアップストリームのプロジェクト）では、OAuthによるユーザ認証が実装されていないため osc login を実施した時に、/openshift.local.config/master/admin.kubeconfig にユーザが追加されます。

```
[vagrant@openshiftdev ~]$ cat /openshift.local.config/master/admin.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority: ../../home/vagrant/openshift.local.config/master/ca.crt
    server: https://localhost:8443
  name: localhost-8443
contexts:
- context:
    cluster: localhost-8443
    namespace: default
    user: test-admin
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: test-admin
  user:
    token: MmZhOTU4ZjgtZTcyYS00Y2NmLWEyODEtZjUxNzU4ZGMzZGY4
```



### 管理コンソールへのアクセス（ブラウザ）
ブラウザで https://localhost:8443/console にアクセスします。
ログイン画面で test-admin でログインします（パスワードはなんでもOK）
プロジェクト一覧で　「default」 を選択します。

### Docker Registry の作成
~/opensift.local.config/master/openshift-registry.kubeconfig は、openshift コマンドを実行した root ユーザのみ rw の権限を持っているので、すべてのユーザがread可能となるようにファイルのパーミッションを変更します。
```
[vagrant@openshiftdev ~]$ sudo chmod +r openshift.local.config/master/openshift-registry.kubeconfig
[vagrant@openshiftdev ~]$ sudo chmod +r openshift.local.config/master/admin.kubeconfig
```

ローカルにDocker Registryを作成します。Docker Image の名称が変更されている場合があるので、`docker search` コマンドでイメージを検索してください。
```
[vagrant@openshiftdev ~]$ openshift ex registry --create --credentials=openshift.local.config/master/openshift-registry.kubeconfig --config=openshift.local.config/master/admin.kubeconfig --images=openshift/origin-docker-registry:latest
deploymentConfigs/docker-registry
services/docker-registry
```

### Docker Registry のサービスの確認
```
[vagrant@openshiftdev ~]$ osc describe service docker-registry --config=openshift.local.config/master/admin.kubeconfig
Name:			docker-registry
Labels:			docker-registry=default
Selector:		docker-registry=default
IP:			172.30.130.34
Port:			<unnamed>	5000/TCP
Endpoints:		172.17.0.2:5000
Session Affinity:	None
No events.
```
※ Endpoints が <none>  となっている場合は、registry が起動されていません。少し待ってから、再度 osc describe コマンドで Endpoint を確認してください。

docker-registry を再作成する場合は、pod, service, replicataionController, buildConfig, deploymentConfig を一度削除してからやり直します。
`osc get services`、 `osc get pods`、 `osc get bc`、 `osc get rc` で確認後削除します。

```
[vagrant@openshiftdev ~]$ curl `osc get service docker-registry --template="{{ .spec.portalIP }}:{{ with index .spec.ports 0 }}{{ .port }}{{ end }}" --config=openshift.local.config/master/admin.kubeconfig`
"docker-registry server (dev) (v0.9.0)"
```



## プロジェクト管理
### プロジェクトの作成
「test」という名前のプロジェクトを作成します。
```
osc new-project test --display-name="OpenShift 3 Sample" --description="This is an example project to demonstrate OpenShift v3"
```

```
osc get projects
```

プロジェクトを削除する場合は、以下のコマンドを利用します。
でも、今はやらないでください。
```
osc delete projects
```

### 簡単なアプリケーションのデプロイ
「test」プロジェクトにアプリケーションをデプロイします。
```
[vagrant@openshiftdev ~] $ cd  /data/src/github.com/openshift/origin/examples/hello-openshift
[vagrant@openshiftdev hello-openshift]$ osc create -f hello-pod.json  -n test
[vagrant@openshiftdev hello-openshift]$ curl http://localhost:6061
```

### frontend サービスの追加
```
[vagrant@openshiftdev hello-openshift]$ https://raw.githubusercontent.com/akubicharm/OpenShiftOriginHandsOn/master/v0.5/hello-openshift/hello-frontend.json

```

### 複雑なアプリケーションのデプロイ
```
osc process -f application-template-stibuild.json  > ruby.json
osc create -f ruby.json
```
ブラウザで frontend の IP アドレスとポート番号を確認します。例えば、frontend の IP アドレスが 172.30.126.82:5432 の場合、もう一つターミナルを開き、次のコマンドを実行します。
```
vagrant ssh -- -L 9999:172.30.17.4:5432
```
ブラウザで http://localhost:9999 にアクセスします。

```
$ docker pull openshift/origin-haproxy-router
$ sudo chmod +r openshift.local.config/master/openshift-router.kubeconfig
$ openshift ex router --create --credentials=openshift.local.config/master/openshift-router.kubeconfig --config=openshift.local.config/master/admin.kubeconfig --images=openshift/origin-haproxy-router
deploymentConfigs/router
services/router

$ osc project default
$ osc describe dc router
```
