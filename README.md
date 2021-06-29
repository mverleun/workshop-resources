# Kubernetes resource management

Resource management is important in a cluster. Proper resource settings help the Kubernetes scheduler to identify a suitable node that meets the resource requirements.
In turn the scheduler will rewards us with a stable deployment.

## Prequisites

In order to experience the behaviour of resource limits the following setup is expected:

- minikube running local with 4 Cores and 8GiB of memory. (`minikube start --memory=8G --cpus=4`)
- minikube metrics-server enabled (`minikube addons enable metrics-server`)
- network connectivity to download images

> Create the minikube cluster if you haven't done it yet. Once it is started open a terminal and enter the command `kubectl get events --watch` to receive constant updates on what is happening inside the cluster.
> Minikube is a controller and worker at the same time, unlike production systems. You'll learn soon why they have to be seperated. If your system becomes totally unusable enter the following commands to free up resources, you may have to repeat the `sudo killall stress` command a few times.

``` bash
❯ minikube ssh
                         _             _
            _         _ ( )           ( )
  ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __
/' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
| ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
(_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)

$ sudo killall stress
```

Eventually cleanup the deployment (don' try this now)

``` bash
❯ kubectl delete -f 03-resources-garantueed.yaml
deployment.apps "memory-waster" deleted
```

## QoS Levels

Kubernetes supports three levels of QoS:

- Best Effort, like the name suggests there are no garantuees
- Burstable, more stable than 'Best Effort' but not fully garantueed
- Garantueed, resources are allocated and reserved.

## Resource shortage (or protection?)

When a Kubernetes cluster experiences a shortage in resources it has to act. In case of CPU shortage things might slow down, in case of memory shortage an OOM (Out Of Memory) condition occurs which causes a Pod to die.

OOMs can be caused because a node is running out of memory or because a Pod is trying to consume more memory that it is allowed to use.

The QoS of a deployment plays an important role in here.

Pods with a QoS 'Garantueed' will not be terminated in case of memory shortage on the node itself. Hence the reserved memory is garantueed to be available for these pods. (The same applies to CPU which causes a slowdown).

Burstable pods do reserve a minimum amount of resources but are allowed to request and use more resources if they are available. The downside is that they might be terminated when a shortage occurs and they are using more than they intially requested.

## Imposing limits

Limits on resources can be imposed in three different levels:

- The amount of resources available to a node
- The amount of resources allowed for all deployments in a namespace
- Per container which is accumulated to a demand per pod

Whichever limit is reached first determines the boundary.

## Get to know your system

Before we proceed it is nice to see certain aspects of the current system. Not all information is shown below and the information on your system may differ.

Make sure Minikube is running and notice the current load (commands are entered after the ❯ sign, the rest is output):

``` bash
❯ kubectl get node
... omitted
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Thu, 24 Jun 2021 20:25:54 +0200   Tue, 15 Jun 2021 15:22:03 +0200   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Thu, 24 Jun 2021 20:25:54 +0200   Tue, 15 Jun 2021 15:22:03 +0200   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Thu, 24 Jun 2021 20:25:54 +0200   Tue, 15 Jun 2021 15:22:03 +0200   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Thu, 24 Jun 2021 20:25:54 +0200   Tue, 22 Jun 2021 08:35:43 +0200   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  192.168.99.109
  Hostname:    minikube
Capacity:
  cpu:                4
  ephemeral-storage:  17784752Ki
  hugepages-2Mi:      0
  memory:             8161812Ki
  pods:               110
Allocatable:
  cpu:                4
  ephemeral-storage:  17784752Ki
  hugepages-2Mi:      0
  memory:             8161812Ki
  pods:               110
... omitted
Non-terminated Pods:          (16 in total)
  Namespace                   Name                                     CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                     ------------  ----------  ---------------  -------------  ---
  kube-system                 coredns-74ff55c5b-hdk4m                  100m (2%)     0 (0%)      70Mi (0%)        170Mi (2%)     9d
  kube-system                 etcd-minikube                            100m (2%)     0 (0%)      100Mi (1%)       0 (0%)         9d
  kube-system                 kube-apiserver-minikube                  250m (6%)     0 (0%)      0 (0%)           0 (0%)         3d12h
  kube-system                 kube-controller-manager-minikube         200m (5%)     0 (0%)      0 (0%)           0 (0%)         3d12h
  kube-system                 kube-proxy-br9p6                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d12h
  kube-system                 kube-scheduler-minikube                  100m (2%)     0 (0%)      0 (0%)           0 (0%)         3d12h
  kube-system                 storage-provisioner                      0 (0%)        0 (0%)      0 (0%)           0 (0%)         9d

Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                900m (22%)    4400m (110%)
  memory             1036Mi (12%)  2468Mi (30%)
  ephemeral-storage  100Mi (0%)    0 (0%)
... omitted
```

## QoS Best effort

First lets look at a deployment that runs in a Best Effort setting. This means that no resources settings are present in the deployment manifest file.

Deploy the manifest file `01-resources-best-effort.yaml` to create a deployment that consumes quite some memory (500 M). It's poor in responding. Next display information about the pod:

``` bash
❯ kubectl apply -f 01-resources-best-effort.yaml
deployment.apps/memory-waster created

❯ kubectl describe pod -l app=memory-waster
Name:         memory-waster-6b8478554c-gd29b
Namespace:    default
... omitted
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
... omitted
QoS Class:       BestEffort
... omitted
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  91s   default-scheduler  Successfully assigned default/memory-waster-6b8478554c-gd29b to minikube
  Normal  Pulling    90s   kubelet            Pulling image "polinux/stress"
  Normal  Pulled     88s   kubelet            Successfully pulled image "polinux/stress" in 1.751600699s
  Normal  Created    88s   kubelet            Created container memory-demo-ctr
  Normal  Started    87s   kubelet            Started container memory-demo-ctr
```

> Question: What is the QoS Class of this pod?

Check the memory consumption of the pod:

``` bash
❯ kubectl top pod -l app=memory-waster
W0624 20:50:24.640026   52234 top_pod.go:140] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME                             CPU(cores)   MEMORY(bytes)
memory-waster-6b8478554c-gd29b   0m           502Mi

❯ kubectl top node
W0624 20:52:57.232172   52292 top_node.go:119] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube   1365m        34%    3106Mi          38%
```

Time to scale the application and check resource usage again:

``` bash
❯ kubectl scale deployment memory-waster --replicas 10
deployment.apps/memory-waster scaled

❯ kubectl top pod -l app=memory-waster
W0624 20:54:12.621871   52341 top_pod.go:140] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME                             CPU(cores)   MEMORY(bytes)
memory-waster-6b8478554c-4mhvx   37m          501Mi
memory-waster-6b8478554c-97zcx   56m          502Mi
memory-waster-6b8478554c-bz9ss   0m           1Mi
memory-waster-6b8478554c-cccvf   70m          501Mi
memory-waster-6b8478554c-gd29b   0m           502Mi
memory-waster-6b8478554c-glskr   69m          502Mi
memory-waster-6b8478554c-mzdgd   0m           0Mi
memory-waster-6b8478554c-tzdw2   79m          501Mi
memory-waster-6b8478554c-wkh6d   0m           0Mi
memory-waster-6b8478554c-wqqcl   56m          501Mi

❯ kubectl top node
W0624 20:54:28.883166   52346 top_node.go:119] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube   2727m        68%    7210Mi          90%
```

Did you keep an eye on the terminal with the events?

> Question: Why is the image being pulled multiple times?

Scale again and see what happens:

``` bash
❯ kubectl scale deployment memory-waster --replicas 20
deployment.apps/memory-waster scaled
```

Pretty soon the system is in a lot of pain.... The other terminal shows many errors (In the case of Minikube we have a single node that is both controller and worker).

``` bash
❯ kubectl get events --watch
LAST SEEN   TYPE      REASON              OBJECT                                MESSAGE
... omitted
1s          Normal    ScalingReplicaSet   deployment/memory-waster              Scaled up replica set memory-waster-6b8478554c to 10
0s          Normal    SuccessfulCreate    replicaset/memory-waster-6b8478554c   Created pod: memory-waster-6b8478554c-tzdw2
0s          Normal    SuccessfulCreate    replicaset/memory-waster-6b8478554c   Created pod: memory-waster-6b8478554c-4mhvx
0s          Normal    SuccessfulCreate    replicaset/memory-waster-6b8478554c   Created pod: memory-waster-6b8478554c-mzdgd
0s          Normal    Scheduled           pod/memory-waster-6b8478554c-tzdw2    Successfully assigned default/memory-waster-6b8478554c-tzdw2 to minikube
0s          Normal    Scheduled           pod/memory-waster-6b8478554c-4mhvx    Successfully assigned default/memory-waster-6b8478554c-4mhvx to minikube
... omitted
0s          Normal    ScalingReplicaSet   deployment/memory-waster              Scaled up replica set memory-waster-6b8478554c to 20
0s          Normal    SuccessfulCreate    replicaset/memory-waster-6b8478554c   (combined from similar events): Created pod: memory-waster-6b8478554c-c6rdm
0s          Normal    SuccessfulCreate    replicaset/memory-waster-6b8478554c   (combined from similar events): Created pod: memory-waster-6b8478554c-9g89s
0s          Normal    SuccessfulCreate    replicaset/memory-waster-6b8478554c   (combined from similar events): Created pod: memory-waster-6b8478554c-k4m8k
0s          Normal    SuccessfulCreate    replicaset/memory-waster-6b8478554c   (combined from similar events): Created pod: memory-waster-6b8478554c-p5q5v
0s          Normal    SuccessfulCreate    replicaset/memory-waster-6b8478554c   (combined from similar events): Created pod: memory-waster-6b8478554c-ztz6n
0s          Normal    SuccessfulCreate    replicaset/memory-waster-6b8478554c   (combined from similar events): Created pod: memory-waster-6b8478554c-9jzc6
0s          Normal    SuccessfulCreate    replicaset/memory-waster-6b8478554c   (combined from similar events): Created pod: memory-waster-6b8478554c-tgqfp
0s          Normal    Scheduled           pod/memory-waster-6b8478554c-9g89s    Successfully assigned default/memory-waster-6b8478554c-9g89s to minikube

0s          Normal    SuccessfulCreate    replicaset/memory-waster-6b8478554c   (combined from similar events): Created pod: memory-waster-6b8478554c-kxjdh
0s          Normal    NodeNotReady        node/minikube                         Node minikube status is now: NodeNotReady
0s          Warning   NodeNotReady        pod/memory-waster-6b8478554c-wkh6d    Node is not ready
1s          Warning   NodeNotReady        pod/memory-waster-6b8478554c-4mhvx    Node is not ready
1s          Warning   NodeNotReady        pod/memory-waster-6b8478554c-97zcx    Node is not ready
0s          Warning   NodeNotReady        pod/memory-waster-6b8478554c-gd29b    Node is not ready
46s         Normal    Pulling             pod/memory-waster-6b8478554c-z9h5g    Pulling image "polinux/stress"
46s         Normal    Pulling             pod/memory-waster-6b8478554c-9g89s    Pulling image "polinux/stress"
45s         Normal    Pulling             pod/memory-waster-6b8478554c-ztz6n    Pulling image "polinux/stress"
44s         Normal    Pulling             pod/memory-waster-6b8478554c-k4m8k    Pulling image "polinux/stress"
44s         Normal    Pulling             pod/memory-waster-6b8478554c-jfjzq    Pulling image "polinux/stress"
45s         Normal    Pulling             pod/memory-waster-6b8478554c-tgqfp    Pulling image "polinux/stress"
45s         Normal    Pulled              pod/memory-waster-6b8478554c-p5q5v    Successfully pulled image "polinux/stress" in 4.453962809s
107s        Normal    Created             pod/memory-waster-6b8478554c-p5q5v    Created container memory-demo-ctr
107s        Normal    Pulled              pod/memory-waster-6b8478554c-9jzc6    Successfully pulled image "polinux/stress" in 5.816763092s
108s        Normal    Created             pod/memory-waster-6b8478554c-9jzc6    Created container memory-demo-ctr
108s        Normal    Started             pod/memory-waster-6b8478554c-p5q5v    Started container memory-demo-ctr
... omitted
78s         Warning   SystemOOM           node/minikube                         System OOM encountered, victim process: stress, pid: 465488
78s         Warning   SystemOOM           node/minikube                         System OOM encountered, victim process: stress, pid: 465125
... omitted
```

Quickly scale down, it might take some time and maybe several attempts:

``` bash
❯ kubectl scale deployment memory-waster --replicas 1
deployment.apps/memory-waster scaled

❯ kubectl get pods
NAME                             READY   STATUS        RESTARTS   AGE
memory-waster-6b8478554c-4mhvx   0/1     Terminating   0          14m
... omitted
memory-waster-6b8478554c-kxjdh   0/1     Terminating   0          12m
memory-waster-6b8478554c-mzdgd   0/1     Terminating   0          14m
... omitted
```

On a production cluster the controller would be more responsive but the pain on the workers would be the same.

> Why is the cluster suffering from these deployments?
> Why is the cluster accepting this amount of pods to be scaled?

Clean up the deployment:

``` bash
❯ kubectl delete deployments.apps memory-waster
deployment.apps "memory-waster" deleted
```

## QoS Class Burstable

Let's repeat some of the steps above with another manifest file. This file is almost the same but has resource settings.
Both limits and requests are defined for memory and cpu. Let's see if you can spot the differences in what is happening. Keep an eye on the event log as well.

``` bash
❯ kubectl apply -f 02-resources-burstable.yaml
deployment.apps/memory-waster configured

❯ kubectl describe pod -l app=memory-waster
Name:         memory-waster-6f787b8b7b-cmwlh
Namespace:    default
... omitted
    Limits:
      cpu:     10m
      memory:  1Gi
    Requests:
      cpu:        10m
      memory:     256Mi
... omitted
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
... omitted
QoS Class:       Burstable
... omitted
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  107s  default-scheduler  Successfully assigned default/memory-waster-6f787b8b7b-cmwlh to minikube
  Normal  Pulling    67s   kubelet            Pulling image "polinux/stress"
  Normal  Pulled     66s   kubelet            Successfully pulled image "polinux/stress" in 1.315099762s
  Normal  Created    65s   kubelet            Created container memory-demo-ctr
  Normal  Started    59s   kubelet            Started container memory-demo-ctr
```

Scale again and see what is happening now...

``` bash
❯ kubectl scale deployment memory-waster --replicas 10

❯ kubectl top pod -l app=memory-waster
W0624 21:29:55.735186   53105 top_pod.go:140] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME                             CPU(cores)   MEMORY(bytes)
memory-waster-6f787b8b7b-9khqh   11m          344Mi
memory-waster-6f787b8b7b-bg27q   10m          346Mi
memory-waster-6f787b8b7b-bgtlj   10m          309Mi
memory-waster-6f787b8b7b-cmwlh   0m           501Mi
memory-waster-6f787b8b7b-lq8fq   10m          217Mi
memory-waster-6f787b8b7b-r6wqs   11m          397Mi
memory-waster-6f787b8b7b-ssz2z   10m          365Mi
memory-waster-6f787b8b7b-vz27k   10m          349Mi
memory-waster-6f787b8b7b-w7qfc   11m          256Mi
memory-waster-6f787b8b7b-ws8nr   11m          297Mi
❯ kubectl top pod -l app=memory-waster
W0624 21:29:58.879050   53108 top_pod.go:140] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME                             CPU(cores)   MEMORY(bytes)
memory-waster-6f787b8b7b-9khqh   8m           502Mi
memory-waster-6f787b8b7b-bg27q   11m          477Mi
memory-waster-6f787b8b7b-bgtlj   10m          477Mi
memory-waster-6f787b8b7b-cmwlh   0m           501Mi
memory-waster-6f787b8b7b-lq8fq   11m          438Mi
memory-waster-6f787b8b7b-r6wqs   11m          397Mi
memory-waster-6f787b8b7b-ssz2z   10m          482Mi
memory-waster-6f787b8b7b-vz27k   8m           501Mi
memory-waster-6f787b8b7b-w7qfc   5m           501Mi
memory-waster-6f787b8b7b-ws8nr   11m          491Mi
❯ kubectl top pod -l app=memory-waster
W0624 21:30:01.566915   53111 top_pod.go:140] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME                             CPU(cores)   MEMORY(bytes)
memory-waster-6f787b8b7b-9khqh   8m           502Mi
memory-waster-6f787b8b7b-bg27q   11m          477Mi
memory-waster-6f787b8b7b-bgtlj   10m          477Mi
memory-waster-6f787b8b7b-cmwlh   0m           501Mi
memory-waster-6f787b8b7b-lq8fq   11m          438Mi
memory-waster-6f787b8b7b-r6wqs   11m          397Mi
memory-waster-6f787b8b7b-ssz2z   10m          482Mi
memory-waster-6f787b8b7b-vz27k   8m           501Mi
memory-waster-6f787b8b7b-w7qfc   5m           501Mi
memory-waster-6f787b8b7b-ws8nr   11m          491Mi

❯ kubectl top node
W0624 21:30:56.247442   53140 top_node.go:119] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube   555m         13%    7264Mi          91%
```

Notice that the memory consumption is going slower. This is because of the CPU limit that is set to 10m (1% of a core). Eventually all pods are there and the system has memory left. You have to be patient...

A limit on CPU usage does not terminate the pods. It slows them down.

Let's scale up a bit more to consume more memory than available. We have to be patient due to the CPU limits before we see what is happening.

``` bash
❯ kubectl scale deployment memory-waster --replicas 15
deployment.apps/memory-waster scaled
```

Eventually the system will show OOM Errors again in the event window. Delete this deployment to free the resources:

``` bash
❯ kubectl delete -f 02-resources-burstable.yaml
deployment.apps "memory-waster" deleted
```

> Why do we still have OOM Errors?

## QoS Class Garantueed

Again we repeat the previous steps with the third manifest file. Have a look at the steps in sequence:

``` bash
❯ kubectl apply -f 03-resources-garantueed.yaml
deployment.apps/memory-waster created
❯ kubectl scale deployment memory-waster --replicas 10
❯ kubectl describe pod -l app=memory-waster
Name:         memory-waster-77cd759d9f-h9fv8
Namespace:    default
... omitted
QoS Class:       Guaranteed
... omitted

❯ kubectl scale deployment memory-waster --replicas 10
deployment.apps/memory-waster scaled
❯ kubectl top pod -l app=memory-waster
W0624 21:55:23.836522   53597 top_pod.go:140] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME                             CPU(cores)   MEMORY(bytes)
memory-waster-77cd759d9f-27mxc   0m           0Mi
memory-waster-77cd759d9f-59kgn   0m           0Mi
memory-waster-77cd759d9f-h9fv8   0m           502Mi
memory-waster-77cd759d9f-hf7mb   0m           0Mi
memory-waster-77cd759d9f-k9tj6   0m           1Mi
memory-waster-77cd759d9f-lhmrm   0m           0Mi
memory-waster-77cd759d9f-qzvlb   0m           0Mi
memory-waster-77cd759d9f-rgst7   0m           0Mi
memory-waster-77cd759d9f-v8w2x   0m           1Mi
memory-waster-77cd759d9f-xswj8   0m           0Mi

❯ kubectl top pod -l app=memory-waster
W0624 21:55:31.645426   53600 top_pod.go:140] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME                             CPU(cores)   MEMORY(bytes)
memory-waster-77cd759d9f-27mxc   0m           1Mi
memory-waster-77cd759d9f-59kgn   0m           1Mi
memory-waster-77cd759d9f-h9fv8   0m           502Mi
memory-waster-77cd759d9f-hf7mb   0m           1Mi
memory-waster-77cd759d9f-k9tj6   25m          501Mi
memory-waster-77cd759d9f-lhmrm   0m           1Mi
memory-waster-77cd759d9f-qzvlb   37m          501Mi
memory-waster-77cd759d9f-rgst7   0m           1Mi
memory-waster-77cd759d9f-v8w2x   0m           1Mi
memory-waster-77cd759d9f-xswj8   0m           0Mi

❯ kubectl scale deployment memory-waster --replicas 15
deployment.apps/memory-waster scaled
```

Now something interesting is showing up in the event log:

``` bash
0s          Normal    Scheduled                 pod/memory-waster-77cd759d9f-2rxjx    Successfully assigned default/memory-waster-77cd759d9f-2rxjx to minikube
0s          Warning   FailedScheduling          pod/memory-waster-77cd759d9f-s5qk7    0/1 nodes are available: 1 Insufficient memory.
0s          Normal    Scheduled                 pod/memory-waster-77cd759d9f-vzd6t    Successfully assigned default/memory-waster-77cd759d9f-vzd6t to minikube
1s          Normal    SuccessfulCreate          replicaset/memory-waster-77cd759d9f   (combined from similar events): Created pod: 
0s          Normal    Pulling                   pod/memory-waster-77cd759d9f-vzd6t    Pulling image "polinux/stress"
0s          Normal    Pulled                    pod/memory-waster-77cd759d9f-2rxjx    Successfully pulled image "polinux/stress" in 1.516383751s
0s          Warning   FailedScheduling          pod/memory-waster-77cd759d9f-c4hbd    0/1 nodes are available: 1 Insufficient memory.
0s          Warning   FailedScheduling          pod/memory-waster-77cd759d9f-k6rnn    0/1 nodes are available: 1 Insufficient memory.
0s          Warning   FailedScheduling          pod/memory-waster-77cd759d9f-s5qk7    0/1 nodes are available: 1 Insufficient memory.
... omitted
```

> On a production system you would now see a who list of pods in a waiting state... Why?

Unfortunate Minikube is in a lot of pain... Let's try to scale down. It is becoming more difficult to recover because our deployments have a QoS Class garantueed... Kubernetes has sacrificed other pods (which are system critical... :-( ).

You might want to break into Minikube as described in the beginning to free up resources. Do it from a different terminal and try to scale down from the other terminal. We want the Kubernets API service to become responsive again so it can honor our scaling request.

## Recap

As we have seen there are three QoS Classes:

- Best effort
- Burstable
- Garantueed

A pod is garantueed if _all_ containers have the proper resource settings. That means all containers have resource values where the values of the limits and the request are the same. In fact you do not have to specifiy a request value for a QoS Class deployment Garantueed.

As soon as there is an (implicit) resource setting the scheduler will try to identify a node that meets the requirements. Requested values are reserved for a pod and not available for others.

## To be safe (or sorry?)

Enter the following commands and have a look at what is happening:

``` bash
❯ kubectl top node
W0624 22:14:15.829512   53963 top_node.go:119] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube   359m         8%     2065Mi          25%

❯ kubectl apply -f 04-resources-burstable.yaml
deployment.apps/memory-waster created

❯ kubectl top pod -l app=memory-waster
W0624 22:14:54.403864   53979 top_pod.go:140] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME                             CPU(cores)   MEMORY(bytes)
memory-waster-59c86c9647-6499q   0m           1Mi

❯ kubectl scale deployment memory-waster --replicas 10
deployment.apps/memory-waster scaled

❯ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
memory-waster-59c86c9647-2jvrv   0/1     Pending   0          97s
memory-waster-59c86c9647-6499q   1/1     Running   0          4m3s
memory-waster-59c86c9647-c76qq   0/1     Pending   0          97s
memory-waster-59c86c9647-cfrsz   1/1     Running   0          97s
memory-waster-59c86c9647-hqdhw   0/1     Pending   0          97s
memory-waster-59c86c9647-j7kcl   0/1     Pending   0          97s
memory-waster-59c86c9647-l56vt   0/1     Pending   0          97s
memory-waster-59c86c9647-ptk7j   1/1     Running   0          97s
memory-waster-59c86c9647-vbz7m   0/1     Pending   0          97s
memory-waster-59c86c9647-vfvsr   0/1     Pending   0          97s
```

Have a look at the event logging as well. You'll see that we run the same deployment as before only now with a much larger request value for memory. As a result the system is reserving a lot of memory that is not being used which prevents other pods from deploying.
These pods remain in a pending state for an infinite amount of time.

``` bash
❯ kubectl describe pod -l app=memory-waster
... omitted

❯ kubectl top pod -l app=memory-waster
W0624 22:23:33.528661   54157 top_pod.go:140] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME                             CPU(cores)   MEMORY(bytes)
memory-waster-59c86c9647-6499q   0m           502Mi
memory-waster-59c86c9647-cfrsz   0m           501Mi
memory-waster-59c86c9647-ptk7j   0m           501Mi

❯ kubectl delete -f 04-resources-burstable.yaml
deployment.apps "memory-waster" deleted
```

As you can see for each pod a reservation of 2G has been made. The actual usage of the pod is about .5G... We're wasting valuable resources...

## And the opposite

``` bash
❯ kubectl apply -f 05-resources-garantueed.yaml
deployment.apps/memory-waster created
```

Watch the events. What is happening now? We can not start a single container!

``` bash
❯ kubectl describe pod -l app=memory-waster
Name:         memory-waster-78fdf77ddf-frcg9
Namespace:    default
... omitted
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    1
      Started:      Thu, 24 Jun 2021 22:28:47 +0200
      Finished:     Thu, 24 Jun 2021 22:28:50 +0200
    Ready:          False
    Restart Count:  3
    Limits:
      cpu:     100m
      memory:  256Mi
    Requests:
      cpu:        100m
      memory:     256Mi
    Environment:  <none>
... omitted
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
... omitted
QoS Class:       Guaranteed
... omitted
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  77s                default-scheduler  Successfully assigned default/memory-waster-78fdf77ddf-frcg9 to minikube
  Normal   Pulled     72s                kubelet            Successfully pulled image "polinux/stress" in 1.512656599s
  Normal   Pulled     66s                kubelet            Successfully pulled image "polinux/stress" in 1.31126612s
  Normal   Pulled     49s                kubelet            Successfully pulled image "polinux/stress" in 1.413620931s
  Normal   Pulling    18s (x4 over 73s)  kubelet            Pulling image "polinux/stress"
  Normal   Created    17s (x4 over 72s)  kubelet            Created container memory-demo-ctr
  Normal   Pulled     17s                kubelet            Successfully pulled image "polinux/stress" in 1.3325018s
  Normal   Started    16s (x4 over 71s)  kubelet            Started container memory-demo-ctr
  Warning  BackOff    2s (x5 over 62s)   kubelet            Back-off restarting failed container
  ```

  Ahhh... Another OOM kill... By now we know the requirements of our app pretty well and it for sure wants to consume more memory than is allowed by the resource limits....

  Let's clean up...

 ``` bash
 ❯ kubectl delete -f 05-resources-garantueed.yaml
deployment.apps "memory-waster" deleted
```

## Final, important remark

Our application has been crashing for various reasons. This crash was detected by Kubernetes which is a good thing. If Kubernetes is aware of a crash it could reschedule a pod on another node with more resources etc. etc.

Please do not try to handle errors as a result of resource shortage in a container. This will mask the events from Kubernetes giving the wrong impression of the status of a pod.
If things go sour.... Crash! And crash fast. Let Kubernetes deal with it. (At the same time it saves you from writing additional code...)

## Answers

> Question: What is the QoS Class of this pod?

The QoS Class is reported as: BestEffort which is the default if no resource settings are specified.

> Question: Why is the image being pulled multiple times?

The tag of the image is not specified. It is configured as `polinux/stress` which leads kubernetes to assume that it has to pull `polinux/stress:latest`. Because `latest` can change at any moment Kubernetes has no choice but to pull this image every time.

> Why is the cluster suffering from these deployments?

The combined total of memory required by the scheduled pods exceeds what the cluster has available.

> Why is the cluster accepting this amount of pods to be scaled?

Scaling deployments is an declaritive operation. There is not garantuee that this desired state will be achieved.

> Why do we still have OOM Errors?

Looking at the output you'll notice that there is a resource request that is less then the amount of memory required by the pod. The scheduler is using this value to schedule the deployment on a node. Eventually the demand for memory exceeds what has been reserved resulting in OOM kills due to resource shortage on the node.
