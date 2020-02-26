# How to collect ECK cluster diags on GKE

A quick guide on the ways to collect ECK cluster diagnostics bundle on GKE.

There are basically two scenarios to consider here: Public access to Elasticsearch http service is `enabled`, or `disabled`. Both will be using the same support diagnostics tool [1] but in a different way.

### Public access to Elasticsearch http service is enabled
1. Make sure the Elasticsearch http service is public accessible and note down the external IP (which is `35.244.127.175` in this case) :
```
% kubectl get svc
NAME                              TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
apm-server-sample-apm-http        ClusterIP      10.32.3.32     <none>           8200/TCP         2d22h
elasticsearch-sample-es-default   ClusterIP      None           <none>           <none>           2d22h
elasticsearch-sample-es-http      LoadBalancer   10.32.4.127    35.244.127.175   9200:30357/TCP   2d22h
kibana-sample-kb-http             LoadBalancer   10.32.10.249   35.189.39.129    5601:31290/TCP   2d22h
kubernetes                        ClusterIP      10.32.0.1      <none>           443/TCP          2d23h
```

2. You should have the secret for superuser `elastic` already. If not, run the following command to find out (remove the trailing `%`):
```
% kubectl get secret elasticsearch-sample-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
```

3. Run diagnostics tool from your computer and this will save the diagnostics bundle to your disk: 
```
% ./diagnostics.sh --host <elaticsearch_service_external_ip> --type remote -u elastic -p --ssl --noVerify
```
For example:
```
% ./diagnostics.sh --host 35.244.127.175 --type remote -u elastic -p --ssl --noVerify
```

### Public access to Elasticsearch http service is disabled
1. Check the public access to Elasticsearch http service is not available (`EXTERNAL-IP` should shows `<none>`). Note down the `CLUSTER-IP` of Elasticsearch http service :
```
% kubectl get svc
NAME                              TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
apm-server-sample-apm-http        ClusterIP      10.32.3.32     <none>          8200/TCP         2d4h
elasticsearch-sample-es-default   ClusterIP      None           <none>          <none>           2d4h
elasticsearch-sample-es-http      ClusterIP      10.32.4.127    <none>          9200/TCP         2d4h
kibana-sample-kb-http             LoadBalancer   10.32.10.249   35.189.39.129   5601:31290/TCP   2d4h
kubernetes                        ClusterIP      10.32.0.1      <none>          443/TCP          2d5h
```

2. You should have the secret for superuser `elastic` already. If not, run the following command to find out (remove the trailing `%`):
```
% kubectl get secret elasticsearch-sample-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
```

3. Exec into any of the Elasticsearch pods :
```
kubectl exec -it elasticsearch-sample-es-default-2 -- /bin/bash
```

4. Download support diagnostics tool to the pod and extract:
```
# yum install wget
# wget https://github.com/elastic/support-diagnostics/releases/download/7.1.5/support-diagnostics-7.1.5-dist.zip
# tar -zxvf support-diagnostics-7.1.5-dist.zip
```
5. Set `JAVA_HOME`. You can use the java which runs elasticsearch :
```
# export JAVA_HOME=/usr/share/elasticsearch/jdk
```
6. Run diagnostics tool from the pod and this will save the diagnostics bundle to the pod:
```
# ./diagnostics.sh --host <elaticsearch_service_cluster_ip> --type remote -u elastic -p --ssl --noVerify
```
For example:
```
# ./diagnostics.sh --host 10.32.4.127 --type remote -u elastic -p --ssl --noVerify
```
7. Use `kubectl cp` to copy the bundle to local disk, for example this will copy the budle to local `/tmp` folder:
```
% kubectl cp elasticsearch-sample-es-default-2:/tmp/support-diagnostics-7.1.5/remote-diagnostics-20200226-054502.tar.gz /tmp/remote-diagnostics-20200226-054502.tar.gz
tar: Removing leading `/' from member names
```
You can ignore the warning regarding `tar`, the command will still run:
```
% ls -l *.tar.gz
-rw-r--r--  1 vincent  wheel  146718 27 Feb 09:07 remote-diagnostics-20200226-054502.tar.gz
```
   
   
[1] https://github.com/elastic/support-diagnostics
