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
3. Ubuntu 虛擬機：安裝 microk8s。
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

#### Login Ubuntu Machine(microk8s-vm) 

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
```

####  Enable kubeflow

現在來到最關鍵的一步，也就是開啟 microk8s 的 kubeflow。這邊 kubeflow 選用的版本是`cs:kubeflow-245`，也是嘗試加上爬文後，確認可行的版本。

```bash
$ microk8s enable kubeflow --bundle=cs:kubeflow-245
Enabling dns...
Enabling storage...
Enabling ingress...
Enabling metallb:10.64.140.43-10.64.140.49...
Waiting for other addons to finish initializing...
Addon setup complete. Checking connectivity...
Bootstrapping...
Bootstrap complete.
Successfully bootstrapped, deploying...
Kubeflow deployed.
Waiting for operator pods to become ready.
Waited 0s for operator pods to come up, 24 remaining.
Waited 15s for operator pods to come up, 21 remaining.
Waited 30s for operator pods to come up, 21 remaining.
...
Waited 645s for operator pods to come up, 1 remaining.
Operator pods ready.
Congratulations, Kubeflow is now available.

The dashboard is available at http://10.64.140.43.nip.io

    Username: admin
    Password: GVCELKGPEE2I9JAT6RV670MUM7LOK9

To see these values again, run:

    microk8s juju config dex-auth static-username
    microk8s juju config dex-auth static-password



To tear down Kubeflow and associated infrastructure, run:

    microk8s disable kubeflow
```

> 可會在 `Operator pods ready.` 的階段等很久，因為其實還有很多需要的 pod 正在建立中。你可以透過以下指令，確認目前進度如何：
>
> ```bash
> $ microk8s kubectl -n kubeflow get pods | grep 0/
> kubeflow-profiles-5db9ffd44f-c6gr4             0/2     PodInitializing   0          22m
> pipelines-viewer-5679b95c8d-h8767              0/1     PodInitializing   0          16m
> pipelines-scheduledworkflow-79cf4c6649-mzv8f   0/1     PodInitializing   0          17m
> tf-job-operator-85f9558966-cslz2               0/1     PodInitializing   0          16m
> ...
> 
> ```
>
> 如果一直卡在某個 pod 出現 ImagePullBackOff 的狀況，基本上也是網路的問題。這種情況下，就刪除這個 pod，給他一個重新 Pull Image（改過自新）的機會吧：
>
> ```bash
> $ microk8s kubectl -n kubeflow get pods | grep 0/
> pipelines-visualization-67b6995765-8fs6k       0/1     ImagePullBackOff   0          30m
> $ kubectl delete pod pipelines-visualization-67b6995765-8fs6k -n kubeflow
> pod "pipelines-visualization-67b6995765-8fs6k" deleted
> ```

執行後，大概會需要 30 - 50 分鐘以上的時間，才會完成建立，就看網路的狀況而定。接下來，由於**此本版的 kubeflow UI 會缺少功能選單**，所以還需要再下次指令，讓 UI 上的選單長回來：

```bash
$ microk8s kubectl apply -n kubeflow -f https://raw.githubusercontent.com/kubeflow/manifests/master/apps/centraldashboard/upstream/base/configmap.yaml
```

現在 kubeflow 已經安裝好囉！接下來就是要開啟 kubeflow 的 Web UI 來操作 kubeflow。

### Setting Sock Proxy for Opening kubeflow UI from microk8s-vm

為了要開啟虛擬機內的 Web，首先需要建立與 microk8s-vm 間的通道，再進一步透過 Sock Proxy 的方式與 microk8s-vm 內的 microk8s 網路互通。建立通道的方式，就是要透過 `ssh -D`。但要怎麼改用 ssh 連線至 microk8s-vm 呢？我們只需要對  microk8s-vm 動一些手腳即可：

```bash
$ cd ~/.ssh
# 備份原本的 key 以備不時之需
$ cp authorized_keys authorized_keys.backup
# 用自己的公鑰進行替換
echo "<your public key>" > authorized_keys
```

動完手腳後，你已經可以透過 ssh 連進 microk8s-vm 囉！那我們就來建立通道吧！

```bash
# 回到 Mac 的 Terminal
$ exit
logout
Connection to 192.168.64.2 closed.
# 建立通道在 9999 port 上
$ ssh -D9999 ubuntu@192.168.64.2
```

成功連進去後，就可以開始在 Mac 上設定 Sock Proxy。設定方式為：

```bash
開啟'系統偏好設定' 
-> 點選'網路' 
-> 選擇正在使用的網路 
-> 點選右下角的'進階' 
-> 點選倒數第二個選項'代理伺服器' 
-> 選擇左方清單中的'SOCKS 代理伺服器'
-> 在右方的 IP 及 Port 欄位鍵入'127.0.0.1':'9999' # 也就是我們剛剛透過 ssh 建立的通道 
```

此時，你的網路已經開始透過 microk8s-vm 進行轉送。這時候就可以在 Browser 開起，剛剛 enable kubeflow 後顯示的 URL 及帳號密碼，登入 kubeflow 的 UI 介面。

```bash
The dashboard is available at http://10.64.140.43.nip.io

    Username: admin
    Password: GVCELKGPEE2I9JAT6RV670MUM7LOK9
```

如果畫面一切正常，那就恭喜你，在 Mac 上架好 kubeflow 的環境囉！
