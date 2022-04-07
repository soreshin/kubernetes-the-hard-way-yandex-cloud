# Smoke Test

In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.

## Data Encryption

In this section you will verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

```
{
  CONTROLLER0_IP=$(yc compute instance get --name controller-0 --format json | jq -r '.network_interfaces[0].primary_v4_address.one_to_one_nat.address')

  ssh yc-user@${CONTROLLER0_IP} \
    "sudo ETCDCTL_API=3 etcdctl get \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/etcd/ca.pem \
    --cert=/etc/etcd/kubernetes.pem \
    --key=/etc/etcd/kubernetes-key.pem\
    /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
}
```

> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a e9 71 6d 37 73 1c 7e  |:v1:key1:.qm7s.~|
00000050  a9 50 ed 59 e7 73 42 b6  29 38 28 5b d2 35 27 f9  |.P.Y.sB.)8([.5'.|
00000060  b4 eb 1a 7b ed 17 d0 f5  bf 25 4e f1 34 b9 2a 75  |...{.....%N.4.*u|
00000070  a3 55 df b8 36 11 ec 84  08 98 e9 b4 ec c5 82 38  |.U..6..........8|
00000080  03 81 a1 93 ee 56 bf c9  06 f0 81 e2 46 38 12 eb  |.....V......F8..|
00000090  5d 07 de 1c a6 35 a3 be  e3 63 4b ac f7 b0 0c e0  |]....5...cK.....|
000000a0  53 54 af db f9 56 f0 10  68 f0 96 ac c5 ef b1 1c  |ST...V..h.......|
000000b0  28 e6 3b ae da 50 50 8e  4c c1 88 3a 8f 2f 9f 32  |(.;..PP.L..:./.2|
000000c0  b8 5e ce 34 ab 0d 34 af  83 c9 30 4c d1 87 e1 c0  |.^.4..4...0L....|
000000d0  cf e4 a0 f6 d8 4e ca 79  bc b1 a9 7f c7 b7 d0 68  |.....N.y.......h|
000000e0  63 d4 24 39 04 0e a2 36  d6 35 05 ec d6 6b 97 63  |c.$9...6.5...k.c|
000000f0  c9 e5 fa 9c 8e fb 6c f8  c7 cb 69 18 94 2c fc 45  |......l...i..,.E|
00000100  4e f1 7a 1a f2 33 79 20  60 81 9b 8f a4 03 a7 ba  |N.z..3y `.......|
00000110  53 d6 78 3e 86 37 0c af  66 1e 76 b8 5f 60 0b 58  |S.x>.7..f.v._`.X|
00000120  b1 fe 61 5e 62 3b 5a 31  0a 62 cc f4 31 f5 ea b1  |..a^b;Z1.b..1...|
00000130  96 be ff d4 39 7c 73 80  07 c1 fd 9c ad 3e 0b e6  |....9|s......>..|
00000140  f3 74 a3 1c 02 27 11 74  12 b0 bd c5 20 2d 25 98  |.t...'.t.... -%.|
00000150  2c 54 29 2d c2 1e c8 91  aa 0a                    |,T)-......|
0000015a
```

The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

## Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```
kubectl create deployment nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```
kubectl get pods -l app=nginx
```

> output

```
NAME                    READY   STATUS    RESTARTS   AGE
nginx-f89759699-kpn5m   1/1     Running   0          10s
```

### Port Forwarding

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```
kubectl port-forward $POD_NAME 8080:80
```

> output

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```
curl --head http://127.0.0.1:8080
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Tue, 05 Apr 2022 17:21:47 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 25 Jan 2022 15:03:52 GMT
Connection: keep-alive
ETag: "61f01158-267"
Accept-Ranges: bytes
```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```
kubectl logs $POD_NAME
```

> output

```
...
127.0.0.1 - - [05/Apr/2022:17:21:47 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.77.0" "-"
```

### Exec

In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```
nginx version: nginx/1.19.10
```

## Services

In this section you will verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```
kubectl expose deployment nginx --port 80 --type NodePort
```

> The LoadBalancer service type can not be used because your cluster is not configured with [cloud provider integration](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider). Setting up cloud provider integration is out of scope for this tutorial.

Retrieve the node port assigned to the `nginx` service:

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Retrieve the external IP address of a worker instance:

```
EXTERNAL_IP=$(yc compute instance get --name worker-0 --format json | jq -r '.network_interfaces[0].primary_v4_address.one_to_one_nat.address')
```

Make an HTTP request using the external IP address and the `nginx` node port:

```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.19.10
Date: Sun, 02 May 2021 05:31:52 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Apr 2021 15:13:59 GMT
Connection: keep-alive
ETag: "6075b537-264"
Accept-Ranges: bytes
```

Next: [Cleaning Up](14-cleanup.md)
