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

install `istioctl` following the instruction of the official website.

```
export ISTIO_VERSION=1.9.0
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.9.0/
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y
```

install `strimzi` after download `strimzi-0.21.1.tar.gz`

```
tar -xzf strimzi-0.21.1.tar.gz
cd strimzi-0.21.1/install
kubectl apply -f cluster-operator/
```

Also, original Dockerfile is modified, we use curl to install docker-compose in the stack-buillder

```
## Dockerfile.deploy
sudo curl -L "https://github.com/docker/compose/releases/download/1.28.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose
sudo chmod +x /usr/bin/docker-compose
```



deploy trace grapher

```
cd trace-grapher
docker-compose run stack-builder
# now a shell pops as root in the project directory of the stack-builder container
cd deploy-trace-grapher
make prepare-trace-grapher-namespace
make install-components
```





```
trace-grapher    jupyter-0                                     0/1     Pending            0          8d
trace-grapher    kafka-connect-ui-ff8b9f5c-8w9bb               1/1     Running            0          8d
trace-grapher    neo4j-0                                       0/1     Pending            0          8d
```

But I have no idea how to use this is thing...

Install deployment module:

```
cd scripts
make all
cd python-cat-mba
make env
```

another error shows up, it seems we haven't install `intel-cmt-cat`, which is in `firm/third-party`. I suspect that we have to install all the third parties first.