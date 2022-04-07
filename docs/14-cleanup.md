# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

```
yc compute instance delete \
  controller-0 controller-1 controller-2 \
  worker-0 worker-1 worker-2
```

## Networking

Delete the network load balancer and target groups:

```
{
  yc load-balancer network-load-balancer delete --name kubernetes-load-balancer

  yc load-balancer target-group delete --name kubernetes-target-pool
}
```

Delete the route-tables:

```
{
  yc vpc subnet update --name=kubernetes --disassociate-route-table

  yc vpc route-table delete kubernetes-route-table
}
```

Delete Public IP-Address

```
yc vpc address delete --name kubernetes-the-hard-way
```

Delete virtual subnet

```
yc vpc subnet delete --name kubernetes
```

Delete the `kubernetes-the-hard-way` virtual network

```
yc vpc network delete --name kubernetes-the-hard-way
```