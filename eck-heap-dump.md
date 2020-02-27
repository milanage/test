# How to manually capture Elasticsearch heap dump on GKE



How can we manually capture Elasticsearch heap dump on GKE?

Ideally if we can `kubectl exec` into Kubernetes pods with `elasticsearch` user (the users runs Elasticsearch process), we can simply run `jmap` to take heap dump, similar to what we suggest for docker containers in the KB article [1]. 

However, due to the known `kubectl` limitation [2], it is not possible at the time of this writing -- we'll receive an exception if we run `jmap` as following, which is because the user is `root` rather than `elasticsearch` :

```
# /usr/share/elasticsearch/jdk/bin/jmap -dump:format=b,file=heap.hprof 1
Exception in thread "main" com.sun.tools.attach.AttachNotSupportedException: Unable to open socket file /proc/1/root/tmp/.java_pid1: target process 1 doesn't respond within 10500ms or HotSpot VM not loaded
	at jdk.attach/sun.tools.attach.VirtualMachineImpl.<init>(VirtualMachineImpl.java:100)
	at jdk.attach/sun.tools.attach.AttachProviderImpl.attachVirtualMachine(AttachProviderImpl.java:58)
	at jdk.attach/com.sun.tools.attach.VirtualMachine.attach(VirtualMachine.java:207)
	at jdk.jcmd/sun.tools.jmap.JMap.executeCommandForPid(JMap.java:128)
	at jdk.jcmd/sun.tools.jmap.JMap.dump(JMap.java:218)
	at jdk.jcmd/sun.tools.jmap.JMap.main(JMap.java:114)
```


This will become possible once `kubectl exec` accepts `user` ("-u") flag as `docker exec` does [3]. 


The workaround is to `ssh` to GKE node, then `docker exec` into the actual docker container behind the pod and take the heap dump.

1. Once we have the pod name (the one with Elasticsearch that is running into issues), we can identify the GKE node to connect:
```
% kubectl get pods -o wide
NAME                                            READY   STATUS    RESTARTS   AGE    IP           NODE                                        NOMINATED NODE   READINESS GATES
apm-server-sample-apm-server-6bb799c747-kj7jp   1/1     Running   0          3d3h   10.28.0.13   gke-vk-cluster-default-pool-e0d02f91-9mff   <none>           <none>
elasticsearch-sample-es-default-0               1/1     Running   0          3d3h   10.28.1.12   gke-vk-cluster-default-pool-e0d02f91-vd5d   <none>           <none>
elasticsearch-sample-es-default-1               1/1     Running   0          3d3h   10.28.2.6    gke-vk-cluster-default-pool-e0d02f91-d5dl   <none>           <none>
elasticsearch-sample-es-default-2               1/1     Running   0          3d3h   10.28.0.15   gke-vk-cluster-default-pool-e0d02f91-9mff   <none>           <none>
kibana-sample-kb-68c4b547cf-l5g72               1/1     Running   0          3d3h   10.28.0.12   gke-vk-cluster-default-pool-e0d02f91-9mff   <none>           <none>
```
For example, if we want to inspect `elasticsearch-sample-es-default-2`, the GKE node would be `gke-vk-cluster-default-pool-e0d02f91-9mff` here.

2. `ssh` into the target GKE node using `gcloud compute` :
```
gcloud compute ssh <NODE_NAME> --zone <ZONE>
```
For example,
```
gcloud compute ssh gke-vk-cluster-default-pool-e0d02f91-9mff --zone australia-southeast1-b
```

3. Once logged in the node, identify the docker container in question. Note down the container ID (as `d6e646b2a793` in this case) :
```
$ docker ps
CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS              PORTS               NAMES
d6e646b2a793        911f580307ae                                          "/usr/local/bin/dock…"   3 days ago          Up 3 days                               k8s_elasticsearch_elasticsearch-sample-es-default-2_default_8c0fc714-568f-11ea-b638-42010a980049_0
......
......
```

4. Follow the KB article to capture heap dump in docker container :
```
$ docker exec -u 1000:0 -ti d6e646b2a793 /bin/bash
$ /usr/share/elasticsearch/jdk/bin/jmap -dump:format=b,file=heap.hprof 1
```

5. Use `kubectl cp` to copy the generated heap dump to local disk, for example this will copy the budle to local `/tmp` folder :
```
% kubectl cp elasticsearch-sample-es-default-2:/tmp/heap.hprof /tmp/heap.hprof
tar: Removing leading `/' from member names
```
You can ignore the warning regarding `tar`, the command will still run:
```
% ls -l
total xxxxxx
-rw-r--r--  1 vincent  wheel  306880281 27 Feb 11:13 heap.hprof
```

[1] https://support.elastic.co/customers/s/article/ka04M000000BXHk  
[2] https://github.com/kubernetes/kubernetes/issues/30656  
[3] https://github.com/kubernetes/kubernetes/pull/81883
