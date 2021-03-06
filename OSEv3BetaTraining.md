# はじめに
このハンズオンでは、仮想OSに OpenShift Origin のオールインワンの環境を構築し、アプリケーションのデプロイを実施します。

## 概要

### 作業手順

1. 環境構築
2. ユーザとプロジェクトの作成
3. アプリケーションのデプロイ 



### 利用するツール
* vagrant
* virtualbox
* git

---
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

OpenShift Origin v0.5.x(beta3)のソースコードのアーカイブを GitHub から入手します。
2015年6月1日現在、v0.5.3 で動作確認済みです。
```
wget https://github.com/openshift/origin/archive/v0.5.x.zip
unzip v0.5.x.zip
```
origin-0.5.x というディレクトリが作成されていることを確認します。

![OpenShiftOrigin Release](images/github.top.png)
![OpenShiftOrigin Release](images/github.releases.png)


最新版を取得する場合は、以下のコマンドで入手します。
```
git clone https://github.com/openshift/origin.git
```

---
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
* 設定ファイルの確認 /etc/sysconfig/docker
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
[vagrant@openshiftdev ~]$ osadm policy add-role-to-user admin test-admin --config=openshift.local.config/master/admin.kubeconfig
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
![login](images/origin_login_with_test-admin-1.png)
![login](images/origin_login_with_test-admin-2.png)
![login](images/origin_login_with_test-admin-3.png)

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
※ Endpoints が `<none>`  となっている場合は、registry が起動されていません。少し待ってから、再度 osc describe コマンドで Endpoint を確認してください。

docker-registry を再作成する場合は、pod, service, replicataionController, buildConfig, deploymentConfig を一度削除してからやり直します。
`osc get services`、 `osc get pods`、 `osc get bc`、 `osc get rc` で確認後削除します。

```
[vagrant@openshiftdev ~]$ curl `osc get service docker-registry --template="{{ .spec.portalIP }}:{{ with index .spec.ports 0 }}{{ .port }}{{ end }}" --config=openshift.local.config/master/admin.kubeconfig`
"docker-registry server (dev) (v0.9.0)"
```
※ 404 Page Not Found となりますが、docker-registry は正常に動作しているようです。。。


## プロジェクト管理
### プロジェクトの作成
「test」という名前のプロジェクトを作成し、確認します。
```
[vagrant@openshiftdev ~] $ osc new-project test --display-name="OpenShift 3 Sample" --description="This is an example project to demonstrate OpenShift v3"
[vagrant@openshiftdev ~] $  osc get projects
```

プロジェクトを削除する場合は、以下のコマンドを利用します。
でも、今はやらないでください。
```
[vagrant@openshiftdev ~] $ osc delete projects
```

### 簡単なアプリケーションのデプロイ
「test」プロジェクトにアプリケーションをデプロイします。
```
[vagrant@openshiftdev ~] $ cd /data/src/github.com/openshift/origin/examples/hello-openshift
[vagrant@openshiftdev hello-openshift]$ osc create -f hello-pod.json  -n test
[vagrant@openshiftdev hello-openshift]$ curl http://localhost:6061
```

### 複雑なアプリケーションのデプロイ
```
[vagrant@openshiftdev sample-app]$ cd /data/src/github.com/openshift/origin/examples/sample-app
[vagrant@openshiftdev sample-apps] $ osc process -f application-template-stibuild.json  > ruby.json
[vagrant@openshiftdev sample-apps] $ osc create -f ruby.json
```
ビルドの状態は、`osc get builds` で確認できます。pollingで確認数場合は--watch オプションを利用します。
```
[vagrant@openshiftdev sample-app]$ osc get builds --watch
NAME                  TYPE      STATUS    POD
ruby-sample-build-1   Source    Pending   ruby-sample-build-1
NAME                  TYPE      STATUS    POD
```

ビルドのログは `osc build-logs ruby-sample-build-1` で確認できます。
```
[vagrant@openshiftdev sample-app]$ osc build-logs ruby-sample-build-1 
I0531 17:18:30.280040       1 sti.go:392] ---> Installing application source
I0531 17:18:30.291989       1 sti.go:392] ---> Building your Ruby application from source
I0531 17:18:30.292022       1 sti.go:392] ---> Running 'bundle install --deployment'
I0531 17:18:34.532721       1 sti.go:392] Fetching gem metadata from https://rubygems.org/..........
I0531 17:18:36.452446       1 sti.go:392] Installing rake (10.3.2) 
I0531 17:18:36.655133       1 sti.go:392] Installing i18n (0.6.11) 
I0531 17:18:38.147289       1 sti.go:392] Installing json (1.8.1) 
I0531 17:18:38.367357       1 sti.go:392] Installing minitest (5.4.2) 
I0531 17:18:38.647045       1 sti.go:392] Installing thread_safe (0.3.4) 
I0531 17:18:38.937859       1 sti.go:392] Installing tzinfo (1.2.2) 
I0531 17:18:39.300163       1 sti.go:392] Installing activesupport (4.1.7) 
I0531 17:18:39.469672       1 sti.go:392] Installing builder (3.2.2) 
I0531 17:18:39.669629       1 sti.go:392] Installing activemodel (4.1.7) 
I0531 17:18:39.881080       1 sti.go:392] Installing arel (5.0.1.20140414130214) 
I0531 17:18:40.229913       1 sti.go:392] Installing activerecord (4.1.7) 
I0531 17:18:42.932239       1 sti.go:392] Installing mysql2 (0.3.16) 
I0531 17:18:43.673486       1 sti.go:392] Installing rack (1.5.2) 
I0531 17:18:43.841714       1 sti.go:392] Installing rack-protection (1.5.3) 
I0531 17:18:44.045783       1 sti.go:392] Installing tilt (1.4.1) 
I0531 17:18:44.402413       1 sti.go:392] Installing sinatra (1.4.5) 
I0531 17:18:44.888498       1 sti.go:392] Installing sinatra-activerecord (2.0.3) 
I0531 17:18:44.891529       1 sti.go:392] Using bundler (1.3.5) 
I0531 17:18:44.894559       1 sti.go:392] Your bundle is complete!
I0531 17:18:44.894742       1 sti.go:392] It was installed into ./bundle
I0531 17:18:44.924633       1 sti.go:392] ---> Cleaning up unused ruby gems
I0531 17:18:51.404719       1 sti.go:92] Pushing 172.30.204.63:5000/test/origin-ruby-sample:latest image ...
I0531 17:20:08.301894       1 sti.go:97] Successfully pushed 172.30.204.63:5000/test/origin-ruby-sample:latest
```

管理画面では![STIBuildの結果](images/stibuild-result.png)


ブラウザで frontend の IP アドレスとポート番号を確認します。例えば、frontend の IP アドレスが 172.30.126.82:5432 の場合、もう一つターミナルを開き、次のコマンドを実行します。
```
vagrant ssh -- -L 9999:172.30.17.4:5432
```
ブラウザで http://localhost:9999 にアクセスします。



<!--
```
$ docker pull openshift/origin-haproxy-router
$ sudo chmod +r openshift.local.config/master/openshift-router.kubeconfig
$ openshift ex router --create --credentials=openshift.local.config/master/openshift-router.kubeconfig --config=openshift.local.config/master/admin.kubeconfig --images=openshift/origin-haproxy-router
deploymentConfigs/router
services/router

$ osc project default
$ osc describe dc router
```
-->
