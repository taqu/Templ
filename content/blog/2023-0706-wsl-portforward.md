---
title: "WSL ポートフォワーディングと自動化"
date: 2023-07-06T00:00:00+09:00
archives:
    - 2023-07
categories: ["blog"]
tags: ["note", "wsl"]
draft: false
---

# はじめに
WSLのサーバ等に外部からアクセスさせたいとき, Windowsのポートフォワーディングを設定する必要があります.
WSLのインスタンスのIPアドレスは固定できないので, インスタンスを立ち上げたあと, 都度調べて設定する必要があります.

# ポートフォワーディング

次のようなシェルスクリプトをWSL側に保存しておきます. ipコマンドからアドレスを切り出して, Windowsのnetsh.exeを呼び出してポートフォワーディングの設定をしています.
```bash
#!/bin/bash
IP=$(ip address show eth0 | awk '/inet / {print $2}' | awk -F / '{print $1}')
echo "own ip is " $IP
netsh.exe interface portproxy delete v4tov4 listenport=9090
netsh.exe interface portproxy add    v4tov4 listenport=9090 connectaddress=$IP
echo "portforward done"
```

Windows側からは次のように呼び出せば設定できます. 適当にスタートアップに設定するなどします.

```bash
$ wsl -d debian -u root --exec /bin/bash /home/user/portforward.sh
```

