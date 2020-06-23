# 重新认识docker容器

## build一个镜像

### 创建Flask相关文件

* app.py

```python
from flask import Flask
import socket
import os

app = Flask(__name__)

@app.route('/')
def hello():
    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>"           
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname())
    
if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```

* requirements.txt

```txt
Flask
```

### 创建Dockerfile

```dockerfile
FROM python:2.7-slim

WORKDIR /app

# app.py和requirements.txt都放在当前文件夹的app目录下
ADD ./app /app

# 使用pip命令安装这个应用所需要的依赖
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# 允许外界访问容器的80端口
EXPOSE 80

# 设置环境变量
ENV NAME World

# 设置容器进程为：python app.py，即：这个Python应用的启动命令
CMD ["python", "app.py"]
```

当前目录结构

```bash
$ tree .
.
├── app
│   ├── app.py
│   └── requirements.txt
└── Dockerfile

1 directory, 3 files
```

### build docker镜像

```bash
$ docker build -t flaskapp .
```

当build镜像时：

1. docker每执行一行，会以上一层为基础拉起一个容器
2. 然后在这个容器里执行对应的命令
3. 完成后，将这一层提交成一个image

### 查看镜像的build history

```bash
$ docker history 06e1a19665ce
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
06e1a19665ce        5 weeks ago         /bin/sh -c apk update && apk add nginx          2.98MB
f70734b6a266        2 months ago        /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>           2 months ago        /bin/sh -c #(nop) ADD file:b91adb67b670d3a6f…   5.61MB
```

## docker exec的本质

### 进入一个namespace

* `set_ns.c`

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE);} while (0)

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

```bash
$ gcc -o set_ns set_ns.c
```

### 启动一个container

```bash
$ docker run -it -d --rm ubuntu
59ed1b0423ac42b5659e9c3d1759000e934c8383f605875d86db42b6ae7bf098
$ docker inspect 59ed1b0423a | grep \"Pid\"
            "Pid": 14123,
```

### 查看进程相关ns

```bash
$ ls /proc/14123/ns/ -l
total 0
lrwxrwxrwx 1 root root 0 Jun 16 07:34 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jun 16 07:34 ipc -> 'ipc:[4026532571]'
lrwxrwxrwx 1 root root 0 Jun 16 07:34 mnt -> 'mnt:[4026532569]'
lrwxrwxrwx 1 root root 0 Jun 16 07:32 net -> 'net:[4026532574]'
lrwxrwxrwx 1 root root 0 Jun 16 07:34 pid -> 'pid:[4026532572]'
lrwxrwxrwx 1 root root 0 Jun 16 07:34 pid_for_children -> 'pid:[4026532572]'
lrwxrwxrwx 1 root root 0 Jun 16 07:34 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jun 16 07:34 uts -> 'uts:[4026532570]'
```

### 以net的namespace运行ifconfig

```bash
$ ./set_ns /proc/14123/ns/net /bin/bash
$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 13  bytes 1046 (1.0 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

### 分别用set_ns和docker exec查看

```bash
$ ps -aux | grep /bin/bash
root     14123  0.0  0.3   4112  3288 pts/0    Ss+  07:32   0:00 /bin/bash
root     14682  0.0  5.8 706028 58772 pts/3    Sl+  07:47   0:00 docker exec -it 59ed1b0423ac /bin/bash
root     14698  0.0  0.3   4112  3376 pts/1    Ss+  07:47   0:00 /bin/bash
root     14722  0.0  0.3  20416  3792 pts/4    S+   07:52   0:00 /bin/bash
```

```bash
$ ls -l /proc/14123/ns/net
lrwxrwxrwx 1 root root 0 Jun 16 07:32 /proc/14123/ns/net -> 'net:[4026532574]'
$ ls -l /proc/14698/ns/net
lrwxrwxrwx 1 root root 0 Jun 16 07:49 /proc/14698/ns/net -> 'net:[4026532574]'
$ ls -l /proc/14722/ns/net
lrwxrwxrwx 1 root root 0 Jun 16 07:52 /proc/14722/ns/net -> 'net:[4026532574]'
```

> 可以看出，最终都指向了同一个net namespace

> Linux的ip命令也支持创建一个network namespace，如：
>
> ```bash
> $ ip netns add ns_test
> $ ip netns exec ns_test /bin/bash
> $ ifconfig
> # 由于并没有为这个namespace设定接口，所以，这里显示为空
> ```
>
> ```bash
> $ ps -aux | grep /bin/bash
> root     14956  0.0  0.4  20416  4056 pts/0    S+   08:33   0:00 /bin/bash
> $ ls -l /proc/14956/ns/net
> lrwxrwxrwx 1 root root 0 Jun 16 08:33 /proc/14956/ns/net -> 'net:[4026532629]'
> ```
>
> 当然，同样可以使用前面的set_ns的工具进行查看
>
> ```bash
> $ ./set_ns /proc/14956/ns/net /bin/bash
> $ ifconfig
> # 这里同样没有内容输出
> ```
>
> ```bash
> $ ps -aux | grep /bin/bash
> root     14956  0.0  0.4  20416  4056 pts/0    S+   08:33   0:00 /bin/bash
> root     14992  0.0  0.4  20416  4084 pts/4    S+   08:42   0:00 /bin/bash
> $ ls -l /proc/14992/ns/net
> lrwxrwxrwx 1 root root 0 Jun 16 08:45 /proc/14992/ns/net -> 'net:[4026532629]'
> ```
>
> 可以看到，使用ip命令进入namespace和set_ns进入namespace后的的`/bin/bash`的ns

## Volume的本质

### 启动一个挂载volume的容器

```bash
$ docker run --rm -it -d -v /test ubuntu
65f6facb7c3c8e2f239bed07481e70ca7e0d09fc61617d0ef07e00a066ae7f96
$ docker volume ls
DRIVER              VOLUME NAME
local               ca1683d1e9cc06657c7857d8b8c7196176a59e409d74fc72ff30f0a76f98d614
```

```bash
$ docker inspect 65f6fa
[
    {
        "Id": "65f6facb7c3c8e2f239bed07481e70ca7e0d09fc61617d0ef07e00a066ae7f96",
        ...
        "Mounts": [
            {
                "Type": "volume",
                "Name": "ca1683d1e9cc06657c7857d8b8c7196176a59e409d74fc72ff30f0a76f98d614",
                "Source": "/var/lib/docker/volumes/ca1683d1e9cc06657c7857d8b8c7196176a59e409d74fc72ff30f0a76f98d614/_data",
                "Destination": "/test",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
        ...
]
```

可以看到，在不指定本地目录的时候，docker会自动创建一个volume，且在`/var/lib/docker/volumes`下创建一个目录

### 启动一个挂载本地目录的容器

```bash
$ docker run --rm -it -d -v /test:/test ubuntu
ddbbb888a84a3b700a321db79e0743576d894a2c6c6b9be58f73142d921b60aa
```

```bash
$ docker inspect ddbbb8
[
    {
        "Id": "ddbbb888a84a3b700a321db79e0743576d894a2c6c6b9be58f73142d921b60aa",
        ...
        "Mounts": [
            {
                "Type": "bind",
                "Source": "/test",
                "Destination": "/test",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ],
        ...
]
```

### Volume挂载的真相

**这一段话有必要引述**

> Docker 又是如何做到把一个宿主机上的目录或者文件，挂载到容器里面去呢？难道又是 Mount Namespace 的黑科技吗？
>
> 实际上，并不需要这么麻烦。在《白话容器基础（三）：深入理解容器镜像》的分享中，我已经介绍过，当容器进程被创建之后，尽管开启了 Mount Namespace，但是在它执行 chroot（或者 pivot_root）之前，容器进程一直可以看到宿主机上的整个文件系统。而宿主机上的文件系统，也自然包括了我们要使用的容器镜像。这个镜像的各个层，保存在 /var/lib/docker/aufs/diff 目录下，在容器进程启动后，它们会被联合挂载在 /var/lib/docker/aufs/mnt/ 目录中，这样容器所需的 rootfs 就准备好了。
>
> 所以，我们只需要在 rootfs 准备好之后，在执行 chroot 之前，把 Volume 指定的宿主机目录（比如 /home 目录），挂载到指定的容器目录（比如 /test 目录）在宿主机上对应的目录（即 /var/lib/docker/aufs/mnt/[可读写层 ID]/test）上，这个 Volume 的挂载工作就完成了

由此可见，如果要将一个目录挂载到一个容器里，其操作是：

* 进入mount namespace
* 将需要挂载的目录挂载到容器的目录上
* `chroot`切换到对应的文件系统

#### 示例

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
65f6facb7c3c        ubuntu              "/bin/bash"         14 minutes ago      Up 14 minutes                           recursing_clarke
```

```bash
$ docker inspect 65f6facb7c3c
[
    {
        "Id": "65f6facb7c3c8e2f239bed07481e70ca7e0d09fc61617d0ef07e00a066ae7f96",
        ...
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/9f0bd8e84ddffd8663f01c959a0ced115a4d598b096896be15b3c680039d7754-init/diff:/var/lib/docker/overlay2/17bd5da0cda20e8ecd1d4955d25f49609ff0d7aa72fe45a0388a357fcc5b625f/diff:/var/lib/docker/overlay2/823e415d4256d05fb0101af4dcc42a4389d44cf6467972d654e93e0cc575cd9b/diff:/var/lib/docker/overlay2/37d3e588905fae55c8a0481e9cda7be36177af874631abb15724c893887e260b/diff:/var/lib/docker/overlay2/40d198d6f624e455800254766eb6a7190ce02442fc48f02f6f16f72105cefd0d/diff",
                "MergedDir": "/var/lib/docker/overlay2/9f0bd8e84ddffd8663f01c959a0ced115a4d598b096896be15b3c680039d7754/merged",
                "UpperDir": "/var/lib/docker/overlay2/9f0bd8e84ddffd8663f01c959a0ced115a4d598b096896be15b3c680039d7754/diff",
                "WorkDir": "/var/lib/docker/overlay2/9f0bd8e84ddffd8663f01c959a0ced115a4d598b096896be15b3c680039d7754/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [
            {
                "Type": "volume",
                "Name": "ca1683d1e9cc06657c7857d8b8c7196176a59e409d74fc72ff30f0a76f98d614",
                "Source": "/var/lib/docker/volumes/ca1683d1e9cc06657c7857d8b8c7196176a59e409d74fc72ff30f0a76f98d614/_data",
                "Destination": "/test",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
        ...
]
```

可以看到`Mounts`中的`/var/lib/docker/volumes/ca1683d1e9cc06657c7857d8b8c7196176a59e409d74fc72ff30f0a76f98d614/_data`

`UpperDir`中已经可以看到test目录了

```bash
$ ls /var/lib/docker/overlay2/9f0bd8e84ddffd8663f01c959a0ced115a4d598b096896be15b3c680039d7754/diff/
test
```

现在，在docker中创建一个文件

```bash
$ docker exec -it 65f6facb7c3c touch /test/test.txt
$ ls /var/lib/docker/volumes/ca1683d1e9cc06657c7857d8b8c7196176a59e409d74fc72ff30f0a76f98d614/_data/
test.txt
```

### 绑定挂载机制

可以将一个目录绑定挂载到另外一个目录

```bash
$ ls /test
# 没有挂载前，目录为空
# 挂载tmp目录
$ mount --bind /tmp /test
$ ls /test
systemd-private-0ea54a48e403454ba91e8c8d816d2cbd-systemd-resolved.service-A5ml7i   vmware-root_550-2991137472
systemd-private-0ea54a48e403454ba91e8c8d816d2cbd-systemd-timesyncd.service-qu9LFm
# /test目录与/tmp目录内容一致
$ ls /tmp
systemd-private-0ea54a48e403454ba91e8c8d816d2cbd-systemd-resolved.service-A5ml7i   vmware-root_550-2991137472
systemd-private-0ea54a48e403454ba91e8c8d816d2cbd-systemd-timesyncd.service-qu9LFm
# umount
$ umount /test
$ ls /test
# 目录为空
```

但是，启动容器前host上的目录没有挂载内容，容器启动后，host上挂载目录，查看容器中的内容

```bash
$ ls /test
# 无内容
$ docker run -it -d --rm -v /test:/test ubuntu
5f2aaad59abb7546ecd4b7a47ede09fb6a1541c4bda27b79de893bd27350b93c
$ docker exec -it 5f2 ls /test
# 无内容
$ mount --bind /tmp /test
$ ls /test
systemd-private-0ea54a48e403454ba91e8c8d816d2cbd-systemd-resolved.service-A5ml7i   vmware-root_550-2991137472
systemd-private-0ea54a48e403454ba91e8c8d816d2cbd-systemd-timesyncd.service-qu9LFm
$ docker exec -it 5f2 ls /test
# 依旧没有内容
$ docker exec -it 5f2 touch /test/test.txt
$ ls /test
systemd-private-0ea54a48e403454ba91e8c8d816d2cbd-systemd-resolved.service-A5ml7i   vmware-root_550-2991137472
systemd-private-0ea54a48e403454ba91e8c8d816d2cbd-systemd-timesyncd.service-qu9LFm
# 咦，test.txt去哪儿了呢
$ umount /test
$ ls /test
test.txt
# 这样就有了
```

如果/test目录已经mount了呢？

### 会不会将本地挂载的目录提交到image里面呢？

> **不会**
>
> 容器的镜像操作，比如 docker commit，都是发生在宿主机空间的。而由于 Mount Namespace 的隔离作用，宿主机并不知道这个绑定挂载的存在。所以，在宿主机看来，容器中可读写层的 /test 目录（/var/lib/docker/aufs/mnt/[可读写层 ID]/test），始终是空的。

## 小结

本节主要学习了DockerFile编写、镜像的build方法。以及docker exec和volume的底层实现原理。

通过所有前面几节的实验，不难发现，docker就是通过linux namespace进行隔离，cgroup对资源进行限制，rootfs作为容器的文件系统。无论是docker镜像还是docker容器，以及网络和volume，都是在linux的这些基础功能的基础上实现起来的。
