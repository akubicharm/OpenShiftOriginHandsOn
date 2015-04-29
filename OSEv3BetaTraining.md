# はじめに
## ゴール
### 環境
### 使用するソフトウェア
### 参考文書
### 書体の表記ルール

# 概要

# 準備

## OpenShift Origin の入手
OpenShift Orign は GitHub <http://github.com/openshift/origin> で公開されています。
```
git clone https://github.com/openshift/origin.git
```
## システム要件

### Native インストールの場合
RHEL7.1 (Open vSwitchを利用）
NetworkManager disabled
Minimal インストールオプション

### 仮想環境の場合

### Docker コンテナの場合

## 利用するツール
vagrant
virtualbox
git

go

# OpenShift Origin の実行
## Vagrantを利用する場合
### Vagrantの基本的な操作

### 環境
Vagrant で起動する仮想環境は、次のような構成と成っています。
* ユーザとパスワード
|ユーザ|パスワード|権限|
|vagrant|vagrant|sudoでrootユーザ権限でのコマンド実行が可能|


### 仮想OSの起動とログイン
```
vagrant up
vagrant ssh
```

### OpenShift のインストール
/data/src/github.com/openshift/origin ディレクトリに git clone で取得したGitリポジトリがマウントされていることを確認してください。
```
ls /data/src/github.com/openshift/origin
```
makeコマンドを使って、OpenShift Originをビルドします。 
ビルドが完了すると、All In Oneの構成でOpenShiftを利用する環境が構築されます。
```
cd /data/src/github.com/openshift/origin
make clean build
```

### 環境確認
* Docker の設定
/etc/sysconfig/docker を確認し、Insecure Registry のネットマスクを確認する

* 環境変数

### ネットワークの設定
* firewalld, Network manager の確認
* ネットワークセグメントの確認

## 動作確認


# 管理機能
## 管理Web
## 管理CLI
## ユーザ管理
## プロジェクト管理

# アプリエーションのデプロイ
## 
