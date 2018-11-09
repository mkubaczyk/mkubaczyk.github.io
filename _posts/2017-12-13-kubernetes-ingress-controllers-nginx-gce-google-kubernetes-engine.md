---
layout: post
title: "Kubernetes Ingress controllers – NGINX and GCE on Google Kubernetes Engine"
summary: "There's many of different Ingress controllers, but just two of them really count in the end. In this post I'm describing both: nginx and GCE, with examples based on Kubernetes cluster set up on Google Cloud Platform."
image: /img/blog/2017/12/google-load-balancer-l7-ingress.png
---

Getting to know Kubernetes might be a challenge when stepping out of your comfort zone consisting only of Pods and Deployments, where all Services you expose are just NodePorts or simple Load Balancers created with no knowledge what’s underneath. The same applies to Ingress and its controllers, without which the former can’t even work. In this post I would like to explain some of the least understandable parts of Ingress controllers, especially what’s the difference between the two most popular controllers Google Kubernetes Engine supports at the moment, often incomprehensibly defined using annotation `kubernetes.io/ingress.class`: **nginx** and **gce**.

## Ingress and its controllers
Ingress, being a collection of routing rules for the incoming traffic, might be considered a definition of routing object. A plan, that is about to be executed by something. That something is a controller, a thing that stands behind each Ingress and executes the predefined rules. Generally speaking, Ingress creates routes for incoming traffic so it can reach cluster services easily with predefined conditions. Such a condition can be a specific hostname or a URL path. Ingress controllers, on the other hand, are "applications", let’s just consider them as code that translates the Ingress-shaped definition into working connections. Take a look at an example Ingress manifest:

```
$ cat example-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - backend:
              serviceName: example-backend
              servicePort: 80
            path: /
```

It can be clearly seen that it’s defining an explicit connection between an incoming request with Host set as `example.com` and a Service `example-backend` on port 80. But up to this point Ingress is still not exposed outside the cluster. It can be done with different methods and its exposition depends on the Ingress controller we choose. 

### GCE Ingress controller
In this post it is assumed everything was created within Google Cloud Platform as a Google Kubernetes Engine cluster. The specificity of this kind of cluster is that if Ingress is created without annotation pointing out a specific Ingress controller type, the default approach is to create a GCE Ingress controller. What that gives is a separate Google Load Balancer based on Layer 7 per Ingress. What’s more, there’s no need to create such a controller. It’s done automatically thanks to the use of GKE here.

First, we need to create some application we refer to with Ingress and expose it as NodePort with

```
$ kubectl run example-backend --image=nginx --port=80
$ kubectl expose deploy example-backend --port=80 --type=NodePort
$ kubectl get svc
NAME              TYPE       PORT(S)
example-backend   NodePort   80:30985/TCP
```

It is required by GCE Ingress Controller that all services are exposed as NodePort, not ClusterIP, which is the default configuration if nothing is passed as `--type` flag.

```
$ kubectl apply -f example-ingress.yaml
```

This command creates Ingress. After a minute, there’s a Load Balancer created on Google Cloud Platform, and exposed on a specific IP address on port 80 with "Host and path rules" as defined in the file. The "backend services" section stands for health checks needed by Load Balancer before it can pass requests to the services. The very first backend service here represents the default backend which is presented to the user when there’s no valid route found in Ingress. 

```
$ kubectl get svc -n kube-system
NAME                         TYPE         PORT(S)
default-http-backend   NodePort   80:31946/TCP
```

Each next service would be just a health check for a specific NodePort of service defined in Ingress manifest, like `example-backend` in our case.
To sum it up, the generated Ingress is reflected on Google Cloud Platform as Load Balancer and it consists of the parts described above. The current Load Balancer can be seen on a screenshot below.

![kubernetes ingress google load balancer]({{ "/img/blog/2017/12/google-load-balancer-l7-ingress.png" }})

In this case, the 1st `Backend service` points at `default-http-backend` Service exposed with type NodePort on 31946 created automatically by GKE, and the 2nd one stands for our `example.com` traffic (when there’s a Host Header matched with the rule) to the `example-backend` Service on a NodePort 30985 we exposed a bit earlier.

So when the "Healthy" section for the 2nd backend service is 2/2 in this case (it checks all nodes because NodePort is a type of exposition where service is exposed from every node and the cluster had 2 of them when this post was created), it can be easily checked with

```
$ curl -H "Host: example.com" 35.201.75.133 -I
HTTP/1.1 200 OK
Server: nginx/1.13.7
```

that Load Balancer is exposing our `example-backend` application to the world.

If you have more than one Ingress controller in your cluster, then just annotate a specific Ingress with class definition as in the snippet below.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  annotations: 
    kubernetes.io/ingress.class: “gce”
```
When there’s no such annotation and more than one Ingress controller is present in a cluster, it may lead to some collisions upon which the controller should back up the created Ingresses.

### Nginx Ingress controller
An Nginx-based Ingress controller takes a slightly different approach. The most important advantage that comes with such a choice is that the Nginx Ingress controller is completely agnostic to the cluster setup. This means that no matter how the cluster’s been made, it can be even a bare metal-based configuration, Nginx Ingress controller is fully deployable with `helm` manager or directly with some YAML manifests and will provide comparable functions. 

I really like the way `helm` manager works. Installing Nginx Ingress controller is as easy as just invoking:

```
helm install stable/nginx-ingress --set rbac.create=true
```

Of course you need to have your `helm` fully initialized for your cluster, but that’s not the problem. I’m sending you to https://github.com/kubernetes/charts/tree/master/stable/nginx-ingress for more instructions on how to customize this specific helm release, or get yourself directly to https://github.com/kubernetes/ingress-nginx for documentation of the nginx-based controller.

But why would you even consider this type of controller over the GCE’s one? With the latter the problem that arises right after you start to use it is that with each Ingress created, it automatically invokes GCE Load Balancer creation as well. While this sounds like the best option ever, it may become quite an expensive way to expose applications to the world. This kind of problem doesn’t exist when the Nginx-based controller is used. By default, there’s only one service with `LoadBalancer` type that is responsible for all incoming traffic. No new service is created when Ingress is created with this controller. The Nginx Ingress controller’s pods do the routing based on the Ingresses you created for your namespaces. It’s worth noting that only this service also creates Load Balancer in your Google project, but it has a bit different way of operating. Instead of being aware of paths and having health checks for every service you’re exposing it with, it’s balancing the traffic between nodes of your cluster. We can call this Load Balancer a Layer 4 object of OSI Model. For the record: the one created with GCE Ingress Controller was Layer 7. 

![kubernetes nginx ingress exposed with load balancer ]({{ "/img/blog/2017/12/nginx-ingress-load-balancer.png" }})

This is the Lod Balancer we're exposing our Nginx Ingress controller with.

```
$ kubectl get svc -n kube-system
NAME                                    TYPE           PORT(S)              
monty-python-nginx-ingress-controller   LoadBalancer   80:30743/TCP,443:32531/TCP
monty-python-ingress-default-backend    ClusterIP      80/TCP
```

The helm release I used in this post created default-backend as well. That’s the same kind of object GCE Ingress controller used.

From this point on, the external IP address of `monty-python-nginx-ingress-controller` can be used to access objects defined with Ingresses. If there’s an Ingress with a proper annotation like:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  annotations: 
    kubernetes.io/ingress.class: “nginx”
spec:
  rules:
    - host: example.com
      http:
        paths:
          - backend:
              serviceName: example-backend
              servicePort: 80
            path: /
```

the Nginx Ingress controller will take care of defined routing. 

I use this controller to expose different projects on our `dev` cluster, because it’s the only one IP address I need to add as "A" record for our domain. If I defined two different hosts, let’s say `example.com` and `foobar.com`, both would have the same "A" record and Nginx Ingress controller will direct users based on the Host header. It’s convenient.

## Summary
Ingress is a great thing in Kubernetes. It opens new paths (ha, you got this joke, right?) for ops to expose applications with ease. While it’s a great idea, Ingress controllers might seem scary. I find this the most complicated thing to understand when one tries Kubernetes for the first time. I believe, however,  this post will shed some light on the case. Kubernetes is a great tool, but it should be used with proper knowledge. I hope this knowledge has been passed to you via this article. 
