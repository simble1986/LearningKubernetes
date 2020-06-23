# 为什么需要Pod

## 共享net的container

##### 先来起两个共享net和volume的容器

```bash
$ docker run -it -d --name test1 busybox
15dafb04a3fae857c0db7ce09c02d6ebc565de71a49df2511eded48457e0945e
$ docker run -it -d --network container:test1 --volumes-from test1 --name test2 busybox
6e15995e63a4f2cc0b293be848a448c2e34a1502477645c434e529245ffaa1f3
```

##### 查看两个container的信息

```bash
$ docker inspect test1
[
    {
        "Id": "15dafb04a3fae857c0db7ce09c02d6ebc565de71a49df2511eded48457e0945e",
            ...
            "Pid": 2282,
            ...
        },
        ...
            "Networks": {
                "bridge": {
                    ...
                    "NetworkID": "5bdb44786ae497e27f9626360b59ff35eead5c02f6f6da4392306132e8910ad0",
                    "EndpointID": "8a9d1594a9949509e42eee8af6145d30e1162cc4c8384c0f7d7e8d7bece5cfdd",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                }
        ...

$ docker inspect test2
[
    {
        "Id": "6e15995e63a4f2cc0b293be848a448c2e34a1502477645c434e529245ffaa1f3",
        ...
            "Pid": 3555,
        ...
            "NetworkMode": "container:15dafb04a3fae857c0db7ce09c02d6ebc565de71a49df2511eded48457e0945e",
        ...
```

##### 查看进程

```bash
$ ls -l /proc/2282/ns/
total 0
lrwxrwxrwx 1 root root 0 Jun 12 06:45 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 12 06:31 ipc -> 'ipc:[4026532571]'
lrwxrwxrwx 1 root root 0 Jun 12 06:31 mnt -> 'mnt:[4026532569]'
lrwxrwxrwx 1 root root 0 Jun 12 06:16 net -> 'net:[4026532574]'
lrwxrwxrwx 1 root root 0 Jun 12 06:31 pid -> 'pid:[4026532572]'
lrwxrwxrwx 1 root root 0 Jun 12 06:45 pid_for_children -> 'pid:[4026532572]'
lrwxrwxrwx 1 root root 0 Jun 12 06:45 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 12 06:31 uts -> 'uts:[4026532570]'
$ ls -l /proc/3555/ns/
total 0
lrwxrwxrwx 1 root root 0 Jun 12 06:45 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 12 06:42 ipc -> 'ipc:[4026532630]'
lrwxrwxrwx 1 root root 0 Jun 12 06:42 mnt -> 'mnt:[4026532628]'
lrwxrwxrwx 1 root root 0 Jun 12 06:42 net -> 'net:[4026532574]'
lrwxrwxrwx 1 root root 0 Jun 12 06:42 pid -> 'pid:[4026532631]'
lrwxrwxrwx 1 root root 0 Jun 12 06:45 pid_for_children -> 'pid:[4026532631]'
lrwxrwxrwx 1 root root 0 Jun 12 06:45 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 12 06:42 uts -> 'uts:[4026532629]'
```

可以看到，cgroup、net和user是相同的

> 实际上，在不指定共享net的container的时候，cgroup和user也都是相同的
>
> ```bash
>$ docker run -it -d ubuntu
> 25b637a2facf0ba713403c14f300644ee21cd107b45fb57d47f7cb4be0fcc3ad
> $ docker inspect 25b637a2f | grep Pid
>          "Pid": 10895,
>          "PidMode": "",
>             "PidsLimit": null,
>    $ ls /proc/10895/ns/ -l
>    total 0
> lrwxrwxrwx 1 root root 0 Jun 15 03:10 cgroup -> 'cgroup:[4026531835]'
> lrwxrwxrwx 1 root root 0 Jun 15 03:10 ipc -> 'ipc:[4026532706]'
> lrwxrwxrwx 1 root root 0 Jun 15 03:10 mnt -> 'mnt:[4026532704]'
> lrwxrwxrwx 1 root root 0 Jun 15 03:10 net -> 'net:[4026532709]'
> lrwxrwxrwx 1 root root 0 Jun 15 03:10 pid -> 'pid:[4026532707]'
> lrwxrwxrwx 1 root root 0 Jun 15 03:10 pid_for_children -> 'pid:[4026532707]'
> lrwxrwxrwx 1 root root 0 Jun 15 03:10 user -> 'user:[4026531837]'
> lrwxrwxrwx 1 root root 0 Jun 15 03:10 uts -> 'uts:[4026532705]'
> ```

##### 查看网络

```bash
$ docker exec -it test1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:19 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1522 (1.4 KiB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

$ docker exec -it test2 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:19 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1522 (1.4 KiB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

```

两个container的eth0的IP，MAC等都相同

## Pod

##### 先起一个pod看看

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  containers:
  - name: my-test-container1
    image: busybox
    imagePullPolicy: Never
    stdin: true
    tty: true
  - name: my-test-container2
    image: busybox
    imagePullPolicy: Never
    stdin: true
    tty: true
```

> 注：因为已经pull了image，所以这里设置了`imagePullPolicy: Never`

```bash
$ kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
two-containers   2/2     Running   0          3s
```

```bash
$ docker ps
CONTAINER ID        IMAGE                                               COMMAND                  CREATED              STATUS              PORTS               NAMES
14aea3a7ec58        1c35c4412082                                        "sh"                     About a minute ago   Up About a minute                       k8s_my-test-container2_two-containers_default_dd628b40-a070-4d9d-9837-7d3031092e86_0
fb9a5a4e5ee2        1c35c4412082                                        "sh"                     About a minute ago   Up About a minute                       k8s_my-test-container1_two-containers_default_dd628b40-a070-4d9d-9837-7d3031092e86_0
bac4514284b2        registry.aliyuncs.com/google_containers/pause:3.2   "/pause"                 About a minute ago   Up About a minute                       k8s_POD_two-containers_default_dd628b40-a070-4d9d-9837-7d3031092e86_0
```

可以看到，分别起了3个container：

* k8s_my-test-container1_two-containers_default
* k8s_my-test-container2_two-containers_default
* k8s_POD_two-containers_default

其中，`k8s_POD_two-containers_default`的image是`pause`

##### 先看看namespace

```bash
$ docker inspect 14aea3a7ec58 | grep \"Pid\"
            "Pid": 359647,
$ docker inspect fb9a5a4e5ee2 | grep \"Pid\"
            "Pid": 359614,
$ docker inspect bac4514284b2 | grep \"Pid\"
            "Pid": 359530,
$ ls -l /proc/359647/ns
total 0
lrwxrwxrwx 1 root root 0 Jun 15 14:41 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 ipc -> 'ipc:[4026532625]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 mnt -> 'mnt:[4026532715]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 net -> 'net:[4026532628]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 pid -> 'pid:[4026532717]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 pid_for_children -> 'pid:[4026532717]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 uts -> 'uts:[4026532716]'
$ ls -l /proc/359614/ns
total 0
lrwxrwxrwx 1 root root 0 Jun 15 14:41 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 ipc -> 'ipc:[4026532625]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 mnt -> 'mnt:[4026532710]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 net -> 'net:[4026532628]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 pid -> 'pid:[4026532714]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 pid_for_children -> 'pid:[4026532714]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 uts -> 'uts:[4026532713]'
$ ls -l /proc/359530/ns
total 0
lrwxrwxrwx 1 root root 0 Jun 15 14:41 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 15 14:38 ipc -> 'ipc:[4026532625]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 mnt -> 'mnt:[4026532623]'
lrwxrwxrwx 1 root root 0 Jun 15 14:38 net -> 'net:[4026532628]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 pid -> 'pid:[4026532626]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 pid_for_children -> 'pid:[4026532626]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 15 14:41 uts -> 'uts:[4026532624]'
```

##### 看看IP

* Pod层面

```bash
$ kubectl describe pod two-containers
Name:         two-containers
Namespace:    default
Priority:     0
Node:         node2/10.160.18.181
Start Time:   Mon, 15 Jun 2020 14:38:10 +0800
Labels:       <none>
Annotations:  Status:  Running
IP:           172.172.1.35
IPs:
  IP:  172.172.1.35
Containers:
  my-test-container1:
    Container ID:   docker://fb9a5a4e5ee2eef15dddc159bdb507b435ebbbc189d3ceefbaca7d7d4dac994d
    Image:          busybox
    ...
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-nlz8h (ro)
  my-test-container2:
    Container ID:   docker://14aea3a7ec58e4f4220d6eea1266799a55fbdd945956d4845d4e062fce6542d7
    Image:          busybox
    ...
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-nlz8h (ro)
...
Volumes:
  default-token-nlz8h:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-nlz8h
    Optional:    false
```

* Container层面

```bash
$ docker exec -it 14aea3a7ec58 ifconfig
eth0      Link encap:Ethernet  HWaddr 26:64:FF:FA:7D:BE
          inet addr:172.172.1.35  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:15 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1198 (1.1 KiB)  TX bytes:42 (42.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

$ docker exec -it fb9a5a4e5ee2 ifconfig
eth0      Link encap:Ethernet  HWaddr 26:64:FF:FA:7D:BE
          inet addr:172.172.1.35  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:15 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1198 (1.1 KiB)  TX bytes:42 (42.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

#### Pod共享volume

##### Pod配置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers-volume
spec:
  volumes:
  - name: shared-data
    hostPath:
      path: /data
  containers:
  - name: c1
    image: busybox
    imagePullPolicy: Never
    stdin: true
    tty: true
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: c2
    image: busybox
    imagePullPolicy: Never
    stdin: true
    tty: true
    volumeMounts:
    - name: shared-data
      mountPath: /data
```

##### 查看pod信息

```bash
$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
two-containers-volume   2/2     Running   0          5s
```

```bash
$ kubectl describe pod two-containers-volume
Name:         two-containers-volume
Namespace:    default
Priority:     0
Node:         node2/10.160.18.181
Start Time:   Mon, 15 Jun 2020 14:53:43 +0800
Labels:       <none>
Annotations:  Status:  Running
IP:           172.172.1.36
IPs:
  IP:  172.172.1.36
Containers:
  c1:
    Container ID:   docker://b5b285305aae0d9155fca8415a1002697b0a446d222ce8f8bb27456c51680bc0
    Image:          busybox
    ...
    Mounts:
      /data from shared-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-nlz8h (ro)
  c2:
    Container ID:   docker://52767f5bfa9e246e79547b11bfca0eb5948f3458850beaf173fbd921e43aa949
    Image:          busybox
    ...
    Mounts:
      /data from shared-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-nlz8h (ro)
...
Volumes:
  shared-data:
    Type:          HostPath (bare host directory volume)
    Path:          /data
    HostPathType:
  default-token-nlz8h:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-nlz8h
    Optional:    false
...
```

> 对比之前的，可以看到，volumes中增加了一个`shared-data`的volume，每个container的`Mounts`中多了一个`/data from shared-data (rw)`

##### 查看container层面的信息

```bash
$ docker ps
CONTAINER ID        IMAGE                                               COMMAND                  CREATED             STATUS              PORTS               NAMES
52767f5bfa9e        1c35c4412082                                        "sh"                     6 minutes ago       Up 6 minutes                            k8s_c2_two-containers-volume_default_a66c7066-35c2-4be6-b58a-0df6ede9d95d_0
b5b285305aae        1c35c4412082                                        "sh"                     6 minutes ago       Up 6 minutes                            k8s_c1_two-containers-volume_default_a66c7066-35c2-4be6-b58a-0df6ede9d95d_0
09b081c04b1b        registry.aliyuncs.com/google_containers/pause:3.2   "/pause"                 6 minutes ago       Up 6 minutes                            k8s_POD_two-containers-volume_default_a66c7066-35c2-4be6-b58a-0df6ede9d95d_0
```

```bash
$ docker inspect 52767f5bfa9e
[
    {
        ...
        "Name": "/k8s_c2_two-containers-volume_default_a66c7066-35c2-4be6-b58a-0df6ede9d95d_0",
        ...
        "HostConfig": {
            ...
            "NetworkMode": "container:09b081c04b1ba2b973d7fb31aadd7d796d8f6aef03113764f4a868022b6332ee",
            ...
        },
        ...
        "Mounts": [
            ...
            {
                "Type": "bind",
                "Source": "/data",
                "Destination": "/data",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
            ...
        ],
        ...
    }
]
$ docker inspect b5b285305aae
[
    {
        ...
        "Name": "/k8s_c1_two-containers-volume_default_a66c7066-35c2-4be6-b58a-0df6ede9d95d_0",
        ...
        "HostConfig": {
            ...
            "NetworkMode": "container:09b081c04b1ba2b973d7fb31aadd7d796d8f6aef03113764f4a868022b6332ee",
            ...
        },
        ...
        "Mounts": [
            {
                "Type": "bind",
                "Source": "/data",
                "Destination": "/data",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
            ...
        ],
        ...
    }
]
$ docker inspect 09b081c04b1b
[
    {
        ...
        "Name": "/k8s_POD_two-containers-volume_default_a66c7066-35c2-4be6-b58a-0df6ede9d95d_0",
        ...
        "HostConfig": {
            ...
            "NetworkMode": "none",
            ...
        },
        ...
        "Mounts": [],
        ...
        "NetworkSettings": {
            ...
            "SandboxKey": "/var/run/docker/netns/cb1ed85e8f94",
            ...
            "Networks": {
                "none": {
                    ...
                    "NetworkID": "d5b75ce86af4c81134a382c1b77ed2de74e8ec9d484a68a5002f52ed3905b239",
                    "EndpointID": "f6ea091a8d1445dae3a4b961c6cfbfa05c671f604dab8c9284eca959d02e7867",
                    ...
                }
            }
        }
    }
]
```

> 说明：
>
> * `k8s_POD_two-containers-volume_default`这个container的`NetworkSettings`字段声明有网络相关内容，但Mounts为空
> * `k8s_c1`和`k8s_c2`的container的`NetworkSettings`为空，但Mounts中则挂载了`/data`目录

## Init container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-container-test
spec:
  volumes:
  - name: app-volume
    emptyDir: {}
  initContainers:
  - image: busybox
    imagePullPolicy: Never
    name: data-container
    stdin: true
    tty: true
    command: ["touch", "/data/test.txt"]
    volumeMounts:
    - mountPath: "/data"
      name: app-volume
  containers:
  - image: busybox
    imagePullPolicy: Never
    name: app-container
    stdin: true
    tty: true
    volumeMounts:
    - mountPath: "/data"
      name: app-volume
```

##### 查看pod状况

```bash
$ kubectl get pod
NAME                  READY   STATUS    RESTARTS   AGE
init-container-test   1/1     Running   0          5s
```

##### 查看container

```bash
$ docker ps
CONTAINER ID        IMAGE                                               COMMAND                  CREATED             STATUS              PORTS               NAMES
ce67fd98bc68        1c35c4412082                                        "sh"                     30 seconds ago      Up 29 seconds                           k8s_app-container_init-container-test_default_5960cae6-00b3-48ae-9991-741d73077dc4_0
e75657cb2079        registry.aliyuncs.com/google_containers/pause:3.2   "/pause"                 31 seconds ago      Up 30 seconds                           k8s_POD_init-container-test_default_5960cae6-00b3-48ae-9991-741d73077dc4_0
```

> 会发现这里只有一个`k8s_app-container`和`k8s_POD`，而没有`init-container`
>
> 是因为
>
> > 在 Pod 中，所有 Init Container 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。
>
> 但k8s并没有在这个container结束运行后重启它

```bash
$ docker ps -a
CONTAINER ID        IMAGE                                                COMMAND                  CREATED             STATUS                      PORTS               NAMES
ce67fd98bc68        1c35c4412082                                         "sh"                     36 seconds ago      Up 36 seconds                                   k8s_app-container_init-container-test_default_5960cae6-00b3-48ae-9991-741d73077dc4_0
bbc45d10357c        1c35c4412082                                         "touch /data/test.txt"   37 seconds ago      Exited (0) 37 seconds ago                       k8s_data-container_init-container-test_default_5960cae6-00b3-48ae-9991-741d73077dc4_0
e75657cb2079        registry.aliyuncs.com/google_containers/pause:3.2    "/pause"                 37 seconds ago      Up 37 seconds                                   k8s_POD_init-container-test_default_5960cae6-00b3-48ae-9991-741d73077dc4_0
```

> 从这里可以看到， `data-container`已经`Exited (0) 37 seconds ago`

```bash
$ kubectl exec init-container-test -- ls /data
test.txt
```

> 通过容器可以看到test.txt已经存在在`init-container-test`的`/data`目录下了

### 总结引用

* 一个运行在虚拟机里的应用，哪怕再简单，也是被管理在 systemd 或者 supervisord 之下的一组进程，而不是一个进程。
* 对于容器来说，一个容器永远只能管理一个进程
* Pod，实际上是在扮演传统基础设施里“虚拟机”的角色；而容器，则是这个虚拟机里运行的用户程序。