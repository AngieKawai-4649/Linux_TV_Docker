# Linux (主にLinuxMint ubuntu系でTVを視聴／録画する為のインストールガイド Docker編)

## 【Dockerとは】
一般的な仮想環境はマシン上のOSとは別にゲストOSを構築し、ホストOSと独立した形で動作  
DockerはホストOSのカーネルを使用し、ファイルシステムを仮想環境とする  
Docker上で動くプロセスはホストOSカーネル下で動くので動作が軽く複数の仮想環境を同時に起動することができる  
ホストOSのファイルシステムは使用せずにOS、ミドルウエア等のパッケージ、アプリケーションは全てゲストOSのファイルシステム上で構築する  
ホストOSにはDockerをインストールし、Dockerコンテナを起動するだけである
- Dockerイメージ: ゲストOSのファイルシステム
- Dockerコンテナ: ホストOS上でDockerイメージを起動する為のプロセス

## 【最新版Dockerインストール】

    1. 既にインストールしている場合は削除
        $ sudo apt remove docker docker-engine docker.io containerd runc
       ボリュームの削除
         $ sudo rm -rf /var/lib/docker
         $ sudo rm -rf /var/lib/containerd
    2. 動作に必要となるパッケージのインストール
        $ sudo apt-get update
        $ sudo apt-get install ca-certificates curl gnupg lsb-release
    3. Dockerの公式GPGキーを取得する
        $ sudo mkdir -p /etc/apt/keyrings
        $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    4. リポジトリの登録（Ubuntu系）
        /etc/apt/sources.list.d/docker.list を作成する
        注意:
        コマンド lsb_release -cs Ubuntuではコードネームとなるが （Ubuntu 22.04 LTSではJammy ）、Ubuntu系他Distributorでは別のコードネームとなる
       （LinuxMint 21.2 では Victoria）
        Ubuntu系は全てUbuntuのコードネームにならないとリポジトリの更新でエラーとなる
        echo $(. /etc/os-release && echo $UBUNTU_CODENAME)で一応Ubuntuのコードネームを取得できるが
        lsb_release -i でDistributorIDを取得しUbuntuとUbuntu系（Mint等）を切り分けることにする
        Ubuntu系以外（Debian等）は別の手法となる
        詳細は https://docs.docker.com/engine/install/ を参照
        以下のコマンドを実行し /etc/apt/sources.list.d/docker.list を作成する
            $ echo  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu" $(if [ `lsb_release -i | awk '{print $3}'` = "Ubuntu" ];then echo $(lsb_release -cs); else  echo $(. /etc/os-release && echo $UBUNTU_CODENAME);fi) "stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
        または
            $ echo  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu" $(if [ `lsb_release -is` = "Ubuntu" ];then echo $(lsb_release -cs); else  echo $(. /etc/os-release && echo $UBUNTU_CODENAME);fi) "stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    5. dockerインストール
    5.1 最新版
        $ sudo apt update
        $ sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    5.2 バージョンを指定したインストール
        $ apt-cache madison docker-ce
        例 5:20.10.13~3-0~ubuntu-jammyをインストール
        $ sudo apt install docker-ce=5:20.10.13~3-0~ubuntu-jammy docker-ce-cli=5:20.10.13~3-0~ubuntu-jammy containerd.io docker-compose-plugin
    6. バージョン確認
        $ sudo docker -v
    7. ユーザーがdockerを使用できるようにdockerグループに追加
        $ sudo gpasswd -a user-name docker
    8. 動作確認
        $ docker run hello-world
    9. docker compose
        docker composeはdocker-compose-pluginパッケージをインストールするとdockerのサブコマンドとして使えるようになっている
        $ docker compose version
    10. docker image保存先を外部ストレージに変更
        docker サービス停止
            $ sudo systemctl stop docker
            $ sudo systemctl stop docker.socket
        ファイルコピー
        例 /opt/docker に変更する
            $ sudo cp -ar /var/lib/docker /opt
        サービスファイルの編集
       /lib/systemd/system/docker.service
        変更前
            ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
        変更後
             Docker Engine 23.0未満
               ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock -g /opt/docker
             Docker Engine 23.0以降
               ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --data-root /opt/docker
        サービス再起動
            $ sudo systemctl daemon-reload
            $ sudo systemctl start docker
         確認
            $ docker info | grep "Docker Root Dir"

