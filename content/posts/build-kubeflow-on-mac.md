---
title: "Mac 上安裝 kubeflow? 其實不太簡單"
date: 2021-10-17T22:30:12+08:00
draft: false
tags:
    - kubeflow
---

# Mac 上安裝 kubeflow？其實不太簡單

小弟因工作的緣故，需要開始熟悉 kubeflow 到底是什麼，所以就異想天開的想在自己的 Mac 上建立一個 kubeflow 的環境（痛苦的開始，往往都是如此）。以下將是我嘗試多種安裝組合後，幸運試出的一種架設方式，希望在讀者的 Mac 上也能順利架起 kubeflow。首先，解整體大概的流程，或許會比較有幫助。

### Overview

1. Homebrew：用來安裝 Multipass。
2. Multipass：用來開啟 Ubuntu 虛擬機。
3. Ubuntu 虛擬機：便於安裝 microk8s。
4. microk8s：透過 add-on 安裝 kubeflow。

> 關於設備的要求，如果你的 Mac 沒有 16G 的 Memory，這篇文章可能沒辦法帶給你有效的幫助。

有了以上的藍圖，接下來就可以開始動手囉！

### Install Homebrew

在 Mac 上安裝套件怎麼能少了 Homebrew 呢？這邊不免俗的帶過一下：

```bash
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Install Multipass

接下來，透過 Homebrew 來安裝 Multipass 吧！

```bash
$ brew install multipass
```

### Launch a Ubuntu Machine

如果上述的安裝都順利完成，那就可以開始建立一個 Ubuntu Machine，以便安裝 microk8s。這編我配了 4 個 CPU 及 12G 的 Memory 給這台 Machine。其中，Memory 要 12G 是 kubeflow 的基本需求，雖然可以透過環境變數（`KUBEFLOW_IGNORE_MIN_MEM=true`），忽略這項要求，但也會讓安裝的時間拉很長，甚至安裝失敗（經驗是這樣告訴我的）。

```bash
$ multipass launch --name microk8s-vm --cpus 4 --mem 12G --disk 50G
```

### Install microk8s on the Ubuntu Machine

再來，就是在建立出來的 Ubuntu 上安裝 microk8s，讓 kubeflow 可以建立在上面。

#### 連進 Ubuntu ，準備進行安裝

Multipass 連進虛擬機的方式：

```bash
$ multipass shell microk8s-vm
```

#### Install microk8s

在此，我們需要透過 Ubuntu 的 snap 工具，安裝 microk8s：

```bash
$ sudo snap install microk8s --channel=1.20/edge --classic
```

這邊選擇的 microk8s 的版本是 `1.20/edge`，這也是經驗告訴我的結果。順道一提，如果 microk8s 版本`>=1.22`，是連 kubeflow 的 add-on 都沒有哦！

```bash
$ sudo snap install microk8s --channel=1.20/edge --classic
```

### Build kubeflow by microk8s add-on

終於，我們要開始建立 kubeflow 了。在 enable kubeflow 前，我們需要先執行以下指令，以便順利執行：

```bash
$ sudo usermod -a -G microk8s $USER
$ sudo chown -f -R $USER ~/.kube
# 接下來，需要先登出當前的 terminal，再重新連進來一次，上述指令才會生效
# 登出
$ exit
# 重新連線
$ multipass shell microk8s-vm
# 預先開啟會需要的 plugin
$ microk8s enable dns storage ingress
```

####  Enable kubeflow

現在來到最關鍵的一步，也就是開啟 microk8s 的 kubeflow。這邊 kubeflow 選用的版本是`cs:kubeflow-245`，也是嘗試加上爬文後，確認可行的版本。

```bash
$ microk8s enable kubeflow --bundle=cs:kubeflow-245
```

執行後，大概會需要10分鐘以上的時間，才會完成建立。接下來，由於**此本版的 kubeflow UI 會缺少功能選單**，所以還需要再下次指令，讓 UI 上的選單長回來：

```bash
$ microk8s kubectl apply -n kubeflow -f https://raw.githubusercontent.com/kubeflow/manifests/master/apps/centraldashboard/upstream/base/configmap.yaml
```

#### Setting Connection for Opening kubeflow UI from Virtual Machine(Ubuntu)

首先需要透過 ssh 來建立 Port Forwarding。但要怎麼改用 ssh 連線至 microk8s-vm 呢？我們只需要對  microk8s-vm 棟一些手腳即可：

```bash
# 備份原本的 key 以備不時之需
# 用自己的公鑰進行替換
```

