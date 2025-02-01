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

## 【Dockerイメージセットアップ】
LinuxでTVを視聴／録画する為に以下のイメージを作成する
- mariadb
- mirakurun
- EPGStashon

イメージを作成する為にDockerファイルを作成する  
ここでは以下のディレクトリ構成で環境構築する例で記述する  

        /opt/TV_app/docker/
                      |--/mariadb                              docker mariadb root
                      |--/Mirakurun                            docker mirakurun root
                      |--/docker-mirakurun-epgstation          docker compose root
                                           |--epgstation       docker epgstation root

    1. mariadb
      公式イメージをベースとして文字セットをutfmb4を組み込んだオリジナルイメージを作成する
      ディレクトリ構成
        /mariadb              docker mariadb root
            |--/db
            |--/conf          configファイル格納
            |--/mariadb_data  データベースデータ

    1.1 Dockerファイルの作成
        conf ディレクトリに文字セットutfmb4を組み込んだ50-server.cnf, 50-client.cnf を配置する
        docker mariadb root ディレクトリにDockerfile_mariadb を作成する
        1.4でコンテナ起動する場合はdocker-compose.ymlを作成する

    [Dockerfile_mariadb 内容例]
      FROM mariadb:10.11.8
      RUN apt-get update && \
      apt-get upgrade -y
      #COPY ./db/conf/mariadb.cnf /etc/mysql/mariadb.cnf
      COPY ./db/conf/50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf
      COPY ./db/conf/50-client.cnf /etc/mysql/mariadb.conf.d/50-client.cnf

    1.2 Dockerイメージの作成
        $ cd /opt/TV_app/docker/mariadb
        $ docker image build -t mariadb:10.11.8.mod -f Dockerfile_mariadb .
          -t TAGを指定 上記例ではRIPOSITORY mariadb TAG 10.11.8.mod となる
          -f Dockerファイル名を指定

    1.3 確認
        $ docker image ls

    以下は任意

    1.4 コンテナ起動
        $ docker run -e MYSQL_ROOT_PASSWORD=root mariadb:10.11.8.mod
          (-eは環境変数の設定、起動するリポジトリ名:タグ  またはイメージIDを指定)
        docker-compose.ymlを使用する場合は
          $ cd /opt/TV_app/docker/mariadb
          $ docker compose up -d

        [docker-compose.yml 内容例]
        version: '3'
        services:
            mariadb:
                image: mariadb:10.11.8.mod
                container_name: mariadb
                ports:
                    - "7777:3306"
                volumes:
                    - ./db/mariadb_data:/var/lib/mysql
        #       restart: always
                environment:
                    MYSQL_ROOT_PASSWORD: root
                    TZ: "Asia/Tokyo"
        #       command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --performance-schema=false --expire_logs_days=1 # for mariadb
        volumes:
            mariadb_data:
            driver: local

    1.5 コンテナにログイン
        コンテナ起動中にホスト側ターミナルでmariadbにログオンする
        コンテナにログイン
        $ docker exec -it コンテナID bash
          (-i:入力をホストと共有 -t:出力をホストと共有) 
        DBにログオン
          # mysql -u root -p
          文字セットの確認
            MariaDB [(none)]> show variables like "chara%";
    1.6 コンテナの停止/削除
        $ docker stop コンテナID
        $ docker rm コンテナID
    1.7 ホストとポートマッピングして起動
        ホスト側でmariadbをデフォルトポート(3306)で起動しているとポートがかち合ってしまうので
        ポートマッピングして起動する
        $ docker run -e MYSQL_ROOT_PASSWORD=root -p 7777:3306 mariadb:10.6
          (7777:ホスト側のポート 3306:コンテナ側のポート）
    1.8 ホストからDocker mariadbにログオン
        $ mysql -P 7777 -u root -p
          文字セットの確認
            MariaDB [(none)]> show variables like "chara%";
    1.9 データベースデータの削除
        コンテナを削除した時はデータベースデータを全て削除する
        また、他のコンテナで起動する場合もデータベースデータを全て削除しておく


