# DigitalOceanK8sChallenge
Digital Ocean k8s challenge

## Challenge: Deploy a virtual cluster solution
Install vcluster to test upgrades (eg. 1.20 to 1.21 DOKS version) of your cluster. With a virtual cluster, you can create a new Kubernetes cluster inside your existing DOKS cluster, test the application in this new vcluster, and then upgrade your original cluster if everything works well with the new version.   https://loft-sh.medium.com/high-velocity-engineering-with-virtual-kubernetes-clusters-7df929ac6d0a

### What is vcluster? 
Virtual clusters, as the name indicates, are fully working k8s clusters that run on top of other k8s clusters. k8s clusters in general have their own node pools but in case of virtual clusters, they utilise the node pool of the underyling cluster to schedule their workloads while having their own control plane. In order to reduce overhead, virtual clusters builds on k3s[https://k3s.io/]. 

### Architecture
Detailed explanation of each of this components can be found at https://www.vcluster.com/docs/what-are-virtual-clusters
![image](https://user-images.githubusercontent.com/9016845/144619100-c8813853-c82a-4487-8253-7548e43886ff.png)[img source: https://www.vcluster.com/docs/what-are-virtual-clusters]

### Benefits and limitations 

#### Some Advantages:

##### Isolation
While k8s namespaces are considered to be a logical separation between applications, they cannot contain cluster scoped resources like nodes, cluster roles, pv's and storage classes. This is not the case in vclusters, they provide a great amount of isolation since the virtual cluster itself creates its own resource objects which the host cluster isn't aware of. This makes it very resilient to failure since breaking of something in one virtual cluster doesn't affect other virtual clusters that're configured. 

##### Multi Tenancy
vclusters can be configured very easily and is independent of the physical cluster. This is very useful while creating multi-tenant applications where in the customers can spin up sandbox environments, new environments very quickly or even to just spin up a test environment for the admin to do some sanity checks.

#### Limitation:
While they provide a good amount of isolation, isolation between physical/standalone clusters is much better than the isolation in case of virtual clusters. In cases when the underlying host clusters is down, all the vclusters would be affected.


### Demo
Goal: We'll create a DOKS(DigitalOcean Kubernetes) with version 1.20 and we'll create a new vcluster with upgraded k3s version(v1.22.4-k3s1) and deploy the existing applications(nginx in this case) to ensure that they're working fine on the new version as well.

#### Prerequisites
- Create a k8s cluster on Digital ocean platform, more details on https://docs.digitalocean.com/products/kubernetes/quickstart/. 
- Set the context to point to the newly created cluster config, follow https://docs.digitalocean.com/products/kubernetes/how-to/connect-to-cluster/.
- Deploy nginx by running the below command. This should also create a load balancer with an external ip which can be accessed via the browser itself. If nginx is up and running, you should see the Welcome to nginx! page.
  ```
  kubectl apply -f https://github.com/Kurento/Kubernetes/blob/master/nginx-deployment-service.yaml
  
  ```
- Install vcluster cli, more details on https://www.vcluster.com/docs/getting-started/setup.

Once the above prerequisites are done, follow the below steps to create a new vcluster with upgraded version

- Check the current version on the host cluster. In this case, it is v1.20.11.
```
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.9", GitCommit:"94f372e501c973a7fa9eb40ec9ebd2fe7ca69848", GitTreeState:"clean", BuildDate:"2020-09-16T13:56:40Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.11", GitCommit:"27522a29febbcc4badac257763044d0d90c11abd", GitTreeState:"clean", BuildDate:"2021-09-15T19:16:25Z", GoVersion:"go1.15.15", Compiler:"gc", Platform:"linux/amd64"}
```
- Create a vcluster with a upgraded version. In this case, we're creating a vcluster named test-vcluster with the k3s image as v1.22.4 in the namespace do.
```
vcluster create "test-vlcuster" --k3s-image v1.22.4+k3s1 -n "do"
```
- Once the above command is run on the host cluster, you should see something like this 
```
[info]   execute command: helm upgrade test-vlcuster vcluster --repo https://charts.loft.sh --version 0.4.5 --kubeconfig /var/folders/2t/y5zmg4qx46d1n5g5ftwhpv7h001_qk/T/157700830 --namespace do --install --repository-config='' --values /var/folders/2t/y5zmg4qx46d1n5g5ftwhpv7h001_qk/T/011263653
[done] √ Successfully created virtual cluster test-vlcuster in namespace do. Use 'vcluster connect test-vlcuster --namespace do' to access the virtual cluster
```
- Verify that the test-vcluster is created
```
kubectl get pods -n do

NAME                                                    READY   STATUS    RESTARTS   AGE
coredns-85cb69466-7dv4l-x-kube-system-x-test-vlcuster   1/1     Running   0          129m
nginx-7848d4b86f-dbstq-x-default-x-test-vlcuster        1/1     Running   0          74m
nginx-7848d4b86f-dstqc-x-default-x-test-vlcuster        1/1     Running   0          74m
nginx-7848d4b86f-x4wf7-x-default-x-test-vlcuster        1/1     Running   0          74m
test-vlcuster-0                                         2/2     Running   5          136m
```
- When a vcluster is created, it also creates a new config in the same folder in which the command is run. In order to access the vcluster, all you've to do is point KUBECONFIG to the newly created config. This can be done by running the below command:
```
export KUBECONFIG=./kubeconfig.yaml
```
- Once the config is set to point to the newly created config. All of the kubectl commands should be pointing to the vcluster. This can be verified by running the kubectl version command again. We can clearly see that version is 1.22.4+k3s1 which is the image that we used while creating the vcluster. 
```
kubectl version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.9", GitCommit:"94f372e501c973a7fa9eb40ec9ebd2fe7ca69848", GitTreeState:"clean", BuildDate:"2020-09-16T13:56:40Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.4+k3s1", GitCommit:"bec170bc813e342cdba9710a1a71f8644c948770", GitTreeState:"clean", BuildDate:"2021-11-29T17:11:06Z", GoVersion:"go1.16.10", Compiler:"gc", Platform:"linux/amd64"}
```
- Now, you can deploy nginx again by the steps mentioned above and then verify that things are working fine in vcluster and voila!
