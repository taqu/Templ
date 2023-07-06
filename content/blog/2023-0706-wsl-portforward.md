---
title: "WSL Portforwardと自動化"
date: 2023-07-06T00:00:00+09:00
archives:
    - 2023-07
categories: ["blog"]
tags: ["note", "wsl"]
draft: true
---

#

参考:[WSL2のポートフォワードを自動化する](https://zenn.dev/fate_shelled/scraps/f6252654277ca0)

# 
WSLのIPアドレスは起動ごとに変わる可能性があり, 固定する方法がないためWSL内で自分のIPアドレスを取得します.

次のコマンドを`get_ipv4.bash`などに保存します.

```bash
ip -f inet -o addr show eth0|cut -d\  -f 7 | cut -d/ -f 1
```


Windows側から次のようにすると指定ディストリビューションのコマンドを呼び出せます.

```shell
$ wsl -d distro -e bash /home/user/get_ipv4.bash
```

