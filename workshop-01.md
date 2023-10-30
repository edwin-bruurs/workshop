# Workshop Kubernetes

## Create your first pod

Lets first check if our Kubernetes cluster is up and running using the `kubectl` command

```bash
$ kubectl get pods
No resources found in default namespace.
```

When you want to create a pod you can use the `kubectl run` command. During this tutorial we will create a pod called podinfo, which is just a simple web application.

```bash
$ kubectl run podinfo --image stefanprodan/podinfo
pod/podinfo created
```

Let's verify the pod is running

```bash
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
podinfo   1/1     Running   0          7s
```

For more information you can use the `-o wide` flag

```bash
$ kubectl get pods -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
podinfo   1/1     Running   0          20s   10.42.0.58   lima-rancher-desktop   <none>           <none>
```

You can also inspect a pod by running the `kubectl describe` command, which outputs a lot more information. 

```bash
$ kubectl describe pod podinfo
Name:             podinfo
Namespace:        default
Priority:         0
Service Account:  default
Node:             lima-rancher-desktop/192.168.5.15
Start Time:       Mon, 30 Oct 2023 14:43:07 +0100
Labels:           run=podinfo
Annotations:      <none>
Status:           Running
IP:               10.42.0.58
IPs:
  IP:  10.42.0.58
Containers:
  podinfo:
    Container ID:   containerd://f4646909ad951a7e982bc25729bc53dddd17bb8f34c5b3a1a16ea32757899052
    Image:          stefanprodan/podinfo
    Image ID:       docker.io/stefanprodan/podinfo@sha256:f0afdfe24a4feb6f09073bfe0f37347405a81b9512f8d6958aff81b6aaf54cd6
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 30 Oct 2023 14:43:13 +0100
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-t5w4g (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-t5w4g:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  42s   default-scheduler  Successfully assigned default/podinfo to lima-rancher-desktop
  Normal  Pulling    42s   kubelet            Pulling image "stefanprodan/podinfo"
  Normal  Pulled     37s   kubelet            Successfully pulled image "stefanprodan/podinfo" in 4.802542919s (4.802559585s including waiting)
  Normal  Created    37s   kubelet            Created container podinfo
  Normal  Started    37s   kubelet            Started container podinfo
```

Pods are by default not exposed to the host network, so to verify the pod is actually running correct we can use the `kubectl port-forward` command to create a port forward from the client to the pod

```bash
$ kubectl port-forward pod/podinfo 9898:9898
Forwarding from 127.0.0.1:9898 -> 9898
Forwarding from [::1]:9898 -> 9898
```

Next open your browser and go to http://localhost:9898 and see if the pod is running.

Finally, we can delete our created pod using the `kubectl delete` command

```bash
$ kubectl delete pod podinfo
pod "podinfo" deleted
```

Kubernetes also has an option to define the desired state of a cluster using YAML files also called the declaritive method. 

Next create a file called `pod.yaml` with the following content, which should result in the same pod as the one we created using the `kubectl run` command.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: podinfo
  namespace: default
spec:
  containers:
    - image: stefanprodan/podinfo
      name: podinfo
```

If you want to read more about the different fields you can use the `kubectl explain pod` command of information on a specific field `kubectl explain pod.<path>`. For example `kubectl explain pod.spec.containers.args`

Now we are going to declare what we want by applying the `pod.yaml` to our cluster

```bash
$ kubectl apply -f pod.yaml
pod/podinfo created
```

Again you can verify if the pod is running corrrect by executing the `kubectl get pods` and `kubectl port-forward` commands.

Afterwards delete your created pod using `kubectl delete -f pod.yaml`.

## Scale our pod

Typically you don't want to have a single pod running your application. What if that single pod crashes, runs out of resources, is updated, ..? In many cases you want to have more replica's of your application running in your cluster. Kubernetes has different kind of resources implemented, each with a specific use case. For this workshop we will take a look at the `Deployment` resource.

Create a new file `deployment.yaml` with the following content

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      containers:
        - image: stefanprodan/podinfo
          name: podinfo
```

Apply this file to your cluster

```bash
$ kubectl apply -f deployment.yaml
deployment.apps/podinfo created
```

Verify the deployment is running by using the `kubectl get deployments` command

```bash
$ kubectl get deployments
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
podinfo   3/3     3            3           50s
```

You can see that there are now actually three pods running

```bash
$ kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
podinfo-85cff4b554-r58fl   1/1     Running   0          65s
podinfo-85cff4b554-tmc8g   1/1     Running   0          65s
podinfo-85cff4b554-lpsz8   1/1     Running   0          65s
```

Try to delete a pod, and see what happens

```bash
$ kubectl delete pod podinfo-85cff4b554-r58fl
pod "podinfo-85cff4b554-r58fl" deleted

$ kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
podinfo-85cff4b554-tmc8g   1/1     Running   0          2m4s
podinfo-85cff4b554-lpsz8   1/1     Running   0          2m4s
podinfo-85cff4b554-b4bn7   1/1     Running   0          20s
```

In case of a `Deployment` Kubernetes will try to always have the exact number of replicas up and running. If one is deleted, crashed, ... it will recreate a new pod.

## Create a service

Now we have three pods running, how do we route traffic to these pods? Kubernetes uses service-discovery for this implemented via the `Service` resources.

First create a client pod which we can use to execute `curl` commands to our pods.

```bash
$ kubectl run client --image=wbitt/network-multitool
pod/client created
```

Next we can execute a `curl` to one of the pods. You can find the pod IP using the `kubectl get pods -o wide` command.

```bash
$ kubectl exec client -- curl -s 10.42.0.62:9898
{
  "hostname": "podinfo-85cff4b554-tmc8g",
  "version": "6.5.3",
  "revision": "d9bc6301e9b614cbe24e190396bc574d51a2427d",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.5.3",
  "goos": "linux",
  "goarch": "arm64",
  "runtime": "go1.21.3",
  "num_goroutine": "6",
  "num_cpu": "2"
}
```

To see what the problem is of this approach, use the following command to redeploy all the pods one-by-one.

```bash
$ kubectl rollout restart deployment podinfo
deployment.apps/podinfo restarted
 ```

 After about 1 min. all pods are restarted, try to execute the same `curl` command as before and see what happens.

 To solve the problem of connecting direct to a pod we can use the build in service discovery from kubernetes.
 Create a new file `service.yaml` with the following content and apply this to your cluster.

 ```yaml
apiVersion: v1
kind: Service
metadata:
  name: podinfo
  namespace: default
spec:
  ports:
    - port: 9898
      protocol: TCP
      targetPort: 9898
  selector:
    app: podinfo
 ```

```bash
$ kubectl apply -f service.yaml
service/podinfo created
```

Execute the following command from the client pod

```bash
$ kubectl exec client -- curl -s podinfo.default.svc.cluster.localkubectl exec client -- curl -s podinfo.default.svc.cluster.local:9898
{
  "hostname": "podinfo-845588c779-dgf2f",
  "version": "6.5.3",
  "revision": "d9bc6301e9b614cbe24e190396bc574d51a2427d",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.5.3",
  "goos": "linux",
  "goarch": "arm64",
  "runtime": "go1.21.3",
  "num_goroutine": "6",
  "num_cpu": "2"
}
```

Note; you can replace the fqdn also by using just `podinfo:9898` or the IP of the service.

Execute the curl command multiple times and see what happens with the hostname.

Verify that once you do a `kubectl rollout restart deployment podinfo` everything is still working.

Afterwards you can delete the `deployment` using `kubectl delete -f deployment.yaml` and `kubectl delete -f service.yaml`