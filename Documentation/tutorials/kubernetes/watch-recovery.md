# Learn Kubernetes Testing Failure and Recovery

This guide is designed to help you learn how Kubernetes components interact together and can failover automatically. Similar to [Kubernetes "the hard way"][hardway], these steps will walk through triggering certain failures and triaging how the cluster handles them.

One of the core features of Kubernetes is that it’s designed to maintain the desired state defined by operators and app admins. A series of **reconciliation loops** constantly works to find the optimal path from the current state to the desired state for all components of the cluster.

This makes the operator's job easy. They can simply declare that a node should be pulled from serving workloads, or that a certain permission should be allowed.

From the system's perspective, handling a container failure is easy because several simple loops are always running. To handle a single failure of a highly available apps, each loop
* ensures that the correct number of containers is running and creates any that aren't;
* ensures that all unassigned containers are assigned to nodes that are healthy;
* ensures that each node is running containers assigned to it; and
* ensures that traffic is load balanced only to healthy backend containers.

The same logic is applied to all desired states, with any impacted component reacting appropriately. These simple loops scale well, and are easier to test and easier to debug than a centralized black box.

## Set up a test workload

First, make sure you are running a sample app. Follow the [Tectonic on AWS tutorials][first-app] to launch a simple app, or use one of your own.

The API server acts as the brains of the cluster and is run as a Deployment with multiple [Pods][pods] to ensure High Availability. Each Deployment runs on a master node. To demonstrate the automated recovery process of a Kubernetes cluster, we will first intentionally kill a copy of the API server on the running app, and then simulate a node failure. After each of these, we will watch as the cluster recovers.

## Simulate Pod failure

First, let’s demonstrate that the Control Plane is run in High Availability by intentionally killing an API server Pod process, and then watch it recover. This will also show the rescheduling process of ReplicaSets and reconciliation loops.

Open a terminal and a browser window with Tectonic Console. Keep them open side by side for this exercise. Kubernetes will rebuild the killed containers quickly.

First, use Tectonic Console to determine which nodes are running the API server.
Go to *Workloads > Pods*, and enter ‘api’ in the search field.

Then, look at the details of one of the nodes listed, and SSH to its IP address.

```
$ ssh core@ip
```

Remember that the kubelet (the Kubernetes node agent) runs on reconciliation loops which work to keep the system in a stable state. When we execute the command to kill this API server, the kubelet will notice immediately.

Test this by killing the active containers running the Kubernetes API server on this node. Because the API server is a DaemonSet, it should be running on all of the master nodes.

From one of the masters, get the IDs of the containers running the API Server:

<!-- {% raw %} -->
```
$ docker ps -f 'name=.*kube-apiserver.*' --format "{{.ID}}"
4ae686927f22
06bb5ba95034
```
<!-- {% endraw %} -->

Then, kill the listed containers:

```
$ docker kill 4ae686927f22 06bb5ba95034
```

Within a few seconds the kubelet will notice that the API server Pod is no longer running, and will relaunch under a new ID:

<!-- {% raw %} -->
```
$ docker ps -f 'name=.*kube-apiserver.*' --format "{{.ID}}"
f4a262619d1d
d964fa94c69b
```
<!-- {% endraw %} -->

In the Console, watch the list of API servers to see that one is disabled, and a new one is launched. Note that the Tectonic Console session offers continuous feedback because the API is configured with High Availability. This is the same Kubernetes feature that enables your apps to remain up and running without constant developer monitoring and interaction.

We just proved that our cluster can not only recover from a single Pod failing, but it can handle failure of the most critical type, its own API server.

## Simulate Node failure

Killing the API server acts as an example of an individual process failure. Next, simulate a power down or network partition by disabling an entire node.

Kubelet uses reconciliation loops to remain in constant contact with the API server and report back status. Use iptables to create a temporary firewall rule and simulate an interruption in this connection.

```
$ iptables -A OUTPUT -p tcp --dport 443  -j DROP
$ systemctl stop dockerk
$ systemctl stop kubelet
$ ps aux | grep kube
```

Use Tectonic Console to inspect the node. Notice that the node is marked ‘unhealthy’, but nothing else has changed about workloads.

Check your app. Even though the node is unhealthy, and the rescheduling process has not yet begun, the app is still up and running as before. Because the cluster runs in High Availability, the app does not go down, and your clients experience no interruption in service. Kubernetes maintains persistent availability automatically.

To ensure that workloads aren't constantly shuffled around for a minor networking outage, the kubelet will wait 5 minutes for transient errors to clear, before taking more drastic action. After 5 minutes, reinspect Console. Notice that new Pods have been started and scheduled to repair the interruption.

Undo the temporary firewall rule to watch the node recover almost immediately.

```
$ iptables -A OUTPUT -p tcp --dport 443  -j ACCEPT
$ systemctl start dockerk
$ systemctl enable kubelet
$ ps aux | grep kube
```

## More learning opportunities

This guide served as a basic introduction to Kubernetes, and the power of Tectonic to manage your clusters. To dig more deeply into the sophistication of these systems, browse both the [Kubernetes][k8s-doc] and the [Tectonic][tectonic-doc] documentation.

Suggested topics:
* [Managing resources for containers][managing-resources]
* [Managing memory requests for namespaces][managing-memory]
* [Autoscaling the number of Pods][autoscaling-pods]


[autoscaling-pods]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale
[first-app]: ../aws/first-app.md
[hardway]: https://github.com/kelseyhightower/kubernetes-the-hard-way
[k8s-doc]: https://kubernetes.io/docs/home
[pods]: https://coreos.com/kubernetes/docs/latest/pods.html
[managing-resources]: https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container
[managing-memory]: https://kubernetes.io/docs/tasks/administer-cluster/memory-default-namespace
[tectonic-doc]: https://coreos.com/tectonic/docs/latest
