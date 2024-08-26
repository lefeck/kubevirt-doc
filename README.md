

# KubeVirt


KubeVirt 是一个用于在 Kubernetes 中运行虚拟机 (VM) 的项目。它允许在 Kubernetes 集群中同时运行容器和虚拟机，这使得用户可以在同一基础架构上运行容器化应用和虚拟化机器。KubeVirt 的架构由多个组件组成，这些组件通过相互协作来管理和运行虚拟机。以下是 KubeVirt 主要组件的交互关系图：

![img](./architecture.png)

组件：

1. Virt-API
   * 作用：Virt-API 是 KubeVirt 的 API 服务器组件，负责处理与虚拟机相关的 API 请求。它将这些请求转发给相应的控制器或处理程序。
   * 交互：Virt-API 与 Kubernetes API Server 交互，监听和处理虚拟机自定义资源（例如 VirtualMachine, VirtualMachineInstance）的创建、更新和删除请求。
2. Virt-Controller
   * 作用：Virt-Controller 是一个控制器，类似于 Kubernetes 的 controller-manager，负责管理虚拟机的生命周期。它通过监控和响应 VirtualMachine 和 VirtualMachineInstance 自定义资源的状态变化来创建和删除虚拟机实例,管理和监控 VMI 对象及其关联的 Pod，对其状态进行更新。
     交互：
      * 与 Virt-API 交互，获取虚拟机配置和状态更新。
      * 与 Kubernetes scheduler，确保虚拟机实例被调度到适当的节点上运行。
      * 与 Virt-Handler 交互，管理虚拟机的实际启动、停止等操作。
3. Virt-Handler
   
   * 作用：Virt-Handler 是以 DaemonSet 运行在每个 Kubernetes 节点上的守护进程，监听 VMI 的状态向上汇报，负责管理 VMI 的生命周期，例如启动、停止和监控虚拟机实例。
     交互：
   
     * 与 Virt-Controller 交互，接收控制器的指令来启动或停止虚拟机实例。
   
     * 直接与虚拟机监控进程 (libvirt/qemu) 交互，执行虚拟机的实际操作。
4. Virt-Launcher
   * 作用：Virt-Launcher 是一个 Kubernetes Pod方式运行，负责在特定节点上运行一个VirtualMachineInstance(虚拟机实例)。每个VirtualMachineInstance 对应一个特定的 Virt-Launcher Pod 。
    交互：
      * 与 Virt-Handler 交互，接收指令并启动虚拟机实例。
      * virt-handler通过将VirtualMachineInstance的CRD对象传递给virt-launcher来通知virt-launcher启动VMI。然后virt-launcher在其容器中使用本地libvirtd实例来启动VirtualMachineInstance。从此开始，virt-launcher将监控VirtualMachineInstance进程，并在VirtualMachineInstance退出后终止；
      * 通过 libvirt 和 QEMU/KVM 管理虚拟机实例的生命周期。

5. libvirtd:
   * 作用：virt-launcher借助于libvirtd管理VirtualMachineInstance生命周期
   * 交互：每个VirtualMachineInstance对应的Pod都有一个libvirtd实例。





###  **VirtualMachine (VM)**

- **定义**: `VirtualMachine` 是一种 Kubernetes 自定义资源定义（CRD），用于定义和管理虚拟机的生命周期。
- **作用**: 它是虚拟机的高级别抽象，提供了虚拟机的配置信息，例如虚拟机的 CPU、内存、存储、网络配置等。
- 特性:
  - 可以包含虚拟机的启动策略，例如虚拟机应何时启动或停止。
  - `VirtualMachine` 可以被视为一种“虚拟机模板”，它不会直接创建虚拟机实例，而是定义虚拟机实例应如何配置和运行。
  - 支持开关操作，如“启动”和“停止”虚拟机。

### **VirtualMachineInstance (VMI)**

- **定义**: `VirtualMachineInstance` 也是一种 Kubernetes 自定义资源定义（CRD），它表示一个实际运行中的虚拟机实例。
- **作用**: `VMI` 是运行的虚拟机状态，包含了运行时状态和配置。
- 特性:
  - 当创建一个 VMI 资源时，KubeVirt 会在 Kubernetes 集群中实际启动一个虚拟机。
  - VMI 是短暂的，表示虚拟机的当前状态，例如是否运行中、是否已分配给某个节点等。
  - `VMI` 的生命周期通常由 `VM` 资源控制，但可以单独使用。



## Requirements

在开始之前需要满足一些要求

Kubernetes 群集或衍生物（例如 OpenShift ），基于最新的三个 Kubernetes 发行版之一，该版本是在 KubeVirt 发布时发行的。
这里 KubeVirt 最新版是 1.2.1 ，K8s 选择 1.29.6
Kubernetes apiserver 必须具有-allow-privileged = true，才能运行Kubevirt的特权守护程序。

kubectl  客户端

* 推荐使用 containerd 或 crio (with runv) 容器运行时


## 验证硬件虚拟化支持
建议使用虚拟化支持的硬件。您可以使用 virt-host validate 来确保您的主机能够运行虚拟化工作负载：

安装 virt-host-validate 命令
```shell
# centos7
yum install -y qemu-kvm libvirt virt-install bridge-utils

# ubuntu
apt-get -y install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils libguestfs-tools virt-viewer virt-manager virtinst
```
验证
```shell
virt-host-validate qemu
QEMU: Checking for hardware virtualization                                 : PASS
QEMU: Checking if device /dev/kvm exists                                   : PASS
QEMU: Checking if device /dev/kvm is accessible                            : PASS
QEMU: Checking if device /dev/vhost-net exists                             : PASS
QEMU: Checking if device /dev/net/tun exists                               : PASS
...
```
在 Kubernetes 上安装 KubeVirt
```shell
# 指定为 v1.2.1 版本
export RELEASE=v1.2.1
# 下载 KubeVirt operator Yaml，并安装
1. KubeVirt Operator
* 作用：KubeVirt Operator 是用于安装和管理 KubeVirt 组件的控制器。它负责部署、升级和维护 KubeVirt 组件，包括 Virt-API、Virt-Controller、Virt-Handler 等。
* 交互：Operator 通过 Kubernetes API 服务器管理和监控所有 KubeVirt 组件的部署和健康状态。
wget https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml
kubectl apply -f kubevirt-operator.yaml
# 下载 KubeVirt CR， 创建 KubeVirt CR（实例部署请求），该 CR 触发实际安装
wget https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml
kubectl apply -f kubevirt-cr.yaml
# 等待所有 KubeVirt 组件都启动
kubectl -n kubevirt wait kv kubevirt --for condition=Available
# 下载 virtctl client
wget https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/virtctl-${RELEASE}-linux-amd64
mv virtctl-${RELEASE}-linux-amd64 /usr/local/bin/virtctl
chmod +x /usr/local/bin/virtctl
```



virtctl image-upload \
--image-path='rhcos-4.15.23-x86_64-live.x86_64.iso' \
--storage-class hostpath-csi \
--pvc-name=iso-rhcos \
--pvc-size=5Gi \
--uploadproxy-url=https://10.96.2.240 \
   --insecure \
   --wait-secs=240





### hostpath 安装

```shell
# hostpath provisioner operator 依赖于 cert manager 提供鉴权能力
$ kubectl create -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.yaml

#创建 hostpah-provisioner namespace
$ kubectl create -f https://raw.githubusercontent.com/kubevirt/hostpath-provisioner-operator/main/deploy/namespace.yaml

#部署 operator
$ kubectl create -f https://raw.githubusercontent.com/kubevirt/hostpath-provisioner-operator/main/deploy/operator.yaml -n hostpath-provisioner
$ kubectl create -f https://raw.githubusercontent.com/kubevirt/hostpath-provisioner-operator/main/deploy/webhook.yaml
```



### 安装 nfs server


| OS           | IP Address     | server     |
|--------------|----------------|------------|
| ubuntu 24.04 | 192.168.10.162 | nfs server |
| ubuntu 24.04 | 192.168.10.163 | master     |
| ubuntu 24.04 | 192.168.10.164 | node01     |
| ubuntu 24.04 | 192.168.10.165 | node02     |

nfs server 节点，安装组件：
```shell
sudo apt-get -y install nfs-kernel-server nfs-common
```

创建数据存储目录：
```shell
sudo mkdir -p /data/volumes 
```
修改配置文件：
```
cat << 'eof' > /etc/exports
/data/volumes  *(rw,sync,no_root_squash)
eof
# *表示允许所有网段访问
```

重启 nfs server 服务并检查
```
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server
sudo showmount -e
```

在所有k8s node节点需要安装组件：
```shell
sudo apt-get -y install nfs-common
```



### 安装nfs存储插件

nfs-subdir-external-provisioner是一个自动配置卷程序，它使用现有的和已配置的 NFS 服务器来支持通过持久卷声明动态配置 Kubernetes 持久卷。

此组件是对 nfs-client-provisioner 的扩展，nfs-client-provisioner 已经不提供更新，且 nfs-client-provisioner 的 Github 仓库已经迁移到 nfs-subdir-external-provisioner 的仓库。

GitHub 地址：https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

创建 ServiceAccount
```shell
kubectl apply -f nfs-provisioner/nfs-rbac.yaml
```
部署 NFS-Subdir-External-Provisioner
```shell
kubectl apply -f nfs-provisioner/nfs-deployment.yaml -n provisioner
```
创建 NFS SotageClass
```shell
kubectl apply -f nfs-provisioner/storageclass-nfs.yaml -n provisioner
```



# 部署一个简单的 VM

部署完 KubeVirt 后，可以通过创建一个测试虚拟机来验证 KubeVirt 是否正常工作。

# 部署虚拟机

VM 是命名空间中的资源，可以创建一个命名空间：

```bash
vagrant@master01:~$ kubectl create ns vm
namespace/vm created
```

KubeVirt 官网提供了一个简单的 [vm](https://kubevirt.io/labs/manifests/vm.yaml) 资源定义：

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: testvm
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: testvm
    spec:
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: default
            masquerade: {}
        resources:
          requests:
            memory: 64M
      networks:
      - name: default
        pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/cirros-container-disk-demo
        - name: cloudinitdisk
          cloudInitNoCloud:
            userDataBase64: SGkuXG4=
```

部署 vm：

```bash
vagrant@master01:~$ kubectl apply -f https://kubevirt.io/labs/manifests/vm.yaml  -n vm
virtualmachine.kubevirt.io/testvm created
```

查看状态：

```bash
vagrant@master01:~$ kubectl get vm -n vm
NAME     AGE   STATUS    READY
testvm   34s   Stopped   False
```

`testvm` 处于 Stopped 的状态，是因为在资源定义中，设置了 `running: false`，所以需要手动启动。
同时可以看到在 VM 没有启动的时候是不存在 `virt-launcher` Pod 和 VMI 资源的。

```bash
vagrant@master01:~$ kubectl get vmi -n vm
No resources found in testvm namespace.
vagrant@master01:~$ kubectl get pod -n vm
No resources found in testvm namespace.
```

使用 `virtctl` 进行启动 VM：

```bash
vagrant@master01:~$ virtctl start testvm -n vm
VM testvm was scheduled to start
```

启动后，查看资源状态：

```bash
vagrant@master01:~$ kubectl get vm,vmi,pod -n vm
NAME                                AGE     STATUS    READY
virtualmachine.kubevirt.io/testvm   3m44s   Running   True

NAME                                        AGE   PHASE     IP              NODENAME   READY
virtualmachineinstance.kubevirt.io/testvm   30s   Running   10.244.241.89   master01   True

NAME                             READY   STATUS    RESTARTS   AGE
pod/virt-launcher-testvm-nlwd4   3/3     Running   0          30s
```

# 访问虚拟机

借助 `virtctl` 可以通过 console vnc 或 ssh 访问虚拟机。

## serial console

通过 `virtctl console` 连接到虚拟机：

```bash
vagrant@master01:~$ virtctl  console vm -n vm
Successfully connected to testvm console. The escape sequence is ^]

login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
testvm login: cirros
Password: 
$ uname -a
Linux testvm 4.4.0-28-generic #47-Ubuntu SMP Fri Jun 24 10:09:13 UTC 2016 x86_64 GNU/Linux
$ lsblk
NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda     253:0    0  44M  0 disk 
|-vda1  253:1    0  35M  0 part /
`-vda15 253:15   0   8M  0 part 
vdb     253:16   0   1M  0 disk 
```

通过 `ctrl + ]` 退出。

## ssh

借助 `virtctl ssh` 可以通过 SSH 连接到虚拟机：

```bash
vagrant@master01:~$ virtctl ssh cirros@testvm.vm
The authenticity of host 'vmi/testvm.testvm:22 (192.168.121.1:7890)' can't be established.
ECDSA key fingerprint is SHA256:DlsqNiPTxNrBsy+VldgVLJT7qKbyynH+Dxqj2s4Q7oU.
Are you sure you want to continue connecting (yes/no)? yes
cirros@vmi/testvm.vm's password:
$ lsblk 
NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda     253:0    0  44M  0 disk 
|-vda1  253:1    0  35M  0 part /
`-vda15 253:15   0   8M  0 part 
vdb     253:16   0   1M  0 disk
```

`cirros@testvm.vm` 中 `cirros@testvm` 代表使用 `cirros` 用户登陆到 `testvm` 虚拟机中，`.vm` 代表虚拟机位于 `vm` 命名空间中，也可以通过 `-n` 来指定。

## vnc

使用 `virtctl vnc testvm -n vm` 可以打开 vnc 控制台，这个依赖于图形化环境和 remote-viewer 软件包，目前我节点没有图形化，这个不演示了。

# 关闭和删除虚拟机

关闭虚拟机可以使用 `virtctl stop` 命令：

```bash
vagrant@master01:~$ virtctl stop testvm -n vm
VM testvm was scheduled to stop
vagrant@master01:~$ kubectl get vm,vmi -n vm
NAME                                AGE   STATUS    READY
virtualmachine.kubevirt.io/testvm   11m   Stopped   False
```

要删除虚拟机，需要删除 VM 资源，如果不删除 VM 资源，仅删除 VMI 资源，会导致 VMI 重新构建。

### 只删除 VMI

为了更好的演示删除 VMI 会自动创建 VMI，首选需要将 VM 中的 `running: false` 修改为 `true`：

```bash
vagrant@master01:~$ kubectl patch vm testvm --type=merge --patch '{"spec":{"running": true}}' -n vm
virtualmachine.kubevirt.io/testvm patched
```

删除 VMI ：

```bash
vagrant@master01:~$ kubectl get vmi  -n vm
NAME     AGE   PHASE     IP              NODENAME   READY
testvm   31s   Running   10.244.241.90   master01   True
vagrant@master01:~$ kubectl delete vmi testvm -n vm
virtualmachineinstance.kubevirt.io "testvm" deleted
vagrant@master01:~$ kubectl get vmi  -n vm
NAME     AGE   PHASE     IP              NODENAME   READY
testvm   18s   Running   10.244.241.91   master01   True
```

可以看到删除 VMI 后，又重新构建了 VMI，IP 是不一样的。

### 删除 VM

正确的流程应该是先手动停止 VMI，然后将 VM 删除：

```bash
vagrant@master01:~$ virtctl stop testvm -n vm
VM testvm was scheduled to stop
vagrant@master01:~$ kubectl get vmi -n vm
No resources found in testvm namespace.
vagrant@master01:~$ kubectl delete vm testvm -n vm
virtualmachine.kubevirt.io "testvm" deleted
vagrant@master01:~$ kubectl get vm,vmi -n vm
No resources found in testvm namespace.
```

```sh
root@master:~/kubevirt# kubectl get  svc -n cdi
NAME                     TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
cdi-api                  ClusterIP   10.96.1.139   <none>        443/TCP    9d
cdi-prometheus-metrics   ClusterIP   10.96.2.3     <none>        8080/TCP   9d
cdi-uploadproxy          ClusterIP   10.96.1.196   <none>        443/TCP    9d
root@master:~/kubevirt#
```



创建pvc，并且上传rhcos镜像

```sh
# virtctl image-upload dv rhcos-dv \
--image-path=/root/rhcos-4.15.23-x86_64-live.x86_64.iso \
--storage-class nfs-client \ # 指定存储类的名称
--size=20G \
--uploadproxy-url=https://10.96.1.196 \
--insecure \
--wait-secs=240

PVC default/rhcos-dv not found
DataVolume default/rhcos-dv created
Waiting for PVC rhcos-dv upload pod to be ready...
Pod now ready
Uploading data to https://10.96.1.196

 1.09 GiB / 1.09 GiB [=================================================================================================================================================================================] 100.00% 25s

Uploading data completed successfully, waiting for processing to complete, you can hit ctrl-c without interrupting the progress
Processing completed successfully
```



创建rhcos虚拟机

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/os: linux
  name: rhcos-vm
spec:
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        kubevirt.io/domain: rhcos-vm
    spec:
      domain:
        cpu:
          cores: 2         #根据需求，确定cpu的数量
        devices:
          disks:
          - disk:
              bus: virtio
            name: disk0
          - cdrom:
              bus: sata
              readonly: true
            name: cloudinitdisk
        resources:
          requests:
            memory: 12G          #根据需求，确定内存的数量
      volumes:
      - name: disk0
        persistentVolumeClaim:
          claimName: rhcos-dv
      - cloudInitNoCloud:
        name: cloudinitdisk
```





## Containerized Data Importer

COntainerd Data Importer(CDI) 是能够将多种来源的数据导入到 Kubernetes 中的 PVC，随后用做 VM 的磁盘。可以通过 DataVolumes 将 pvc (Persistent Volume Claims) 用作 KubeVirt 虚拟机的磁盘。三个主要的CDI 用例是：

* 从 web 服务器或容器注册中心导入磁盘映像到 DataVolume
* 将现有的 PVC 克隆到数据卷
* 上传本地磁盘映像到数据卷

### 安装 CDI

用以下命令安装最新版本的 CDI 通过 Operator：

```shell
# 指定 v1.59.0 版本
export VERSION=v1.59.0
# 下载 Yaml 并创建
wget https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
wget https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml
kubectl create -f cdi-operator.yaml
kubectl create -f cdi-cr.yaml
```

等待 Operator 部署完成，然后检查状态：

```shell
root@master:~/kubevirt# kubectl get cdi cdi -n cdi
NAME   AGE    PHASE
cdi    7d2h   Deployed
root@master:~/kubevirt# kubectl get pod -n cdi
NAME                               READY   STATUS    RESTARTS   AGE
cdi-apiserver-75bb86c8c4-znvpj     1/1     Running   0          7d2h
cdi-deployment-645dc8bc99-mvtbt    1/1     Running   0          7d2h
cdi-operator-67d655447d-hhdlv      1/1     Running   0          7d2h
cdi-uploadproxy-67f586465b-2mgsn   1/1     Running   0          7d2h
root@master:~/kubevirt#



```

部署完成后，会引进 `DataVolumes` ，通过创建该资源，可以抽象的创建 PVC。



# 使用导入的 PVC 创建 VM



**PersistentVolumeClaim:** 将可用的 PVC 附加到 VM。将现有 VM 导入到 Kubernetes/OpenShift 中的方法是使用 CDI 将现有 VM 的磁盘或容器镜像中的磁盘导入到 PVC 中，然后将 PVC 附加到 VM 实例。



#### 创建pvc并导入操作系统镜像

```yaml
root@master:~/kubevirt# cat pvc-centos7.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: centos7-vm
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-2003.qcow2.xz"
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: nfs-client
```

创建PVC实例

```
kubectl  apply -of pvc-centos7.yaml
```

#### 创建虚拟机模版实例

```yaml
cat << eof > vm-centos7.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/os: linux
  name: vm1
spec:
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        kubevirt.io/domain: vm1
    spec:
      domain:
        cpu:
          cores: 2
        devices:
          disks:
          - disk:
              bus: virtio
            name: disk0
          - cdrom:
              bus: sata
              readonly: true
            name: cloudinitdisk
        resources:
          requests:
            memory: 1024M
      volumes:
      - name: disk0
        persistentVolumeClaim:
          claimName: centos7          #与pvc-centos.yaml中的名称对应
      - cloudInitNoCloud:
          userData: |
            #cloud-config
            hostname: fnode21           #与pv中的主机名对应
            ssh_pwauth: True
            disable_root: false
            ssh_authorized_keys:
            - ssh-rsa YOUR_SSH_PUB_KEY_HERE
        name: cloudinitdisk
eof
```

导入密钥使当前服务器可以免密登录，并创建虚机执行

```sh
ssh-keygen
PUBKEY=`cat ~/.ssh/id_rsa.pub`
sed -i "s%ssh-rsa.*%$PUBKEY%" centos_vm.yml
kubectl create -f centos_vm.yaml
```

运行成功后，执行

```shell
kubectl get vmi
```


执行 ssh centos@10.42.130.135 登录到该主机，可以使用sudo passwd修改root密码



# 使用 CDI 导入磁盘映像



**DataVolume:** 可动态创建 PVC 并将外部磁盘、镜像、容器数据导入到这些 PVC 中。为了使用 dataVolume 卷，必须借助 OpenShift Virtualization Operator 安装的 Containerized Data Importer (CDI) 功能。如果不使用 dataVolume 卷，那么可以使用 persistentVolumeClaim 卷，即将一个可用的 PVC 分配给 VM。

通过声明 DadaVolume 资源对象来动态创建 PVC，并将拉取的镜像源导入到该 PVC 中：

```bash
cat <<EOF > dv_fedora.yml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: fedora
spec:
  storage:
    resources:
      requests:
        storage: 5Gi
    storageClassName: nfs-client
  source:
    http:
      url: "https://download.fedoraproject.org/pub/fedora/linux/releases/37/Cloud/x86_64/images/Fedora-Cloud-Base-37-1.7.x86_64.raw.xz"
EOF
```

这里 dataVolume 拉取镜像的 `source` 是 `http` ,除此之外还可以使用 `pvc` 、`registry` 、`s3` 等，具体可以通过 `kubectl explain DataVolume.spec.source` 来进行查看。

```bash
vagrant@master01:~$ kubectl create -f dv_fedora.yml  -n cdi-demo 
```

然后，它就报错了。

```shell
vagrant@master01:~$ kubectl get pod -n cdi-demo
NAME                                                  READY   STATUS             RESTARTS      AGE
importer-prime-4d87fc83-0832-42df-bd2c-f970098b75ac   0/1     CrashLoopBackOff   4 (45s ago)   2m35s

vagrant@master01:~$ kubectl logs importer-prime-4d87fc83-0832-42df-bd2c-f970098b75ac -n cdi-demo
I0428 04:53:21.137080       1 importer.go:103] Starting importer
E0428 04:53:21.138511       1 importer.go:133] exit status 1, blockdev: cannot open /dev/cdi-block-volume: Permission denied

kubevirt.io/containerized-data-importer/pkg/util.GetAvailableSpaceBlock
    pkg/util/util.go:130
kubevirt.io/containerized-data-importer/pkg/util.GetAvailableSpaceByVolumeMode
    pkg/util/util.go:100
main.main
    cmd/cdi-importer/importer.go:131
runtime.main
    GOROOT/src/runtime/proc.go:267
```

查看 PVC 和 DataVolume 的状态：

```bash
vagrant@master01:~$ kubectl get pvc -n cdi-test
NAME                                         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      VOLUMEATTRIBUTESCLASS   AGE
fedora                                       Pending                                                                        rook-ceph-block   <unset>                 3m27s
prime-ac1aca3c-15cc-4c8b-ab77-3ef2cb044846   Bound     pvc-e995b1ec-c7eb-41ed-9fae-7f6ce7a5ebdd   5Gi        RWX            rook-ceph-block   <unset>  

vagrant@master01:~$ kubectl get datavolume -n cdi-test
NAME     PHASE              PROGRESS   RESTARTS   AGE
fedora   ImportInProgress   N/A        7          16m
```

经过查找资料，解决方法参考 [Non-root Containers And Devices](https://kubernetes.io/blog/2021/11/09/non-root-containers-and-devices/)

如果使用的 Containerd ，修改配置中的：

```toml
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    device_ownership_from_security_context = true
```

如果使用的是 CRI-O，修改配置中的：

```toml
[crio.runtime]
device_ownership_from_security_context = true
```

根据使用的 Container Runtime 修改不同的配置文件，然后重启，就正常了。

查看状态：

```bash
vagrant@master01:~$ kubectl get dv -n cdi-test
NAME     PHASE              PROGRESS   RESTARTS   AGE
fedora   ImportInProgress   86.27%     11         35m
```

等待导入状态为Succeeded成功后，查看资源状态：

```bash
vagrant@master01:~$ kubectl get DataVolume,pvc -n cdi-test
NAME                                PHASE       PROGRESS   RESTARTS   AGE
datavolume.cdi.kubevirt.io/fedora   Succeeded   100.0%     11         46m

NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/fedora   Bound    pvc-e995b1ec-c7eb-41ed-9fae-7f6ce7a5ebdd   5Gi        RWX            rook-ceph-block   <unset>                 46m
```

可以发现在, 创建DataVolume过程中, 可以动态创建 PVC 并将外部磁盘、镜像、容器数据导入到这些 PVC 中。



## 使用导入的 PVC 创建 VM



定义VM资源类型创建名称为fedora-vm的虚拟机，并且关联DataVolume动态创建出的PVC。

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/os: linux
  name: fedora-vm
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/domain: fedora-vm
    spec:
      domain:
        cpu:
          cores: 1
        devices:
          disks:
          - disk:
              bus: virtio
            name: disk0
          - cdrom:
              bus: sata
              readonly: true
            name: cloudinitdisk
        machine:
          type: q35
        resources:
          requests:
            memory: 512M
      volumes:
      - name: disk0
        persistentVolumeClaim:
          claimName: fedora
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            hostname: fedora-vm
            ssh_pwauth: True
            disable_root: false
            user: fedora
            password: 123456
            ssh_authorized_keys:
        name: cloudinitdisk
```

创建 vm：

```bash
vagrant@master01:~$ kubectl -n cdi-test apply -f fedora-vm-pvc.yml 
virtualmachine.kubevirt.io/fedora-vm created
```

查看状态：

```bash
vagrant@master01:~$ kubectl get vmi -n cdi-test
NAME        AGE   PHASE     IP               NODENAME   READY
fedora-vm   28s   Running   10.244.241.115   master01   True
```

连接到虚拟机：

```bash
vagrant@master01:~$ virtctl ssh -i .ssh/id_rsa fedora@fedora-vm.cdi-test
The authenticity of host 'vmi/fedora-vm.cdi-test:22 (192.168.121.1:7890)' can't be established.
ECDSA key fingerprint is SHA256:qmiiDH3244ZW1cfwYav30En+1CoLOR/dSY3ixx5Ul0Y.
Are you sure you want to continue connecting (yes/no)? yes
[fedora@fedora-vm ~]$ lsblk 
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0     11:0    1    1M  0 rom  
zram0  251:0    0  435M  0 disk [SWAP]
vda    252:0    0    5G  0 disk 
├─vda1 252:1    0    1M  0 part 
├─vda2 252:2    0 1000M  0 part /boot
├─vda3 252:3    0  100M  0 part /boot/efi
├─vda4 252:4    0    4M  0 part 
└─vda5 252:5    0  3.9G  0 part /home
                                /
```

VM 可以正常使用 CDI 导入的 PVC 用作根。







Volume

KubeVirt 的 VirtualMachine 可以使用以下几种常用的存储卷类型：

* Ephemeral - 其对应的卷 (PVC) 的数据不会以发生变化，因为所有的写入都保存在位于本地存储的临时镜像中。当 VM 停止、重启或删除时，便会丢弃临时镜像。
* ContainerDisk - 从 registry 中拉取镜像到运行 VM 的主机节点上，并在 VM 启动时作为磁盘附加到 VM。containerDisk 提供了在容器镜像注册表中存储和分发 VM 磁盘的能力。不过 containerDisk 是临时的，数据将在 VM 停止、重启或删除时丢弃。
* DataVolume - 可动态创建 PVC 并将外部磁盘、镜像、容器数据导入到这些 PVC 中。为了使用 dataVolume 卷，必须借助 OpenShift Virtualization Operator 安装的 Containerized Data Importer (CDI) 功能。如果不使用 dataVolume 卷，那么可以使用 persistentVolumeClaim 卷，即将一个可用的 PVC 分配给 VM。
* PersistentVolumeClaim - 将可用的 PVC 附加到 VM。将现有 VM 导入到 OpenShift 中的方法是使用 CDI 将现有 VM 的磁盘或容器镜像中的磁盘导入到 PVC 中，然后将 PVC 附加到 VM 实例。
* EmptyDisk - 为 VM 创建额外的 qcow2 磁盘。当 VM 重启，数据会保留下来，但当重新创建 VM，数据将会被丢弃。
* CloudInitNoCloud - 将所引用的 cloudInitNoCloud 数据源附加给磁盘，以便在 VM 启动后自动执行脚本。VM 内部需要安装 cloud-init。
  

通过声明 DadaVolume 资源定义，来创建 PVC，并将一个 mini 的映像导入到该 PVC 中：

```yaml
cat <<EOF > dv_mini.yml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: mini
spec:
  storage:
    resources:
      requests:
        storage: 5Gi
    storageClassName: nfs-client
  source:
    http:
      url: "http://archive.ubuntu.com/ubuntu/dists/focal/main/installer-amd64/current/legacy-images/netboot/mini.iso"
EOF
```

