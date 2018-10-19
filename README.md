# GO (TCP) Echo

A Simple go TCP echo server. Written to learn and test [Kubernetes | Istio] TCP networking.

## Build the Docker image
Clone this repo and switch to this directory, and then run from a terminal:

>> Note: you should replace the tag with your own tag.

```bash
docker build -t registry.cn-hangzhou.aliyuncs.com/wangxining/tcptest:0.1 .
```

And push this image:
```bash
docker push registry.cn-hangzhou.aliyuncs.com/wangxining/tcptest:0.1
```

## Test with [Docker] before deploying to Kubernetes and Istio

Run the container from a terminal:
```bash
docker run --rm -it -e TCP_PORT=3333 -e NODE_NAME="EchoNode" -p 3333:3333 registry.cn-hangzhou.aliyuncs.com/wangxining/tcptest:0.1
```

In another terminal run:
```bash
nc localhost 3333
Welcome, you are connected to node EchoNode.
hello
hello
```

## Test with [Kubernetes and Istio]

Create the deployment:
```bash
cd k8s
kubectl apply -f deployment.yml
```

Create the service:
```
kubectl apply -f service.yml
```

You should now have two TCP echo containers running:

```bash
kubectl get pods --selector=app=tcp-echo
```

```bash
NAME                           READY     STATUS    RESTARTS   AGE
tcp-echo-v1-7c775f57c9-frprp   2/2       Running   0          1m
tcp-echo-v2-6bcfd7dcf4-2sqhf   2/2       Running   0          1m
```

You should also have a service:

```bash
kubectl get service --selector=app=tcp-echo
```

```bash
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
tcp-echo   ClusterIP   172.19.46.255   <none>        3333/TCP   17h

```

Create the destination rule for this service:
```bash
kubectl apply -f destination-rule-all.yaml
```

Create the gateway and virtual service:
```bash
kubectl apply -f gateway.yaml
```

Echo some data, replace INGRESSGATEWAY_IP with the external IP of Istio Ingress Gateway:
```
nc INGRESSGATEWAY_IP 31400
```

After connecting, type the word hello and hit return:
```bash
Welcome, you are connected to node cn-beijing.i-2zeij4aznsu1dvd4mj5c.
Running on Pod tcp-echo-v1-7c775f57c9-frprp.
In namespace default.
With IP address 172.16.2.90.
Service default.
hello, app1
hello, app1
continue..
continue..
```

Verify the logs from version 1 POD:
```bash
kubectl logs -f tcp-echo-v1-7c775f57c9-frprp -c tcp-echo-container | grep Received
2018/10/17 07:32:29 6c7f4971-40f1-4f72-54c4-e1462a846189 - Received Raw Data: [104 101 108 108 111 44 32 97 112 112 49 10]
2018/10/17 07:32:29 6c7f4971-40f1-4f72-54c4-e1462a846189 - Received Data (converted to string): hello, app1
2018/10/17 07:34:40 6c7f4971-40f1-4f72-54c4-e1462a846189 - Received Raw Data: [99 111 110 116 105 110 117 101 46 46 10]
2018/10/17 07:34:40 6c7f4971-40f1-4f72-54c4-e1462a846189 - Received Data (converted to string): continue..
```

Switch to another port 31401, replace INGRESSGATEWAY_IP with the external IP of Istio Ingress Gateway:
```
nc INGRESSGATEWAY_IP 31401
```

After connecting, type the word hello and hit return:
```bash
Welcome, you are connected to node cn-beijing.i-2zeij4aznsu1dvd4mj5b.
Running on Pod tcp-echo-v2-6bcfd7dcf4-2sqhf.
In namespace default.
With IP address 172.16.1.95.
Service default.
hello, app2
hello, app2
yes,this is app2
yes,this is app2
```

```bash
kubectl logs -f tcp-echo-v2-6bcfd7dcf4-2sqhf -c tcp-echo-container | grep Received
2018/10/17 07:36:29 1a70b9d4-bbc7-471d-4686-89b9234c8f87 - Received Raw Data: [104 101 108 108 111 44 32 97 112 112 50 10]
2018/10/17 07:36:29 1a70b9d4-bbc7-471d-4686-89b9234c8f87 - Received Data (converted to string): hello, app2
2018/10/17 07:36:37 1a70b9d4-bbc7-471d-4686-89b9234c8f87 - Received Raw Data: [121 101 115 44 116 104 105 115 32 105 115 32 97 112 112 50 10]
2018/10/17 07:36:37 1a70b9d4-bbc7-471d-4686-89b9234c8f87 - Received Data (converted to string): yes,this is app2
```

## Resources
- [Expose Pod Information to Containers Through Environment Variables]
- [Docker]
- [Kubernetes]
- [Istio]


[Expose Pod Information to Containers Through Environment Variables]: https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/
[Docker]: https://www.docker.com/
[Kubernetes]: https://kubernetes.io/
[Istio]: https://istio.io/