---
title: "WSL Docker"
date: 2023-07-06T00:00:00+09:00
archives:
    - 2023-07
categories: ["blog"]
tags: ["note", "wsl"]
draft: false
---

# はじめに
今更ですがまとめておきます.

# Dockerインストール

必要なパッケージをインストール
```bash
$ sudo apt update
$ sudo apt install ca-certificates curl gnupg lsb-release
```

GPG鍵をaptに追加
```bash
$ sudo mkdir -p /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

パッケージソースリストに追加
```bash
$ echo \
 "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
 $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt update
```

Dockerインストール
```bash
$ sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

# サービス起動

```bash
$ sudo touch /etc/fstab
$ sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
$ sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```

# ログイン時自動起動

`sudo visudo`でsudosersにユーザの`/usr/sbin/service`実行権限を追加する.

```
xxx ALL=(ALL) NOPASSWD: /usr/sbin/service
```

.bashrcに追加
```bash .bashrc
sudo service docker start
```

