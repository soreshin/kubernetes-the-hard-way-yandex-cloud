# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network.

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

Print the internal IP address and Pod CIDR range for each worker instance:

```
for instance in worker-0 worker-1 worker-2; do
  yc compute instance get ${instance} --full --format json \
  | jq -r '.network_interfaces[0].primary_v4_address.address + " " + .metadata."pod-cidr"'
done
```

> output

```
10.240.0.20 10.200.0.0/24
10.240.0.21 10.200.1.0/24
10.240.0.22 10.200.2.0/24
```

## Routes

Create network routes and attach them to subnet:

```
{
  yc vpc route-table create --name=kubernetes-route-table --network-name=kubernetes-the-hard-way \
    --route destination=10.200.0.0/24,next-hop=10.240.0.20 \
    --route destination=10.200.1.0/24,next-hop=10.240.0.21 \
    --route destination=10.200.2.0/24,next-hop=10.240.0.22

  yc vpc subnet update --name=kubernetes --route-table-name=kubernetes-route-table
}
```

List the routes:

```
yc vpc route-table list
```

> output

```
+----------------------+------------------------+-------------+----------------------+
|          ID          |          NAME          | DESCRIPTION |      NETWORK-ID      |
+----------------------+------------------------+-------------+----------------------+
| enp9gqplvejn4mbngetj | kubernetes-route-table |             | enpm8gice0t1h7547lb8 |
+----------------------+------------------------+-------------+----------------------+
```

And verify the route-table created in previous step is attached to subnet:

```
yc vpc subnet list
```

> output

```
+----------------------+------------+----------------------+----------------------+---------------+-----------------+
|          ID          |    NAME    |      NETWORK ID      |    ROUTE TABLE ID    |     ZONE      |      RANGE      |
+----------------------+------------+----------------------+----------------------+---------------+-----------------+
| e9bpp5qqk1f10e4lkiu9 | kubernetes | enpm8gice0t1h7547lb8 | enp9gqplvejn4mbngetj | ru-central1-a | [10.240.0.0/24] |
+----------------------+------------+----------------------+----------------------+---------------+-----------------+
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
