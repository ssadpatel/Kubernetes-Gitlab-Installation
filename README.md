# Kubernetes-Gitlab-Installation
Deploying Gitlab on the Kubernetes cluster.

# 1. Deploying Gitlab-CE on the Kubernetes cluster.

First, we need to create a `GitLab` namespace.

`$ kubectl create ns gitlab`

Creating PV and PVC for storing `GitLab` data and configuration.

Creating `gitlab-pv`. Replace nfs-server and export path.

```
cat <<EOF | tee gitlab-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gitlab-claim
  namespace: gitlab
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  storageClassName: gitlab-claim
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /nfs-export-path
    server: nfs-server
    readOnly: true
EOF
```

`$ kubectl apply -f gitlab-pv.yaml`

Creating `gitlab-pvc`.

```
cat <<EOF | tee gitlab-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-claim
  namespace: gitlab
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: gitlab-claim
  resources:
     requests:
       storage: 10Gi
EOF
```

`$ kubectl apply -f gitlab-pvc.yaml`


Creating service with NodePort.
```
cat <<EOF | tee service.yaml
apiVersion: v1
kind: Service
metadata:
  name: gitlab-svc
  namespace: gitlab
spec:
  ports:
  - port: 80
    targetPort: 30088
    nodePort: 30088
    protocol: TCP
    name: http
  - port: 22
    targetPort: 22
    nodePort: 30022
    protocol: TCP
    name: tcp
  selector:
    app: gitlab
  type: NodePort
EOF
```

`$ kubectl apply -f service.yaml`


Creating gitlab deployment.
```
cat <<EOF | tee gitlab-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab
  namespace: gitlab
  labels:
    app: gitlab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitlab
  template:
    metadata:
      labels:
        app: gitlab
    spec:
      containers:
      - name: gitlab
        image: gitlab/gitlab-ce
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 30088
        volumeMounts:
        - name: gitlab-data
          mountPath: /etc/gitlab
          subPath: git-config
        - name: gitlab-data
          mountPath: /var/log/gitlab
          subPath: git-logs
        - name: gitlab-data
          mountPath: /var/opt/gitlab
          subPath: gitlab-data
      volumes:
      - name: gitlab-data
        persistentVolumeClaim:
          claimName: gitlab-claim
EOF
```

`$ kubectl apply -f gitlab-deployment.yaml`



# 2. Deploying Gitlab-Runner on the Kubernetes cluster.

We need a service account and cluster-admin label role to access all the resources for gitlab-runner.

```
cat <<EOF | gitlab-runner-sa-cluster-role.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-admin
  namespace: gitlab
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: gitlab-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: gitlab-admin
  namespace: gitlab
EOF
```

`$ kubectl apply -f gitlab-runner-sa-cluster-role.yaml`

Create a configamp for gitlab-runner. Change the registration-token and service-account name with your. (Remeber the tag name {--tag-list k8s} it will be use in gitlab-ci.

```
cat <<EOF |tee gitlab-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitlab-runner-config
  namespace: gitlab
  entrypoint: |
    #!/bin/bash
    set -xe
    # Register the runner (((Using Shared runner for this deployment--get the shared token from root user-->in admin tab)))
    /entrypoint register --non-interactive \
      --url http://gitlab-ip:port/ci \
      --registration-token KRxpJXsfllasflXDDD \
      --name k8s-runner \
      --kubernetes-image alpine \
      --tag-list k8s \
      --executor kubernetes \
      --kubernetes-namespace gitlab \
      --kubernetes-namespace_overwrite_allowed ".*" \
      --kubernetes-pod_annotations_overwrite_allowed ".*" \
      --kubernetes-service-account gitlab-admin \
      --kubernetes-service_account_overwrite_allowed ".*" \
      --kubernetes-privileged true

    # Change concurrent value from 1 to 10 after register runner
    grep -rl 'concurrent = 1' /etc/gitlab-runner/config.toml | xargs sed -i "s|concurrent = 1|concurrent = 10|g"

    # Start the runner
    /entrypoint run --config "/etc/gitlab-runner/config.toml"
EOF
```

`$ kubectl apply -f gitlab-configmap.yaml`


And finaly deploying gitlab-runner. Service must be the same in deployement(serviceAccountName: gitlab-admin).

```
cat <<EOF | tee gitlab-runner.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab-runner
  namespace: gitlab
  labels:
    name: kubernetes-runner
    app: gitlab-runner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitlab-runner
      name: kubernetes-runner
  template:
    metadata:
      labels:
        name: kubernetes-runner
        app: gitlab-runner
    spec:
      serviceAccountName: gitlab-admin
      containers:
      - name: gitlab-runner
        image: gitlab/gitlab-runner
        command: ["/bin/bash", "/config/entrypoint"]
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: gitlab-runner-config
          mountPath: /config
      volumes:
      - name: gitlab-runner-config
        configMap:
          name: gitlab-runner-config
          defaultMode: 0755
EOF
```

`$ kubectl apply -f gitlab-runner.yaml`


Get the pod status by `kubectl get po -n gitlab`

```
kubectl get po -n gitlab

NAME                             READY   STATUS    RESTARTS   AGE
gitlab-7c6fdbcc97-d8rfj          1/1     Running   0          5m
gitlab-runner-865d779469-gbxkt   1/1     Running   0          120s
```
