# kubectl exec实现原理 

## docker exec

docker exec 的原理，大致概况一句话是：一个进程可以选择加入到某个进程已有的 Namespace 当中，从而达到“进入”这个进程所在容器的目的。

![image-20230807202707866](https://cn-yangrunchun-oss-typora.oss-cn-hangzhou.aliyuncs.com/typoraimage-20230807202707866.png)

Docker 容器其实就是若干进程构成，容器内 pid为1的进程为容器主进程，如果要加入 exec 运行一个命令，等于是新增一个process为/bin/bash, 其父进程为 Docker Daemon，新的process加入了容器主进程 P1 所在的隔离环境（namespaces），与 P1共享 `Network`、`Mount`、`IPC` 等所有资源，且与该容器内的进程一样受到资源限制（cgroup）。

docker exec加入已有namespace的过程：

查看现有容器的 pid

```shell
$ docker inspect --format '{{ .State.Pid }}'  4ddf4638572d25686
```

通过查看宿主机的 proc 文件，看到这个 25686 进程的所有 Namespace 对应的文件， 进程的每种 Linux Namespace，都在它对应的 /proc/{进程号}/ns 下有一个对应的虚拟文件，并且链接到一个真实的 Namespace 文件上。

```shell
$ls -l  /proc/25686/ns
lrwxrwxrwx  cgroup -> cgroup:[4026531835]
lrwxrwxrwx  ipc -> ipc:[4026532278]
lrwxrwxrwx  mnt -> mnt:[4026532276]
lrwxrwxrwx  net -> net:[4026532281]
lrwxrwxrwx  pid -> pid:[4026532279]
lrwxrwxrwx  pid_for_children -> pid:[4026532279]
lrwxrwxrwx  user -> user:[4026531837]
lrwxrwxrwx  uts -> uts:[4026532277]
```

要加入25686所在的 namespace，执行的是叫 setns()的 Linux系统调用。

```shell
int setns(int fd, int nstype);
fd: 是一个指向一个命名空间的文件描述符，位于/proc/PID/ns/目录。
nstype: 指定了允许进入的命名空间，设置为0表示允许进入所有命名空间。
int main(int argc, char *argv[]) {
    int fd;

    fd = open(argv[1], O_RDONLY);
    if (setns(fd, 0) == -1) {
        errExit("setns");
    }
    execvp(argv[2], &argv[2]); 
    errExit("execvp");
}
```

这段代码的操作就是通过 open() 系统调用打开了指定的 Namespace 文件，并把这个文件的描述符 fd 交给 setns() 使用。在 setns() 执行后，当前进程就加入了这个文件对应的 Linux Namespace 当中了。加入后，可以查看两个pid 的 namespace 是同一个。

```shell
$ ls -l /proc/28499/ns/net
lrwxrwxrwx  /proc/28499/ns/net -> net:[4026532281]

$ ls -l  /proc/25686/ns/net
lrwxrwxrwx/proc/25686/ns/net -> net:[4026532281]
```

setns 操作是docker exec 的基础，加入 namespace 之后，只需要将 std 通过 stream 暴露接口出去，被上层调用。docker 代码

```go
## moby/daemon/exec.go
attachConfig := stream.AttachConfig{
        TTY:        ec.Tty,
        UseStdin:   cStdin != nil,
        UseStdout:  cStdout != nil,
        UseStderr:  cStderr != nil,
        Stdin:      cStdin,
        Stdout:     cStdout,
        Stderr:     cStderr,
        DetachKeys: ec.DetachKeys,
        CloseStdin: true,
    }
    ec.StreamConfig.AttachStreams(&attachConfig)
    attachErr := ec.StreamConfig.CopyStreams(ctx, &attachConfig)

    // Synchronize with libcontainerd event loop
    ec.Lock()
    c.ExecCommands.Lock()
    systemPid, err := d.containerd.Exec(ctx, c.ID, ec.ID, p, cStdin != nil, ec.InitializeStdio)
    // the exec context should be ready, or error happened.
    // close the chan to notify readiness
    close(ec.Started)
```

## kubectl  exec

 在k8s中，可以使用 kubectl exec 来进入 pod 中的容器，如：

```shell
$ kubectl exec 123456-7890 -c ruby-container date
```

执行kubectl exec时首先会向 apiserver 发起请求，由 apiserver 转发给pod 所在机器上的kubelet进程，然后再转发给 runtime 的exec接口

![image-20230807203143254](https://cn-yangrunchun-oss-typora.oss-cn-hangzhou.aliyuncs.com/typoraimage-20230807203143254.png)

请求时apiserver 中可以看到这种日志

```shell
handler.go:143] kube-apiserver: POST "/api/v1/namespaces/default/pods/exec-test-nginx-6558988d5-fgxgg/exec" satisfied by gorestful with webservice /api/v1
upgradeaware.go:261] Connecting to backend proxy (intercepting redirects) https://192.168.205.11:10250/exec/default/exec-test-nginx-6558988d5-fgxgg/exec-test-nginx?command=sh&input=1&output=1&tty=1
Headers: map[Connection:[Upgrade] Content-Length:[0] Upgrade:[SPDY/3.1] User-Agent:[kubectl/v1.12.10 (darwin/amd64) kubernetes/e3c1340] X-Forwarded-For:[192.168.205.1] X-Stream-Protocol-Version:[v4.channel.k8s.io v3.channel.k8s.io v2.channel.k8s.io channel.k8s.io]]
```

```shell
HTTP 请求中包含了协议升级的请求. 101 upgrade
SPDY 允许在单个 TCP 连接上复用独立的 stdin/stdout/stderr/spdy-error 流。
```

API Server 收到请求后，找到需要转发的 node 地址，即 nodeip:10255端口，然后开始连接

```go
//GetConnectionInfo retrieves connection info from the status of a Node API object.
func (k *NodeConnectionInfoGetter) GetConnectionInfo(ctx context.Context, nodeName types.NodeName) (*ConnectionInfo, error) {
        node, err := k.nodes.Get(ctx, string(nodeName), metav1.GetOptions{})
        if err != nil {
                return nil, err
        }

       ....

        return &ConnectionInfo{
                Scheme:    k.scheme,
                Hostname:  host,
                Port:      strconv.Itoa(port),
                Transport: k.transport,
        }, nil
}
....

location, transport, err := pod.ExecLocation(r.Store, r.KubeletConn, ctx, name, execOpts)
        if err != nil {
                return nil, err
        }
        return newThrottledUpgradeAwareProxyHandler(location, transport, false, true, true, responder), nil
```

kubelet 接到 apiserver 的请求后，调用各运行时的RuntimeServiceServer的实现，包括 exec 接口的实现，给 apiserver返回一个连接端点

```go
 // Exec prepares a streaming endpoint to execute a command in the container.
Exec(context.Context, *ExecRequest) (*ExecResponse, error)
```

最后，容器运行时在工作节点上执行命令，如：

```go
if v, found := os.LookupEnv("XDG_RUNTIME_DIR"); found {
                execCmd.Env = append(execCmd.Env, fmt.Sprintf("XDG_RUNTIME_DIR=%s", v))
        }
        var cmdErr, copyError error
        if tty {
                cmdErr = ttyCmd(execCmd, stdin, stdout, resize)
        } else {
                if stdin != nil {
```

![image-20230807203405867](https://cn-yangrunchun-oss-typora.oss-cn-hangzhou.aliyuncs.com/typoraimage-20230807203405867.png)

kubectl exec 流程完成。

# scheduler的水平扩展 

这块没有了解过在网上找的教程，

<https://www.cnblogs.com/gaoyuechen/p/16528329.html>

# statefulset/deployment如何滚动升级 

## deployment 滚动升级

```yaml
# 升级前yaml
# nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  nginx-deploy
  namespace: default
spec:
  replicas: 3  # 期望的 Pod 副本数量，默认值为1
  selector:  # Label Selector，必须匹配 Pod 模板中的标签
    matchLabels:
      app: nginx
  template:  # Pod 模板
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

#更新后yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  nginx-deploy
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  strategy:
    type: RollingUpdate  # 指定更新策略：RollingUpdate和Recreate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
#后前面相比较，除了更改了镜像之外，还指定了更新策略：
minReadySeconds: 5
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1

```

1、minReadySeconds：表示 Kubernetes 在等待设置的时间后才进行升级，如果没有设置该值，Kubernetes 会假设该容器启动起来后就提供服务了，如果没有设置该值，在某些极端情况下可能会造成服务不正常运行，默认值就是0。
2、type=RollingUpdate：表示设置更新策略为滚动更新，可以设置为Recreate和RollingUpdate两个值，Recreate表示全部重新创建，默认值就是RollingUpdate。
3、maxSurge：表示升级过程中最多可以比原先设置多出的 Pod 数量，例如：maxSurage=1，replicas=5，就表示Kubernetes 会先启动一个新的 Pod，然后才删掉一个旧的 Pod，整个升级过程中最多会有5+1个 Pod。
4、maxUnavaible：表示升级过程中最多有多少个 Pod 处于无法提供服务的状态，当maxSurge不为0时，该值也不能为0，例如：maxUnavaible=1，则表示 Kubernetes 整个升级过程中最多会有1个 Pod 处于无法服务的状态。

```shell
#直接更新上面的 Deployment 资源对象
kubectl apply -f nginx-deploy.yaml
# kubectl rollout status 命令来查看此次滚动更新的状态
kubectl rollout status deployment/nginx-deploy
```

## statefulset 滚动更新

statefulset 支持两种升级策略：`onDelete` 和 `RollingUpdate`，同样可以通过设置 `.spec.updateStrategy.type` 进行指定。

- `OnDelete`: 该策略表示当更新了 `StatefulSet` 的模板后，只有手动删除旧的 Pod 才会创建新的 Pod。
- `RollingUpdate`：该策略表示当更新 StatefulSet 模板后会自动删除旧的 Pod 并创建新的Pod，如果更新发生了错误，这次“滚动更新”就会停止。不过需要注意 StatefulSet 的 Pod 在部署时是顺序从 0~n 的，而在滚动更新时，这些 Pod 则是按逆序的方式即 n~0 一次删除并创建。

另外`SatefulSet` 的滚动升级还支持 `Partitions`的特性，可以通过`.spec.updateStrategy.rollingUpdate.partition` 进行设置，在设置 partition 后，SatefulSet 的 Pod 中序号大于或等于 partition 的 Pod 会在 StatefulSet 的模板更新后进行滚动升级，而其余的 Pod 保持不变，statefulset 很少更新故只写大概流程

# pause容器用处

Pod 是一组紧密关联的`容器集合`，它们共享 PID、IPC、Network 和 UTS namespace，是 Kubernetes 调度的`基本单位`。Pod 的设计理念是支持多个容器在一个 Pod 中共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。容器本质上就是进程，那么 Pod 实际上就是进程组了，只是这一组进程是作为一个整体来进行调度的。

# 蓝绿发布，金丝雀发布



## 蓝绿部署

一些应用程序只需要部署一个新版本，并需要立即切到这个版本。因此，我们需要执行蓝/绿部署。在进行蓝/绿部署时，应用程序的一个新副本（绿）将与现有版本（蓝）一起部署。然后更新应用程序的入口/路由器以切换到新版本（绿）。然后，您需要等待旧（蓝）版本来完成所有发送给它的请求，但是大多数情况下，应用程序的流量将一次更改为新版本；Kubernetes不支持内置的蓝/绿部署。目前最好的方式是创建新的部署，然后更新应用程序的服务（如service）以指向新的部署；蓝绿部署是不停老版本，部署新版本然后进行测试，确认OK后将流量逐步切到新版本。蓝绿部署无需停机，并且风险较小。

## 金丝雀发布

金丝雀发布一般是先发1台机器，或者一个小比例，例如2%的服务器，主要做流量验证用，也称为金丝雀 (Canary) 测试，国内常称灰度测试。以前旷工下矿前，会先放一只金丝雀进去用于探测洞里是否有有毒气体，看金丝雀能否活下来，金丝雀发布由此得名。简单的金丝雀测试一般通过手工测试验证，复杂的金丝雀测试需要比较完善的监控基础设施配合，通过监控指标反馈，观察金丝雀的健康状况，作为后续发布或回退的依据。如果金丝测试通过，则把剩余的 V1 版本全部升级为 V2 版本。如果金丝雀测试失败，则直接回退金丝雀，发布失败。



# 优雅删除pod

## 发现问题

对于Kubernetes Deployment的每次部署过程，都是新版本的Pod创建、老版本的Pod删除的过程。

![image-20230807205159516](https://cn-yangrunchun-oss-typora.oss-cn-hangzhou.aliyuncs.com/typoraimage-20230807205159516.png)

在这个过程中如果不使用优雅退出，则会引发两个问题：

- 问题1：可能会出现Pod未将正在处理的请求处理完成的情况下被删除，如果该请求不是幂等性的，则会导致状态不一致的bug。
- 问题2：可能会出现Pod已经被删除，Kubernetes仍然将流量导向该Pod，从而出现用户请求处理失败，带来比较差的用户体验。

### 分析问题

在Kubernetes Pod 的删除过程中，同时会存在两条并行的时间线，如下图所示，其中一条时间线是网络规则的更新过程，另一条时间线是Pod的删除过程。

![image-20230807205306797](https://cn-yangrunchun-oss-typora.oss-cn-hangzhou.aliyuncs.com/typoraimage-20230807205306797.png)

当用户执行 `kubectl delete pod` 命令时，

网络规则生效流程：

1. Kube-apiserver会收到Pod删除的请求，在Etcd中更新Pod的状态为Terminating；
2. Endpoint Controller将该Pod的ip从Endpoint对象中删除；
3. Kube-proxy根据Endpoint对象的改变更新iptables规则，不再将流量路由到被删除的Pod。

Pod删除流程：

1. Kube-apiserver 会收到Pod删除的请求，在Etcd中更新Pod的状态为Terminating；
2. Kubelet在节点上清理容器的相关资源，例如存储，网络；
3. Kubelet发送SIGTERM进程给容器，如果容器中的进程未做任何配置，则容器立即退出；
4. 如果容器未在默认的30秒时间内退出，Kubelet发送SIGKILL给容器，强制让容器退出。

从Pod的删除过程可以知道，如果不对容器内的进程进行任何配置，容器会立即退出，会导致问题1出现。

由于网络规则的更新和Pod的删除是并行的，所以并不能保证网络规则的更新时间一定会早于Pod的删除时间，所以，有可能出现问题2。

## 解决问题

如果要解决以上两个问题，需要做如下配置

1. 设置容器中进程的优雅退出；
2. 增加preStopHook；
3. 修改terminationGracePeriodSeconds。

配置后的时间线如下图所示：

![image-20230807205436435](https://cn-yangrunchun-oss-typora.oss-cn-hangzhou.aliyuncs.com/typoraimage-20230807205436435.png)

### 设置容器中进程的优雅退出

在spring boot项目配置文件中加入配置

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

进行该配置后，Springboot会保证在接收到SIGTERM之后不会再接受新的请求[^1]， 并在超时时间内处理完所有正在处理的请求，如果不能处理完成，也会打印出相应的信息并强制退出。超时时间的具体值应该参考系统中最大允许的请求时长，所以理论上所有的请求都应该在30s内处理完，对于没有在30s内处理完成的请求，我们可以通过监控日志然后发Alert的方式，根据实际情况去处理。通过增加此配置，可以解决问题1。对于使用其它的语言和框架的项目，应该也存在类似的配置

### 增加preStopHook

针对问题2，需保证网络规则更新后，也就是说新的流量不再路由到要删除的Pod后，再开始Pod的删除。所以需要在 Kubernetes 的Yaml文件中增加 preStopHook,让Kubelet接收到Pod删除事件后等待一段时间，给Kube-proxy足够的时间更新iptables网络规则后，再开始删除Pod。

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 10"]  # set prestop hook
```

这里在项目中设置的10s是参照Springboot官网的配置

### 修改terminationGracePeriodSeconds

参照之前分析的Pod删除的流程，Kubernetes会给容器最大的删除时长是30秒[^3]，如果我们在Spring中优雅退出的超时时长和在Kubernetes中的preStopHook时长大于30s，则可能会出现Springboot还未处理完所有的请求Kubernetes已经开始强制删除容器。所以如果这个时长大于30秒，我们需要修改 terminationGracePeriodSeconds使其大于Springboot的优雅退出超时时间和preStopHook之和

```yaml
terminationGracePeriodSeconds: 45   
```

最终Kubernetes中的Yaml文件如下所示：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graceful-shutdown-test-exit-graceful-30s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: graceful-shutdown-test-exit-graceful-30s
  template:
    metadata:
      labels:
        app: graceful-shutdown-test-exit-graceful-30s
    spec:
      containers:
        - name: graeceful-shutdonw-test
          image: graceful-shutdown-test-exit-graceful-30s:latest
          ports:
            - containerPort: 8080
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]  # set prestop hook
      terminationGracePeriodSeconds: 45    # terminationGracePeriodSeconds
```

可以看出，通过设置Springboot的优雅退出，保证了正在处理的请求能够处理完成，通过设置preStopHook，保证了Pod删除和网络规则更新的时序关系。通过配置terminationGracePeriodSeconds，给了容器中进程足够的时间处理所有的请求。综合以上三个步骤，可以解决之前发现的两个问题。

