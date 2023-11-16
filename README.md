## Setup 
It looks like grpc streams are not closed properly when Docker process is restart in below setup

Docker K8s zone <-> Nginx gateway <-> Envoy <-> Envoy <-> Global control-plane

but when I remove gateway from the chain Envoy detects and close active requests

Potential fix is setting `stream_idle_timeout` but this causes another issue. We are using Bi-directional GRPC and we don't want break the connection when there is no data for longer period of time.

## Reproduction
1. Download kuma
```bash
 curl -L https://kuma.io/installer.sh | sh -
```
2. Start docker kubernetes cluster
3. Start global control-plane locally (no k8s)

```bash
KUMA_MULTIZONE_GLOBAL_KDS_TLS_ENABLED=false KUMA_MODE=global ./kuma-cp run
```
4. Install nginx 
   
```bash
brew install nginx
```
5. Start nginx

```bash 
nginx -c $(pwd)/nginx.conf 
```

6. Start envoy

```bash
./envoy -c config.yaml
```

7. Start another envoy

```bash
./envoy -c config2.yaml
```

8. Install zone on k8s

```bash
./kumactl install control-plane \
  --set "kuma.controlPlane.mode=zone" \
  --set "kuma.controlPlane.zone=test-zone" \
  --set "kuma.ingress.enabled=true" \
  --set "kuma.controlPlane.kdsGlobalAddress=grpc://host.docker.internal:20000" \
  --set "kuma.controlPlane.tls.kdsZoneClient.skipVerify=true" \
  --set "kuma.experimental.deltaKds=true" \
  | kubectl apply -f -
```

9. Restart docker using UI function `Restart`

You can notice that metric control-plane grows 
```
kds_delta_streams_active{zone="Global"} 2
```

and envoy metrics has more active requests

```
grpc_service::127.0.0.1:5685::rq_active::10
```

with another restart it gets more