---
title: Kubernetes (Allinone)
taxonomy:
    category: docs
---

### Deploy Using Kubernetes
NeuVector requires and supports Kubernetes network plug-ins such as flannel, weave, or calico. 

You can use Kubernetes to deploy the allinone and enforcer and make sure that all new nodes have an enforcer deployed. Sample files follow the commands below. There is a separate section for OpenShift instructions, and Docker EE on Kubernetes has some special steps described in the Docker section. For a more production ready deployment using multiple controllers, see the Production Deployment section.

Note: This sample section does not include creating the NeuVector Customer Resource Definition (CRD). Please see Deploying NeuVector for the CRD samples.


<ol><li>Create the NeuVector namespace
<pre>
<code>kubectl create namespace neuvector</code></pre>
</li>
<li>Add read permission to access the kubernetes API. RBAC is supported in kubernetes 1.8+ officially. Admission control is supported in kubernetes 1.9+. 
<pre>
<code>kubectl create clusterrole neuvector-binding-app --verb=get,list,watch,update --resource=nodes,pods,services,namespaces
kubectl create clusterrole neuvector-binding-rbac --verb=get,list,watch --resource=rolebindings.rbac.authorization.k8s.io,roles.rbac.authorization.k8s.io,clusterrolebindings.rbac.authorization.k8s.io,clusterroles.rbac.authorization.k8s.io
kubectl create clusterrolebinding neuvector-binding-app --clusterrole=neuvector-binding-app --serviceaccount=neuvector:default
kubectl create clusterrolebinding neuvector-binding-rbac --clusterrole=neuvector-binding-rbac --serviceaccount=neuvector:default
kubectl create clusterrole neuvector-binding-admission --verb=get,list,watch,create,update,delete --resource=validatingwebhookconfigurations,mutatingwebhookconfigurations
kubectl create clusterrolebinding neuvector-binding-admission --clusterrole=neuvector-binding-admission --serviceaccount=neuvector:default
kubectl create clusterrole neuvector-binding-customresourcedefinition --verb=watch,create,get --resource=customresourcedefinitions
kubectl create clusterrolebinding  neuvector-binding-customresourcedefinition --clusterrole=neuvector-binding-customresourcedefinition --serviceaccount=neuvector:default
kubectl create clusterrole neuvector-binding-nvsecurityrules --verb=list,delete --resource=nvsecurityrules,nvclustersecurityrules
kubectl create clusterrolebinding neuvector-binding-nvsecurityrules --clusterrole=neuvector-binding-nvsecurityrules --serviceaccount=neuvector:default
kubectl create clusterrolebinding neuvector-binding-view --clusterrole=view --serviceaccount=neuvector:default
kubectl create rolebinding neuvector-admin --clusterrole=admin --serviceaccount=neuvector:default -n neuvector</code>
</pre>
NOTE: If upgrading NeuVector from a previous install, you may need to delete the old binding as follows:
<pre>
<code>kubectl delete clusterrolebinding neuvector-binding
kubectl delete clusterrole neuvector-binding</code>
</pre>
</li>
<li>Run the following command to check if the neuvector/default service account is added successfully.
<pre>
<code>kubectl get ClusterRoleBinding neuvector-binding-app neuvector-binding-rbac neuvector-binding-admission -o wide</code>
</pre>
<pre>
<code>NAME                          AGE       ROLE                                      USERS     GROUPS    SERVICEACCOUNTS
neuvector-binding-app         1m        ClusterRole/neuvector-binding-app                             neuvector/default
neuvector-binding-rbac        1m        ClusterRole/neuvector-binding-rbac                            neuvector/default
neuvector-binding-admission   50d       ClusterRole/neuvector-binding-admission                       neuvector/default</code>
</pre>
</li>
<li>Add a nvallinone label on one of the worker nodes where the allinone will be deployed (note: by default Kubernetes will not schedule pods on the master)
<pre>
<code>kubectl label nodes nodename nvallinone=true</code></pre>
</li>
<li>Create the neuvector services and pods using the sample below
<pre>
<code>kubectl create -f neuvector.yaml</code></pre>

</li></ol>
NOTE: The nodeport service specified in the neuvector.yaml file will open a random port on all kubernetes nodes for the NeuVector management web console port. If you want to see which port is open on the host nodes, please do the following commands.
```
kubectl get svc -n neuvector
```
And you will see something like:
```
NAME                          CLUSTER-IP      EXTERNAL-IP   PORT(S)                                          AGE
neuvector-service-webui     10.100.195.99     <nodes>       8443:30257/TCP                                   15m
```

Note: If you are using the Allinone container for testing NeuVector, deploy only one Allinone for your cluster. Multiple Manager (Allinone) instances are not supported on Kubernetes. To test high availability for the Controller refer to the Deploying in Production section.

NOTE: If using the containerd run-time, change the volumeMounts for controller and enforcer pods in the sample yamls from docker.sock to:
```
            - mountPath: /run/containerd/containerd.sock
              name: runtime-sock
              readOnly: true
```
And change the volumes from docker.sock to:
```
       - name: runtime-sock
          hostPath:
            path: /run/containerd/containerd.sock
```

Note: PKS is field tested and requires enabling privileged containers to the plan/tile, and changing the yaml hostPath as follows for Allinone, Controller, Enforcer:
<pre>
<code>      hostPath:
            path: /var/vcap/sys/run/docker/docker.sock</code>
</pre>

#### Sample Files
The sample file below deploys an Allinone (Manager, Controller, Enforcer) on one labeled nodeSelector and Enforcers on all others. It creates a webui service for the console using NodePort, but use LoadBalancer if your environment supports it. It also adds a toleration to schedule an Enforcer on the Master Node, which should be updated if other taints on the master exist. This deployment does NOT save configuration to a persistent data store. See Deploying in Production section for persistent data configuration.

If you have created your own namespace instead of using “neuvector”:
1. Replace all instances of “namespace: neuvector” with your namespace.
2. Search for all instances of ”neuvector-svc-allinone.neuvector” in the files below. Then replace the “neuvector” (after the .) with the namespace you use.

**Kubernetes v1.19+**: Here is the sample configuration file:
```
apiVersion: v1
kind: Service
metadata:
  name: neuvector-svc-admission-webhook
  namespace: neuvector
spec:
  ports:
  - port: 443
    targetPort: 20443
    protocol: TCP
    name: admission-webhook
  type: ClusterIP
  selector:
    app: neuvector-allinone-pod

---

apiVersion: v1
kind: Service
metadata:
  name: neuvector-service-webui
  namespace: neuvector
spec:
  ports:
    - port: 8443
      name: manager
      protocol: TCP
  type: LoadBalancer
  selector:
    app: neuvector-allinone-pod

---

apiVersion: v1
kind: Service
metadata:
  name: neuvector-svc-allinone
  namespace: neuvector
spec:
  ports:
  - port: 18300
    protocol: "TCP"
    name: "cluster-tcp-18300"
  - port: 18301
    protocol: "TCP"
    name: "cluster-tcp-18301"
  - port: 18301
    protocol: "UDP"
    name: "cluster-udp-18301"
  clusterIP: None
  selector:
    app: neuvector-allinone-pod

---

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: neuvector-enforcer-pod
  namespace: neuvector
spec:
  selector:
    matchLabels:
      app: neuvector-enforcer-pod
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: neuvector-enforcer-pod
    spec:
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                - key: "nvallinone"
                  operator: NotIn
                  values: ["true"]
      hostPID: true
      containers:
        - name: neuvector-enforcer-pod
          image: neuvector/enforcer:<version>
          securityContext:
            privileged: true
          env:
            - name: CLUSTER_JOIN_ADDR
              value: neuvector-svc-allinone.neuvector
            - name: CLUSTER_ADVERTISED_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: CLUSTER_BIND_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - mountPath: /lib/modules
              name: modules-vol
              readOnly: true
            - mountPath: /var/run/docker.sock
              name: docker-sock
              readOnly: true
            - mountPath: /host/proc
              name: proc-vol
              readOnly: true
            - mountPath: /host/cgroup
              name: cgroup-vol
              readOnly: true
      terminationGracePeriodSeconds: 1200
      restartPolicy: Always
      volumes:
        - name: modules-vol
          hostPath:
            path: /lib/modules
        - name: docker-sock
          hostPath:
            path: /var/run/docker.sock
        - name: proc-vol
          hostPath:
            path: /proc
        - name: cgroup-vol
          hostPath:
            path: /sys/fs/cgroup
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: neuvector-allinone-pod
  namespace: neuvector
spec:
  selector:
    matchLabels:
      app: neuvector-allinone-pod
  minReadySeconds: 60
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: neuvector-allinone-pod
    spec:
      hostPID: true
      containers:
        - name: neuvector-allinone-pod
          image: neuvector/allinone:<version>
          securityContext:
            privileged: true
          readinessProbe:
            exec:
              command:
              - cat
              - /tmp/ready
            initialDelaySeconds: 5
            periodSeconds: 5
          env:
            - name: CLUSTER_JOIN_ADDR
              value: neuvector-svc-allinone.neuvector
            - name: CLUSTER_ADVERTISED_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: CLUSTER_BIND_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - mountPath: /lib/modules
              name: modules-vol
              readOnly: true
            - mountPath: /var/neuvector
              name: nv-share
              readOnly: false
            - mountPath: /var/run/docker.sock
              name: docker-sock
              readOnly: true
            - mountPath: /host/proc
              name: proc-vol
              readOnly: true
            - mountPath: /host/cgroup
              name: cgroup-vol
              readOnly: true
            - mountPath: /etc/config
              name: config-volume
              readOnly: true
      terminationGracePeriodSeconds: 300
      nodeSelector:
        nvallinone: "true"
      restartPolicy: Always
      volumes:
        - name: modules-vol
          hostPath:
            path: /lib/modules
        - name: nv-share
          hostPath:
            path: /var/neuvector
        - name: docker-sock
          hostPath:
            path: /var/run/docker.sock
        - name: proc-vol
          hostPath:
            path: /proc
        - name: cgroup-vol
          hostPath:
            path: /sys/fs/cgroup
        - name: config-volume
          configMap:
            name: neuvector-init
            optional: true

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: neuvector-scanner-pod
  namespace: neuvector
spec:
  selector:
    matchLabels:
      app: neuvector-scanner-pod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  replicas: 2
  template:
    metadata:
      labels:
        app: neuvector-scanner-pod
    spec:
      containers:
        - name: neuvector-scanner-pod
          image: neuvector/scanner
          imagePullPolicy: Always
          env:
            - name: CLUSTER_JOIN_ADDR
              value: neuvector-svc-controller.neuvector
      restartPolicy: Always

---

apiVersion: batch/v1
kind: CronJob
metadata:
  name: neuvector-updater-pod
  namespace: neuvector
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: neuvector-updater-pod
        spec:
          containers:
          - name: neuvector-updater-pod
            image: neuvector/updater
            imagePullPolicy: Always
            command:
            - /bin/sh
            - -c
            - TOKEN=`cat /var/run/secrets/kubernetes.io/serviceaccount/token`; /usr/bin/curl -kv -X PATCH -H "Authorization:Bearer $TOKEN" -H "Content-Type:application/strategic-merge-patch+json" -d '{"spec":{"template":{"metadata":{"annotations":{"kubectl.kubernetes.io/restartedAt":"'`date +%Y-%m-%dT%H:%M:%S%z`'"}}}}}' 'https://kubernetes.default/apis/apps/v1/namespaces/neuvector/deployments/neuvector-scanner-pod'
            env:
              - name: CLUSTER_JOIN_ADDR
                value: neuvector-svc-controller.neuvector
          restartPolicy: Never
```

**Kubernetes v1.6-1.18**: Here is the sample configuration file:
```
apiVersion: v1
kind: Service
metadata:
  name: neuvector-svc-admission-webhook
  namespace: neuvector
spec:
  ports:
  - port: 443
    targetPort: 20443
    protocol: TCP
    name: admission-webhook
  type: ClusterIP
  selector:
    app: neuvector-allinone-pod

---

apiVersion: v1
kind: Service
metadata:
  name: neuvector-service-webui
  namespace: neuvector
spec:
  ports:
    - port: 8443
      name: manager
      protocol: TCP
  type: NodePort
  selector:
    app: neuvector-allinone-pod

---

apiVersion: v1
kind: Service
metadata:
  name: neuvector-svc-allinone
  namespace: neuvector
spec:
  ports:
  - port: 18300
    protocol: "TCP"
    name: "cluster-tcp-18300"
  - port: 18301
    protocol: "TCP"
    name: "cluster-tcp-18301"
  - port: 18301
    protocol: "UDP"
    name: "cluster-udp-18301"
  clusterIP: None
  selector:
    app: neuvector-allinone-pod

---

apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: neuvector-allinone-pod
  namespace: neuvector
spec:
  minReadySeconds: 60
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: neuvector-allinone-pod
    spec:
      hostPID: true
      containers:
        - name: neuvector-allinone-pod
          image: neuvector/allinone:<version>
          securityContext:
            privileged: true
          readinessProbe:
            exec:
              command:
              - cat
              - /tmp/ready
            initialDelaySeconds: 5
            periodSeconds: 5
          env:
            - name: CLUSTER_JOIN_ADDR
              value: neuvector-svc-allinone.neuvector
            - name: CLUSTER_ADVERTISED_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: CLUSTER_BIND_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - mountPath: /lib/modules
              name: modules-vol
              readOnly: true
            - mountPath: /var/neuvector
              name: nv-share
              readOnly: false
            - mountPath: /var/run/docker.sock
              name: docker-sock
              readOnly: true
            - mountPath: /host/proc
              name: proc-vol
              readOnly: true
            - mountPath: /host/cgroup
              name: cgroup-vol
              readOnly: true
            - mountPath: /etc/config
              name: config-volume
              readOnly: true
      terminationGracePeriodSeconds: 300
      nodeSelector:
        nvallinone: "true"
      restartPolicy: Always
      volumes:
        - name: modules-vol
          hostPath:
            path: /lib/modules
        - name: nv-share
          hostPath:
            path: /var/neuvector
        - name: docker-sock
          hostPath:
            path: /var/run/docker.sock
        - name: proc-vol
          hostPath:
            path: /proc
        - name: cgroup-vol
          hostPath:
            path: /sys/fs/cgroup
        - name: config-volume
          configMap:
            name: neuvector-init
            optional: true

---

apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: neuvector-enforcer-pod
  namespace: neuvector
spec:
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: neuvector-enforcer-pod
    spec:
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                - key: "nvallinone"
                  operator: NotIn
                  values: ["true"]
      hostPID: true
      containers:
        - name: neuvector-enforcer-pod
          image: neuvector/enforcer:<version>
          securityContext:
            privileged: true
          env:
            - name: CLUSTER_JOIN_ADDR
              value: neuvector-svc-allinone.neuvector
            - name: CLUSTER_ADVERTISED_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: CLUSTER_BIND_ADDR
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - mountPath: /lib/modules
              name: modules-vol
              readOnly: true
            - mountPath: /var/run/docker.sock
              name: docker-sock
              readOnly: true
            - mountPath: /host/proc
              name: proc-vol
              readOnly: true
            - mountPath: /host/cgroup
              name: cgroup-vol
              readOnly: true
      terminationGracePeriodSeconds: 1200
      restartPolicy: Always
      volumes:
        - name: modules-vol
          hostPath:
            path: /lib/modules
        - name: docker-sock
          hostPath:
            path: /var/run/docker.sock
        - name: proc-vol
          hostPath:
            path: /proc
        - name: cgroup-vol
          hostPath:
            path: /sys/fs/cgroup

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: neuvector-scanner-pod
  namespace: neuvector
spec:
  selector:
    matchLabels:
      app: neuvector-scanner-pod
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  replicas: 2
  template:
    metadata:
      labels:
        app: neuvector-scanner-pod
    spec:
      containers:
        - name: neuvector-scanner-pod
          image: neuvector/scanner
          imagePullPolicy: Always
          env:
            - name: CLUSTER_JOIN_ADDR
              value: neuvector-svc-controller.neuvector
      restartPolicy: Always

---

apiVersion: batch/v1
kind: CronJob
metadata:
  name: neuvector-updater-pod
  namespace: neuvector
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: neuvector-updater-pod
        spec:
          containers:
          - name: neuvector-updater-pod
            image: neuvector/updater
            imagePullPolicy: Always
            command:
            - /bin/sh
            - -c
            - TOKEN=`cat /var/run/secrets/kubernetes.io/serviceaccount/token`; /usr/bin/curl -kv -X PATCH -H "Authorization:Bearer $TOKEN" -H "Content-Type:application/strategic-merge-patch+json" -d '{"spec":{"template":{"metadata":{"annotations":{"kubectl.kubernetes.io/restartedAt":"'`date +%Y-%m-%dT%H:%M:%S%z`'"}}}}}' 'https://kubernetes.default/apis/apps/v1/namespaces/neuvector/deployments/neuvector-scanner-pod'
            env:
              - name: CLUSTER_JOIN_ADDR
                value: neuvector-svc-controller.neuvector
          restartPolicy: Never
```

**Master Node Taints and Tolerations**
All taint info must match to schedule Enforcers on nodes. To check the taint info on a node (e.g. Master):
```
$ kubectl get node taintnodename -o yaml
```
Sample output:
```
spec:
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
  # there may be an extra info for taint as below
  - effect: NoSchedule
    key: mykey
    value: myvalue
```

If there is additional taints as above, add these to the sample yaml tolerations section:
```
spec:
  template:
    spec:
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
        # if there is an extra info for taints as above, please add it here. This is required to match all the taint info defined on the taint node. Otherwise, the Enforcer won't deploy on the taint node
        - effect: NoSchedule
          key: mykey
          value: myvalue
```
