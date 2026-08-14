# k8s 浅入门（Apple Silicon）

# 前情提要
对互联网云服务等行业有过一定了解的朋友，肯定都知道掌握k8s容器化技术，是踏入云原生大门的敲门砖。k8s 全称 Kubernetes，是谷歌公司基于自家多年的生产经验，并结合社区中最先进的理念和实践，打造出来的一款开源系统，主要用于自动化容器化应用的部署、扩展和管理。笔者以手头上的一台 Macbook Air 为准，对 k8s 进行简单的入门学习。
# 准备工作
1. 首先选择适合你自己的一款代码编辑器，因为 k8s 的各种资源都是通过 YAML 文件定义的，而 YAML 对缩进极其严格，如果用纯终端工具编辑会比较麻烦。我自己使用的是 VS Code + YAML 扩展插件。
2. 其次还需要部署好本地的 Docker 环境，建议放弃沉重的 Docker Desktop，在 Mac 上使用更轻量、底层网络优化更好的 **OrbStack** 作为容器引擎。
3. 最后安装 k8s 的两个核心命令行工具：`kubectl` 和 `kind` 。
    可以在 Mac 系统的终端里使用 Homebrew 一键安装：
    ```Bash
    brew install kubectl kind
    ```
# 正式开始
## 拉起集群与初次部署
1. 打开 **OrbStack** 并完成首次引导，看到状态栏图标就是底层容器引擎在线，让它在后台运行即可。 
2. 选择一个本地文件夹作为作为工作目录，新建一个文件，命名为 `kind-cluster.yaml` ，填入以下内容：
    ```YAML
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
      - role: control-plane
      - role: worker
      - role: worker
    ```
3. 在终端中执行一下命令拉起集群：
    ```Bash
    kind create cluster --config kind-cluster.yaml --name mac-k8s
    ```
4. 等待进度条跑完，看到 `mac-k8s` 集群上线。接着新建一个文件，命名为 `nginx-demo.yaml` ，填入以下内容：
    ```YAML
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-nginx
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
    spec:
      selector:
        app: nginx
      ports:
        - port: 80
          targetPort: 80
    ```
5. 在终端中执行部署命令：
    ```Bash
    kubectl apply -f nginx-demo.yaml
    ```
    等待几分钟后，应用应该已经跑在 k8s 的内网里了。可以把它穿透到本地浏览器来看一眼，在终端中接着执行下面的命令：
    ```Bash
    kubectl port-forward service/nginx-service 8080:80
    ```
6. 打开Mac本地的浏览器，访问 `http://localhost:8080` 。如果看到了 "Welcome to nginx!"的类似界面，那么就代表第一个应用部署成功了。
## 核心对象与网络进阶 (Core Objects & Networking)
- 在实际生产环境中，应用代码与配置必须解耦。硬编码密码或配置文件会导致每次修改都需要重新构建镜像，这违背了 SRE 的基本运维原则。
1. 下面将使用单一 YAML 文件部署一个包含配置注入与机密管理的复合架构。
    在工作目录中新建文件并命名为 `advanced-nginx.yaml` ，填入以下内容：
    ```YAML
    apiVersion: v1
    kind: Secret
    metadata:
      name: db-secret
    type: Opaque
    stringData:
      DB_PASSWORD: "super-secret-password-2026"
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: app-config
    data:
      custom-nginx.conf: |
        server {
            listen 80;
            location / {
                default_type text/plain;
                return 200 'Configured by Kubernetes ConfigMap!\n';
            }
        }
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: advanced-nginx
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: advanced-nginx
      template:
        metadata:
          labels:
            app: advanced-nginx
        spec:
          containers:
          - name: nginx
            image: nginx:alpine
            ports:
            - containerPort: 80
            env:
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
            volumeMounts:
            - name: config-volume
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: custom-nginx.conf
          volumes:
          - name: config-volume
            configMap:
              name: app-config
    ```

| **核心组件 🧱**      | **作用与机制**                                                        |
| ---------------- | ---------------------------------------------------------------- |
| **Secret**       | 存储敏感数据（如密码）。K8s 会在底层对其进行 Base64 编码，避免在配置文件中明文暴露。                 |
| **ConfigMap**    | 存储非机密配置。此处定义了一个自定义的 Nginx 路由规则，直接返回 200 状态码和文本。                  |
| **env**          | 将 Secret 中的 `DB_PASSWORD` 键值提取，映射为容器内的 `DATABASE_PASSWORD` 环境变量。 |
| **volumeMounts** | 利用数据卷技术，将 ConfigMap 中的配置文件精准挂载并覆盖到容器内的指定路径下。                     |
2. 在终端执行部署命令：
    ```Bash
    kubectl apply -f advanced-nginx.yaml
    ```
3. 等待部署完成后，我们需要验证配置是否生效；
    要确认 `DATABASE_PASSWORD` 环境变量已被正确注入，需要**进入到这个新运行的 Pod 内部（开启一个交互式 shell)** ；
    在 Kubernetes 体系中，与 Docker 进入容器的命令非常相似，排查问题最常用的命令之一就是 `kubectl exec` ，作用是直接进入正在运行的容器内部执行操作；
    1. 获取 Pod 的精确名称
        由于 Deployment 每次创建 Pod 时都会在名字后面加上随机字符，我们需要先查出它的全名；
        在终端执行：
        ```Bash
        kubectl get pods
        ```
        在输出结果中找到一个以 `advanced-nginx-` 开头、状态为 `Running` 的 Pod 名，完整复制下来。
    2. 进入 Pod 内部
        将上一步中复制的名字替换掉下面的 `<你的Pod名字>` 并执行：
        ```Bash
        kubectl exec -it <你的Pod名字> -- sh
        ```
        `-it` 参数代表交互式，`sh` 代表我们要打开的命令行外壳程序
    3. 打印环境变量
        在容器内部的命令行执行：
        ```Bash
        echo $DATABASE_PASSWORD
        ```
        看到 `super-secret-password-2026` 的输出结果即验证成功

- 完成配置与代码解耦后， SRE 的下一个核心任务是保障应用的高可用性； Kubernetes 默认只检查容器进程是否在运行，但运行不代表能正常处理请求（例如发生了死锁或连接池耗尽）。
    为了解决这个问题， Kubernetes 引入了 **健康检查探针(Probes)** ；日常工作中最常接触的是以下两种：
    1.  **Liveness Probe (存活探针)** ：检测应用是否处于健康状态；如果探测失败， Kubernetes 会直接强制重启该容器。
    2.  **Readiness Probe (就绪探针)** ：检测应用是否准备好接收网络流量；如果探测失败， Kubernetes 会将该 Pod 从 Service 的流量分发名单中暂时剔除，直到它恢复正常。
1. 为 Nginx 应用增加一个基于 HTTP 请求的存活探针
    代码块如下：
    ```YAML
            livenessProbe:
              httpGet:
                path: /
                port: 80
              initialDelaySeconds: 5
              periodSeconds: 10
    ```
    这段代码的含义是：在容器启动 5 秒后，每隔 10 秒向容器的 80 端口发送一次 HTTP GET 请求，如果返回 200-399 的状态码，则认为容器存活。
2. 将探针代码块加入配置文件 `advanced-nginx.yaml` 
    需要明确的一点是 **探针是针对具体容器生效的** ，因此它需要作为 `nginx` 容器的属性，和 `name` 、 `image` 、 `env` 保持严格的左侧对齐；
3. 故意注入一个错误的探针配置
    光学会配置探针远远不够，更重要的是要清楚当应用真正“假死”或探针检测失败时， Kubernetes 会作何反应；
    进行一次故障演练，将 `advanced-nginx.yaml` 中的探针检测路径修改为 Nginx 并不存在的一个地址 `/healthz` ：
    ```YAML
        spec:
          containers:
          - name: nginx
            image: nginx:alpine
            ports:
            - containerPort: 80
            livenessProbe:       # 与 ports, env 等保持对齐
              httpGet:
                path: /healthz   # 故意写错路径，模拟检测失败
                port: 80
              initialDelaySeconds: 5
              periodSeconds: 10
            env:
            # ... (下方保留原有的 env 和 volumeMounts 配置不变)
    ```
    保存文件后，在终端再次执行部署命令以更新应用：
    ```Bash
    kubectl apply -f advanced-nginx.yaml
    ```
    更新完成后， Kubernetes 现在会在容器启动 5 秒后，每隔 10 秒去请求一次 `/healthz` 路径；
    由于我们之前在 ConfigMap 中并没有配置这个路径，正常情况下 Nginx 会返回 404 错误代码（不属于 200-399 的健康范围）。
4. 实际观察故障的发生
    1. Kubernetes 非常严谨，它不会因为一次网络抖动或请求失败就立刻重启容器；
        Liveness 探针默认有一个 `failureThreshold: 3` （失败阈值）的设定；这意味着，它必须连续 3 次探测失败，才会执行重启容器的操作；再结合我们前面配置的 `initialDelaySeconds: 5` 和 `periodSeconds: 10` ，大概需要经过 35 秒以上，重启机制才会被真正触发。
        等待足够的时长后，每隔十几秒执行一次以下命令：
        ```Bash
        kubectl get pods
        ```
        重点关注输出结果中以 `advanced-nginx-` 开头 Pod 的 `RESTARTS` （重启次数）列，观察数据的前后变化
    2.  Kubernetes 的滚动更新（Rolling Update）机制
        每次修改YAML后再次 `apply` 时， Deployment 并不会直接修改原来的 Pod ；为了保证服务不中断，它会启动全新的 Pod ，并逐步销毁旧的 Pod 。如果执行 `kubectl get pods` ，可能会同时看到旧 Pod （没有探针，运行正常， RESTARTS 为 0 ）和新 Pod 都在列表中，影响对故障的观察；
        这时候就需要查看 Pod 内部的日志来获取详细信息，执行以下命令：
        ```Bash
        kubectl describe pod <你的新Pod名字>
        ```
        重点关注输出结果中 `Events:` 这一段记录，找一找类似于 `Liveness probe failed` 以及包含 `404` 状态码的警告（Warning）信息
    事实上，本次的故障演练从开始就是失败的；
	不管是 `RESTARTS` 数据，还是 Pod 内部日志，从始至终都是一切正常的；
	因为我们无意中触发了 Nginx 的一个核心路由机制，这也是日常工作中排查问题时经常遇到的”配置逻辑陷阱“；
	回看一下之前 `advanced-nginx.yaml` 在 ConfigMap 里的 Nginx 配置：
	```YAML
	location / {
	    default_type text/plain;
	    return 200 'Configured by Kubernetes ConfigMap!\n';
	}
	```
	在 Nginx 的语法中， `location /` 是一个 **前缀匹配** ，意味着”匹配所有以 `/` 开头的路径“；
	所以，当探针去请求 `/healthz` 时， Nginx 依然把它匹配到了这里，并成功返回了 200 状态码
5. 重新注入一个错误的探针配置
	打开 `advanced-nginx.yaml` ，将 `livenessProbe` 下的 `port: 80` 修改为 Nginx 没有监听的端口，比如 `port: 81` ：
	```YAML
	        livenessProbe:
	          httpGet:
	            path: /healthz
	            port: 81
	```
	保存并再次执行：
	```Bash
	kubectl apply -f advanced-nginx.yaml
	```
6. 再次观察故障的发生
    等待足够的时长后，再次执行：
    ```Bash
    kubectl get pods
    ```
    或者执行：
    ```Bash
    kubectl describe pod <新Pod名字>
    ```
    这次应该就可以在输出结果中看到 `RESTARTS` 列的数据变化或者 `Events` 段记录中的 `Liveness probe failed` 报错信息
7. 观察结果的复盘
    在 `kubectl get pods` 执行后的输出结果中，以 `advanced-nginx-` 开头 Pod 的 `STATUS` 列从正常运行时的 `Running` 变成了 `CrashLoopBackOff` ；这正是我们预期的结果，同时也是日常运维中最著名的错误状态之一；
    1.  **Crash (崩溃/被杀)** ：探针连续多次无法连接到 81 端口，判定应用已死，于是 Kubelet 强制结束了该容器进程
    2.  **Loop (循环)** ： Kubernetes 的 Deployment 核心机制是”确保应用永远运行“，所以它会立刻重启容器；但探针依然会去请求 81 端口，导致再次失败被杀
    3.  **BackOff (退避延迟)** ：如果无限循环秒级重启，会极大地消耗服务器的 CPU 和磁盘 I/O ；因此， Kubernetes 引入了”指数退避“机制（等待 10 秒、 20 秒、 40 秒...直至 5 分钟）；在这个等待重启的期间， Pod 的状态就是 `CrashLoopBackOff` 
    在 `advanced-nginx.yaml` 中进行相应的修正并重新部署，使 Pod 恢复至正常运行状态

## SRE 可观测性实践 (Observability Ecosystem)
在生产环境中，应用“活着”（Running）并不代表“活得好”；如果 Nginx 突然占用了 100% 的 CPU ，或者内存泄露即将导致服务器崩溃，单靠 `kubectl get pods` 是无法提前预警的； SRE 需要一双“透视眼”，这就是可观测性。
可观测性体系通常包含三大支柱： **指标(Metrics)、日志(Logs)和链路追踪(Traces)** 。
- 先从最基础也是最重要的 **资源指标(Metrics)** 开始。
    为了让 `mac-k8s` 集群具备收集底层硬件指标的能力，我们需要安装一个基础监控插件： **Metrics Server** 。
1. 配置核心工具 Helm
    面对复杂的基础组件配置，业界标准的做法是使用 Kubernetes 的“包管理器”--- **Helm** 。
    这里通过 Homebrew 安装 Helm ，在终端中执行：
    ```Bash
    brew install helm
    ```
    安装完成后，进行一些简单的配置，在终端中依次执行：
    ```Bash
    # 1. 告诉 Helm 去哪里找 metrics-server 的安装包
    helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
    
    # 2. 更新本地的软件源列表
    helm repo update
    ```
2. 安装部署监控基座
    在终端中执行：
    ```Bash
    helm install my-metrics-server metrics-server/metrics-server \
      --namespace kube-system \
      --set args={--kubelet-insecure-tls}
    ```

|**代码片段**|**含义**|
|---|---|
|`helm install`|📦 **核心指令**：告诉 Helm（包管理器）去安装一个新的应用。|
|`my-metrics-server`|🏷️ **发布名称 (Release Name)**：你为当前这个安装实例定义的名称。后续排查日志、更新或卸载时都需要通过这个名字来指定。|
|`metrics-server/metrics-server`|🗺️ **包来源路径**：格式为 `<仓库名>/<图表名>`。它告诉 Helm 去本地缓存的 `metrics-server` 仓库中寻找同名的安装包。|
|`\`|↵ **转义符/换行符**：在 Linux/Mac 终端中，用于将一条长命令折叠成多行，提高可读性。|
|`--namespace kube-system`|📁 **命名空间隔离**：Kubernetes 的逻辑隔离区。这条指令将 Metrics Server 强制安装到专供系统底层核心组件使用的 `kube-system` 命名空间中，与你的业务应用（如 Nginx）分开。|
|`--set args={--kubelet-insecure-tls}`|⚙️ **配置覆盖**：修改默认配置。因为本地测试环境（如 OrbStack/Kind）通常没有配置权威机构颁发的安全证书，此参数指令 Metrics Server 放弃验证 Kubelet 的 TLS 证书，否则它会因安全校验失败而无法抓取任何数据。|
3. 验证监控数据
    部署完成后再等待 1-2 分钟，执行：
    ```Bash
    kubectl top pods
    ```
    输出结果中可以看到各个 Pod 所占用的 CPU 核心数和内存，就意味着 Metrics Server 已经开始正常采集和聚合底层节点的硬件状态了
4. 设置容器资源边界
    Kubernetes 资源管控依赖两个核心维度：
    -  Requests (请求值)：容器运行所需的保底资源；K8s调度器会根据这个值来计算节点剩余容量，并决定把 Pod 放到哪个物理节点上。
    -  Limits (限制值)：容器被允许使用的资源绝对上限。
    Kubernetes 资源处理机制的严格区分：
    - 内存（不可压缩资源）：一旦容器的内存使用量超过 `limits` 的绝对上限，系统内核会立刻触发 `OOMKilled` (Out Of Memory Killed) 强制终止该容器进程，随后 Kubelet 才会将其重启。
    - CPU（可压缩资源）：如果容器使用的 CPU 超过了限制，它不会被杀，而是会被 **限流(Throttling)** 。这会导致应用处理请求变慢，但进程依然存活。
    现在，将以下 `resources` 配置加入 `advanced-nginx.yaml` ，作为 `nginx` 容器的属性，与 `ports` 、 `env` 、 `livenessProbe` 保持同级别的左侧对齐：
    ```YAML
            resources:
              requests:
                memory: "64Mi"
                cpu: "50m"
              limits:
                memory: "128Mi"
                cpu: "100m"
    ```
    保存文件并执行：
    ```Bash
    kubectl apply -f advanced-nginx.yaml
    ```
5. 验证容器资源边界
    等待更新部署完成后，在终端中执行：
    ```Bash
    kubectl describe pod <你的新Pod名字>
    ```
    查看输出结果中的详细信息，找到 `Limits:` 和 `Requests:` 字段，并确认上一步设定的数值是否已经成功生效

---
<center>未完待续</center>
