

### Kubernetes 概念

Kubernetes 是自动化容器编排的开源平台

<img src="pictures/image-20210622190228210.png" alt="image-20210622190228210" style="zoom: 67%;" />

通过部署Nginx服务来说明Kubernetes系统各个组件调用关系：

1. 首先需要明确，一旦Kubernetes环境启动之后，master和node都会将自身的信息存储到etcd数据库中。
2. Nginx服务的安装请求首先会被发送到master节点上的API Server组件。
3. API Server组件会调用Scheduler组件来决定到底应该把这个服务安装到那个node节点上。此时，它会从etcd中读取各个node节点的信息，然后按照一定的算法进行选择，并将结果告知API Server。
4. API Server调用Controller-Manager去调用Node节点安装Nginx服务。
5. Kubelet接收到指令后，会通知Docker，然后由Docker来启动Nginx的Pod。Pod是Kubernetes的最小操作单元，容器必须跑在Pod中。
6. Nginx服务就运行了，如果需要访问Nginx，就需要通过kube-proxy来对Pod产生访问的代理，这样，外界用户就可以访问集群中的Nginx服务了。



#### **Master**

```
- Master 集群控制平台，负责协调集群中的所有活动。
- Master 组件包括：
	- apiserver：提供 Kubernetes API, 是控制平台的前端接口
	- etcd：存储配置信息和集群状态数据
	- kube-scheduler：提供调度服务，选择合适的节点执行Pod
	- controller：控制器，监控 apiserver 当前状态，不断修正接近用户yaml文件声明的期望值
        - kube-controller-manager：统一运行控制器，包括
            - Node controller：负责监听节点停机事件
            - Replicas controller：负责维护期望的Pod副本数
            - Endpoints controller：负责为端点对象（连接Service和Pod）赋值
            - Namespace controller：负责为新的名称空间创建 Service Account 和 API Access Token
        - cloud-controller-manager：由云产商定制的控制器
            - route controller
            - service controller
```

#### **Worker/Node**  

``` 
- 节点是虚拟机或物理计算机，充当k8s集群中的工作计算机。（外网IP）
- Node 组件包括：
	- kubelet：管理Worker节点并负责与Master节点通信，管理 Pod 内运行的 Container 。
	- kube-proxy：维护节点的网络规则，实现Pod网络通信，是实现Service的重要部分
	- container-engine：容器引擎，如 Docker cri-o rktlet
```

<img src="pictures/image-20210606215411690.png" alt="image-20210606215411690" style="zoom:80%;" />

#### **Pod**

```
- Pod（容器组）是 k8s 集群上的最基本的单元。（内网IP）
- 存放一组 container（可包含一个或多个 container 容器，即图上正方体。
- 存放 container 容器的共享资源，包括：
	- 共享存储，称为 Volumes 卷，即图上紫色圆柱
	- 网络，每个 Pod（容器组）在集群中有个唯一的 IP，pod（容器组）中的 container（容器）共享该IP地址
	- container（容器）的基本信息，例如容器的镜像版本，对外暴露的端口
```

> Node             :  物理机
> Pod                :  虚拟机
> Container     :  容器

<img src="https://kuboard.cn/assets/img/module_03_nodes.38f0ef71.svg" width=50% />

#### **Deployment**

```
- Pod 控制器：Deployment
- 在kubernetes中，Pod是最小的控制单元，但是kubernetes很少直接控制Pod，都是通过Pod控制器来完成
```

![](https://kuboard.cn/assets/img/module_02_first_app.25e902c4.svg)

+ **操作：部署 Deployment**

``` yaml
- 创建文件 nginx-deployment.yaml
apiVersion: apps/v1	
kind: Deployment 	
metadata:	        
  name: nginx-deployment	
  labels:	    
    app: nginx	
spec:	        # Deployment specific
  replicas: 1	
  selector:	   
    matchLabels: 
      app: nginx
  template:	    # Pod template
    metadata:	
      labels:	
        app: nginx
    spec:	    # Pod specific
      containers:	
      - name: nginx
        image: nginx:1.7.9
```

#### **Service** 

``` 
- Pod IP会随着 Pod 的重建产生变化，同时 Pod IP仅仅是集群内部可见的虚拟的IP，外部无法访问。
- Service 屏蔽后端系统的 Pod 在销毁、创建过程中所带来的 IP 地址的变化, 实现对前端透明统一。
- Service 使 Pod 之间的相互依赖解耦（原本从一个 Pod 中访问另外一个 Pod，需要知道对方的 IP 地址）.
- Service 使用 Labels、LabelSelector 匹配一组 Pod.
```



<img src="pictures/image-20210622200224118.png" alt="image-20210622200224118" style="zoom: 67%;" />



#### **LabelSelector** 

```
- Service 通过 LabelSelector 将多组 Pod 统一入口，把这些 Pod 的指定端口公布到到集群外部，并支持负载均衡和服务发现。
- 上图中有两个服务Service A(黄色虚线)和Service B(蓝色虚线)
- Service A 将请求转发到 IP 为 10.10.10.1 的Pod上， Service B 将请求转发到 IP 为 10.10.10.2、10.10.10.3、10.10.10.4 的Pod上。
```

<img src="https://kuboard.cn/assets/img/module_04_labels.3255e3d0.svg" width="50%" />

#### **Lable**

```
- Labels（标签）是附加到 Kubernetes 对象（Node、Deployment、Pod、Service等） 的键/值对
	- 将 Kubernetes 对象指派用于开发、测试、生产环境
	- 嵌入版本标签，使用标签区别不同应用软件版本
	- 使用标签对 Kubernetes 对象进行分类

- 上图体现了 Labels（标签）和 LabelSelector（标签选择器）之间的关联关系
    - Deployment B 含有 LabelSelector 为 app=B, 通过此方式，将含有 app=B 标签的 Pod 与之关联
    - 通过 Deployment B 创建的 Pod 包含标签为 app=B
    - Service B 通过标签选择器 app=B 选择可以路由的 app=B 的 Pod
```



+ **操作：为 Deployment 创建 Service**

``` yaml
- 创建文件 nginx-deployment.yaml
apiVersion: apps/v1	
kind: Deployment	# Deployment template
metadata:	       
  name: nginx-deployment	
  labels:	    
    app: nginx	
spec:	         # Deployment specific
  replicas: 1	
  selector:	    
    matchLabels: 
      app: nginx
  template:	     # Pod template
    metadata:	
      labels:	
        app: nginx
    spec:	     # Pod specific
      containers:	
      - name: nginx	
        image: nginx:1.7.9	

- 创建文件 nginx-service.yaml
apiVersion: v1
kind: Service      # Service template
metadata:
  name: nginx-service	
  labels:     	
    app: nginx	
spec:	    # Service specific
  selector:	   
    app: nginx	
  ports:
  - name: nginx-port	
    protocol: TCP	    
    port: 80	        # 集群内的其他容器组可通过 80 端口访问 Service
    nodePort: 32600   # 外部通过主机32600端口访问 Service
    targetPort: 80	# 将请求转发到匹配 Pod 的 80 端口
  type: NodePort	

# ClusterIP：默认值，它是kubernetes系统自动分配的虚拟IP，只能在集群内部访问。
# NodePort：将Service通过指定的Node上的端口暴露给外部，通过此方法，就可以在集群外部访问服务。
# LoadBalancer：使用外接负载均衡器完成到服务的负载分发，注意此模式需要外部云环境的支持。
# ExternalName：把集群外部的服务引入集群内部，直接使用。

- 启动 service 服务进程
kubectl apply -f nginx-service.yaml
- 访问服务
curl [anyNode ip]:32600
```

#### Namespaces

```
命名空间，可对资源进行分组隔离
```

#### **Scaling**

``` 
- 我们创建了一个 Deployment，然后通过 Service 服务 提供访问 Pod 的方式。我们发布的 Deployment 只创建了一个 Pod 来运行我们的应用程序。当流量增加时，我们需要对应用程序进行伸缩操作以满足系统性能需求。
- 下图所示：Service A 只将访问流量转发到 IP 为 10.0.0.5 的Pod上，修改了 Deployment 的 replicas 为 4 后，Kubernetes 又为该 Deployment 创建了 3 新的 Pod，这 4 个 Pod 有相同的标签。因此Service A通过标签选择器与新的 Pod建立了对应关系，将访问流量通过负载均衡在 4 个 Pod 之间进行转发。
```

<img src="https://kuboard.cn/assets/img/module_05_scaling1.d9d22450.svg" width="50%" />

<img src="https://kuboard.cn/assets/img/module_05_scaling2.3f74dfba.svg" width="50%" />

+ **操作：将 Deployment 扩容到 4 个副本**

``` yaml
- 通过更改 Deployment 中的 replicas（副本数）来完成扩展
spec:
  replicas: 4    #使用该Deployment创建两个应用程序实例
```

#### **Rolling Update**

``` 
- 通过使用新版本的 Pod 逐步替代旧版本的 Pod 来实现 Deployment 的更新, 这个过程中，Service 能够监视 Pod 的状态，将流量始终转发到可用的 Pod 上。
- Rolling Update 支持：
	- 将应用程序从准上线环境升级到生产环境（通过更新容器镜像）
	- 回滚到以前的版本
	- 持续集成和持续交付应用程序，无需停机
- Rolling Update 流程：
	- 更新完 Deployment 部署文件中的镜像版本后，master 节点选择了一个 worker 节点，并根据新的镜像版本创建 Pod（紫色容器）。新 Pod 拥有唯一的新的 IP。同时，master 节点选择一个旧版本的 Pod 将其移除。重复以上操作直至所有 Pod 均替换。
```

<img src="https://kuboard.cn/assets/img/module_06_rollingupdates1.3f74dfba.svg" width="50%" />

<img src="https://kuboard.cn/assets/img/module_06_rollingupdates2.62afd5b1.svg" width="50%" />

+ **操作：Deployment 滚动更新** 

``` yaml
- Deployment 运行多个应用程序实例 (replicas)，就可以在不停机的情况下执行滚动更新
- 更新 Image 配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 4
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
        image: nginx:1.8   #使用镜像nginx:1.8替换原来的nginx:1.7.9
        ports:
        - containerPort: 80

- 查看 Pod 替换过程
watch kubectl get pods -l app=nginx
```



### Kubernetes 组件

<img src="https://kuboard.cn/assets/img/image-20190910232615991.12423fd9.png" alt="Kubernetes教程：控制器概念结构" style="zoom:45%;" />

#### Object

```yaml
K8s Objects指集群中的实体 .yaml 数据描述
每个 K8s Object 包含 spec（目标状态） 和 status（实际状态）
由 Controller 动态修正，利用负反馈原理不断使 实际状态 接近 目标状态

apiVersion: v1
kind: Pod
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    ports:
    - containerPort: 80
```

+ **标签和选择器**

  ```yaml
  # 标签（Label）是Kubernetes对象上的一组键值对，标签可以标识Kubernetes对象
  apiVersion: v1
  kind: Pod
  metadata:
    name: label-demo
    labels:
      environment: production
      app: nginx
  
  # 标签选择器（Label Selector）可以分类和选择相同标签的一组Kubernetes对象
  apiVersion: v1
  kind: Pod
  metadata:
    name: cuda-test
  spec:
    containers:
      - name: cuda-test
        image: "k8s.gcr.io/cuda-vector-add:v0.1"
        resources:
          limits:
            nvidia.com/gpu: 1
    nodeSelector:     #  Pod 可以被调度到包含标签 accelerator = nvidia-tesla-p100 的节点上
      accelerator: nvidia-tesla-p100
  ```

+ **注解**

  ```yaml
  # 注解（annotation）可以为对象添加一些元数据，用于客户端读取
  apiVersion: v1
  kind: Pod
  metadata:
    name: annotations-demo
    annotations:
      imageregistry: "https://hub.docker.com/"
  ```



#### Pod

```
Pod 是 Kubernetes 中最小调度单元
Pod 为 组内容器 提供两种共享资源，包括 网络和存储：
	- 网络：每个 Pod 分配独立的 IP地址；Pod中所有容器共享网络名称空间，容器IP相同，端口不同，可用localhost:port通信
	- 存储：Pod 定义共享数据卷，组内容器可以访问共享数据。
```

```
每个 Pod 中都包含一个或者多个容器，这些容器可以分为两类：
- 应用容器：用户程序所在的容器，数量可多可少
- 根容器：每个 Pod 都会有的一个根容器，作用有两个
	- 可以以它为依据，评估整个Pod的健康状况。
	- 可以在根容器上设置IP地址，其它容器都共享此IP（Pod的IP）以实现Pod内部的网络通信（采用虚拟二层网络技术来实现）
```



+ **相关定义**

  ``` yaml
  apiVersion: v1     # 必选，版本号，例如v1
  kind: Pod       　 # 必选，资源类型，例如 Pod
  metadata:       　 # 必选，元数据
    name: string     # 必选，Pod名称
    namespace: string  # Pod所属的命名空间,默认为"default"
    labels:       　　  # 自定义标签列表
      key: value      　
  spec:  # 必选，Pod定义
    # 容器列表
    containers:
      # 容器基本配置
      - name: string   # 必选，容器名称
        image: string  # 必选，容器的镜像名称
        imagePullPolicy: [ Always|Never|IfNotPresent ]  # 获取镜像的策略 
        command: [string]   # 容器的启动命令列表，如不指定，使用打包时使用的启动命令
        args: [string]      # 容器的启动命令参数列表
        workingDir: string  # 容器的工作目录
        volumeMounts:       # 挂载到容器内部的存储卷配置
          - name: string      # 引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
            mountPath: string # 存储卷在容器内mount的绝对路径，应少于512字符
            readOnly: boolean # 是否为只读模式
        ports: # 需要暴露的端口库号列表
          - name: string        # 端口的名称
            containerPort: int  # 容器需要监听的端口号
            hostPort: int       # 容器所在主机需要监听的端口号，默认与Container相同
            protocol: string    # 端口协议，支持TCP和UDP，默认TCP
        env:   # 容器运行前需设置的环境变量列表
          - name: string  # 环境变量名称
            value: string # 环境变量的值
        # 资源限制和请求的设置 
        # k8s会根据容器跑起来后，分配的资源是否满足，将Qos分为三类 
        # [Guaranteed 全满足 | Burstable 满足一部分 | BestEffort 全不满足]
        # 一旦出现OOM，k8s为保证服务可用，会按顺序先后删除 BestEffort Burstable Guaranteed 的 Pod 
        resources: 
          limits:  # 资源限制的设置
            cpu: string     # Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
            memory: string  # 内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
          requests: # 资源请求的设置
            cpu: string    # Cpu请求，容器启动的初始可用数量
            memory: string # 内存请求,容器启动的初始可用数量
        securityContext:
          privileged: true # 使用特权模式运行容器
          
        # 容器生命周期钩子
        lifecycle: 
    		postStart: # 容器启动后立即执行此钩子,如果执行失败,会根据重启策略进行重启
    		  exec:
    		    command:
    		preStop: # 容器终止前执行此钩子,无论结果如何,容器都会终止
    		  exec:
    		    command:
              
        # 容器探测 对容器健康检查的设置，当探测无响应几次后将自动重启该容器
        livenessProbe:   # 
          exec:       　 # 方式一：exec方式
            command: [string]  
          httpGet:       # 方式二：HttpGet方式，需要制定Path、port
            path: string
            port: number
            host: string
            scheme: string
            HttpHeaders:
              - name: string
                value: string
          tcpSocket:     # 方式三：tcpSocket方式
            port: number
          initialDelaySeconds: 0       # 容器启动完成后首次探测的时间，单位为秒
          timeoutSeconds: 0    　　    # 对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
          periodSeconds: 0     　　    # 对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
          successThreshold: 0         # 连续探测成功多少次才被认定为成功。默认是1
          failureThreshold: 0         # 连续探测失败多少次才被认定为失败。默认是3。最小值是1
          
    # 初始化容器列表：    
    # 初始化容器必须运行完成直至结束，如果某个初始化容器运行失败，那么k8s需要重启它直至成功完成。
    # 初始化容器必须按照定义的顺序执行，当且仅当前一个成功之后，后面的一个才能运行
    initContainers: [属性类似于containers]        
  
    # Pod的重启策略
    restartPolicy: [Always | Never | OnFailure]
    
    # Pod调度策略
    # 方式一：定向调度 
    nodeName:  # 设置NodeName表示将该Pod调度到指定到名称的node节点上
    nodeSelector:  # 设置NodeSelector表示将该Pod调度到包含这个label的node上
    # 方式二：亲和性调度
    affinity:
      nodeAffinity:   # 以Node为目标，解决Pod可以调度到那些Node的问题
      podAffinity:  # 以Pod为目标，解决Pod可以和那些已存在的Pod部署在同一个拓扑域中的问题
      podAntiAffinity:   # 以Pod为目标，解决Pod不能和那些已经存在的Pod部署在同一拓扑域中的问题
    # 方式三：污点和容忍
    # 通过在Node上添加污点属性，来决定是否运行Pod调度过来
    # key=value:effect [PreferNoSchedule | NoSchedule | NoExecute]
    # kubectl taint node k8s-node1 k1=v1:PreferNoSchedule  设置污点
    # kubectl taint node k8s-node1 k1:PreferNoSchedule-    取消污点
    
    
    # kubectl create secret docker-registry xxx-key \
    #                       --docker-server=registry.xxxxxx.com  \
    #                       --docker-username=admin \
    #                       --docker-password=123456
    # Pull镜像时使用的 secret 名称
    imagePullSecrets:
      - name: xxx-key
      
    # 是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络  
    hostNetwork: false   
    
    # pod 定义共享存储卷列表
    volumes:   
      - name: string    # 共享存储卷名称 （volumes类型有很多种）
        emptyDir: {}       # 类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
        hostPath: string   # 类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
          path: string     # Pod所在宿主机的目录，将被用于同期中mount的目录
        secret:       　　　# 类型为secret的存储卷，挂载集群与定义的secret对象到容器内部
          scretname: string
          items:
            - key: string
              path: string
        configMap:         # 类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
          name: string
          items:
            - key: string
              path: string
  ```

  

+ **生命周期**

  ```
  Pod 生命周期包含下面过程：
      - Pod 创建
      - 运行初始化容器（init container）过程
      - 运行主容器（main container）
      - 容器启动后钩子（post start）、容器终止前钩子（pre stop）
      - 容器的存活性探测（liveness probe）、就绪性探测（readiness probe）
  	- Pod 终止
  ```

  <img src="pictures/image-20210629160545634.png" alt="image-20210629160545634" style="zoom: 50%;" />

  ```
  在整个生命周期中，Pod会出现5种状态（相位），分别如下：
  	挂起（Pending）：API Server已经创建了Pod资源对象，但它尚未被调度完成或者仍处于下载镜像的过程中。
  	运行中（Running）：Pod已经被调度到某节点，并且所有容器都已经被kubelet创建完成。
  	成功（Succeeded）：Pod中的所有容器都已经成功终止并且不会被重启。
  	失败（Failed）：所有容器都已经终止，但至少有一个容器终止失败，即容器返回了非0值的退出状态。
  	未知（Unknown）：API Server无法正常获取到Pod对象的状态信息，通常由于网络通信失败所导致。
  ```

  <img src="pictures/image-20210622190228210.png" alt="image-20210622190228210" style="zoom: 67%;" />

  ``` 
  Pod 创建过程
      1. 用户通过kubectl或其他的api客户端提交需要创建的Pod信息给API Server
      2. API Server开始生成Pod对象的信息，并将信息存入etcd，然后返回确认信息至客户端
      3. API Server开始反映etcd中的Pod对象的变化，其它组件使用watch机制来跟踪检查API Server上的变动
      4. Scheduler发现有新的Pod对象要创建，开始为Pod分配主机并将结果信息更新至API Server
      5. Node节点上的kubelet发现有Pod调度过来，尝试调度Docker启动容器，并将结果回送至API Server
      5. API Server将接收到的Pod状态信息存入到etcd中
  ```

  ``` 
  Pod 终止过程
      1. 用户向API Server发送删除Pod对象的命令
      2. API Server中的Pod对象信息会随着时间的推移而更新，在宽限期内（默认30s），Pod被视为dead
      3. 将Pod标记为terminating状态
      4. kubelet在监控到Pod对象转为terminating状态的同时，启动Pod关闭过程
      5. controller监控到Pod对象的关闭行为时，将其从所有匹配到此端点的service资源的端点列表中移除
      6. 如果当前Pod对象定义了preStop钩子处理器，则在其标记为terminating后会以同步的方式启动执行
      7. Pod对象中的容器进程收到停止信号
      8. 宽限期结束后，如果Pod中还存在运行的进程，那么Pod对象会收到立即终止的信号
      9. kubectl请求API Server将此Pod资源的宽限期设置为0从而完成删除操作，此时Pod对于用户已经不可用了
  ```

  

#### Controller

```
在k8s中，按照 Pod的创建方式可以将其分为两类：
	1. 自主式Pod：k8s直接创建出来的Pod，此时Pod删除后就没有了，也不会重建
	2. 控制器创建Pod：通过Pod控制器创建的Pod，此时Pod删除之后还会自动重建
```

```
在k8s中，每种控制器都有自己的适合的场景，常见的有：
    1. ReplicaSet：保证指定数量的Pod运行，并支持Pod数量变更，镜像版本变更。
    2. Deployment：通过控制ReplicaSet来控制Pod，并支持滚动升级、版本回退。
    3. Horizontal Pod Autoscaler：可以根据集群负载自动调整Pod的数量，实现削峰填谷。
    4. DaemonSet：在集群中的指定Node上都运行一个副本，一般用于守护进程类的任务。
    5. Job：它创建出来的Pod只要完成任务就立即退出，用于执行一次性任务。
    6. CronJob：它创建的Pod会周期性的执行，用于执行周期性的任务。
    7. StatefulSet：管理有状态的应用。
```



+ **ReplicaSet**

  RS的主要作用是保证一定数量的Pod能够正常运行，它会持续监听这些Pod的运行状态，一旦Pod发生故障，就会重启或重建。

  同时它还支持对Pod数量的扩缩容和版本镜像的升级。 通过修改：kubectl edit rs pc-replicaset -n dev

  ```yaml
  apiVersion: apps/v1 # 版本号 
  kind: ReplicaSet # 类型 
  metadata: # 元数据 
    name: # rs名称
    namespace: # 所属命名空间 
    labels: #标签 
      controller: rs 
  spec: # 详情描述 
    replicas: 3 # 副本数量 
    selector: # 选择器，通过它指定该控制器管理哪些po
      matchLabels: # Labels匹配规则 
        app: nginx-pod 
      matchExpressions: # Expressions匹配规则 
        - {key: app, operator: In, values: [nginx-pod]} 
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本 
    metadata: 
      labels: 
        app: nginx-pod 
    spec: 
      containers: 
        - name: nginx 
          image: nginx:1.17.1 
          ports: 
          - containerPort: 80
  ```



+ **Deployment**

  Deployment管理ReplicaSet，ReplicaSet管理Pod

  除了有RS的功能之外，deploy还支持发布的停止、继续（金丝雀灰度发布），支持版本滚动更新和版本回退

  ```yaml
  apiVersion: apps/v1 # 版本号 
  kind: Deployment # 类型 
  metadata: # 元数据 
    name: # rs名称 
    namespace: # 所属命名空间 
    labels: #标签 
      controller: deploy 
  spec: # 详情描述 
    replicas: 3 # 副本数量 
    revisionHistoryLimit: 3 # 保留历史版本，默认为10 
    paused: false # 暂停部署，默认是false 
    progressDeadlineSeconds: 600 # 部署超时时间（s），默认是600 
    strategy: # 策略 
      type: RollingUpdate # 滚动更新策略 
      rollingUpdate: # 滚动更新 
        maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数 maxUnavailable: 30% # 最大不可用状态的    Pod 的最大值，可以为百分比，也可以为整数 
    selector: # 选择器，通过它指定该控制器管理哪些pod 
      matchLabels: # Labels匹配规则 
        app: nginx-pod 
      matchExpressions: # Expressions匹配规则 
        - {key: app, operator: In, values: [nginx-pod]} 
    template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本 
      metadata: 
        labels: 
          app: nginx-pod 
      spec: 
        containers: 
        - name: nginx 
          image: nginx:1.17.1 
          ports: 
          - containerPort: 80
  
  # 更新 Deployment
  kubectl set image nginx nginx=nginx:1.9.1 -record=true
  
  # 回滚 Deployment  
  1. 检查更新历史  kubectl rollout[滚动] history[历史] nginx
  2. 回滚指定版本  kubectl rollout history nginx -revision=2
  
  # 伸缩 Deployment
  kubectl scale nginx -replicas=10
  kubectl autoscale nginx -min=10 -max=15 -cpu-percent=80 （基于CPU利用率自动伸缩）
  
  # 暂停和继续 Deployment
  1. 暂停 Deployment  kubectl rollout pause[暂停] nginx
  2. 修改 Deployment（暂停状态下未更新）   kubectl set image nginx nginx=nginx:1.9.1
  3. 继续 Deployment（对修改做更新）   kubectl rollout resume[继续] nginx
  
  # 查看 Deployment 状态  
  kubectl rollout status 
      - Progressing：正在执行滚动更新
      - Complete：滚动完成
      - False：滚动失败
  ```



+ **StatefulSet**

  StatefulSet是Kubernetes提供的管理有状态应用的负载管理控制器。

  + 无状态应用：认为Pod都是一样的。没有顺序要求。不用考虑在哪个Node节点上运行。随意进行伸缩和扩展。

  + 有状态应用：有顺序的要求。认为每个Pod都是不一样的。需要考虑在哪个Node节点上运行。需要按照顺序进行伸缩和扩展。让每个Pod都是独立的，保持Pod启动顺序和唯一性。

  StatefulSet 部署需要 HeadLinessService（无头服务）

  StatefulSet中要求Pod必须是有序 ，每个Pod不能被随意取代，Pod重建后标识还是一样的，而Pod IP是变化的，所以是以Pod名称来作为唯一标志，这时候需要无头服务，它可以给每个Pod一个唯一的名称 

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: service-headliness
    namespace: dev
  spec:
    selector:
      app: nginx-pod
    clusterIP: None # 将clusterIP设置为None，即可创建headliness Service
    type: ClusterIP
    ports:
      - port: 80 # Service的端口
        targetPort: 80 # Pod的端口
  ---
  
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: pc-statefulset
    namespace: dev
  spec:
    replicas: 3
    serviceName: service-headliness
    selector:
      matchLabels:
        app: nginx-pod
    template:
      metadata:
        labels:
          app: nginx-pod
      spec:
        containers:
          - name: nginx
            image: nginx:1.17.1
            ports:
              - containerPort: 80
  ```



+ **DaemonSet**

  DS类型的控制器可以保证集群中的每一台（或指定）节点上都运行一个副本，一般适用于日志收集、节点监控等场景。

  - 每向集群中添加一个节点的时候，指定的Pod也将添加到该节点上

  - 当节点从集群中移除的时候，Pod也会被垃圾回收

  ```yaml
  apiVersion: apps/v1 # 版本号
  kind: DaemonSet # 类型
  metadata: # 元数据
    name: # 名称
    namespace: #命名空间
    labels: #标签
      controller: daemonset
  spec: # 详情描述
    revisionHistoryLimit: 3 # 保留历史版本
    updateStrategy: # 更新策略
      type: RollingUpdate # 滚动更新策略
      rollingUpdate: # 滚动更新
        maxUnavailable: 1 # 最大不可用状态的Pod的最大值，可用为百分比，也可以为整数
    selector: # 选择器，通过它指定该控制器管理那些Pod
      matchLabels: # Labels匹配规则
        app: nginx-pod
      matchExpressions: # Expressions匹配规则
        - key: app
          operator: In
          values:
            - nginx-pod
    template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
       metadata:
         labels:
           app: nginx-pod
       spec:
         containers:
           - name: nginx
             image: nginx:1.17.1
             ports:
               - containerPort: 80
  ```



+ **Job**

  Job主要用于负责批量处理短暂的一次性任务。

  - 当Job创建的Pod执行成功结束时，Job将记录成功结束的Pod数量。
  - 当成功结束的Pod达到指定的数量时，Job将完成执行。

  ```yml
  apiVersion: batch/v1 # 版本号
  kind: Job # 类型
  metadata: # 元数据
    name:  # 名称
    namespace:  #命名空间
    labels: # 标签
      controller: job
  spec: # 详情描述
    completions: 1 # 指定Job需要成功运行Pod的总次数，默认为1
    parallelism: 1 # 指定Job在任一时刻应该并发运行Pod的数量，默认为1
    activeDeadlineSeconds: 30 # 指定Job可以运行的时间期限，超过时间还没结束，系统将会尝试进行终止
    backoffLimit: 6 # 指定Job失败后进行重试的次数，默认为6
    manualSelector: true # 是否可以使用selector选择器选择Pod，默认为false
    selector: # 选择器，通过它指定该控制器管理那些Pod
      matchLabels: # Labels匹配规则
        app: counter-pod
      matchExpressions: # Expressions匹配规则
        - key: app
          operator: In
          values:
            - counter-pod
    template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
       metadata:
         labels:
           app: counter-pod
       spec:
         restartPolicy: Never # 重启策略只能设置为Never或OnFailure
         containers:
           - name: counter
             image: busybox:1.30
             command: ["/bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1;do echo $i;sleep 20;done"]
             
  # 关于模板中的重启策略的说明：
  # 如果设置为OnFailure，则Job会在Pod出现故障的时候重启容器，而不是创建Pod，failed次数不变。
  # 如果设置为Never，则Job会在Pod出现故障的时候创建新的Pod，并且故障Pod不会消失，也不会重启，failed次数+1。
  # 如果指定为Always的话，就意味着一直重启，意味着Pod任务会重复执行，这和Job的定义冲突，所以不能设置为Always。
  ```

  

+ **CronJob**

  CronJob控制器以Job控制器为其管控对象，并借助它管理Pod资源对象。

  Job定义的作业任务在其控制器资源创建之后便会立即执行，CronJob可以在特定的时间点反复去执行Job任务。

  ``` yaml
  apiVersion: batch/v1beta1 # 版本号
  kind: CronJob # 类型
  metadata: # 元数据
    name:  # 名称
    namespace:  #命名空间
    labels:
      controller: cronjob
  spec: # 详情描述
    schedule: # cron格式的作业调度运行时间点，用于控制任务任务时间执行
    concurrencyPolicy: # 并发执行策略
    failedJobsHistoryLimit: # 为失败的任务执行保留的历史记录数，默认为1
    successfulJobsHistoryLimit: # 为成功的任务执行保留的历史记录数，默认为3
    jobTemplate: # job控制器模板，用于为cronjob控制器生成job对象，下面其实就是job的定义
      metadata: {}
      spec:
        completions: 1 # 指定Job需要成功运行Pod的总次数，默认为1
        parallelism: 1 # 指定Job在任一时刻应该并发运行Pod的数量，默认为1
        activeDeadlineSeconds: 30 # 指定Job可以运行的时间期限，超过时间还没结束，系统将会尝试进行终止
        backoffLimit: 6 # 指定Job失败后进行重试的次数，默认为6
        template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
          spec:
            restartPolicy: Never # 重启策略只能设置为Never或OnFailure
            containers:
              - name: counter
                image: busybox:1.30
                command: [ "/bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1;do echo $i;sleep 20;done" ]
                
  
  # concurrencyPolicy：并发执行策略
  # Allow：运行Job并发运行（默认）。
  # Forbid：禁止并发运行，如果上一次运行尚未完成，则跳过下一次运行。
  # Replace：替换，取消当前正在运行的作业并使用新作业替换它。
  ```

  

#### Service

```
Service 解决 Deployment 管理的 Pod IP 动态变化问题
	- Pod 有各自的 IP 地址，Service 提供统一的入口地址
	- Service 通过 label selector 选定一组 Pod，并将流程负载均衡到这组 Pod 中
```

<img src="pictures/image-20210622200224118.png" alt="image-20210622200224118" style="zoom: 67%;" />

+ **kube-proxy**

  对于 Service，真正起作用的其实是 kube-proxy 服务进程。每个 Node 节点上都运行了 kube-proxy 的服务进程。当创建 Service 的时候会通过 API Server 向 etcd 写入创建的 Service 的信息，而 kube-proxy 会基于监听的机制发现这种 Service 的变化，然后它会将最新的 Service 信息转换为对应的访问规则。

  kube-proxy目前支持三种工作模式：  kubectl edit cm kube-proxy -n kube-system  -->  mode 字段

  + **userspace模式**

    + userspace模式下，kube-proxy会为每个Service创建监听端口，发向Cluster IP的请求被iptables规则重定向到kube-proxy监听的端口上，kube-proxy根据LB算法（负载均衡算法）选择提供服务的Pod并和其建立连接，以便将请求转发到Pod上。
    + 该模式下，kube-proxy充当了四层负载均衡器的角色。由于kube-proxy运行在userspace中，在进行转发处理的时候会增加内核和用户空间之间的数据拷贝，虽然比较稳定，但是效率非常低下。

    <img src="pictures/image-20210629184358341.png" alt="image-20210629184358341" style="zoom:80%;" />

  + **iptables模式 （默认）**

    + iptables模式下，kube-proxy为Service后端的每个Pod创建对应的iptables规则，直接将发向Cluster IP的请求重定向到一个Pod的IP上。
    + 该模式下kube-proxy不承担四层负载均衡器的角色，只负责创建iptables规则。该模式的优点在于较userspace模式效率更高，但是不能提供灵活的LB策略，当后端Pod不可用的时候无法进行重试。

    <img src="pictures/image-20210629184516125.png" alt="image-20210629184516125" style="zoom:80%;" />

  + **ipvs模式**

    + ipvs模式和iptables类似，kube-proxy监控Pod的变化并创建相应的ipvs规则。ipvs相对iptables转发效率更高，除此之外，ipvs支持更多的LB算法。
    + 开启需安装ipvs内核模块，否则会降级为iptables。  ipvsadm -Ln 查看映射规则

    <img src="pictures/image-20210629184637581.png" alt="image-20210629184637581" style="zoom:80%;" />



+ **yml 定义**

  ```yaml
  apiVersion: v1 # 版本
  kind: Service # 类型
  metadata: # 元数据
    name: # 资源名称
    namespace: # 命名空间
  spec:
    selector: # 标签选择器，用于确定当前Service代理那些Pod
      app: nginx
    type: ClusterIP # Service的类型，指定Service的访问方式
    clusterIP: # 虚拟服务的IP地址
    sessionAffinity: # session亲和性
    ports: # 端口信息
      - port: 8080 # Service端口
        protocol: TCP # 协议
        targetPort : # Pod端口
        nodePort:  # 主机端口
        
  # 负载分发策略 sessionAffinity
  # none: 默认使用kube-proxy的策略，比如随机、轮询等
  # ClientIP: 来自同个客户端发起的所有请求都会转发到固定的Pod上
  ```

  + **ClusterIP**

    默认值，它是kubernetes系统自动分配的虚拟IP，只能在集群内部访问。

    ``` yml
    apiVersion: v1
    kind: Service
    metadata:
      name: service-clusterip
      namespace: dev
    spec:
      selector:
        app: nginx-pod
      clusterIP: 10.97.97.97 # service的IP地址，如果不写，默认会生成一个
      type: ClusterIP
      ports:
        - port: 80 # Service的端口
          targetPort: 80 # Pod的端口
    ```

  + **HeadLiness**

    在某些场景中，开发人员可能不想使用Service提供的负载均衡功能，而希望（Controller）来控制负载均衡策略。K8s提供了HeadLinesss，此Service不会分配Cluster IP，如果想要访问Service，只能通过Service的域名进行查询。 （StatefulSet 需要）

    ``` yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: service-headliness
      namespace: dev
    spec:
      selector:
        app: nginx-pod
      clusterIP: None # 将clusterIP设置为None，即可创建headliness Service
      type: ClusterIP
      ports:
        - port: 80 # Service的端口
          targetPort: 80 # Pod的端口
    ```

  + **NodePort**

    前面两种Service只能在集群内部才能访问。如果希望Service暴露给集群外部使用，则可由 NodePort 将 Service 的端口映射到 Node 的端口上，然后就可以用 NodeIP:NodePort 来访问 Service

    <img src="pictures/image-20210629191247167.png" alt="image-20210629191247167" style="zoom:80%;" />

    ``` yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: service-nodeport
      namespace: dev
    spec:
      selector:
        app: nginx-pod
      type: NodePort # Service类型为NodePort
      ports:
        - port: 80 # Service的端口
          targetPort: 80 # Pod的端口
          nodePort: 30002 # 指定绑定的node的端口（默认取值范围是30000~32767），如果不指定，会默认分配
    ```

  + **LoadBalancer**

    LoadBalancer和NodePort很相似，目的都是向外部暴露一个端口，区别在于LoadBalancer会在集群的外部再来做一个负载均衡设备，而这个设备需要外部环境的支持，外部服务发送到这个设备上的请求，会被设备负载之后转发到集群中。

    <img src="pictures/image-20210629191541333.png" alt="image-20210629191541333" style="zoom:80%;" />

  + **ExternalName**

    ExternalName类型的Service用于引入集群外部的服务，它通过externalName属性指定一个服务的地址，然后在集群内部访问此Service就可以访问到外部的服务了。

    ``` yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: service-externalname
      namespace: dev
    spec:
      type: ExternalName # Service类型为ExternalName
      externalName: www.baidu.com # 改成IP地址也可以
    ```



+ **Ingress**

  Service对集群之外暴露服务的主要方式有两种：NodePort 和 LoadBalancer，都具有一定缺点

  + NodePort方式的缺点是会占用很多集群机器的端口
  + LoadBalancer的缺点是每个Service都需要一个LB，需要k8s之外的设备的支持

  基于这种现状，k8s提供了Ingress资源对象，Ingress只需要一个NodePort或者一个LB就可以满足暴露多个Service的需求。

  实际上，Ingress相当于一个七层的负载均衡器，是kubernetes对反向代理的一个抽象，包括：

  + Ingress：kubernetes中的对象，作用是定义请求如何转发到Service的规则。
  + Ingress Controller：具体实现反向代理及负载均衡的程序，对Ingress定义的规则进行解析，根据配置的规则来实现请求转发，实现的方式有很多，比如Nginx，Contour，Haproxy等。

  Ingress（以Nginx）的工作原理如下：

  1. 用户编写Ingress规则，说明那个域名对应kubernetes集群中的那个Service
  2. Ingress Controller动态感知Ingress服务规则的变化，然后生成一段对应的Nginx的反向代理配置
  3. Ingress Controller会将生成的Nginx配置写入到一个运行着的Nginx服务中，并动态更新
  4. 到此为止，其实真正在工作的就是一个Nginx了，内部配置了用户定义的请求规则

  <img src="pictures/image-20210629193518262.png" alt="image-20210629193518262" style="zoom:90%;" />

  + **yaml 定义**

    ``` yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: ingress-http
      namespace: dev
    spec:
      rules:
      - host: nginx.xudaxian.com
        http:
          paths:
          - path: /
            backend:
              serviceName: nginx-service
              servicePort: 80
      - host: tomcat.xudaxian.com
        http:
          paths:
          - path: /
            backend:
              serviceName: tomcat-service
              servicePort: 8080
    ```



+ **Network Policy**

  作用：网络隔离。定制网络流量 **进出 / 流出** Pod的规则，其采用的是白名单模式，符合规则的通过，不符合规则的拒绝。

  ``` yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: network-policy
    namespace: default
  spec:
    podSelector:
      matchLabels:
        key: value   # Pod 选择器
    policyTypes:
    - Ingress    # 进入 Pod 的流量规则
    - Egress     # 流出 Pod 的流量规则
    ingress:
    - from:
      - ipBlock:
          cidr: 172.17.0.0/16
          except:
          - 172.17.1.0/24
      - namespaceSelector:
          matchLabels:
            project: myproject
      - podSelector:
          matchLabels:
            role: frontend
      ports:
      - protocol: TCP
        port: 6379
    egress:
    - to:
      - ipBlock:
          cidr: 10.0.0.0/24
      ports:
      - protocol: TCP
        port: 5978
  ```

  





#### Volume

K8s数据存储：Volume是Pod中能够被多个容器访问的共享目录，挂载到Host节点上具体的文件目录下，实现持久化存储。

``` 
Volume支持多种类型，如：
    - 简单存储：EmptyDir、HostPath、NFS。
    - 高级存储：PV、PVC。
    - 配置存储：ConfigMap、Secret。
```

+ **EmptyDir**

  EmptyDir是最基础的Volume类型，一个EmptyDir就是Host上的一个空目录

  EmptyDir是在Pod被分配到Node时创建的，它的初始内容为空，并且无须指定宿主机上对应的目录文件，自动分配。当Pod销毁时，EmptyDir中的数据也会被永久删除。

  ``` yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: volume-emptydir
    namespace: dev
  spec:
    containers:
      - name: nginx
        image: nginx:1.17.1
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 80
        volumeMounts: # 将logs-volume挂载到nginx容器中对应的目录，该目录为/var/log/nginx
          - name: logs-volume
            mountPath: /var/log/nginx
      - name: busybox
        image: busybox:1.30
        imagePullPolicy: IfNotPresent
        command: ["/bin/sh","-c","tail -f /logs/access.log"] # 初始命令，动态读取指定文件
        volumeMounts: # 将logs-volume挂载到busybox容器中的对应目录，该目录为/logs
          - name: logs-volume
            mountPath: /logs
    volumes: # 声明volume，name为logs-volume，类型为emptyDir
      - name: logs-volume
        emptyDir: {}
  ```

+ **HostPath**

  EmptyDir中的数据不会被持久化，会随着Pod的结束而销毁。如果想要简单的将数据持久化到主机中，可以选择HostPath。

  ``` yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: volume-hostpath
    namespace: dev
  spec:
    containers:
      - name: nginx
        image: nginx:1.17.1
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 80
        volumeMounts: # 将logs-volume挂载到nginx容器中对应的目录，该目录为/var/log/nginx
          - name: logs-volume
            mountPath: /var/log/nginx
      - name: busybox
        image: busybox:1.30
        imagePullPolicy: IfNotPresent
        command: ["/bin/sh","-c","tail -f /logs/access.log"] # 初始命令，动态读取指定文件
        volumeMounts: # 将logs-volume挂载到busybox容器中的对应目录，该目录为/logs
          - name: logs-volume
            mountPath: /logs
    volumes: # 声明volume，name为logs-volume，类型为hostPath
      - name: logs-volume
        hostPath:
          path: /root/logs
          type: DirectoryOrCreate # 目录存在就使用，不存在就先创建再使用
  
  # type的值的说明：
  #   DirectoryOrCreate：目录存在就使用，不存在就先创建后使用。
  #   Directory：目录必须存在。
  #   FileOrCreate：文件存在就使用，不存在就先创建后使用。
  #   File：文件必须存在。
  #   Socket：unix套接字必须存在。
  #   CharDevice：字符设备必须存在。
  #   BlockDevice：块设备必须存在。
  ```

+ **NFS**

  NFS是网络文件存储系统，可以搭建NFS服务器，然后将Pod中的存储直接连接到NFS系统上，可以实现Node故障时，Pod转移到其它节点仍可访问数据。

  ``` yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: volume-nfs
    namespace: dev
  spec:
    containers:
      - name: nginx
        image: nginx:1.17.1
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 80
        volumeMounts: # 将logs-volume挂载到nginx容器中对应的目录，该目录为/var/log/nginx
          - name: logs-volume
            mountPath: /var/log/nginx
      - name: busybox
        image: busybox:1.30
        imagePullPolicy: IfNotPresent
        command: ["/bin/sh","-c","tail -f /logs/access.log"] # 初始命令，动态读取指定文件
        volumeMounts: # 将logs-volume挂载到busybox容器中的对应目录，该目录为/logs
          - name: logs-volume
            mountPath: /logs
    volumes: # 声明volume
      - name: logs-volume
        nfs:
          server: 192.168.18.100 # NFS服务器地址
          path: /root/data/nfs # 共享文件路径
  ```

  

+ **PV PVC**

  为了能够屏蔽底层存储实现的细节，方便用户使用，k8s引入了 PV 和 PVC 两种资源对象

  PV（Persistent Volume）是持久化卷的意思，是对底层的共享存储的抽象。PV由kubernetes管理员进行创建和配置，它和底层具体的共享存储技术有关，并通过插件完成和共享存储的对接。

  PVC（Persistent Volume Claim）是持久化卷声明的意思，是用户对于存储需求的声明。换言之，PVC其实就是用户向8s系统发出的资源需求申请。

  从此：存储由底层工程师维护，PV由K8s管理员维护，PVC由K8s用户维护

  <img src="pictures/image-20210629195353066.png" alt="image-20210629195353066" style="zoom: 67%;" />

  + **PV**

    ``` yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv2
    spec:
      nfs: # 存储类型，和底层正则的存储对应
        path:
        server:
      capacity: # 存储能力，目前只支持存储空间的设置
        storage: 2Gi
      accessModes: # 访问模式
        -
      storageClassName: # 存储类别
      persistentVolumeReclaimPolicy: # 回收策略
      
    # 关键配置参数说明
    # 存储类型：底层实际存储的类型，kubernetes支持多种存储类型，每种存储类型的配置有所不同
    # 存储能力（capacity）：目前只支持存储空间的设置（storage=1Gi），不过未来可能会加入IOPS、吞吐量等指标的配置。
    # 访问模式（accessModes）：用来描述用户应用对存储资源的访问权限（不同类型支持的可能不同）
    # 		ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载
    # 		ReadOnlyMany（ROX）：只读权限，可以被多个节点挂载。
    #		ReadWriteMany（RWX）：读写权限，可以被多个节点挂载
    # 回收策略（persistentVolumeReclaimPolicy）：当PV不再被使用之后，对其的处理方式（不同类型支持的可能不同）
    # 		Retain（保留）：保留数据，需要管理员手动清理数据。
    #		Recycle（回收）：清除PV中的数据，效果相当于rm -rf /volume/*
    #		Delete（删除）：和PV相连的后端存储完成volume的删除操作
    # 存储类别（storageClassName）：PV可以通过storageClassName参数指定一个存储类别
    #		具有特定类型的PV只能和请求了该类别的PVC进行绑定
    #		未设定类别的PV只能和不请求任何类别的PVC进行绑定
    # 状态（status）：一个PV的生命周期，可能会处于4种不同的阶段
    #		Available（可用）：表示可用状态，还未被任何PVC绑定
    #		Bound（已绑定）：表示PV已经被PVC绑定
    #		Released（已释放）：表示PVC被删除，但是资源还没有被集群重新释放
    #		Failed（失败）：表示该PV的自动回收失败
    ```

  + **PVC**

    ``` yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: pvc
      namespace: dev
    spec:
      accessModes: # 访客模式
        - 
      selector: # 采用标签对PV选择
      storageClassName: # 存储类别
      resources: # 请求空间
        requests:
          storage: 5Gi
    
    # 关键配置参数说明：
    # 访客模式（accessModes）：用于描述用户应用对存储资源的访问权限。
    # 选择条件（selector）：通过Label Selector的设置，可使PVC对于系统中已存在的PV进行筛选。
    # 存储类别（storageClassName）：PVC在定义时可以设定需要的后端存储的类别，只有设置了该class的pv才能被系统选出。
    # 资源请求（resources）：描述对存储资源的请求。
    ```

  + **Pod**

    ``` yaml
    ---
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv1
    spec:
      nfs: # 存储类型吗，和底层正则的存储对应
        path: /root/data/pv1
        server: 192.168.18.100
      capacity: # 存储能力，目前只支持存储空间的设置
        storage: 1Gi
      accessModes: # 访问模式
        - ReadWriteMany
      persistentVolumeReclaimPolicy: Retain # 回收策略
    
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: pvc1
      namespace: dev
    spec:
      accessModes: # 访客模式
        - ReadWriteMany
      resources: # 请求空间
        requests:
          storage: 1Gi
    
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod1
      namespace: dev
    spec:
      containers:
      - name: busybox
        image: busybox:1.30
        command: ["/bin/sh","-c","while true;do echo pod1 >> /root/out.txt; sleep 10; done;"]
        volumeMounts:
        - name: volume
          mountPath: /root/
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: pvc1
            readOnly: false
    ```

  + **生命周期**

    PVC和PV是一一对应的，PV绑定到PVC上，就会被这个PVC独占，不能再和其他的PVC进行绑定。

    <img src="pictures/image-20210629200916658.png" alt="image-20210629200916658" style="zoom:80%;" />

+ **ConfigMap**

  ConfigMap主要用来存储配置信息

  ``` yaml
  ---
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: configmap
    namespace: dev
  data:
    info:
      username:admin
      password:123456
      
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-configmap
    namespace: dev
  spec:
    containers:
      - name: nginx
        image: nginx:1.17.1
        volumeMounts:
          - mountPath: /configmap/config
            name: config
    volumes:
      - name: config
        configMap:
          name: configmap
  ```

  ```shell
  # properties文件中的每一行必须为 VAR=VAL 格式， VAR可自定义，多个英文单词.号隔开
  # enemies=aliens
  # lives=3
  # enemies.cheat=true
  # enemies.cheat.level=noGoodRotten
  
  # 从一个目录中创建ConfigMap
  kubectl create configmap cm1 --from-file=configure-pod-container/configmap/
  # 从一个文件中创建ConfigMap,并自定义ConfigMap中data.key的名称
  kubectl create configmap cm2 --from-file=keyName=configure-pod-container/configmap/game.properties
  # 从环境变量文件创建ConfigMap 
  kubectl create cm cm3 --from-env-file=configure-pod-container/configmap/env-file.properties
  # 在命令行根据键值对创建ConfigMap
  kubectl create cm cm4 --from-literal=special.how=very --from-literal=special.type=charm
  
  # 使用ConfigMap定义容器环境变量
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: test-pod
  spec:
    containers:
      - name: test-container
        image: busybox
        command: [ "/bin/sh", "-c", "env" ]
        env:
          # 定义环境变量
          - name: SPECIAL_LEVEL_KEY
            valueFrom:
              configMapKeyRef:
                # ConfigMap的名称
                name: cm4
                # ConfigMap的key 不指定则将所有键值对配置为容器环境变量
                key: special.how
    restartPolicy: Never
  
  # 使用存储在ConfigMap中的数据填充容器
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: test-pod
  spec:
    containers:
      - name: test-container
        image: busybox
        command: [ "/bin/sh", "-c", "ls /etc/config/" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
    volumes:
      - name: config-volume
        configMap:
          name: cm4
    restartPolicy: Never
  ```

+ **Secret**

  与ConfigMap类似，用来存储敏感信息（base64编码）

  ``` yaml
  ---
  apiVersion: v1
  kind: Secret
  metadata:
    name: secret
    namespace: dev
  type: Opaque
  data:  # 手动编码
    username: YWRtaW4=
    password: MTIzNDU2
  stringData: # 交给K8s编码
    username: admin
    password: 123456
    
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-secret
    namespace: dev
  spec:
    containers:
      - name: nginx
        image: nginx:1.17.1
        volumeMounts:
          - mountPath: /secret/config
            name: config
    volumes:
      - name: config
        secret:
          secretName: secret
  ```

  imagePullSecrets：拉取镜像仓库

  ``` yaml
  kubectl create secret docker-registry docker-harbor-registrykey \
  			--docker-server=ip:port \
            	--docker-username=username \
            	--docker-password=password \
            	--docker-email=email
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: redis
  spec:
    containers:
      - name: redis
        image: ip:port/uri # 这是Harbor的镜像私有仓库地址
    imagePullSecrets:
      - name: docker-harbor-registrykey
  ```

  其它操作与ConfigMap类似



#### Role

API Server是访问和管理资源对象的唯一入口。任何一个请求访问API Server，都要经过下面的三个流程

<img src="pictures/image-20210701092043043.png" alt="image-20210701092043043" style="zoom:80%;" />

+ 认证管理：

  + 通过用户名+密码识别用户
  + 通过Token来识别合法用户
  + HTTPS基于CA根证书签名的双向数字证书认证方式

  

+ 授权管理：

  + AlwaysDeny：表示拒绝所有请求，一般用于测试。

  - AlwaysAllow：允许接收所有的请求，相当于集群不需要授权流程（kubernetes默认的策略）。

  - ABAC：基于属性的访问控制，表示使用用户配置的授权规则对用户请求进行匹配和控制。

  - Webhook：通过调用外部REST服务对用户进行授权。

  - Node：是一种专用模式，用于对kubelet发出的请求进行访问控制。

  - RBAC：基于角色的访问控制（kubeadm安装方式下的默认选项）。

  

+ **RABC**（Role Based Access Control）：基于角色的访问控制，涉及以下概念：

  + 对象：User、Groups、ServiceAccount。

  - 角色：Role 代表着一组定义在资源上的可操作的动作（权限）的集合。

  - 绑定：RoleBinding 将定义好的角色和用户绑定在一起。

    <img src="pictures/image-20210701173735556.png" alt="image-20210701173735556" style="zoom: 60%;" />

  - RBAC还引入了4个顶级资源对象：

    - Role、ClusterRole：角色，用于指定一组权限。
    - RoleBinding、ClusterRoleBinding：角色绑定，用于将角色（权限的集合）赋予给对象。

  

+ **Role、ClusterRole**

  一个角色就是一组权限的集合，这里的权限都是许可形式的（白名单）

  ``` yaml
  ---
  # Role只能对命名空间的资源进行授权，需要指定namespace
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: authorization-role
    namespace: dev
  rules:
    - apiGroups: [""] # 支持的API组列表，""空字符串，表示核心API群
      resources: ["pods"] # 支持的资源对象列表
      verbs: ["get","watch","list"]
  
  ---
  # ClusterRole可以对集群范围内的资源、跨namespace的范围资源、非资源类型进行授权
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: authorization-clusterrole
  rules:
    - apiGroups: [""] # 支持的API组列表，""空字符串，表示核心API群
      resources: ["pods"] # 支持的资源对象列表
      verbs: ["get","watch","list"]
      
  
  # rules中的参数说明：
  # apiGroups：支持的API组列表。"","apps","autoscaling","batch"。
  # resources：支持的资源对象列表。"services","endpoints","pods","secrets","configmaps","crontabs","deployments","jobs","nodes","rolebindings","clusterroles","daemonsets","replicasets","statefulsets","horizontalpodautoscalers","replicationcontrollers","cronjobs"。
  # verbs：对资源对象的操作方法列表。"get", "list", "watch", "create", "update", "patch", "delete", "exec"。
  ```



+ **RoleBinding、ClusterRoleBinding**

  角色绑定用来把一个角色绑定到一个目标对象上，绑定目标可以是User、Group或者ServiceAccount。

  ``` yaml
  ---
  # RoleBinding可以将同一namespace中的subject对象绑定到某个Role下，则此Subject具有该Role定义的权限
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: authorization-role-binding
    namespace: dev
  subjects:
    - kind: User
      name: xudaxian
      apiGroup: rbac.authorization.k8s.io  
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: authorization-role
    
  ---
  # ClusterRoleBinding在整个集群级别和所有namespaces将特定的subject与ClusterRole绑定，授予权限
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: authorization-clusterrole-binding
  subjects:
    - kind: User
      name: xudaxian
      apiGroup: rbac.authorization.k8s.io
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: authorization-clusterrole
    
    
  ---
  # 虽然authorization-clusterrole是一个集群角色，但是因为使用了RoleBinding，所以xudaxian只能读取dev命名空间中的资源
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: authorization-clusterrole-binding
  subjects:
    - kind: User
      name: xudaxian
      apiGroup: rbac.authorization.k8s.io
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: authorization-clusterrole
  ```

  



###  Kubernetes 资源

#### kubectl 命令

```shell
kubectl [command] [resourceType] [name] [flags -n -A -o]   # kubectl集群命令行工具
```

1. 基本命令

   | 命令         |          | 命令作用                     |
   | ------------ | -------- | ---------------------------- |
   | create       | 创建     | 创建一个资源                 |
   | edit         | 编辑     | 编辑一个资源                 |
   | get          | 获取     | 获取一个资源                 |
   | patch        | 更新     | 更新一个资源                 |
   | delete       | 删除     | 删除一个资源                 |
   | explain      | 解释     | 展示资源文档                 |
   | cluster-info | 集群信息 | 显示集群信息                 |
   | version      | 版本     | 显示当前Client和Server的版本 |

2. 运行和调试

   | 命令      |          | 命令作用               |
   | --------- | -------- | ---------------------- |
   | run       | 运行     | 在集群中运行指定的镜像 |
   | expose    | 暴露     | 暴露资源为Service      |
   | describe  | 描述     | 显示资源内部信息       |
   | logs      | 日志     | 输出容器在Pod中的日志  |
   | attach    | 缠绕     | 进入运行中的容器       |
   | exec      | 执行     | 进入容器中执行命令     |
   | cp        | 复制     | 在Pod内外复制文件      |
   | rollout   | 首次展示 | 管理资源的发布         |
   | scale     | 规模     | 扩（缩）容Pod的数量    |
   | autoscale | 自动调整 | 自动调整Pod的数量      |
   | apply     | 应用     | 通过文件对资源进行配置 |
   | label     | 标签     | 更新资源上的标签       |



#### 资源类型

```shell
kubectl api-resources   # kubernetes中所有的内容都抽象为资源
```

1. 集群容器资源

   | 资源名称   | 缩写 | 资源作用     |
   | ---------- | ---- | ------------ |
   | nodes      | no   | 集群组成部分 |
   | namespaces | ns   | 隔离Pod      |
   | pods       | po   | 装载容器     |

2. Pod资源控制器

   | 资源名称                   | 缩写   | 资源作用    |
   | -------------------------- | ------ | ----------- |
   | replication controllers    | rc     | 控制Pod资源 |
   | replicasets                | rs     | 控制Pod资源 |
   | deployments                | deploy | 控制Pod资源 |
   | daemonsets                 | ds     | 控制Pod资源 |
   | jobs                       |        | 控制Pod资源 |
   | cronjobs                   | cj     | 控制Pod资源 |
   | horizontal pod autoscalers | hpa    | 控制Pod资源 |
   | statefulsets               | sts    | 控制Pod资源 |

3. 服务发现资源

   | 资源名称 | 缩写 | 资源作用        |
   | -------- | ---- | --------------- |
   | services | svc  | 统一Pod对外接口 |
   | ingress  | ing  | 统一Pod对外接口 |

4. 存储资源

   | 资源名称                 | 缩写 | 资源作用 |
   | ------------------------ | ---- | -------- |
   | volume attachments       |      | 存储     |
   | persistent volumes       | pv   | 存储     |
   | persistent volume claims | pvc  | 存储     |

5. 配置资源

   | 资源名称   | 缩写 | 资源作用 |
   | ---------- | ---- | -------- |
   | configmaps | cm   | 配置     |
   | secrets    |      | 配置     |



#### 资源管理方式

1. 命令式：

   kubectl create deployment nginx --image=nginx:1.17.1 --dry-run=client -n dev -o yaml

   kubectl run nginx --image=nginx:1.17.1 --port=80 --namespace=dev

2. 命令式 + 配置：kubectl create/patch -f nginx-pod.yaml

3. 声明式 + 配置： kubectl apply -f nginx-pod.yaml

