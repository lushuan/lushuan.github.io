# 从Docker至Containerd

平时较多使用的是docker项目将容器运行时迁移至了containerd，这里整理下相关的操作命令

## 常用指令对比
ctr是containerd自带的工具，有命名空间的概念
crictl是k8s社区的专用CLI工具，所有操作都在命名空间k8s.io
所以使用ctr操作时注意指定-n k8s.io

![docker-containerd-k8s](../../images/kubernetes/docker-containerd-k8s.png)

## 清理未被使用的镜像，一般用于释放本地空间
docker 场景
```
docker image prune -a
```
containerd场景（需要高版本crictl）
```
crictl rmi --prune
```

## 本地文件和容器内文件间拷贝
docker 场景
```
docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|-
docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH
```

使用kubectl cp 
```
Copy files and directories to and from containers.

Examples:
  # !!!Important Note!!!
  # Requires that the 'tar' binary is present in your container
  # image.  If 'tar' is not present, 'kubectl cp' will fail.
  
  # Copy /tmp/foo_dir local directory to /tmp/bar_dir in a remote pod in the default namespace
  kubectl cp /tmp/foo_dir <some-pod>:/tmp/bar_dir
  
  # Copy /tmp/foo local file to /tmp/bar in a remote pod in a specific container
  kubectl cp /tmp/foo <some-pod>:/tmp/bar -c <specific-container>
  
  # Copy /tmp/foo local file to /tmp/bar in a remote pod in namespace <some-namespace>
  kubectl cp /tmp/foo <some-namespace>/<some-pod>:/tmp/bar
  
  # Copy /tmp/foo from a remote pod to /tmp/bar locally
  kubectl cp <some-namespace>/<some-pod>:/tmp/foo /tmp/bar

Options:
  -c, --container='': Container name. If omitted, the first container in the pod will be chosen
      --no-preserve=false: The copied file/directory's ownership and permissions will not be preserved in the container

Usage:
  kubectl cp <file-spec-src> <file-spec-dest> [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).
```





