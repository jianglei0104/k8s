Hpa基于cpu自动扩缩容

用Deployment来创建一个nginx pod，资源清单如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: "200m"
          requests:
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: hpa-demo
  namespace: dev
  labels:
    app: nginx-pod
spec:
  selector:
    app: nginx-pod
  type: NodePort
  ports:
  - port: 80
    nodePort:
    targetPort: 80
```

直接创建Deployment：

```yaml
[root@master 1.8+]# kubectl create -f hpa-demo-pod.yaml 
deployment.apps/hpa-demo created
service/hpa-demo created
[root@master 1.8+]# kubectl get svc,deploy,pods -n dev
NAME               TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/hpa-demo   NodePort   10.102.85.21   <none>        80:31285/TCP   21s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hpa-demo   1/1     1            1           25s

NAME                            READY   STATUS    RESTARTS   AGE
pod/hpa-demo-788bf4777b-rgl5q   1/1     Running   0          23s
```

创建一个Hpa资源对象，资源清单如下：

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo
  namespace: dev
spec:
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 3
  scaleTargetRef:
    apiVersion:  apps/v1
    kind: Deployment
    name: hpa-demo
```

```yaml
[root@master 1.8+]# kubectl get hpa -n dev
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   0%/3%     1         10        1          26s
```

```yaml
[root@master 1.8+]# kubectl get hpa -n dev
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   22%/3%    1         10        1          16m
[root@master 1.8+]# kubectl get hpa -n dev
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-demo   Deployment/hpa-demo   8%/3%     1         10        8          17m


[root@master ~]# kubectl get deployment -n dev -w
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
hpa-demo   1/1     1            1           19m
hpa-demo   1/4     1            1           20m
hpa-demo   1/4     1            1           20m
hpa-demo   1/4     1            1           20m
hpa-demo   1/4     4            1           20m
hpa-demo   2/4     4            2           20m
hpa-demo   2/8     4            2           20m
hpa-demo   2/8     4            2           20m
hpa-demo   2/8     4            2           20m
hpa-demo   2/8     8            2           20m
hpa-demo   3/8     8            3           20m
hpa-demo   4/8     8            4           20m
hpa-demo   5/8     8            5           21m
hpa-demo   6/8     8            6           21m
hpa-demo   7/8     8            7           21m
hpa-demo   8/8     8            8           21m
[root@master ~]# kubectl get pods -n dev -w
NAME                        READY   STATUS    RESTARTS   AGE
hpa-demo-788bf4777b-rgl5q   1/1     Running   0          20m
hpa-demo-788bf4777b-9ktjr   0/1     Pending   0          0s
hpa-demo-788bf4777b-gcq42   0/1     Pending   0          0s
hpa-demo-788bf4777b-bvsfd   0/1     Pending   0          0s
hpa-demo-788bf4777b-9ktjr   0/1     Pending   0          1s
hpa-demo-788bf4777b-gcq42   0/1     Pending   0          0s
hpa-demo-788bf4777b-bvsfd   0/1     Pending   0          0s
hpa-demo-788bf4777b-bvsfd   0/1     ContainerCreating   0          0s
hpa-demo-788bf4777b-9ktjr   0/1     ContainerCreating   0          1s
hpa-demo-788bf4777b-gcq42   0/1     ContainerCreating   0          0s
hpa-demo-788bf4777b-bvsfd   1/1     Running             0          5s
hpa-demo-788bf4777b-gr8kn   0/1     Pending             0          0s
hpa-demo-788bf4777b-9zhp8   0/1     Pending             0          0s
hpa-demo-788bf4777b-gr8kn   0/1     Pending             0          0s
hpa-demo-788bf4777b-hfs7h   0/1     Pending             0          0s
hpa-demo-788bf4777b-nznjg   0/1     Pending             0          0s
hpa-demo-788bf4777b-9zhp8   0/1     Pending             0          0s
hpa-demo-788bf4777b-hfs7h   0/1     Pending             0          0s
hpa-demo-788bf4777b-nznjg   0/1     Pending             0          0s
hpa-demo-788bf4777b-gr8kn   0/1     ContainerCreating   0          0s
hpa-demo-788bf4777b-hfs7h   0/1     ContainerCreating   0          0s
hpa-demo-788bf4777b-9zhp8   0/1     ContainerCreating   0          0s
hpa-demo-788bf4777b-nznjg   0/1     ContainerCreating   0          0s
hpa-demo-788bf4777b-hfs7h   1/1     Running             0          23s
hpa-demo-788bf4777b-gr8kn   1/1     Running             0          23s
hpa-demo-788bf4777b-nznjg   1/1     Running             0          43s
hpa-demo-788bf4777b-9zhp8   1/1     Running             0          43s
hpa-demo-788bf4777b-9ktjr   1/1     Running             0          60s
hpa-demo-788bf4777b-gcq42   1/1     Running             0          59s

```



Hpa基于内存自动扩缩容

用Deployment来创建一个nginx pod，资源清单如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hpa-mem
  namespace: dev
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        resources:
          requests:
            cpu: 0.01
            memory: 25Mi
          limits:
            cpu: 0.05
            memory: 60Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-mem
  namespace: dev
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
```

创建一个Hpa资源对象，资源清单如下：

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa-mem
  namespace: dev
spec:
    maxReplicas: 10
    minReplicas: 1
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: nginx-hpa-mem
    metrics:
    - type: Resource
      resource:
        name: memory
        targetAverageUtilization: 60
```

```yaml
[root@master 1.8+]# kubectl create -f hpa-demo-memory.yaml 
horizontalpodautoscaler.autoscaling/nginx-hpa-mem created
[root@master 1.8+]# kubectl get hpa -n dev
NAME            REFERENCE                  TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa-mem   Deployment/nginx-hpa-mem   <unknown>/60%   1         10        0          8s
[root@master 1.8+]# kubectl get hpa -n dev
NAME            REFERENCE                  TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa-mem   Deployment/nginx-hpa-mem   5%/60%    1         10        1          17s

```

进入pod结点，生成文件增加内存，进而观察pod变化情况

```yaml
[root@master 1.8+]# kubectl exec -it nginx-hpa-mem-568984d85-wq5n4 -n dev /bin/sh
# dd if=/dev/zero of=/tmp/a


[root@master 1.8+]# kubectl get hpa -n dev
NAME            REFERENCE                  TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa-mem   Deployment/nginx-hpa-mem   63%/60%   1         10        1          2m43s
[root@master 1.8+]# kubectl get hpa -n dev
NAME            REFERENCE                  TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa-mem   Deployment/nginx-hpa-mem   83%/60%   1         10        1          3m3s

[root@master ~]# kubectl get deploy -n dev -w
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
nginx-hpa-mem   1/1     1            1           17m
nginx-hpa-mem   1/2     1            1           17m
nginx-hpa-mem   1/2     1            1           17m
nginx-hpa-mem   1/2     1            1           17m
nginx-hpa-mem   1/2     2            1           17m
nginx-hpa-mem   2/2     2            2           18m

[root@master ~]# kubectl get pod -n dev -w
NAME                            READY   STATUS    RESTARTS   AGE
nginx-hpa-mem-568984d85-wq5n4   1/1     Running   0          17m
nginx-hpa-mem-568984d85-ntjh2   0/1     Pending   0          0s
nginx-hpa-mem-568984d85-ntjh2   0/1     Pending   0          0s
nginx-hpa-mem-568984d85-ntjh2   0/1     ContainerCreating   0          0s
nginx-hpa-mem-568984d85-ntjh2   1/1     Running             0          29s

```

结论：

如果在某一时间段流量突然猛增，因为之前replicas一直处于较低值，hpa无法在一个短期内增到一个理想的副本数，hpa启动需要一定的时间。