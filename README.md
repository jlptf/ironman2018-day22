## Day 22 - 常見問題與建議 (3)

### 本日共賞

* 叢集資源不足
* 應用程式崩潰 (crash)

### 希望你知道

* [與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502)
* [yaml](https://ithelp.ithome.com.tw/articles/10193509)

<br/>

#### 叢集資源不足

在無套用自動擴展 (auto-scale) 的 k8s 中，分配給叢集使用的資源 (CPU, Memory) 可能會有分配完的時候。

> 這裡單純考慮無任何使用資源限制的情況，與 [Day 19 - 常見問題與建議 (2)](https://ithelp.ithome.com.tw/articles/10193946) 中討論的資源限制不同

由於要配合 k8s Scheduler 的關係，通常在部署 Pod 的時候都會指定需要分配多少資源給 Pod。

> 你可以指定分配 `500m` cpu 給 Pod 使用，但不表示 Pod 真的會用到 500m。因此，這裡討論的分配完並不等於使用完。

如果以 1 個 cpu 來討論，你會有單位為 `1000m` 的 cpu 資源可分配給 Pod 使用。

假設你想部署每個都使用 `100m` 的 Pod，如果在不考慮 k8s 系統需要使用的 Pod 的情況下，你最多可以部署 10 個 Pod。

如果想查看 Node 資源使用情況，可使用

```bash
$ kubectl describe nodes
...
Non-terminated Pods:         (5 in total)
  Namespace                  Name                              CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------                  ----                              ------------  ----------  ---------------  -------------
  kube-system                default-http-backend-78p5w        10m (0%)      10m (0%)    20Mi (1%)        20Mi (1%)
  kube-system                kube-addon-manager-minikube       5m (0%)       0 (0%)      50Mi (2%)        0 (0%)
  kube-system                kube-dns-6fc954457d-zt7fd         260m (13%)    0 (0%)      110Mi (5%)       170Mi (8%)
  kube-system                kubernetes-dashboard-hcz6v        0 (0%)        0 (0%)      0 (0%)           0 (0%)
  kube-system                nginx-ingress-controller-z6vn9    0 (0%)        0 (0%)      0 (0%)           0 (0%)
Allocated resources: <=== 資源配置情況
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ------------  ----------  ---------------  -------------
  275m (13%)    10m (0%)    180Mi (9%)       190Mi (9%) 
  ...  
```
  
這裡就會列出目前 Node 資源配置情況，可以看到目前已分配的 cpu 資源是 `275m (13%)` 
  
> 部分資訊有省略，所以請參照實際狀況。而`(13%)`是因為有兩個 cpu 
> 
> 275 / 2000 約等於 13%
  
這裡舉個例子，撰寫 error-exceed.yaml 並宣告分配 1 個 cpu (`1000m`)

```
# error-exceed.yaml

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: err-exceed-nginx
spec:
    replicas: 1
    template:
      metadata:
        labels: 
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx
          ports:
          - containerPort: 80
          resources:
            requests:
              cpu: 1  <=== 這裡宣告配置 1 個 CPU, 佔用 1000m
```

並部署到 k8s 中

```bash
$ kubectl apply -f error-exceed.yaml 
deployment "err-exceed-nginx" created
```

接著再次查看資源分配狀態

```bash
$ kubectl describe nodes
...
Non-terminated Pods:         (6 in total)
  Namespace                  Name                                 CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ---------                  ----                                 ------------  ----------  ---------------  -------------
  default                    err-exceed-nginx-7b6fcd788c-wr5fd    1 (50%)       0 (0%)      0 (0%)           0 (0%)
  kube-system                default-http-backend-78p5w           10m (0%)      10m (0%)    20Mi (1%)        20Mi (1%)
  kube-system                kube-addon-manager-minikube          5m (0%)       0 (0%)      50Mi (2%)        0 (0%)
  kube-system                kube-dns-6fc954457d-zt7fd            260m (13%)    0 (0%)      110Mi (5%)       170Mi (8%)
  kube-system                kubernetes-dashboard-hcz6v           0 (0%)        0 (0%)      0 (0%)           0 (0%)
  kube-system                nginx-ingress-controller-z6vn9       0 (0%)        0 (0%)      0 (0%)           0 (0%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  CPU Requests  CPU Limits  Memory Requests  Memory Limits
  ------------  ----------  ---------------  -------------
  1275m (63%)   10m (0%)    180Mi (9%)       190Mi (9%)   <=== 發生變化了
...
```

這裡看到 `CPU Requests` 變成 `1275m (63%)`。 因為 k8s 分配了 1 cpu 給 `err-exceed-nginx-7b6fcd788c-wr5fd` 這個運行在 `default` 的 Pod 使用。

如果我們再多要求配置一個 Pod，叢集資源不足的情況就發生

> 如果你使用的是超級電腦，請要求配置更多 Pod 才能看得到錯誤

```bash
$ kubectl scale --replicas=2 deployment/err-exceed-nginx
deployment "err-exceed-nginx" scaled

$kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
err-exceed-nginx   2         2         2            1           51m
```

查看 deployment 就會發現 `AVAILABLE: 1`，接著查看 Pod 狀態

```bash
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
err-exceed-nginx-7b6fcd788c-hh9v8   0/1       Pending   0          7m
err-exceed-nginx-7b6fcd788c-wr5fd   1/1       Running   0          52m
```

這裡會發現 `err-exceed-nginx-7b6fcd788c-hh9v8` 的狀態變成 `Pending`，於是再進一步取得 Pod 資料

```bash
$ kubectl describe pods err-exceed-nginx-7b6fcd788c-hh9v8
...
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  7m (x26 over 13m)  default-scheduler  No nodes are available that match all of the predicates: Insufficient cpu (1).
```

在 Event 項目會發現 `Insufficient cpu (1)` 的提示訊息。

*建議方案*

對於這樣的狀況你可以

1. 降低要求的資源，並重新部署到 k8s 中，例如將 `resources.requests.cpu` 設成 `700m` 或更低
2. 請管理者增加資源

<br/>

#### 應用程式崩潰

如果部署到 k8s 的應用程式發生崩潰 (crash)，理所當然 Pod 無法正確運行。讓我們看看底下的 error-crash.yaml

```
# error-crash.yaml

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
    name: err-crash-nginx
spec:
    replicas: 1
    template:
      metadata:
        labels: 
          app: nginx
      spec:
        containers:
        - name: nginx
          image: jlptf/crash-nginx <=== 預先做好會發生崩潰的映像檔
          ports:
          - containerPort: 80
``` 

`jlptf/crash-nginx` 是一個已經預先做好會發生崩潰的映像檔，將它部署到 k8s 中，並查看 Pod 運行狀態

```bash
$ kubectl apply -f error-crash.yaml
deployment "err-crash-nginx" created

$ kubectl get pods
NAME                               READY     STATUS             RESTARTS   AGE
err-crash-nginx-7d755dd668-rc6dc   0/1       CrashLoopBackOff   2          7m
``` 

這邊會看到 `CrashLoopBackOff` 顯示 k8s 嘗試啟動 Pod 但是發生了崩潰的情況。接著檢查 Pod 

```bash
$ kubectl describe pods err-crash-nginx-7d755dd668-rc6dc
...
Containers:
  nginx:
    Container ID:   docker://4fec40ee33ae7736aeb2d6a289eb8939fac15eb42989a224e5906ac4bbc65b38
    Image:          jlptf/crash-nginx
    Image ID:       docker-pullable://jlptf/crash-nginx@sha256:98c0655b12f4e7eb8966887504ed8706f11bcaf00586a87642442f3612cef6d7
    Port:           80/TCP
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated   <=== 狀態顯示終止了
      Reason:       Error
      Exit Code:    1
      Started:      Tue, 09 Jan 2018 10:11:07 +0800
      Finished:     Tue, 09 Jan 2018 10:11:07 +0800
    Ready:          False
    Restart Count:  4
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-ld9jt (ro)
...
```

> 請記得將 `err-crash-nginx-7d755dd668-rc6dc` 換成正確名稱

你會在 `Containers` 的項目看到 `Last State: Terminated` 表示 Pod 崩潰造成運作不正確。

*建議方案*

在實務上，應用程式崩潰的原因有很多，可能是存取不正確的記憶體也有可能是某個必要的檔案不存在等等。一般來說，當遇到崩潰的問題的時候，會利用 log 幫助追蹤問題。

在 k8s 中你可以

```bash
$ kubectl logs [Pod Name] [namespace]
```

例如

```bash
$ kubectl logs err-crash-nginx-7d755dd668-rc6dc
2018/01/09 02:12:02 [emerg] 1#1: BIO_new_file("/etc/nginx/ssl/tls.crt") failed (SSL: error:02001002:system library:fopen:No such file or directory:fopen('/etc/nginx/ssl/tls.crt','r') error:2006D080:BIO routines:BIO_new_file:no such file)
nginx: [emerg] BIO_new_file("/etc/nginx/ssl/tls.crt") failed (SSL: error:02001002:system library:fopen:No such file or directory:fopen('/etc/nginx/ssl/tls.crt','r') error:2006D080:BIO routines:BIO_new_file:no such file)
```

透過 log 可以發現程式崩潰就是因為檔案 `/etc/nginx/ssl/tls.crt` 不存在造成的。

> 崩潰的原因很多，這裡僅僅是舉一個簡單的例子

遇到這種狀況，請將錯誤修正後，再部署到 k8s 中即可。

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day22/](https://jlptf.github.io/ironman2018-day22/)