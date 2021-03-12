# Ansible/Jenkinsを使った実行環境の構築手順

## 1.コントロールノードにAnsibleをインストール
以下コマンドをコントロールノード上で実行
```
# お決まりのパッケージ最新化
$ sudo yum update -y

# amazon-linux-extrasを使用してansibleがインストール出来るか確認
$ sudo amazon-linux-extras info ansible2

# ansibleのインストール
$ sudo amazon-linux-extras install ansible2

# インストールしたansibleのバージョン確認
$ ansible --version
```
あれ、以下コマンドでインストールじゃないの？と思うかもしれない。
```
$ sudo yum install -y ansible
```
amazon linux2のAMIから作成したインスタンス上にansibleをインストールする場合は通常のパッケージインストール方法は対応していないため、専用のコマンドを使う必要がある。

## 2.コントロールノードに秘密鍵を配置
コントロールノード(ansible server)からターゲットノード(jenkins server)に接続しにいくためには、キーペアとなっている秘密鍵を接続元であるコントロールノードに配置しておく必要がある。

この時点では、秘密鍵を知っているのはローカルPCのみなので、ローカルPCからコントロールノードに対してscpコマンドを使用して秘密鍵を配置する。
```
$ scp -i <コントロールノードへのssh接続で使用する秘密鍵> <ローカルの秘密鍵ファイルPATH> <ユーザ名>@<接続先IPv4アドレス>:<秘密鍵の配置先PATH>
```
例えば上記の秘密鍵およびIPv4アドレスが以下の場合は下記コマンドになる。
- 秘密鍵：hoge.pem
- 接続先IPv4アドレス：55.197.200.191
```
$ scp -i ~/.ssh/hoge.pem ~/.ssh/hoge.pem ec2-user@55.197.200.191:/home/ec2-user/.ssh/
```

問題なくサーバへの秘密鍵がコピー出来たら、コントロールノードからターゲットノードへ接続出来るか確認を行う。
```
# コントロールノードへssh接続した状態で、以下コマンドを実行してターゲットノードへ接続出来ればOK。
$ ssh -i ~/.ssh/hoge.pem ec2-user@<ターゲットノードのIPv4アドレス>
```
接続出来たら、そのままyum updateをターゲットノード上で実行してパッケージの最新化を済ませてしまいましょう。

## 3.ansibleの設定
以下を行う。
- コントロールノードからターゲットノードへansibleを介して接続出来るようにする
- ターゲットノードの情報を持つhostsファイルを作成
- ターゲットノードに対して行う作業が記述されたplaybookファイルを作成

### 3-1.コントロールノードからターゲットノードへansibleを介して接続出来るようにする
ansibleを使用してターゲットノードに接続し、作業を行えるようにするためには項2で行った作業だけでは不足している。
具体的な理由としては「ansibleがssh接続を試みる際、認証情報を知らない」から。
今回は~/.ssh/configを用いた定義を行い接続可能な状態にする。

#### ~/.ssh/configで定義
```
# ~/.ssh/configファイルの設定
$ vi ~/.ssh/config
======= 以下を追加 =======
Host <任意の接続先ホスト名>
  User ec2-user
  port 22
  HostName <ターゲットノードのIPv4アドレス>
  IdentityFile <秘密鍵のPATH>
========================

# ~/.ssh/configのアクセス権限設定
$ chmod 600 ~/.ssh/config
```

### 3-2.ターゲットノードの情報を持つhostsファイルを作成
まずは作業用ディレクトリを作成。
```
$ mkdir -p workspace/ansible
$ cd workspace/ansible
```

hostsファイルを作成
```
$ vi hosts
======= 以下を追加 =======
[web]
<ターゲットノードのIPv4アドレス>

[web:vars]
ansible_ssh_port=22
ansible_ssh_user=ec2-user
ansible_ssh_private_key_file=<秘密鍵PATH>
========================
```

### 3-3.ターゲットノードに対して行う作業が記述されたplaybook/tasksファイルを作成
- playbook.yml
```
$ vi playbook.yml
======= 以下を追加 =======
---
- hosts: web
  become: yes
  tasks:
    - include: tasks/main.yml
========================
```

- tasks/main.yml
```
======= 以下を追加 =======
---
- name: installed git
  yum:
    name: git
    state: installed
- name: installed wget
  yum:
    name: wget
    state: installed
- name: installed java
  yum:
    name: java
    state: installed
- name: get jenkins repo
  get_url:
    url: https://pkg.jenkins.io/redhat/jenkins.repo
    dest: /etc/yum.repos.d/jenkins.repo
- name: add the jenkins repo key
  rpm_key:
    key: https://pkg.jenkins.io/redhat/jenkins.io.key
- name: installed jenkins by latest version
  yum:
    name: jenkins
    state: present
- name: jenkins booted and auto boot conf
  service:
    name: jenkins
    state: started
    enabled: yes
========================
```

### 4.ansible実行
ansibleの実行コマンドは以下のような指定を基本的に行い実行する。
```
$ ansible-playbook -i <hostsファイルPATH> <playbookファイルPATH>
```

本実行の前にplaybookの文法チェックを行っておこう。
```
$ ansible-playbook -i hosts playbook.yml --syntax-check
```
文法に問題なければ実行。
```
$ ansible-playbook -i hosts playbook.yml
```

### 5.jenkinsにアクセス
jenkinsにアクセスするにはブラウザから以下URLを入力する。
```
http://<jenkinsが起動しているサーバのIPv4アドレス>:8080
```

暗号化された認証を行うために、以下ファイルを参照し初回起動画面に入力する。
```
# 以下コマンドで出力された値を使用
$ sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Customize Jenkins画面で「install suggested plugins」を選択すれば必要最低限のプラグインをインストールしてくれる。

後からプラグインを追加で導入して拡張していくことも可能なので、特にこだわりがなければ「install suggested plugins」を選択して実行。

初期設定で以下項目を入力。
- ユーザー名: ログインユーザ名
- パスワード: ログインパスワード
- パスワードの確認:	確認用ログインパスワード
- フルネーム: こだわりなければログインユーザ名と一緒で良い。
- メールアドレス: 管理者用のメールアドレス。

Instance Configuration画面では、今後JenkinsにアクセスするためのURLを指定出来る。
特にこだわりがなければそのままでOK。

ここまででJenkinsのセットアップは完了。

### 6.jenkinsの使い方
まずは適当にジョブを作成してみよう。

基本的には「何か特定の処理をjenkinsから実行したい」という要件であれば
「新規ジョブ作成」>「フリースタイル・プロジェクトのビルド」で大体実現出来る。

ジョブを作成する際押さえておきたいのは、"ジョブ名がJenkinsの作業ディレクトリとしてサーバ上に作成される"という点。
なので、くつかの単語をつなげたジョブ名にする場合はアンダースコアもしくはハイフンを使用しよう。

### ~~7.jenkinsにsudo実行権限を追加~~（不要。7-1を実施）
jenkinsからansibleを実行する際、ec2-userで実行するための以下指定を行う。
```
$ sudo -u ec2-user <ec2-userで実行したいコマンド>
```
しかし、jenkinsにはsudo権限(rootで実行する権限)がないため、追加してあげる。
```
$ sudo vi /etc/sudoers.d/jenkins
======= 以下を追加 =======
Defaults:jenkins !requiretty
jenkins ALL=(ALL) NOPASSWD:ALL
========================
```

### 7-1.ジョブ実行ユーザを変更する
以下ファイルのユーザ指定を変更する
```
$ sudo vi /etc/sysconfig/jenkins
======= 以下を変更 =======
JENKINS_USER="jenkins"
 　　　　　↓↓
JENKINS_USER="ec2-user"
========================
```

ジョブ実行に関係するディレクトリおよびファイルの権限変更する
```
$ sudo chown -R ec2-user: /var/lib/jenkins /var/log/jenkins /var/cache/jenkins
```
### 8.jenkinsからの各種ジョブ実行
- git cloneジョブ
  - 「ソースコード管理」>「Git」を選択して、リポジトリURLを入力
    - この時、httpsでのURLを使用すれば認証なしでclone可能
- CFnのstack作成ジョブ
  - 「高度な設定」>「カスタムワークスペースを使用」でcloneしたリポジトリのPATHを設定
  - 「ビルド・トリガ」>「他プロジェクトの後にビルド」>「対象プロジェクト」にgit cloneを行うジョブ名を設定
  - 「ビルド」>「シェルの実行」で以下を設定
    ```
    #!/bin/bash -xe
    export TZ="Asia/Tokyo"

    SUFFIX=`date "+%Y%m%d%H%M%S"`
    STACK_NAME="livecording-${SUFFIX}"
    sudo -u ec2-user aws cloudformation create-stack --stack-name $STACK_NAME --template-body file://$WORKSPACE/service/VPC/only_vpc.yml
    sudo -u ec2-user aws cloudformation wait stack-create-complete --stack-name $STACK_NAME
    ```
  
