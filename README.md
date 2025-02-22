# eks-kappa-lab
Maybe this exercise could be the unofficial manual for deploying Kappa. This example is used for EKS, but would fit with any K8 implementation if you are aware of some of the things to look for while deploying. 

## prerequisites

What you need:
- [kubectl](https://kubernetes.io/docs/tasks/tools/) v1.24.0 or newer. This Allows you to communicate with your cluster via command line.
- [awscli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) 
- Git in order to clone repos to deploy into your cluster
- (mac) homebrew
- Basic understanding of Kubernetes terms e.g. Containers, Pods, namespaces, deployments, daemonsets etc.. 
- Spoof access or an account to log into kentikaz23 in  US prod (contact Ted or Mike K if you do not have one of these)
- You should have some config files that were emailed to you. This will allow you to access your cluster and deploy software on it.

what you DON'T need:
- To be a Kubernetes expert because they don't exist




## Goal of this session
The goal here is to emulate what someone might do if they own a Kubernetes environment but don't have access to the underlying infrastructure, but has rights to deploy on a cluster. This means you will not need to login to the AWS console or connect to any nodes via ssh.


## Configure AWS CLI with provided credentials run the following command and put in us-east-2 as the region
```
aws configure
```

## Installing your k8 credentials
If you already have a config for another cluster, you can use multiple contexts to manage this. Please see [this](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) if you need to do this.

You can automatically install kubectl creds with AWS CLI (if you are using aws cli profiles you can specify --profile=[profilename]
```
aws eks update-kubeconfig --name=[clustername]
```

## Testing your configuration
You can do a basic test of the configuratipn by running the following command:
```
kubectl get pods -A
```
Example:
```% k get pods -A
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   aws-node-4pjv4             1/1     Running   0          29m
kube-system   aws-node-7gwdl             1/1     Running   0          29m
kube-system   aws-node-m962p             1/1     Running   0          29m
kube-system   coredns-5c5677bc78-cbqct   1/1     Running   0          34m
kube-system   coredns-5c5677bc78-vm7fm   1/1     Running   0          34m
kube-system   kube-proxy-cxp6k           1/1     Running   0          29m
kube-system   kube-proxy-jb858           1/1     Running   0          29m
kube-system   kube-proxy-p4x9f           1/1     Running   0          29m
```


This will give you a listing of all the running pods in all Namespaces.

Once we are confident that we can communicate with our assigned clusters, we can move onto deploying Kappa to Kube.

1. Go to your command line and clone the Master Kentik Kappa repository.
```
git clone https://github.com/kentik/kappa-cfg
```
2. Login to the provided Kentik Portal and make note of the plan-id, your kentik email and token.

3. CD into the kappa-cfg directory and use your favorite editor to open up the kustomization file. This is the file that you need to edit in order to point the output to Kentik. There are 2 sections that need to be modified for now.
```
cd kappa-cfg
nano kustomization
```


One section is the "configmap". This helps to define what interfaces we are looking at for traffic.  Plus the plan number (from the Licensing page in the Portal) and finally give the device a unique name (i.e. yourname-kappa_test) 
```
configMapGenerator:
  - name: kappa-config
    literals:
      - capture=ens3|veth.*
      - device=kappa_test
      - region=US
      - plan=1234
      - bytecode=x86_64/kappa_bpf-ubuntu-5.4.o
 ```
 
 The resulting section will look like this:
 ```
configMapGenerator:
  - name: kappa-config
    literals:
      - capture=ens3|veth.*|eth*
      - device=my_device_name
      - region=US
      - plan=45255
      - bytecode=x86_64/kappa_bpf-ubuntu-5.4.o
 ```
 
 
**** Note that you can configure a list of interface names. This is important and depends on the OS used. If you are unsure you can ask your AWS administrator. In mine, I had to add eth* to the list. We will talk about bytecode in a bit.


Second is the kentik credentials used. You can use credentials shown in the kproxy config if you want to be sure no other users will change the token. Alternatively, create your own user account in this portal.  

```
secretGenerator:
  - name: kentik-api-secrets
    literals:
      - email=test@example.com
      - token=asdf1234
```

Once everything has been filled in, it is now time to deploy.

4. From the kappa-cfg directory run the following:
```
kubectl apply -k .
```
*** -k tells kubectl that there is customization to be done. "." just means the local directory.

you'll see output like this if all goes well:
```
% kubectl apply -k .
namespace/kentik created
serviceaccount/kubeinfo created
clusterrole.rbac.authorization.k8s.io/kubeinfo created
clusterrolebinding.rbac.authorization.k8s.io/kubeinfo created
configmap/kappa-bytecode-5mmmmcb2dt created
configmap/kappa-config-4fd6b66m8d created
configmap/kappa-init-5c79hkbb6c created
secret/kentik-api-secrets-8fb929gg68 created
service/kappa-agg created
deployment.apps/kappa-agg created
deployment.apps/kubeinfo created
daemonset.apps/kappa-agent create
```

5. Let's verify things worked. One simple way is to look in the portal to see that the device is now on your list of devices. Or we can verify it is working by looking at pod logs. 
Another way might be to make sure all your pods are running. You can list all pods running in the kentik namespace with the following command:
```
kubectl get pods -n kentik 
```
If healthy, you will see something like this:
```
NAME                         READY   STATUS    RESTARTS   AGE
kappa-agent-5mcvb            1/1     Running   0          118s
kappa-agent-bvqb6            1/1     Running   0          118s
kappa-agent-vbwbf            1/1     Running   0          118s
kappa-agg-55f4bfd9f9-chjvj   1/1     Running   0          118s
kubeinfo-7d676c5444-7jh7s    1/1     Running   0          118s
```

Use kubectl to find the pod with "kappa-agg" in the name. Does anything look off when you list the pods?

Then view the logs for this pod:

```
kubectl logs -name kentik kappa-agg-xxxx-xxxx
```

You will see that it created a device and each of the agents has connected to it.

```
[2023-01-15T19:52:34Z INFO  kappa] initializing kappa 1.1.1
[2023-01-15T19:52:34Z DEBUG kappa::export] creating device aws_kappa_krygeris
[2023-01-15T19:52:34Z DEBUG kappa::export] device Device { id: 144660, name: "aws_kappa_krygeris", kind: "host-nprobe-dns-www", subtype: "kappa", 
...
[2023-01-15T19:52:37Z DEBUG kappa::augment::augment] connection from 10.0.3.140:50420
[2023-01-15T19:52:42Z DEBUG kappa::agg] connection from 10.0.3.223:42098
[2023-01-15T19:52:43Z DEBUG kappa::agg] connection from 10.0.1.29:51348
[2023-01-15T19:52:44Z DEBUG kappa::agg] connection from 10.0.1.65:44682
```

## Let's talk about all the parts.

Kappa has a few moving parts

## namespace/kentik 

This is a unique place that kappa live in inside kubernetes.


## serviceaccount/kubeinfo 

This is the account that kappa uses to interact with kubernetes and gather infomration about kubernetes


## clusterrole.rbac.authorization.k8s.io/kubeinfo 

This the role used by kubeinfo RBAC


## clusterrolebinding.rbac.authorization.k8s.io/kubeinfo

role binding for kubeinfo


## configmap/kappa-bytecode-xxxx

This is the eBPF bytecode that is used to collect stats. Used by kappa agent.


## configmap/kappa-config-xxxx 

This is the configuration that is used by the agents in the daemonset (kappa agents)


## configmap/kappa-init-xxxx

This is mostly to bootstrap eBPF bytecode when starting up

## secret/kentik-api-secrets-xxxx

These are the Kentik credentials that are safely encrypted at rest. 


##service/kappa-agg 

This is the service that makes sure kappa-agg keeps going.

## deployment.apps/kappa-agg

This is the pod that takes all the data from all the nodes and aggregates it. 


### deployment.apps/kubeinfo 


## daemonset.apps/kappa-agent 

This is the actual agent that resides on each node in the cluster. It uses both packets and ebpf to gather data



## About eBPF
eBPF is functionality introduced in the linux kernel that allows you to hook onto any event in the Linux Kernel. It also does a ton of other cool stuff that I won't get into here.

Kappa can generate data based on eBPF both with or with out bytecode. But sometimes you might need to load bytecode to support the correct kernel version. 
eBPF has been develpoed rapidly and some features have changed from kernel to kernel. We counter this by building bytecode for common kernels. 
It is possible to do this yourself if you know how to compile them. Unfortunately we do not provide source to customers.

As of right now, we have not installed bytecode. But we can do this by going to the bytecode instructions.


## Removing Kappa. 
You may have noticed that there isnt much talk about removing it. But sometimes you want to start over. Fortunately this is simple. From your kappa-cfg diretory, simply run the following:
```
kubectl delete -k .
```
