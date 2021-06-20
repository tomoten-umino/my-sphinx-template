# コンテナ型仮想化　その２
## docker-compose
- docker-composeは、コンテナの起動構成をdocker-compose.ymlと呼ばれるymlファイルに記述することができるツールである。
- "サービス"と呼ばれる単位で複数のコンテナの起動構成を記述および起動制御することができる。
- 基本的に、docker-composeなしでdockerを利用するのは苦痛でしかない。

### docker-composeのインストール
- [Docker Compose のインストール](https://docs.docker.jp/compose/install.html) 参照
- インターネットに繋がるLinuxがない場合、直接githubからダウンロードすればいい。

```bash
$ curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

$ cd /usr/local/bin

$ chmod 755 docker-compose

```

### docker-composeを使ってみる : noVNCコンテナをdocker-composeで起動する
※[docker-headless-vnc-container](https://github.com/ConSol/docker-headless-vnc-container) をdocker-composeで起動してみる

- Docker hubから[consol/centos-xfce-vnc](https://hub.docker.com/r/consol/centos-xfce-vnc)をpullする。
```bash
$ docker pull consol/centos-xfce-vnc:1.4.0
```


