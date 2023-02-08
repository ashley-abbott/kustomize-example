# Example

This repo is intended as a brief exammple for [Kustomize](https://kubectl.docs.kubernetes.io/guides/example/multi_base/)

## How to I Kustomize

You can verify the template by using:

```bash
~: kubectl kustomize overlays/development/

apiVersion: v1
kind: Namespace
metadata:
  name: development
---
apiVersion: v1
data:
  nginx.conf: |
    user nginx;
    worker_processes auto;
    error_log /dev/fd/2;

    events {
      worker_connections 1024;
    }

    http {
      log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
      access_log /dev/fd/1 main;

      server {
        listen 80;
        server_name _;

        location / {
          root html;
          index index.html index.htm;
        }
      }

      include /etc/nginx/conf.d/*.conf;
    }
kind: ConfigMap
metadata:
  name: dev-nginx-config
  namespace: development
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: dev-nginx
  namespace: development
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: dev-nginx
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.23.2
        name: nginx
        resources:
          limits:
            cpu: 100m
            memory: 50Mi
        volumeMounts:
        - mountPath: /etc/nginx/nginx.conf
          name: nginx-config
          readOnly: true
          subPath: nginx.conf
      volumes:
      - configMap:
          items:
          - key: nginx.conf
            path: nginx.conf
          name: dev-nginx-config
        name: nginx-config
```

Now we are happy with the output and wish to apply this to the cluster, that is achieved by the following:

```bash
~: microk8s kubectl apply -k overlays/development

namespace/development created
configmap/dev-nginx-config created
service/dev-nginx created
deployment.apps/dev-nginx created
```

To verify we created the desired objects

```bash
~: microk8s kubectl -n development get cm,svc,deploy,po


NAME                         DATA   AGE
configmap/kube-root-ca.crt   1      2m11s
configmap/dev-nginx-config   1      2m11s

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/dev-nginx   ClusterIP   10.152.183.82   <none>        80/TCP    2m11s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dev-nginx   1/1     1            1           2m11s

NAME                             READY   STATUS    RESTARTS   AGE
pod/dev-nginx-5d6b897966-9j5st   1/1     Running   0          2m11s
```

> Note: All objects are prefixed with the `namePrefix` from the kustomization.yaml

## How do I override a base manifest

Say you would like to apply resource limits to the Deployment from base, you would specify a [patch](./overlays/production/kustomization.yaml#L8)

## How do I add more K8s objects

In development it's reasonable to assume that a HorizontalPodAutoscaler isn't required but in production it is, you can supplement your base(s) by adding a manifest under the relevant folder structure and including it in the kustomization.yaml [see kustomization](./overlays/production/kustomization.yaml#L6)

```bash
~: microk8s kubectl kustomize overlays/production

apiVersion: v1
kind: Namespace
metadata:
  name: production
---
apiVersion: v1
data:
  nginx.conf: |
    user nginx;
    worker_processes auto;
    error_log /dev/fd/2;

    events {
      worker_connections 1024;
    }

    http {
      log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
      access_log /dev/fd/1 main;

      server {
        listen 80;
        server_name _;

        location / {
          root html;
          index index.html index.htm;
        }
      }

      include /etc/nginx/conf.d/*.conf;
    }
kind: ConfigMap
metadata:
  name: prod-nginx-config
  namespace: production
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: prod-nginx
  namespace: production
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: prod-nginx
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.22.1
        name: nginx
        resources:
          limits:
            cpu: 500m
            memory: 300Mi
        volumeMounts:
        - mountPath: /etc/nginx/nginx.conf
          name: nginx-config
          readOnly: true
          subPath: nginx.conf
      volumes:
      - configMap:
          items:
          - key: nginx.conf
            path: nginx.conf
          name: prod-nginx-config
        name: nginx-config
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: prod-nginx
  namespace: production
spec:
  maxReplicas: 10
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 50
        type: Utilization
    type: Resource
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: prod-nginx
```

> Note: See we are now creating the HPA object
