# LXD による開発環境のセットアップ

１年近く前に開発用にと思って自作サーバを組んで見たのだけれど、なんやかんやてんやわんやで絶賛放置中だったんだけど、コロナの影響もあったおかげでセットアップする時間が取れる様になったので、LXD で開発環境をセットアップしてみる事した。

１年近く放置中だっただけあってログインパスワードを忘れてるし、何をインストールしたのかも忘れてしまった。まぁ、普通忘れるよね。

とりあえず `Ubuntu 18.04 LTS Server` をインストールする事から開始したんだけど、外付け HDD に書き込んでる間に `20.04 LTS` がリリースされてるのに気づいた。不運だ。しょうがないので `18.04` をインストール後にアップグレードしたので環境は `20.04` になった。

```shell
$ cat /etc/lsb-release 
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=20.04
DISTRIB_CODENAME=focal
DISTRIB_DESCRIPTION="Ubuntu 20.04 LTS"
```

LXD は既に入ってたのだけれど、`lxd --version` を実行すると `usbid: failed to load: "/usr/share/misc/usb.ids" no such file or directory` なんてがエラー出るので確認してみると、存在してる読み込めるしナニコレ状態だったので、あらためて公式ドキュメントに従ってインストールと初期化を行う。

- Linux Containers - LXD - はじめに - コマンドライン  
  https://linuxcontainers.org/ja/lxd/getting-started-cli/

そんなこんなで LXD をインストールすることができた。

```shell
$ lxd --version
4.0.1
```


## 仮想マシン用のプロファイルを設定する

初期化後はデフォルトのプロファイル `default` が作成されているはずなので確認してみる。

```shell
$ lxc profile show default
config: {}
description: Default LXD profile
devices:
  eth0:
    name: eth0
    network: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: default
used_by: []
```


それから仮想マシン用のプロファイル `vm` を作成する。

```shell
$ lxc profile create vm
```

そうすると以下の様なテンプレートが作成される。


```yml
### This is a YAML representation of the profile.
### Any line starting with a '# will be ignored.
###
### A profile consists of a set of configuration items followed by a set of
### devices.
###
### An example would look like:
### name: onenic
### config:
###   raw.lxc: lxc.aa_profile=unconfined
### devices:
###   eth0:
###     nictype: bridged
###     parent: lxdbr0
###     type: nic
###
### Note that the name is shown but cannot be changed

config: {}
description: ""
devices: {}
name: vm
used_by: []
```

プロファイルの設定項目について詳しくは以下の公式ドキュメントを参照。

- Instance configuration - LXD - system container manager  
  https://lxd.readthedocs.io/en/latest/instances/


空っぽのプロファイルを以下の様に書き換えて `cloud-init` での初期化に対応させる。

```shell
$ lxc profile edit vm
```


```yml
### This is a YAML representation of the profile.
### Any line starting with a '# will be ignored.
###
### A profile consists of a set of configuration items followed by a set of
### devices.
###
### An example would look like:
### name: onenic
### config:
###   raw.lxc: lxc.aa_profile=unconfined
### devices:
###   eth0:
###     nictype: bridged
###     parent: lxdbr0
###     type: nic
###
### Note that the name is shown but cannot be changed

config:
  user.user-data: |
    #cloud-config
    disable_root: true
    ssh_pwauth: yes
    users:
      - name: ubuntu
        groups: lxd
        shell: /bin/bash
        passwd: "$6$rP7tNg6vq.AK7U9p$3zaRN3PriMB6p/fv4cDnNGvWaqqNhO9wJkoQzA/9IHd8fwFERPriV5f9tDSZB/gckNQhkBEee2CQJjH2J.ac91"
        lock_passwd: false
        sudo: ALL=(ALL) ALL
description: LXD profile for virtual machines
devices:
  config:
    type: disk
    source: cloud-init:config
name: vm
used_by: []
```

`root` ユーザーでは触りたくないので `disable_root` で禁止にし、公開鍵認証だと秘密鍵をうっかり無くしてしまったらログイン出来なくなるので `ssh_pwauth` でパスワード認証を有効にしている。そうさ、私はうっかりさんなのだ。

それから、パスワードログイン可能なユーザー `ubuntu` を作成して、`sudo` コマンドを実行可能にする。パスワードは以下の様に `openssl passwd -6` で `SHA512` でハッシュ化されたパスワードを設定する。

```shell
$ openssl passwd -6 <password>
```

`cloud-init` ついて詳しくは公式ドキュメントを参照。

- Modules &mdash; cloud-init 20.1 documentation  
  https://cloudinit.readthedocs.io/en/latest/topics/modules.html


## 仮想マシンを作成する

実際に `ubuntu 18.04` の仮想マシンを作成してみようと思う。

仮想マシンの作成自体はコマンド一発で作成できてしまうのだけれど、それだけを記載しても備忘録のために記事にしている意味が無いので、簡単な仕組みだけでも調べて記載しておく事にする。

そもそも「元のイメージはどこからやってくるのさ？」なんて疑問が浮かんだのだけれど、それは以下のコマンドで確認できた。

```shell
$ lxc remote list
+-----------------+------------------------------------------+---------------+-------------+--------+--------+
|      NAME       |                   URL                    |   PROTOCOL    |  AUTH TYPE  | PUBLIC | STATIC |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+
| images          | https://images.linuxcontainers.org       | simplestreams | none        | YES    | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+
| local (default) | unix://                                  | lxd           | file access | NO     | YES    |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+
| ubuntu          | https://cloud-images.ubuntu.com/releases | simplestreams | none        | YES    | YES    |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+
| ubuntu-daily    | https://cloud-images.ubuntu.com/daily    | simplestreams | none        | YES    | YES    |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+
```

この中の `https://cloud-images.ubuntu.com/releases` からイメージを取得する事にして、使用したいイメージ `18.04` が存在するのかを以下のコマンドで確認する。

```shell
$ lxc image list ubuntu: '18.04'
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
|                ALIAS                | FINGERPRINT  | PUBLIC |              DESCRIPTION               | ARCHITECTURE |      TYPE       |   SIZE   |          UPLOAD DATE          |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04 (7 more)               | 15d8ea2efae1 | yes    | Ubuntu bionic amd64 (20200423_07:42)   | x86_64       | CONTAINER       | 94.46MB  | Apr 23, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04 (7 more)               | 8909f04f8b60 | yes    | Ubuntu bionic amd64 (20200423_07:42)   | x86_64       | VIRTUAL-MACHINE | 220.38MB | Apr 23, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/arm64 (3 more)         | 4dcff94827e1 | yes    | Ubuntu bionic arm64 (20200423_07:42)   | aarch64      | VIRTUAL-MACHINE | 215.06MB | Apr 23, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/arm64 (3 more)         | febff738fc4d | yes    | Ubuntu bionic arm64 (20200423_07:42)   | aarch64      | CONTAINER       | 87.35MB  | Apr 23, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/armhf (3 more)         | 16fc34bbe5d6 | yes    | Ubuntu bionic armhf (20200423_07:42)   | armv7l       | CONTAINER       | 87.52MB  | Apr 23, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/cloud (3 more)         | 6a1ee0ffefc0 | yes    | Ubuntu bionic amd64 (20200424_02:00)   | x86_64       | CONTAINER       | 105.25MB | Apr 24, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/cloud (3 more)         | 545af47c5fef | yes    | Ubuntu bionic amd64 (20200424_02:00)   | x86_64       | VIRTUAL-MACHINE | 237.31MB | Apr 24, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/cloud/arm64 (1 more)   | bcb07da59dc2 | yes    | Ubuntu bionic arm64 (20200421_07:42)   | aarch64      | VIRTUAL-MACHINE | 231.81MB | Apr 21, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/cloud/arm64 (1 more)   | c812bf93037c | yes    | Ubuntu bionic arm64 (20200421_07:42)   | aarch64      | CONTAINER       | 97.54MB  | Apr 21, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/cloud/armhf (1 more)   | 4d9868d7b6b5 | yes    | Ubuntu bionic armhf (20200423_07:42)   | armv7l       | CONTAINER       | 97.53MB  | Apr 23, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/cloud/i386 (1 more)    | b1b163d2499c | yes    | Ubuntu bionic i386 (20200424_02:00)    | i686         | CONTAINER       | 106.10MB | Apr 24, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/cloud/ppc64el (1 more) | 702643b30fb8 | yes    | Ubuntu bionic ppc64el (20200423_07:42) | ppc64le      | CONTAINER       | 107.92MB | Apr 23, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/cloud/s390x (1 more)   | 1a3cc8410fcb | yes    | Ubuntu bionic s390x (20200423_07:42)   | s390x        | CONTAINER       | 81.22MB  | Apr 23, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/i386 (3 more)          | 370a34ab6bf2 | yes    | Ubuntu bionic i386 (20200424_02:00)    | i686         | CONTAINER       | 95.38MB  | Apr 24, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/ppc64el (3 more)       | a62eaa2eee42 | yes    | Ubuntu bionic ppc64el (20200423_07:42) | ppc64le      | CONTAINER       | 96.66MB  | Apr 23, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
| ubuntu/18.04/s390x (3 more)         | 77dda0247401 | yes    | Ubuntu bionic s390x (20200423_07:42)   | s390x        | CONTAINER       | 71.03MB  | Apr 23, 2020 at 12:00am (UTC) |
+-------------------------------------+--------------+--------+----------------------------------------+--------------+-----------------+----------+-------------------------------+
```

`NAME` が `ubuntu/18.04` で `TYPE` が `VIRTUAL-MACHINE` のイメージがリストアップされているのが確認できたので、以下のコマンドで実際に仮想マシン `hello-vm` を作成してみる。

```shell
$ lxc launch ubuntu:18.04 hello-vm --vm --profile default --profile vm
Creating hello-vm
Starting hello-vm
```

`--vm` フラグを渡す事で仮想マシンを作成をする。そして、`default` プロファイルに従ってネットワーク周りの設定を行い、`vm` プロファイルに従って `cloud-init` による初期化を行っている。多分。

しばらくすると終わるので確認してみる。

```shell
$ lxc list
+----------+---------+---------------------+-----------------------------------------------+-----------------+-----------+
|   NAME   |  STATE  |        IPV4         |                     IPV6                      |      TYPE       | SNAPSHOTS |
+----------+---------+---------------------+-----------------------------------------------+-----------------+-----------+
| hello-vm | RUNNING | 10.74.152.51 (eth0) | fd42:d488:1c1e:a215:216:3eff:fe26:54c4 (eth0) | VIRTUAL-MACHINE | 0         |
+----------+---------+---------------------+-----------------------------------------------+-----------------+-----------+
```

動いている様なので `ssh` でログインしてみる。

```shell
$ ssh ubuntu@10.74.152.51
The authenticity of host '10.74.152.51 (10.74.152.51)' can't be established.
ECDSA key fingerprint is SHA256:iOTgUrphpoA3DQ/QMDX9C2LXXjwLAWoWqvxGCSFKPQs.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.74.152.51' (ECDSA) to the list of known hosts.
ubuntu@10.74.152.51's password: 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-96-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Apr 24 08:41:18 UTC 2020

  System load:  0.0               Processes:             100
  Usage of /:   11.0% of 8.86GB   Users logged in:       0
  Memory usage: 12%               IP address for enp5s0: 10.74.152.51
  Swap usage:   0%


0 packages can be updated.
0 updates are security updates.


Last login: Fri Apr 24 08:34:23 2020
ubuntu@hello-vm:~$
```

無事確認できた！　やった！　終わった！　なんか疲れた！

そう。あっさり出来た様に見えるけれど、実際には公式ドキュメントを漁ったり色んなサイトを参照して丸二日くらいかかってしまったよ。きっとコンテナ全盛だからなのだろうけど、なかなかコレといった情報にたどり着かなくて結構疲れた。。。

あとはネットワーク周りを調整したら終わりかな。  
あー、、終わってなかった。。
