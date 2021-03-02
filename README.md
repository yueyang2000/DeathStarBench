# DeathStarBench on Kubernetes

Open-source benchmark suite for cloud microservices. DeathStarBench includes five end-to-end services, four for cloud systems, and one for cloud-edge systems running on drone swarms. 

Released services: 

* Social Network
* Media Service
* Hotel Reservation 

More details on the applications and a characterization of their behavior can be found at ["An Open-Source Benchmark Suite for Microservices and Their Hardware-Software Implications for Cloud and Edge Systems"](http://www.csl.cornell.edu/~delimitrou/papers/2019.asplos.microservices.pdf), Y. Gan et al., ASPLOS 2019. 

## Basic Setup

1. let iptables see bridged traffic

   ```bash
   lsmod | grep br_netfilter
   cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
   br_netfilter
   EOF
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   EOF
   sudo sysctl --system
   ```

2. install docker and setup daemon.

   ```bash
   sudo su
   # install docker
   apt-get update
   apt install docker.io
   # setup daemon
   cat > /etc/docker/daemon.json <<EOF
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "100m"
     },
     "storage-driver": "overlay2"
   }
   EOF
   exit
   ```

   Edit /etc/group, add yourself to `docker` group

   restart docker, make sure the service is online

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   sudo systemctl enable docker.service
   ```

   run a image clean-up job

   ```bash
   docker run -d --restart=always \
     -v /var/run/docker.sock:/var/run/docker.sock:rw \
     -v /var/lib/docker:/var/lib/docker:rw \
     -e "CLEAN_PERIOD=86400" \
     meltwater/docker-cleanup:latest
   ```

3. Disable swapoff and install kubernetes packages with the following commands:

   ```bash
   # disable swapoff temporary
   swapoff -a
   # install k8s packages
   curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
   cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
   deb http://apt.kubernetes.io/ kubernetes-xenial main
   EOF
   apt-get update
   apt-get install -y kubelet kubeadm kubectl nfs-common
   apt-mark hold kubelet kubeadm kubectl
   sudo systemctl daemon-reload
   sudo systemctl restart kubelet
   ```

Now you should have kubeadm and kubectl available on your linux machine.

## Create Kubernetes Cluster

You need at least 2 linux instances to run a k8s cluster. (one for master and one for slave) Follow the steps below to create a cluster:

1. Run basic setup on every node. Specify a master node. (others are slave nodes)

2. (on master node) Init cluster with `kubeadm`, copy the `kubeadm join` command printed on the terminal.

   ```bash
   sudo kubeadm init --pod-network-cidr=10.244.0.0/16
   # create directory for the cluster
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

4. (on slave node) paste the `kubeadm join` command that was saved in step 2

   ```bash
   # this is the command in my cluster's case
   sudo kubeadm join 10.0.0.7:6443 --token tmdxkp.m9712eq7zeplnif6 \
       --discovery-token-ca-cert-hash sha256:8a2e2259e82d7b8d4022c3f8323bb4a22f8d55fce6c000f78a308ef6b304a3ad
   ```

5. (on master node) install fannel pod-network

   ```bash
   # apply pod network
   sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   ```
   
you can check if the pod-network is online by `kubectl get pods --all-namespaces`
   
5. (on master node) Now the cluster is successfully created. You can check the status of every node by `kubectl get nodes` and you will be greeted by something like this:

   ```
   NAME        STATUS   ROLES                  AGE     VERSION
   autosys-4   Ready    control-plane,master   3m51s   v1.20.2
   autosys-5   Ready    <none>                 28s     v1.20.2
   ```

   `kub-dns` service is already online, check its cluster IP by `kubectl get service -n kube-system`, which may come in handy later on.

   ```
   NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
   kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   11m
   ```

## Deploy Social Network

1. Place this repo **in the same path** on every node. Build a local image of the service with namespace `local` and tag `v1`.

   ```bash
   git clone https://github.com/yueyang2000/DeathStarBench.git
   # build social-network local image
   cd DeathStarBench/socialNetwork
   docker build -t local/social-network-microservices:v1 .
   ```

2. Change every `/home/yueyang` to `<path-to-repo>` in `k8s-yml/media-frontend.yaml` and `k8s-yml/nginx-thrift.yaml`

3. Install dependencies

   ```bash
   sudo apt install libssl-dev
   sudo apt install libz-dev
   sudo apt install luarocks
   sudo luarocks install luasocket
   ```

4. Apply `social-network-ns.yaml` to create namespace, and then apply other yamls.

   ```bash
   # create namespace
   kubectl apply -f socialNetwork/k8s-yaml/social-network-ns.yaml
   # deploy everything
   kubectl apply -f socialNetwork/k8s-yaml/
   ```

5. Wait a few moment and every pod should be online. Check pod status with `kubectl -n social-network get pod` .

   ```
   NAME                                          READY   STATUS    RESTARTS   AGE
   compose-post-redis-8df45b9d9-9szgs            1/1     Running   0          52m
   compose-post-service-596d8d9f89-hstxg         1/1     Running   0          52m
   home-timeline-redis-6f4c5d55fc-6jpnx          1/1     Running   0          52m
   home-timeline-service-79849956fc-bjv8d        1/1     Running   0          52m
   jaeger-79df655c6-mmgcx                        1/1     Running   0          52m
   media-frontend-679c864bcf-4z5wd               1/1     Running   0          91s
   media-memcached-7d9ff5d6bb-m7l7j              1/1     Running   0          52m
   media-mongodb-5c7b85c65d-c45jx                1/1     Running   0          52m
   media-service-dfc4b58c6-qmw87                 1/1     Running   0          52m
   nginx-thrift-64cf998856-cjk7x                 1/1     Running   0          6m36s
   post-storage-memcached-67b5c87bdb-plkkk       1/1     Running   0          52m
   post-storage-mongodb-695cd587f6-2h6wx         1/1     Running   0          52m
   post-storage-service-5688c98894-zkm4h         1/1     Running   0          52m
   social-graph-mongodb-84d498dc7b-56r8s         1/1     Running   0          52m
   social-graph-redis-6686bb4f78-gmskx           1/1     Running   0          52m
   social-graph-service-7f8d4fb55-6j8vk          1/1     Running   0          52m
   text-service-59fc7bd9bb-hwntn                 1/1     Running   0          50m
   unique-id-service-579cc5f997-sxpnp            1/1     Running   0          52m
   url-shorten-memcached-588494c4c7-bn2g8        1/1     Running   0          52m
   url-shorten-mongodb-8678cd5b77-rhtj6          1/1     Running   0          52m
   url-shorten-service-5dccc4c9-5q9p6            1/1     Running   0          52m
   user-memcached-6b8c6fb85f-mr6cj               1/1     Running   0          52m
   user-mention-service-57b579cc79-x9kjz         1/1     Running   0          52m
   user-mongodb-669f794897-2sf5l                 1/1     Running   0          52m
   user-service-5849995cf4-tm7h7                 1/1     Running   0          52m
   user-timeline-mongodb-7d5c79b677-d55br        1/1     Running   0          52m
   user-timeline-redis-6b54b58777-p7t99          1/1     Running   0          52m
   user-timeline-service-585fd7cf96-l4snd        1/1     Running   0          52m
   write-home-timeline-rabbitmq-fdc74669-9qsgn   1/1     Running   0          52m
   write-home-timeline-service-9c77cc4cb-d9zjc   1/1     Running   3          52m
   ```

6. use command `kubectl get svc -n social-network` to check the port maps.

   ```
   nginx-thrift                   NodePort    10.99.196.255    <none>        8080:30283/TCP
   ```
   
   `ssh -L <your-port>:127.0.0.1:30283 <server>` Then you can visit frontend at `localhost:<your-port>` 
   
## Reference

> [DeathStarBench](https://github.com/delimitrou/DeathStarBench)
>
> [FIRM's benchmark setup](https://gitlab.engr.illinois.edu/DEPEND/firm)
>
> [kubernetes official doc](https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/)

