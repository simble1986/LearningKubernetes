# Pod的基本概念

#### `hostAliases`测试

* Pod配置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-first-test
spec:
  hostAliases:
  - ip: 10.9.8.7
    hostnames:
    - "host.test.site"
    - "host.aliases.test.site"
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

* 查看容器内的hosts信息

```bash
$ kubectl exec my-first-test -c my-test-container1 -- cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
172.172.1.41	my-first-test

# Entries added by HostAliases.
10.9.8.7	host.test.site	host.aliases.test.site
$ kubectl exec my-first-test -c my-test-container2 -- cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
172.172.1.41	my-first-test

# Entries added by HostAliases.
10.9.8.7	host.test.site	host.aliases.test.site
```

#### `shareProcessNamespace`

##### 不配置`shareProcessNamespace`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-share-process-ns-test
spec:
  containers:
  - name: shell
    image: busybox
    imagePullPolicy: Never
    stdin: true
    tty: true
  - name: web
    image: nginx:latest
    imagePullPolicy: Never
```

* 通过shell的container查看进程

```bash
$ kubectl exec no-share-process-ns-test -c shell -- ps
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    6 root      0:00 ps
# shell的容器仅能看到自己容器内的进程
```

* container层面信息

```bash
$ docker ps
CONTAINER ID        IMAGE                                               COMMAND                  CREATED              STATUS              PORTS               NAMES
117497ca101c        2622e6cca7eb                                        "/docker-entrypoint.…"   About a minute ago   Up About a minute                       k8s_web_no-share-process-ns-test_default_94884606-ea15-4038-9fa0-0e3b11ca4f31_0
ca8b6e6a1796        1c35c4412082                                        "sh"                     About a minute ago   Up About a minute                       k8s_shell_no-share-process-ns-test_default_94884606-ea15-4038-9fa0-0e3b11ca4f31_0
eb0fd0f91677        registry.aliyuncs.com/google_containers/pause:3.2   "/pause"                 About a minute ago   Up About a minute                       k8s_POD_no-share-process-ns-test_default_94884606-ea15-4038-9fa0-0e3b11ca4f31_0
$ docker inspect ca8b6e6a1796 | grep \"Pid\"
            "Pid": 1561218,
$ docker inspect 117497ca101c | grep \"Pid\"
            "Pid": 1561252,
$ ls -l /proc/1561218/ns/
total 0
lrwxrwxrwx 1 root root 0 Jun 17 14:43 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 17 14:41 ipc -> 'ipc:[4026532625]'
lrwxrwxrwx 1 root root 0 Jun 17 14:41 mnt -> 'mnt:[4026532718]'
lrwxrwxrwx 1 root root 0 Jun 17 14:41 net -> 'net:[4026532628]'
lrwxrwxrwx 1 root root 0 Jun 17 14:41 pid -> 'pid:[4026532723]'
lrwxrwxrwx 1 root root 0 Jun 17 14:43 pid_for_children -> 'pid:[4026532723]'
lrwxrwxrwx 1 root root 0 Jun 17 14:43 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 17 14:41 uts -> 'uts:[4026532721]'
$ ls -l /proc/1561252/ns/
total 0
lrwxrwxrwx 1 root root 0 Jun 17 14:43 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 17 14:43 ipc -> 'ipc:[4026532625]'
lrwxrwxrwx 1 root root 0 Jun 17 14:43 mnt -> 'mnt:[4026532724]'
lrwxrwxrwx 1 root root 0 Jun 17 14:43 net -> 'net:[4026532628]'
lrwxrwxrwx 1 root root 0 Jun 17 14:43 pid -> 'pid:[4026532726]'
lrwxrwxrwx 1 root root 0 Jun 17 14:43 pid_for_children -> 'pid:[4026532726]'
lrwxrwxrwx 1 root root 0 Jun 17 14:43 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 17 14:43 uts -> 'uts:[4026532725]'
```

> 两个container的`pid`和`pid_for_children`不同

##### 开启shareProcessNamespace

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: share-process-ns-test
spec:
  shareProcessNamespace: true
  containers:
  - name: shell
    image: busybox
    imagePullPolicy: Never
    stdin: true
    tty: true
  - name: web
    image: nginx:latest
    imagePullPolicy: Never
```

* 通过shell的container查看进程

```bash
$ kubectl exec share-process-ns-test -c shell -- ps
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    6 root      0:00 sh
   12 root      0:00 nginx: master process nginx -g daemon off;
   39 101       0:00 nginx: worker process
   40 root      0:00 ps
```

> 可以看到，名为shell的container中也能看到nginx的进程

* container层面

```bash
$ docker ps
CONTAINER ID        IMAGE                                               COMMAND                  CREATED             STATUS              PORTS               NAMES
955d3e3b9cba        2622e6cca7eb                                        "/docker-entrypoint.…"   2 minutes ago       Up 2 minutes                            k8s_web_share-process-ns-test_default_d1b8348e-5399-4d30-9d53-de91ffdb9746_0
d065b6907db0        1c35c4412082                                        "sh"                     2 minutes ago       Up 2 minutes                            k8s_shell_share-process-ns-test_default_d1b8348e-5399-4d30-9d53-de91ffdb9746_0
79a59e6e245c        registry.aliyuncs.com/google_containers/pause:3.2   "/pause"                 2 minutes ago       Up 2 minutes                            k8s_POD_share-process-ns-test_default_d1b8348e-5399-4d30-9d53-de91ffdb9746_0
$ docker inspect 955d3e3b9cba | grep \"Pid\"
            "Pid": 1564637,
$ docker inspect d065b6907db0 | grep \"Pid\"
            "Pid": 1564603,
$ ls -l /proc/1564637/ns/
total 0
lrwxrwxrwx 1 root root 0 Jun 17 14:51 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 17 14:51 ipc -> 'ipc:[4026532625]'
lrwxrwxrwx 1 root root 0 Jun 17 14:51 mnt -> 'mnt:[4026532724]'
lrwxrwxrwx 1 root root 0 Jun 17 14:51 net -> 'net:[4026532628]'
lrwxrwxrwx 1 root root 0 Jun 17 14:51 pid -> 'pid:[4026532626]'
lrwxrwxrwx 1 root root 0 Jun 17 14:51 pid_for_children -> 'pid:[4026532626]'
lrwxrwxrwx 1 root root 0 Jun 17 14:51 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 17 14:51 uts -> 'uts:[4026532725]'
$ ls -l /proc/1564603/ns/
total 0
lrwxrwxrwx 1 root root 0 Jun 17 14:51 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 17 14:48 ipc -> 'ipc:[4026532625]'
lrwxrwxrwx 1 root root 0 Jun 17 14:48 mnt -> 'mnt:[4026532721]'
lrwxrwxrwx 1 root root 0 Jun 17 14:48 net -> 'net:[4026532628]'
lrwxrwxrwx 1 root root 0 Jun 17 14:48 pid -> 'pid:[4026532626]'
lrwxrwxrwx 1 root root 0 Jun 17 14:51 pid_for_children -> 'pid:[4026532626]'
lrwxrwxrwx 1 root root 0 Jun 17 14:51 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 17 14:48 uts -> 'uts:[4026532723]'
```

> shell的container和web的container的`pid`和`pid_for_children`都是`pid:[4026532626]`的namespace

#### `hostNetwork`练习

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: host-net-test
spec:
  hostNetwork: true
  containers:
  - name: shell
    image: busybox
    imagePullPolicy: Never
    stdin: true
    tty: true
  - name: web
    image: nginx:latest
    imagePullPolicy: Never
```

```bash
$ kubectl exec host-net-test -c shell -- ifconfig
cni0      Link encap:Ethernet  HWaddr 16:80:F0:F9:A0:B4
          inet addr:172.172.1.1  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::1480:f0ff:fef9:a0b4/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:31 errors:0 dropped:0 overruns:0 frame:0
          TX packets:600 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:868 (868.0 B)  TX bytes:43824 (42.7 KiB)

docker0   Link encap:Ethernet  HWaddr 02:42:BB:9B:62:76
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          inet6 addr: fe80::42:bbff:fe9b:6276/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:19 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:1666 (1.6 KiB)

...
```

> container中看到的网卡信息与host相同

#### `lifecycle`练习

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    imagePullPolicy: Never
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```

```bash
$ kubectl exec lifecycle-demo -- cat /usr/share/message
Hello from the postStart handler
```

> 可以看到container启动后就打印了

### 小结

这章节主要是理解哪些操作是属于pod层面，而哪些操作是属于容器层面。比如hostAliases和shareProcessNamespace，都是属于整个pod层面，而imagePullPolicy和lifecycle则属于container层面。

再次引用原文中的总结：

> **凡是 Pod 中的容器要共享宿主机的 Namespace，也一定是 Pod 级别的定义**