# FIRM Deployment

### install dependencies

```bash
sudo apt-get install libssl-dev
sudo apt-get install libz-dev
sudo apt-get install luarocks
sudo luarocks install luasocket
```

### work load generation

```bash
./wrk -D exp -t 3 -c 20 -R 500 -d 1m -L -s ./scripts/social-network/compose-post.lua http://10.99.196.255:8080/wrk2-api/post/compose
```

### run anomaly injector 

`python3 injector.py`

- without cpu anomaly

```
3 threads and 20 connections
  Thread calibration: mean lat.: 3506.452ms, rate sampling interval: 12427ms
  Thread calibration: mean lat.: 3595.895ms, rate sampling interval: 12730ms
  Thread calibration: mean lat.: 3533.292ms, rate sampling interval: 12705ms
  Thread Stats   Avg      Stdev     99%   +/- Stdev
    Latency    20.38s     6.17s   28.74s    67.76%
    Req/Sec    47.40      1.56    50.00     90.00%
  Latency Distribution (HdrHistogram - Recorded Latency)
 50.000%   22.58s
 75.000%   25.54s 
 90.000%   26.66s
 99.000%   28.74s
 99.900%   29.20s
 99.990%   29.31s
 99.999%   29.33s
100.000%   29.33s

#[Mean    =    20381.570, StdDeviation   =     6167.817]
#[Max     =    29310.976, Total count    =         7187]
#[Buckets =           27, SubBuckets     =         2048]
----------------------------------------------------------
  8707 requests in 1.00m, 1.79MB read
  Socket errors: connect 0, read 0, write 0, timeout 3
Requests/sec:    145.10
Transfer/sec:     30.46KB
```

- with cpu anomaly

```
  3 threads and 20 connections
  Thread calibration: mean lat.: 4112.276ms, rate sampling interval: 14655ms
  Thread calibration: mean lat.: 4119.843ms, rate sampling interval: 14589ms
  Thread calibration: mean lat.: 4131.951ms, rate sampling interval: 14663ms
  Thread Stats   Avg      Stdev     99%   +/- Stdev
    Latency    27.07s    10.50s   42.93s    54.47%
    Req/Sec    29.67      2.45    32.00     77.78%
  Latency Distribution (HdrHistogram - Recorded Latency)
 50.000%   28.36s 
 75.000%   37.13s 
 90.000%   39.85s
 99.000%   42.93s
 99.900%   44.07s
 99.990%   44.92s
 99.999%   44.92s
100.000%   44.92s

#[Mean    =    27070.772, StdDeviation   =    10503.577]
#[Max     =    44892.160, Total count    =         4535]
#[Buckets =           27, SubBuckets     =         2048]
----------------------------------------------------------
  5503 requests in 1.00m, 1.13MB read
  Socket errors: connect 0, read 0, write 0, timeout 81
Requests/sec:     91.70
Transfer/sec:     19.25KB
```

The result shows that anomaly injector indeed takes effect.

### deploy firm

try to deploy everything given by firm's document

```
export NAMESPACE='monitoring'
kubectl create -f manifests/setup
kubectl create namespace observability
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/crds/jaegertracing.io_jaegers_crd.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/service_account.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role_binding.yaml
kubectl create -n observability -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/operator.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/cluster_role.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/cluster_role_binding.yaml
kubectl create -f manifests/
```

strimzi and istio CRD needs to be install before deploying trace-grapher.

- install `istioctl` following the instruction of the official website.

```
export ISTIO_VERSION=1.9.0
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.9.0/
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y
```

- install `strimzi`  following the quickstart guide

```
tar -xzf strimzi-0.21.1.tar.gz
```

create namespace kafka and modify the deployment files

```
kubectl create ns kafka
sed -i 's/namespace: .*/namespace: kafka/' install/cluster-operator/*RoleBinding*.yaml
```

create trace-grapher namespace

```
kubectl create trace-grapher
```

edit `install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml` and set `STRIMZI_NAMESPACE` to `trace-grapher`

```
env:
- name: STRIMZI_NAMESPACE
  value: trace-grapher
```

deploy CRDs

```
kubectl apply -f install/cluster-operator/ -n kafka
```

give permissions

```
kubectl apply -f install/cluster-operator/020-RoleBinding-strimzi-cluster-operator.yaml -n trace-grapher
kubectl apply -f install/cluster-operator/032-RoleBinding-strimzi-cluster-operator-topic-operator-delegation.yaml -n trace-grapher
kubectl apply -f install/cluster-operator/031-RoleBinding-strimzi-cluster-operator-entity-operator-delegation.yaml -n trace-grapher
```

- deploy trace grapher

Original Dockerfile is modified, we use curl to install docker-compose in the stack-buillder, so we don't need `py-pip, python-dev`

```
## Dockerfile.deploy
sudo curl -L "https://github.com/docker/compose/releases/download/1.28.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose
sudo chmod +x /usr/bin/docker-compose
```

run the following commands: 

```
cd trace-grapher
docker-compose run stack-builder
# now a shell pops as root in the project directory of the stack-builder container
cd deploy-trace-grapher
make prepare-trace-grapher-namespace
make install-components
```

Now `kubectl get pods --all-namespaces` should return something like this:

```
NAMESPACE        NAME                                          READY   STATUS             RESTARTS   AGE
istio-system     istio-egressgateway-65b9c8b54f-wgtx8          1/1     Running            0          53m
istio-system     istio-ingressgateway-56d9b7fdb-5pgxf          1/1     Running            0          53m
istio-system     istiod-89dc6db9c-j87bn                        1/1     Running            0          53m
kafka            strimzi-cluster-operator-644bcc4d44-ht2vc     1/1     Running            6          30m
kube-system      coredns-74ff55c5b-b8g7n                       1/1     Running            0          63m
kube-system      coredns-74ff55c5b-n6v62                       1/1     Running            0          63m
kube-system      etcd-autosys-4                                1/1     Running            0          63m
kube-system      kube-apiserver-autosys-4                      1/1     Running            0          63m
kube-system      kube-controller-manager-autosys-4             1/1     Running            0          63m
kube-system      kube-flannel-ds-2jt9b                         1/1     Running            0          62m
kube-system      kube-flannel-ds-nk89j                         1/1     Running            0          60m
kube-system      kube-proxy-ms96m                              1/1     Running            0          60m
kube-system      kube-proxy-vhd9h                              1/1     Running            0          63m
kube-system      kube-scheduler-autosys-4                      1/1     Running            0          63m
monitoring       alertmanager-main-0                           2/2     Running            0          54m
monitoring       alertmanager-main-1                           2/2     Running            0          54m
monitoring       alertmanager-main-2                           2/2     Running            0          54m
monitoring       grafana-7f567cccfc-z2tdk                      1/1     Running            0          54m
monitoring       kube-state-metrics-5f9c597bb9-86xrc           3/3     Running            0          54m
monitoring       node-exporter-87hfd                           2/2     Running            0          54m
monitoring       node-exporter-fvmd9                           2/2     Running            0          54m
monitoring       prometheus-adapter-557648f58c-dhlb8           1/1     Running            0          54m
monitoring       prometheus-k8s-0                              3/3     Running            1          54m
monitoring       prometheus-k8s-1                              3/3     Running            1          54m
monitoring       prometheus-operator-66558f76d9-cxlfk          2/2     Running            0          54m
observability    jaeger-operator-64bc449b74-9gvbg              1/1     Running            0          55m
social-network   compose-post-redis-8df45b9d9-k44nq            1/1     Running            0          58m
social-network   compose-post-service-596d8d9f89-pnbfc         1/1     Running            0          58m
social-network   home-timeline-redis-6f4c5d55fc-8lfjb          1/1     Running            0          58m
social-network   home-timeline-service-79849956fc-l482v        1/1     Running            0          58m
social-network   jaeger-79df655c6-fxvjb                        1/1     Running            0          58m
social-network   media-frontend-555bf4f69b-x7mx2               1/1     Running            0          58m
social-network   media-memcached-7d9ff5d6bb-m7m6t              1/1     Running            0          58m
social-network   media-mongodb-5c7b85c65d-2gs8k                1/1     Running            0          58m
social-network   media-service-dfc4b58c6-hbwcv                 1/1     Running            0          58m
social-network   nginx-thrift-7c8f5b4479-tmpg6                 1/1     Running            0          58m
social-network   post-storage-memcached-67b5c87bdb-44nlk       1/1     Running            0          58m
social-network   post-storage-mongodb-695cd587f6-9ktrs         1/1     Running            0          58m
social-network   post-storage-service-5688c98894-6wsfs         1/1     Running            0          58m
social-network   social-graph-mongodb-84d498dc7b-6qws9         1/1     Running            0          58m
social-network   social-graph-redis-6686bb4f78-rm7bp           1/1     Running            0          58m
social-network   social-graph-service-7f8d4fb55-j9wmd          1/1     Running            0          58m
social-network   text-service-59fc7bd9bb-c2cwg                 1/1     Running            0          58m
social-network   unique-id-service-579cc5f997-pxl6s            1/1     Running            0          58m
social-network   url-shorten-memcached-588494c4c7-rt98q        1/1     Running            0          58m
social-network   url-shorten-mongodb-8678cd5b77-msmzs          1/1     Running            0          58m
social-network   url-shorten-service-5dccc4c9-gbt76            1/1     Running            0          58m
social-network   user-memcached-6b8c6fb85f-gm6pk               1/1     Running            0          58m
social-network   user-mention-service-57b579cc79-65b7m         1/1     Running            0          58m
social-network   user-mongodb-669f794897-zmkmb                 1/1     Running            0          58m
social-network   user-service-5849995cf4-pltsv                 1/1     Running            0          58m
social-network   user-timeline-mongodb-7d5c79b677-5j9l8        1/1     Running            0          58m
social-network   user-timeline-redis-6b54b58777-vwtgt          1/1     Running            0          58m
social-network   user-timeline-service-585fd7cf96-tjb46        1/1     Running            0          58m
social-network   write-home-timeline-rabbitmq-fdc74669-rfrhw   1/1     Running            0          58m
social-network   write-home-timeline-service-9c77cc4cb-tk9nb   1/1     Running            3          58m
trace-grapher    jupyter-0                                     0/1     Pending            0          12m
trace-grapher    kafka-connect-connect-55469c56d5-wm4g6        0/1     CrashLoopBackOff   6          8m30s
trace-grapher    kafka-connect-ui-ff8b9f5c-5fh8x               1/1     Running            0          10m
trace-grapher    neo4j-0                                       0/1     Pending            0          12m
```

It seems that `kafka connect` operator is still malfunctional. A brief of the pod log:

```
Failed to create new KafkaAdminClient
Caused by:
No resolvable bootstrap urls given in bootstrap.servers
```



- Install deployment module:

```
cd scripts
make all
cd python-cat-mba
make env
```

`make env` failed, the error is:

```
created virtual environment CPython3.6.9.final.0-64 in 608ms
  creator CPython3Posix(dest=/home/yueyang/DeathStarBench/firm/scripts/python-cat-mba/env, clear=False, no_vcs_ignore=False, global=False)
  seeder FromAppData(download=False, pip=bundle, setuptools=bundle, wheel=bundle, via=copy, app_data_dir=/home/yueyang/.local/share/virtualenv)
    added seed packages: pip==21.0.1, setuptools==52.0.0, wheel==0.36.2
  activators BashActivator,CShellActivator,FishActivator,PowerShellActivator,PythonActivator,XonshActivator
make[1]: *** ../../lib/python: No such file or directory.  Stop.
```

