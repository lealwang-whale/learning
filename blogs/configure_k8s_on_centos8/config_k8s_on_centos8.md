# Configure kubernetes cluster on CentOS8
## server preparation
- CentOS8 servers on cloud (1 master node and 1 worker node)
> here, I prepare two servers running CentOS8
> server|public IP|private IP
> --|:--:|--:
> master-node|43.156.25.56|10.0.8.14
> node-1|43.156.30.247|10.0.8.10
- Internet access and connectivity capacity on all nodes, either on public or private network. Make sure all nodes are able to connect to others. You can use ***ping command*** to verify this.  
```shell
#on master-node
# ping 10.0.8.10
# ping 43.156.30.247
#on node-1
# ping 10.0.8.14
# ping 43.156.25.56
```

## configuration and installation

*NOTICE:*
 1. step 1 - 3 configure for all nodes;
 2. step 4 configure for master node only;
 3. step 5 configure for worker node only.

### step 1: environment configuration **[on all nodes]**

1. set hostname on nodes.
    - master-node server
        ```shell 
        # hostnamectl set-hostname master-node
        ```
    - worker-node server
        ```shell 
        # hostnamectl set-hostname node-1
        ```
2. turn off the system firewall and set the selinux state.
    ```shell
    # systemctl stop firewalld && systemctl disable firewalld && setenforce 0
    ```
3. modify the /etc/hosts files.
    ```shell
    # vim /etc/hosts
    10.0.8.14 master-node
    10.0.8.10 node-1 worker-node-1
    ```
4. reboot the servers.
    ```shell
    # reboot
    ```
    After rebooting, you can see that the hostname has changed to the set hostname, like the below:  
    ```shell
    #on master node
    [root@master-node ~]#
    #on worker node 
    [root@node-1 ~]#
    ```
    you can run ***ping command*** again to check your hostname configuration.
    ```shell
    #on master-node
    # ping node-1
    #on node-1
    # ping master-node
    ```


5. configure port rules  
    open kubernetes commonly used ports.
    ```shell
    # firewall-cmd --permanent --add-port=6443/tcp
    # firewall-cmd --permanent --add-port=10248-10255/tcp
    # firewall-cmd --permanent --add-port=2379-2380/tcp
    ```
    reload firewall ruless
    ```shell
    # firewall-cmd --reload
    ```
    
Please **NOTES** that the port access rule of above ports on the servers need to be disclosed on your cloud console. For AWS users, you can configure the security group to open the port access.

### step2: docker installation **[on all nodes]**

1. add the docker repository.
    ```shell
    # dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
    ```
2. install *docker-ce* (Community Edition) via ***dnf command***.
   ```shell
    # dnf install docker-ce
    ```

3. [*optional*] ensure docker configure the same *cgroupdriver* as kubernetes.The default *cgroupdriver* of kubernetes is *systemd*. You could edit file */etc/docker/daemon.json* to modify it with adding: 
    ```json
    {
        "exec-opts": ["native.cgroupdriver=systemd"]
    }
    ```

4. setup and enable docker service.
    ```shell
    # systemctl enable docker
    # systemctl start docker
    ```

### step3: kubernetes component setup **[on all nodes]**

1. configure kubernetes yum source .
    ```shell
    # cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF
    ```
2. install *kubeadm*.
    ```shell
    # dnf install kubeadm -y 
    ```
3. startup *kubelet* service.
    ```shell
    # systemctl enable kubelet
    # systemctl start kubelet
    ```
### step4: setup control-plane on master **[on master node only]**

1. disable swap space of system.
    ```shell
    # swapoff -a
    ```
2. initialize the kubernetes master.
    ```shell
    # kubeadm init
    ```
    If running successfully, you would see the log showing the following command:
    ```shell
    kubeadm join 10.0.8.14:6443 --token girs1w.c1hyfyr3qk933995 --discovery-token-ca-cert-hash sha256:3963d4dcbd2ba003397c50ba023bca0b534d2c9732db26e3d4faee19f2881823
    ```
    The generated command would be further executed on slaves to join the cluster. Also you can run command to check all valid tokens on your cluster.
    ```shell
    # kubeadm token list
    ```
3. configure pod network via *Weavenet*
    ```shell
    # export kubever=$(kubectl version | base64 | tr -d '\n')
    # kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$kubever"
    ```

### step5: salves join the cluster **[on worker nodes only]**
1. execute the generated command from master.
    ```shell
    # kubeadm join 10.0.8.14:6443 --token girs1w.c1hyfyr3qk933995 --discovery-token-ca-cert-hash sha256:3963d4dcbd2ba003397c50ba023bca0b534d2c9732db26e3d4faee19f2881823
    ```

### step6: verify cluster status

Now, you can verify your cluster status on master node via:  
```shell
# kubectl get nodes
```
If all steps go well, you would see all nodes are ready in the cluster.
```shell
NAME          STATUS   ROLES                  AGE   VERSION
master-node   Ready    control-plane,master   27h   v1.23.5
node-1        Ready    <none>                 22h   v1.23.5
```
