+++
date = '2025-09-27T00:00:40+08:00'
draft = false
title = '架設 Kubernetes cluster'
+++
```sh
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

https://stackoverflow.com/questions/52893111/no-endpoints-available-for-service-kubernetes-dashboard

### nodes:

https://120.108.205.137:9999/ui/

```bash
192.168.1.115
```

```bash
ssh k1 # ssh user@192.168.1.11
```

```bash
ssh k2 # ssh user@192.168.1.12 
```

```bash
ssh k3 # ssh user@192.168.1.13
```

```bash
ssh k4 # ssh user@192.168.1.14
```

```bash
sudo vim /etc/hosts
```

```docker
# /etc/hosts
192.168.1.115 asus
192.168.1.11 k1
192.168.1.12 k2
192.168.1.13 k3
192.168.1.14 k4
```

```yaml
# for cluster on virtualbox
192.168.56.1 thinkpad 	# my thinkpad
192.168.56.101 node1 	# vm on virtualbox sudo systemctl restart systemd-resolved
```

### docker installation

- https://docs.docker.com/engine/install/ubuntu/

CRI (container runtime interface) 是 container 用來互相溝通的 API 

也可以想成是 kubernetes 跟 container runtime 溝通的 API
docker 預設是使用 `containerd`, 所以載完 docker 後可以用

```bash
sudo systemctl status containerd.service
```

看到 `containerd` 的存在

- cri-dockerd dependency
    
    ```bash
    sudo apt install make 
    ```
    
    ```bash
    sudo snap install go --classic
    ```
    

可是 k8s 這裡要使用 `cri-dockerd`(https://github.com/Mirantis/cri-dockerd) 來跟 docker 做溝通，照著 github 上面的指示可以安裝後，這時就可以用

```bash
sudo systemctl status cri-docker.service
```

看到 `cri-docker.service` 的存在，現在有了 `containerd` 跟 `cri-dockerd` 這兩種 container runtime interface 同時使用，所以在下一些指令的時候會需要多加

```bash
--cri-socket=unix:///var/run/cri-dockerd.sock
```

這個 flag

如果覺得每次都要加這個 flag 覺得很麻煩了話，去修改 `kubelet` 的設定檔，就不用每次都要自己加 `--cri-socket=unix:///var/run/cri-dockerd.sock` 了

```bash
sudo vim /var/lib/kubelet/kubeadm-flags.env
```

### kubernetes installation

- go
    
    ```bash
    sudo snap install go --classic
    ```
    
- https://docs.docker.com/engine/install/ubuntu/ install docker
- https://github.com/Mirantis/cri-dockerd install `cri-dockerd` (release/0.3) (`00fe8f2013317dbed8308e8171c83fc4f51d640a`)
This adapter provides a shim for Docker Engine that lets you control Docker via the Kubernetes Container Runtime Interface.
    
    ```bash
    ARCH=amd64 make cri-dockerd
    ```
    
    ```bash
    # Run these commands as root
    
    cd cri-dockerd
    mkdir -p /usr/local/bin
    install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
    install packaging/systemd/* /etc/systemd/system
    sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
    systemctl daemon-reload
    systemctl enable --now cri-docker.socket
    ```
    
    ```bash
    sudo systemctl status cri-docker.service
    ```
    
- `kubectl` https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
- `kubeadm` https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### set up cgroup driver

cgroup driver (control group driver) 是 Linux kernel 用來分配資源的一個元件，有 `cgroupfs` 跟 `systemd` 兩種選擇，這裡選擇用 `systemd` 記得 kubernetes 跟 docker 都要做設定

https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/

- kubelet
    
    ```bash
    sudo vim /var/lib/kubelet/config.yaml # might be empty first
    ```
    
    ```yaml
    kind: ClusterConfiguration
    apiVersion: kubeadm.k8s.io/v1beta3
    kubernetesVersion: v1.21.0
    ---
    kind: KubeletConfiguration
    apiVersion: kubelet.config.k8s.io/v1beta1
    cgroupDriver: systemd
    ```
    
    ```bash
    sudo systemctl restart kubelet.service
    ```
    
- docker
    
    ```yaml
    sudo vim /etc/docker/daemon.json
    ```
    
    ```yaml
    {
      "exec-opts": ["native.cgroupdriver=systemd"]
    }
    ```
    
    ```bash
    sudo systemctl restart docker
    ```
    
    ```bash
    docker info | grep "Cgroup Driver"
    ```
    

### initialize control plane

```bash
 sudo kubeadm init --apiserver-advertise-address=192.168.50.250 --pod-network-cidr=192.168.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
```

- 120.108.204.5

```bash
 sudo kubeadm init --apiserver-advertise-address=120.108.204.5 --pod-network-cidr=192.168.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
```

- 192.168.0.101

```bash
 sudo kubeadm init --apiserver-advertise-address=192.168.0.101 --pod-network-cidr=192.168.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock
```

上面的指令中

- `--apiserver-advertise-address=192.168.1.115`: 說明了 API 的位置，這裡設在 master node
- `--cri-socket=unix:///var/run/cri-dockerd.sock` : 用來指定要用 `cri-dockerd` 作為 CRI
- `--pod-network-cidr=<ip-of-container-network-interface>` : 這裡設為 cluster 所使用的網域，以我的例子來說是 `192.168.0.0/16`

```jsx
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- result
    
    ```docker
    $ kubectl get pod -A -o wide                                                                                                                                           1557ms
    NAMESPACE     NAME                           READY   STATUS              RESTARTS   AGE     IP              NODE   NOMINATED NODE   READINESS GATES
    kube-system   coredns-5dd5756b68-hbsrs       0/1     ContainerCreating   0          4m25s   <none>          asus   <none>           <none>
    kube-system   coredns-5dd5756b68-t2598       0/1     ContainerCreating   0          4m25s   <none>          asus   <none>           <none>
    kube-system   etcd-asus                      1/1     Running             0          4m37s   192.168.1.115   asus   <none>           <none>
    kube-system   kube-apiserver-asus            1/1     Running             0          4m40s   192.168.1.115   asus   <none>           <none>
    kube-system   kube-controller-manager-asus   1/1     Running             0          4m37s   192.168.1.115   asus   <none>           <none>
    kube-system   kube-proxy-qc5zv               1/1     Running             0          4m25s   192.168.1.115   asus   <none>           <none>
    kube-system   kube-scheduler-asus            1/1     Running             0          4m37s   192.168.1.115   asus   <none>           <none>
    
    $ kubectl get node 
    NAME   STATUS   ROLES           AGE     VERSION
    asus   Ready    control-plane   4m50s   v1.28.2
    ```
    

### join user plane

- print command from master first
    
    ```bash
    kubeadm token create --print-join-command
    ```
    

```bash
sudo kubeadm join 192.168.56.1:6443 --token uglng4.n0uc7ux5kbl7dijs \
        --discovery-token-ca-cert-hash sha256:5eecc5904cb3595cebfc9374252bcef31084bdc3885203d32e271a8021262464 \
 --cri-socket=unix:///var/run/cri-dockerd.sock 
```

### cni plugin (calico)

- https://ithelp.ithome.com.tw/articles/10295266 **(use Calico)**

```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/calico.yaml
```

```bash
kubectl apply -f calico.yaml
```

- result
    
    ```docker
    $ kubectl get pod -A -o wide
    NAMESPACE     NAME                                      READY   STATUS              RESTARTS   AGE     IP              NODE   NOMINATED NODE   READINESS GATES
    kube-system   calico-kube-controllers-9d57d8f49-lpmmb   0/1     ContainerCreating   0          12s     <none>          asus   <none>           <none>
    kube-system   calico-node-2rhsg                         0/1     Init:1/3            0          12s     192.168.1.12    k2     <none>           <none>
    kube-system   calico-node-5smzc                         1/1     Running             0          12s     192.168.1.115   asus   <none>           <none>
    kube-system   calico-node-nwg87                         0/1     Init:1/3            0          12s     192.168.1.11    k1     <none>           <none>
    kube-system   coredns-5dd5756b68-hbsrs                  1/1     Running             0          16m     192.168.1.129   asus   <none>           <none>
    kube-system   coredns-5dd5756b68-t2598                  0/1     Terminating         0          16m     <none>          asus   <none>           <none>
    kube-system   coredns-766bcb9968-dpdv7                  0/1     ContainerCreating   0          2m25s   <none>          k1     <none>           <none>
    kube-system   coredns-766bcb9968-s76jr                  0/1     ContainerCreating   0          2m25s   <none>          k2     <none>           <none>
    kube-system   etcd-asus                                 1/1     Running             0          16m     192.168.1.115   asus   <none>           <none>
    kube-system   kube-apiserver-asus                       1/1     Running             0          16m     192.168.1.115   asus   <none>           <none>
    kube-system   kube-controller-manager-asus              1/1     Running             0          16m     192.168.1.115   asus   <none>           <none>
    kube-system   kube-proxy-9wspp                          1/1     Running             0          9m29s   192.168.1.11    k1     <none>           <none>
    kube-system   kube-proxy-b7bsr                          1/1     Running             0          8m37s   192.168.1.12    k2     <none>           <none>
    kube-system   kube-proxy-qc5zv                          1/1     Running             0          16m     192.168.1.115   asus   <none>           <none>
    kube-system   kube-scheduler-asus                       1/1     Running             0          16m     192.168.1.115   asus   <none>           <none>
    ```
    
    在 namespace `kube-system` 中，新增了以 `calico-` 為開頭的一些 pod
    

### CNI plugin (flannel)

[https://hackmd.io/@Momentary/rkd1MlSci#2K8S建置-1x步驟](https://hackmd.io/@Momentary/rkd1MlSci#2K8S%E5%BB%BA%E7%BD%AE-1x%E6%AD%A5%E9%A9%9F)

### Kubernetes Dashboard

- apply kubernetes dashboard
    
    ```jsx
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
    ```
    
    ```jsx
    kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
    ```
    

```jsx
kubectl proxy 
```

`kubectl proxy` 之後可以用以下連結連上 dashboard

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login

- token
    
    ```jsx
    kubectl apply -f - << EOF
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: admin
      annotations:
        rbac.authorization.kubernetes.io/autoupdate: "true"
    roleRef:
      kind: ClusterRole
      name: cluster-admin
      apiGroup: rbac.authorization.k8s.io
    subjects:
    - kind: ServiceAccount
      name: admin
      namespace: kube-system
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin
      namespace: kube-system
      labels:
        kubernetes.io/cluster-service: "true"
        addonmanager.kubernetes.io/mode: Reconcile
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: admin-token
      namespace: kube-system
      annotations:
        kubernetes.io/service-account.name: "admin"
    type: kubernetes.io/service-account-token
    EOF
    ```
    
    ```jsx
    kubectl describe secret -n kube-system admin-token
    ```

---

bugs:

- firewall setting
    
    ```bash
    sudo ufw status
    ```
    
    ```bash
    sudo ufw allow 8001/tcp
    ```
    
    ```bash
    sudo ufw reload
    ```
    
    ```bash
    sudo ufw disable
    ```
    
- proxy 遇到問題
    - `docker0` is DOWN (後來發現這應該不是問題
        
        ```bash
        $ ip -c a
        1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
            link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
            inet 127.0.0.1/8 scope host lo
               valid_lft forever preferred_lft forever
            inet6 ::1/128 scope host
               valid_lft forever preferred_lft forever
        2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
            link/ether 04:7c:16:cc:fd:3d brd ff:ff:ff:ff:ff:ff
            inet 192.168.1.115/24 brd 192.168.1.255 scope global dynamic noprefixroute enp3s0
               valid_lft 65759sec preferred_lft 65759sec
            inet6 fe80::ace0:c102:f096:7b5c/64 scope link noprefixroute
               valid_lft forever preferred_lft forever
        3: br-free5gc: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
            link/ether 02:42:31:ee:58:0c brd ff:ff:ff:ff:ff:ff
            inet 10.100.200.1/24 brd 10.100.200.255 scope global br-free5gc
               valid_lft forever preferred_lft forever
        4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
            link/ether 02:42:f9:f2:60:d3 brd ff:ff:ff:ff:ff:ff
            inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
               valid_lft forever preferred_lft forever
        6: cali5fd577e8f5d@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default
            link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
            inet6 fe80::ecee:eeff:feee:eeee/64 scope link
               valid_lft forever preferred_lft forever
        7: cali0d056499591@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default
            link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
            inet6 fe80::ecee:eeff:feee:eeee/64 scope link
               valid_lft forever preferred_lft forever
        8: calib4ed37b0593@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default
            link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 3
            inet6 fe80::ecee:eeff:feee:eeee/64 scope link
               valid_lft forever preferred_lft forever
        9: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1480 qdisc noqueue state UNKNOWN group default qlen 1000
            link/ipip 0.0.0.0 brd 0.0.0.0
            inet 172.18.155.0/32 scope global tunl0
               valid_lft forever preferred_lft forever
        ```
        
    
    https://stackoverflow.com/questions/52893111/no-endpoints-available-for-service-kubernetes-dashboard
    

### KubeFlow

```jsx
kubectl apply -k "github.com/kubeflow/training-operator/manifests/overlays/standalone?ref=v1.5.0"
```

```jsx
git clone https://github.com/kubeflow/training-operator.git
```

```bash
cd ~/training-operator/examples/tensorflow/dist-mnist
```

```jsx
# dockerfile
FROM tensorflow/tensorflow:1.5.0
ADD . /var/tf_dist_mnist
ENTRYPOINT ["python", "/var/tf_dist_mnist/dist_mnist.py"]
```

```jsx
docker build -f Dockerfile -t kubeflow/tf-dist-mnist-test:1.0 ./
```

- push the image to docker hub
    
    ```bash
    # 先把想要 push 上去的 image 的 repository 改成自己的帳號
    docker images
    docker tag 2c7731561d8e 9501sam/tf-dist-mnist-test
    # 登入 docker 之後就可以把本機上的 image push 到自己的 docker hub 上
    docker login
    docker push 9501sam/tf-dist-mnist-test
    ```
    
    ```bash
    # 把 yaml file 中的 image 給指定到自己帳號的 repo, 並且限定一定要 pull
    # 注意後面的 version 要從 1.0 改成 latest
    vim tf_job_mnist.yaml
    ```
    
    ```docker
    apiVersion: "kubeflow.org/v1"
    kind: "TFJob"
    metadata:
      name: "dist-mnist-for-e2e-test"
    spec:
      tfReplicaSpecs:
        PS:
          replicas: 1
          restartPolicy: Always
          template:
            spec:
              containers:
                - name: tensorflow
                  image: 9501sam/tf-dist-mnist-test:latest
        Worker:
          replicas: 1
          restartPolicy: Always
          template:
            spec:
              containers:
                - name: tensorflow
                  image: 9501sam/tf-dist-mnist-test:latest
    ```
    

```jsx
kubectl apply -f ./tf_job_mnist.yaml
```

```jsx
kubectl get pod -A
```

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/affc92d5-39d7-4c69-8d6f-04de07b427d7/f79e150e-d224-43a0-a24e-d0ef8dfd897e/Untitled.png)

---

# 🔧useful tricks

- reset k8s
    
    ```bash
    sudo kubeadm reset --cri-socket=unix:///var/run/cri-dockerd.sock
    sudo rm -rf $HOME/.kube/config
    ```
    
    ```bash
    sudo kubeadm token delete ettz10.pp7nk6qvpw6lhxcv
    ```
    
    ```bash
    kubeadm token create --print-join-commandcordon node
    ```
    
- cordon
    
    ```bash
    kubectl cordon NODE_NAME
    ```
    
    ```bash
    kubectl uncordon NODE_NAME
    ```
    
- 預設 control-pane 不是一個 worker node 這裡讓 control-pane 變成 worker node
    - gpt
        
        To configure your node (`ws690t`) as both a **control-plane** node and a **worker node**, you simply need to allow the node to schedule workloads in addition to running control-plane services. This is done by removing the taint that restricts regular workloads from being scheduled on control-plane nodes.
        
        Here's how you can make your control-plane node (`ws690t`) also act as a worker node:
        
        ### Steps:
        
        1. **Check the current taints on the node**:
        First, let’s check if your node has the default taints that prevent workloads from being scheduled:
            
            ```bash
            kubectl describe node ws690t | grep Taints
            ```
            
            The default taint usually looks like:
            
            ```sh
            node-role.kubernetes.io/control-plane:NoSchedule
            ```
            
        2. **Remove the control-plane taint**:
        To allow the node to accept worker workloads, remove the `NoSchedule` taint with the following command:
            
            ```bash
            kubectl taint nodes ws690t node-role.kubernetes.io/control-plane:NoSchedule-
            
            ```
            
            This will allow regular workloads to be scheduled on the node, while it continues running control-plane services.
            
        3. **(Optional) Assign a worker role label**:
        You can also label the node as both `control-plane` and `worker`, though this is not required for it to function as a worker. The labels are purely for categorization and organization:
            
            ```bash
            kubectl label node ws690t node-role.kubernetes.io/worker=
            
            ```
            
        
        ### Verify:
        
        - After these steps, verify that workloads are being scheduled on the node:
            
            ```bash
            kubectl get nodes -o wide
            
            ```
            
        
        Now your `ws690t` node will act as both a control-plane node and a worker node, scheduling and running workloads while still functioning as part of the control plane.
        
    
    ```bash
    kubectl taint nodes ws690t node-role.kubernetes.io/control-plane:NoSchedule-
    ```
    
    ```bash
    kubectl label node ws690t node-role.kubernetes.io/worker=
    ```
    
    The output shows that the node `ws690t` is currently tainted with `node.kubernetes.io/disk-pressure:NoSchedule`. This means Kubernetes will not schedule new pods on this node due to disk pressure.
    
    To make the node act as both a **control-plane** and a **worker node**, you need to resolve the disk pressure issue (since it's blocking pod scheduling), and there is no `control-plane` taint present that would otherwise need to be removed.
    
    ### Steps to proceed:
    
    1. **Check Disk Usage**:
    First, inspect the disk space usage on the node to see if there's indeed a disk pressure issue:
        
        ```bash
        df -h
        ```
        
        - If a particular mount is running out of space, clean up any unnecessary files, logs, or containers that are consuming disk space.
    2. **Clear the `disk-pressure` Taint**:
    Once you’ve resolved the disk space issue, you can manually remove the `disk-pressure` taint to allow new workloads to be scheduled:
        
        ```bash
        kubectl taint nodes ws690t node.kubernetes.io/disk-pressure:NoSchedule-
        ```
        
    3. **Verify**:
    After removing the taint, verify the node status and make sure it can schedule workloads:
        
        ```bash
        kubectl get nodes -o wide
        ```
        
    
    Once the `disk-pressure` condition is cleared and the taint is removed, the node should be able to act as both a control-plane and worker node.
    
- docker clear storage
    
    ```bash
    docker system prune --all --volumes
    ```
    
- 卡在 Terminating
    
    ```bash
    kubectl delete pod <pod-name> -n skywalking-showcase --force --grace-period=0
    ```
    
    ```bash
    for pod in $(kubectl get pods -n skywalking-showcase | grep Terminating | awk '{print $1}'); do
        kubectl delete pod $pod -n skywalking-showcase --force --grace-period=0
    done
    ```
    

```docker
--cri-socket=unix:///var/run/cri-dockerd.sock
```

---

### bug

- couldn't get current server API group list
    
    ```bash
    $ kubectl get node                                                                                                                                                                                                                        414ms
    E0927 00:00:45.897876  898352 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
    E0927 00:00:45.898033  898352 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
    E0927 00:00:45.899185  898352 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
    E0927 00:00:45.900304  898352 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
    E0927 00:00:45.901847  898352 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
    The connection to the server localhost:8080 was refused - did you specify the right host or port?
    ```
    
    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
    
- stuck on `kubeadm join`
    - token issue:
        
        ```bash
        kubeadm token create
        ```
        
        ```bash
        openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
        ```
        
        ```bash
        kubeadm token create --print-join-command
        ```
        
    - open port 6443 to firewall
        
        ```bash
        sudo ufw enable
        ```
        
        ```bash
        sudo ufw allow 6443/tcp
        ```
        
        ```bash
        sudo ufw status # list opened portstatus
        ```
        
        ```bash
        sudo ufw reload
        ```
        
        ```bash
        sudo systemctl restart ufw
        ```
        
        ```bash
        telnet your_server_ip 6443 # check port status 
        ```
        
- cri-docker `00fe8f2013317dbed8308e8171c83fc4f51d640a`
    
    我實驗室電腦用的是
    
    ```bash
    00fe8f2013317dbed8308e8171c83fc4f51d640a
    ```
    
- **calico/node is not ready: BIRD is not ready: BGP not established with**
    
    我自己的解法：reset cluster, apply calico before join the worker node
    
    https://blog.csdn.net/qq_19734597/article/details/128083040
    
    ```bash
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: enp5s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
        link/ether d4:5d:64:3f:e6:6d brd ff:ff:ff:ff:ff:ff
    3: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether d4:5d:64:3f:e6:6c brd ff:ff:ff:ff:ff:ff
        altname enp0s31f6
        inet 192.168.0.101/24 brd 192.168.0.255 scope global noprefixroute eno1
           valid_lft forever preferred_lft forever
        inet6 fe80::24d8:e2d5:bfd:f732/64 scope link noprefixroute
           valid_lft forever preferred_lft forever
    4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
        link/ether 02:42:21:09:61:f8 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
           valid_lft forever preferred_lft forever
    6: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
        link/ether 52:54:00:b7:37:09 brd ff:ff:ff:ff:ff:ff
        inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
           valid_lft forever preferred_lft forever
    15: wlx3c52a124ce8b: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 3c:52:a1:24:ce:8b brd ff:ff:ff:ff:ff:ff
        inet 192.168.50.250/24 brd 192.168.50.255 scope global dynamic noprefixroute wlx3c52a124ce8b
           valid_lft 77934sec preferred_lft 77934sec
        inet6 fe80::3f72:d59c:8366:e53b/64 scope link noprefixroute
           valid_lft forever preferred_lft forever
    16: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1480 qdisc noqueue state UNKNOWN group default qlen 1000
        link/ipip 0.0.0.0 brd 0.0.0.0
        inet 192.168.29.128/32 scope global tunl0
           valid_lft forever preferred_lft forever
    182: calib9cd446acbc@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default
        link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
        inet6 fe80::ecee:eeff:feee:eeee/64 scope link
           valid_lft forever preferred_lft forever
    183: cali06c7719c614@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default
        link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
        inet6 fe80::ecee:eeff:feee:eeee/64 scope link
           valid_lft forever preferred_lft forever
    ```
    

---

### todo

- train master

---

## references

https://github.com/9501sam/yunikorn-note/blob/main/yunikorn.md

https://chat.openai.com/c/9cbe4d67-7561-4b24-a42b-4ceac7fea53b

https://cwiki.apache.org/confluence/display/PAIMON/Paimon

https://linuxconfig.org/how-to-disable-enable-gui-in-ubuntu-22-04-jammy-jellyfish-linux-desktop

https://ithelp.ithome.com.tw/articles/10294103 (解決 `Found multiple CRI endpoints on the host.` 的問題)

https://ithelp.ithome.com.tw/articles/10235069 (【從題目中學習k8s】-【Day3】建立K8s Cluster環境-以kubeadm為例)

https://ithelp.ithome.com.tw/articles/10295266 **(use Calico)**
