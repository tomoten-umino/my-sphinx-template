# コンテナ型仮想化　その２
## docker-compose
- docker-composeは、コンテナの起動構成をdocker-compose.ymlと呼ばれるymlファイルに記述することができるツールである。
- "サービス"と呼ばれる単位で複数のコンテナの起動構成を記述および起動制御することができる。
- 基本的に、docker-composeなしでdockerを利用するのは苦痛でしかない。

### docker-composeのインストール
- [Docker Compose のインストール](https://docs.docker.jp/compose/install.html) 参照
- インターネットに繋がるLinuxがない場合、直接githubからダウンロードすればいい。

```bash
$ curl -L https://github.com/docker/compose/releases/download/1.28.6/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

$ cd /usr/local/bin

$ chmod 755 docker-compose

```

### docker-composeを使ってみる : noVNCコンテナをdocker-composeで起動する
※[docker-headless-vnc-container](https://github.com/ConSol/docker-headless-vnc-container) をdocker-composeで起動してみる

- Dockerhubから[consol/centos-xfce-vnc](https://hub.docker.com/r/consol/centos-xfce-vnc)をpullする。
```bash
$ docker image pull consol/centos-xfce-vnc:1.3.0
```

- githubまたはDockerhubの説明の通り、Dockerコマンドをそのまま叩いて起動いてみる。
```bash
$ docker container run -d -p 5901:5901 -p 6901:6901 -e VNC_PW=my-pw　consol/centos-xfce-vnc:1.3.00
```

- ブラウザを開き、「http://IPアドレス:6901/vnc.html」を開くと以下のような画面が表示される。
    - vncパスワードで設定したmy-pwを入力すると、ログインできる。
![noVNC-top](./images/noVNC-top.png)

- 上記で実行したコンテナを停止する。
    - コンテナ名を確認したあと、stop -> rmの順に実行する。
```bash
$ docker container ls -a
CONTAINER ID   IMAGE                          COMMAND                  CREATED        STATUS        PORTS                                                                                  NAMES
f1697bd06f35   consol/centos-xfce-vnc:1.3.0   "/dockerstartup/vnc_…"   15 hours ago   Up 15 hours   0.0.0.0:5901->5901/tcp, :::5901->5901/tcp, 0.0.0.0:6901->6901/tcp, :::6901->6901/tcp   goofy_liskov

$ docker container stop goofy_liskov
goofy_liskov

$ docker container rm goofy_liskov
goofy_liskov
```

- 以下のような内容が記述されたdocker-compose.ymlを作成する。
```yaml
virsion: '3.7'

services:
  novnc:
    image: consol/centos-xfce-vnc:1.3.0
    ports:
      - "5901:5901"
      - "6901:6901"
    environment:
      VNC_PW: my-pw
```

- docker-compose -f ./docker-compose.yml config を実行して、ymlファイルが正しく読み込めるか確認する。
```bash
$ docker-compose -f ./docker-compose.yml config
services:
  novnc:
    environment:
      VNC_PW: my-pw
    image: consol/centos-xfce-vnc:1.3.0
    ports:
      - published: 5901
        target: 5901
      - published: 6901
      target: 6901
version: '3.7'
```

- docker-compose up -dで起動する。
```bash
$ docker-compose -f ./docker-compose.yml up -d
Creating network "workspace_default" with the default driver
Creating workspace_novnc_1 ... done
```

- 再度、ブラウザで「http://IPアドレス:6901/vnc.html」を開くと、上記と同じログイン画面となることを確認する。
- 動作が確認できたら、docker-compose downでコンテナを削除する。
```bash
$ docker-compose -f ./docker-compose.yml down
Stopping workspace_novnc_1 ... done
Removing workspace_novnc_1 ... done
Removing network workspace_default
```

### docker-composeで複数コンテナの制御
※具体例として、2重冗長系のGlusterFSコンテナイメージのビルドにdocker-compose.ymlを紹介する。<br>
参考：[portable-gluster](https://github.com/tomoten-umino/portable-gluster)

- GlusterFSの冗長構成の情報をイメージビルド時に保持するため、docker networkにより仮想的なネットワークを構築して1台のサーバ内で冗長構成を仮想的に構築している。
- docker-compose.ymlの実装例は以下の通りである。
```yaml
version: "3.7"
services:
  gluster-server-1:
    hostname: ${HOSTNAME_1}
    image: gluster/gluster-centos:gluster3u10_centos7
    volumes:
      - ${PWD}/volume/gluster-server-1/etc-glusterfs:/etc/glusterfs
      - ${PWD}/volume/gluster-server-1/var-lib-glusterd:/var/lib/glusterd
      - ${PWD}/volume/gluster-server-1/HDD1:/home/HDD1
      - ${PWD}/volume/setup.sh:/setup.sh
      - ${PWD}/.env:/.env
    extra_hosts:
      - ${HOSTNAME_1}:${IP_ADDR_1}
      - ${HOSTNAME_2}:${IP_ADDR_2}
    privileged: true
    networks:
      app_net:
        ipv4_address: ${IP_ADDR_1}

  gluster-server-2:
    hostname: ${HOSTNAME_2}
    image: gluster/gluster-centos:gluster3u10_centos7
    volumes:
      - ${PWD}/volume/gluster-server-2/etc-glusterfs:/etc/glusterfs
      - ${PWD}/volume/gluster-server-2/var-lib-glusterd:/var/lib/glusterd
      - ${PWD}/volume/gluster-server-2/HDD1:/home/HDD1
    extra_hosts:
      - ${HOSTNAME_1}:${IP_ADDR_1}
      - ${HOSTNAME_2}:${IP_ADDR_2}
    privileged: true
    networks:
      app_net:
        ipv4_address: ${IP_ADDR_2}

networks:
  app_net:
    name: app_net
    driver: bridge
    ipam:
     driver: default
     config:
       - subnet: ${NETWORK_ADDR}
```

## 性能を意識したDockerの使い方
- リアルタイム要求のあるプログラムをDocker上で動作させる場合、仮想化特有のジッターの大きさが問題となる。
    - OS仮想化は非常に便利だが、ハードウェアのソフトウェアエミュレートがあるため、単純に遅い。
- コンテナ型仮想化は、OS仮想化に比べて非常に軽いのだが、それでもホストOSに比べるとジッターがある。
- ホストOSと比べて、誤差で済まない範囲のものは以下のものである。
    - コンテナ内でのファイルのWrite
    - ネットワークIO性能
- また、共有メモリ間通信を多用するプログラムは、コンテナによるプロセス分割がしづらく、非常に巨大なコンテナイメージとなり、可搬性が著しく低下する。非推奨だが最後の手段しては以下の対策がある。
    ipc, pidオプション
- 以下では、リアルタイム要求のあるプログラムを扱う上でどのような対処が必要かを提示する。

### ファイルwriteの対策
参考：[Docker でコンテナにマウントできるボリュームについて](https://blog.amedama.jp/entry/docker-mount-volume)
- 対策１：ファイル書き出しする場合は、**volumeマウントした場所**で書くこと
- 対策２：一時的なファイルならば、**tmpfs : RAMディスク**を活用せよ

### ネットワークIO性能
参考：[コンテナでホストに近い速度を出す](https://qiita.com/tukiyo3/items/fbb5dd35b11401b46f30)
- 対策：docker networkは使用せず、ホストネットワーク（--net=hostオプション）を使用すること
   - docker networkは非常に便利なのだが、ホストネットワークとコンテナ内ネットワーク間で[NAPT](https://e-words.jp/w/NAPT.html)変換を実施する。このNAPTがジッターの原因となる。

### ipc, pidオプション
※このオプションは利用方法を謝るとコンテナのパッキング自体を破壊する危険なオプション

- ipcオプションは、コンテナ間で共有できる共有メモリ領域を作成できる。
    - 主な使い方は、ホストの共有メモリをコンテナと共有する、か共有メモリ管理用のコンテナを用意し、その共有メモリコンテナを複数コンテナで共有する、などがある。
    - 要は、コンテナ間に配置したプロセス間でQueueを介して通信したい、場合に使う最終手段
- pidオプションは、コンテナ毎に（自動で）設定される[namespace](https://docs.docker.jp/engine/security/userns-remap.html)を指定できる。
    - 噛み砕いていうと、プロセスIDの空間を他のコンテナやホストと共有できる。
    - 主な使い方は、ホストのPID空間をコンテナと共有する、かPID管理用のコンテナを用意し、そのPIDコンテナを複数コンテナで共有する、などがある。
- 紹介しておいてアレだが、安易に使うべきオプションではない。用法を間違えると、コンテナ側からホスト側を攻撃できる。
    - 他社製MWがこのオプションを指定してきたときは、要注意
