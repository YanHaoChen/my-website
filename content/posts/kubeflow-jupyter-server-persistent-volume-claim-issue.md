---
title: "使用 kubeflow 的你也遇到這個問題了嗎? Jupyter Server: 1 pod has unbound immediate PersistentVolumeClaims?!"
date: 2021-10-23T23:47:34+08:00
draft: false
tags:
    - kubeflow
---

又是一個讓我懷疑人生的問題。經過在 local 建置 kubeflow 的摧殘後（詳情請看：[Mac 上安裝 kubeflow? 其實不太簡單](https://yanhaochen.github.io/posts/build-kubeflow-on-mac/)），以為一切就要風平浪靜，但就在建立第一個 Jupyter Server （它的名字叫做 `playground-0`）時，好奇怪？怎麼 Server 一直無法成功建立，在 Status 上的狀態顯示`playground-0 default-scheduler  0/1 nodes are available: 1 pod has unbound immediate PersistentVolumeClaims`。為了解決這個問題，第一時間想到的就是查看 k8s 的狀況（也沒有其他招了...）：

```bash
# 如果是用 microk8s，後續的指令 'kubectl' 皆改成 'microk8s kubectl' 即可。
$ kubectl get pods -n admin
NAME           READY   STATUS    RESTARTS   AGE
playground-0   1/1     Running   0          6h11m
```

等等，看起來似乎沒有什麼異常。只好再看細一點。describe 這 pod：

```bash
$ kubectl describe pod playground-0 -n admin
Name:         playground-0
Namespace:    admin
Priority:     0
Node:         microk8s-vm/192.168.64.2
Start Time:   Sat, 23 Oct 2021 17:25:05 +0800
Labels:       app=playground
...
workspace-playground:
Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
ClaimName:  workspace-playground
ReadOnly:   false
...
Events:
Type     Reason            Age   From               Message
---------------------------------------------------------------------
Warning  FailedScheduling  25m   default-scheduler  0/1 nodes are available: 1 pod has unbound immediate PersistentVolumeClaims.
Normal   Scheduled         24m   default-scheduler  Successfully assigned admin/playground-0 to microk8s-vm
Normal   Pulled            24m   kubelet            Container image "gcr.io/kubeflow-images-public/tensorflow-1.15.2-notebook-cpu:1.0.0" already present on machine
Normal   Created           24m   kubelet            Created container playground
Normal   Started           24m   kubelet            Started container playground
```

此時，在 Events 的地方找到對應的訊息了。看起來，是在一開始（Age: 25m：0/1 nodes are available: 1 pod has unbound immediate），PersistentVolumeClaims（PVC）並沒有成功進行 bound 。但隨後（Age: 24m：Successfully assigned admin/playground-0 to microk8s-vm）就重新 bound 成功。

感覺上，似乎是這筆訊息，讓 Dashboard 顯示異常。這時候，我的解決方式很日常。透過重啟，洗掉 Event 中的錯誤：

```bash
$ kubectl delete pod playground-0 -n admin
pod "playground-0" deleted
```

> 這邊重啟的方式，並不是透過在 Dashboard 點選“垃圾桶”進行刪除。透過垃圾桶，會把整個 Pod 連根拔起，目前沒有透過這樣的方式，Pass 過這個問題。

等待 pod 重啟後，再次進行 describe：

```bash
ubuntu@microk8s-vm:~$ kubectl describe pod playground-0 -n admin
Name:           playground-0
Namespace:      admin
Priority:       0
Node:           microk8s-vm/192.168.64.2
Start Time:     Sat, 23 Oct 2021 17:51:21 +0800
Labels:         app=playground
...
workspace-playground:
Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
ClaimName:  workspace-playground
ReadOnly:   false
...
Events:
Type    Reason     Age   From               Message
---------------------------------------------------------------------
Normal  Scheduled  3s    default-scheduler  Successfully assigned admin/playground-0 to microk8s-vm
Normal  Pulled     2s    kubelet            Container image "gcr.io/kubeflow-images-public/tensorflow-1.15.2-notebook-cpu:1.0.0" already present on machine
Normal  Created    1s    kubelet            Created container playground
Normal  Started    1s    kubelet            Started container playground
```

 現在這個 Pod 的 Event 中，已經沒有剛剛出現的錯誤訊息。此時，Dashboard 上的 Status 已經變成打勾，也可點擊 `Connect` 連進 Server 囉！

### 總結

Pod 中的 Events 有可能會影響到 kubeflow Dashboard 上的狀態，如果在 Dashboard 上發現一些異常的訊息，但實際查看相關 Pod 並沒有異常，有可能是因為 Events 紀錄到某些的錯誤而導致。
