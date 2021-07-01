# Scaling and Deployment Considerations in AWS EKS

![alt text] (images/eks-elb-presentation.001.png)

Imagine a cluster with hundreds of services each with hundreds of replicas each running on EKS that are exposed with ALBs using the ALB ingress controller (1). In this case, we may also be running a target type of ip (configured in the ingress resource) if there are either a mixture of EC2 and Fargate workloads, or to minimise the number of hops between users and the pods.

```yaml
annotations:
  kubernetes.io/ingress.class: alb
  alb.ingress.kubernetes.io/target-type: ip
```

(1) https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html

![alt text] (images/eks-elb-presentation.002.png)

All services used a rolling deployment strategy, and therefore pod IPs are registered and deregistered from each service’s target group as they come alive and get terminated respectively using the RegisterTargets and DeregisterTargets API calls to the ELB service. Due to high rate of AWS API calls, we will likely get throttled calls which will be retried.

However if some of these API requests are throttled, this means that the ELB does not immediately register pod IPs that are alive and ready to serve traffic. More importantly, it still maintains pod IPs in target groups that (in reality) belong to pods that have already been terminated by the cluster. The actual state of AWS resources lags behind the desired/expected state of resources from Kubernetes' point of view, and it is likely that some users will experience HTTP 5xx errors during a deployment.

![alt text] (images/eks-elb-presentation.003.png)

Under steady state, the architecture looks something like this - Note that the service virtual IP plays no bearing in how traffic is being routed, its going from the ELB straight to the exposed pod IP thanks to the VPC CNI. 

So how can we minimise downtime? Keeping in mind that the whole point of this exercise is to find a way to ensure that the pod process is still running right up to the point where the ELB deregisters its corresponding target from the target group.

![alt text] (images/eks-elb-presentation.004.png)

The preStop hook is one way to keep the pod process alive until the ELB is able to deregister the target, for example by issuing a sleep command to the pod, or enabling a graceful shutdown. This issue here is that it slows down your entire deployment - each batch of pods need to wait for the preStop hooks of the previous batch to complete. With 100s of pods in each service and small batch sizes, this could mean a deployment may take hours.

![alt text] (images/eks-elb-presentation.005.png)

The NodePort approach with using the ingress controller is the way to go in most cases. This is where targets are the EC2 instance node ports for each service. This approach decouples the lifecycle of **pods** from the lifecycle of **nodes** in a cluster - which means you can safely deploy without worrying about API throttling. ELB RegisterTargets and DeregisterTargets API calls are only made when **nodes** (rather than pods) are added or removed from the cluster. Note that this is the default method if no target type is configured in the ingress manifest.

```yaml
annotations:
  kubernetes.io/ingress.class: alb
  alb.ingress.kubernetes.io/target-type: instance
```

![alt text] (images/eks-elb-presentation.006.png)

Here, the ELB is now pointing to service node ports as most are familiar with. The forwarding of traffic from node ports to pods is handled by kube-proxy's iptable rules on each node. Modifying these rules when pods start or stop are also handled by kube-proxy, which is pretty much instantaneous.

![alt text] (images/eks-elb-presentation.007.png)

We should also consider that like pods, nodes can also be stopped and started, like in the case where we enable cluster autoscaling. This introduces complexity where deregsitration delays will play a part, and could potentially cause down time when nodes are selected for scale-in.

This long running github issue describes a variety of use cases where the cluster autoscaler does not take into account for inflight requests between the ALB and the cluster, which will cause request dropping during a scale-in event. This reason why this occurs is that during termination of the EC2 instance, all workloads on that instance have been removed - including kube-proxy, which is responsible for forwarding traffic to other nodes in the cluster. 

A solution is to properly drain instances (ELB drain, not K8s drain), which is possible with the Node Termination handler which was built for spot instances, and the Amazon K8s node drainer. However, this makes your plugin make a DeregisterTarget API call to AWS for EVERY service you have in the cluster - with a large number of services, this will likely also be throttled

(2) https://github.com/kubernetes/autoscaler/issues/1907

![alt text] (images/eks-elb-presentation.008.png)

One of workarounds described in the issue is to change externalTrafficPolicy to Local for your services. This only allows a NodePort to forward traffic to pods running on that node - requests bound for a service which reaches a NodePort without such pods will be dropped. In a steady state, this is fine because the ELB marks nodes without any running pods of that service as unhealthy, and does not forward traffic there. 

However this becomes problematic in the specific scenario where there is indeed service pods running on a particular node. A rolling update or pod eviction causes those pods to be moved onto another node. Traffic is forwarded to that node from the ELB, and cannot be cross-routed to pods running on other nodes. During a rolling deployment, it is possible that some users will experience downtime equivalent to the entire healthcheck period. 

By attempting to come up with a solution, we’ve circled back to our original issue of downtime during a deployment!

![alt text] (images/eks-elb-presentation.009.png)

In short, there is no one perfect solution. We can mitigate delays in the ingress controller by upgrading to the AWS Load Balancer controller, which introduces a webhook mechanism to listen for changes to ingress resources, rather than polling for changes on each ingress.

Ultimately, the question are:
* How many pods in each service? how many are in the cluster in total?
* How often do deployments occur? how often does cluster autoscaling occur?
* Whether sticky sessions will connect users to terminating pods?
* What are your AWS API limits?
* How does your chosen plugin interact with the AWS APIs?
* How much downtime is acceptable for your users?

Ensuring you have a good understanding on the above are key in finding the right measure to minimise downtime in your cluster.

(3) https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/