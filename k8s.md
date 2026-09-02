# Kubernetes Demo

## 1. Install a Kubernetes cluster with Minikube + Docker Desktop
If you are using Docker Desktop on macOS, you can run Minikube with Docker as the driver.

### Prerequisites
- Docker Desktop installed and running
- Minikube installed

### Install Minikube
```bash
brew install minikube
```

### Start cluster with Docker driver
```bash
minikube start --driver=docker
```

### Verify the cluster
```bash
kubectl version
kubectl cluster-info
kubectl get nodes
```

### Stop / delete cluster
```bash
minikube stop
minikube delete
```

Reference:
https://minikube.sigs.k8s.io/docs/start/
https://kubernetes.io/docs/tasks/tools/

## 2. Check cluster status
```bash
kubectl version
kubectl cluster-info
kubectl get nodes
```

## 3. Create a Pod
A Pod is the smallest deployable unit in Kubernetes and runs one or more containers.

Reference:
https://kubernetes.io/docs/concepts/workloads/pods/

Example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod nginx-pod
kubectl delete -f pod.yaml
```

## 4. Create a Deployment
A Kubernetes Deployment defines the desired state of an application and manages Pods for you. It ensures the right number of replicas are running, supports rolling updates, and can roll back if something fails.

Reference:
https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

Example: nginx-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployments
kubectl get pods -l app=nginx
```


## 5. Expose the Deployment with a Service
A Kubernetes Service provides a stable network endpoint for accessing Pods. The service type decides how the application is exposed: inside the cluster, on a node port, or through a cloud load balancer.
Reference:
https://kubernetes.io/docs/concepts/services-networking/service/

## 5.1 ClusterIP (internal only)
A ClusterIP Service exposes the app only inside the cluster, so other Pods can reach it using a fixed internal IP and DNS name.

Example: nginx-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```bash
kubectl apply -f nginx-service.yaml
kubectl get service nginx-service
kubectl get endpoints
kubectl describe service nginx-service
```

This creates an internal service that routes traffic to the Pods selected by the `app: nginx` label.

## 5.2 NodePort (expose on each node)
A NodePort Service opens the app on a static port across all cluster nodes, so it can be reached from outside the cluster using `NodeIP:NodePort`.

Example:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-example
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

This exposes the service on port `30080` on each node, which is useful for local demos and simple external access.

## 5.3 LoadBalancer (cloud-managed external access)
A LoadBalancer Service creates an external load balancer in supported cloud environments and forwards traffic to the service automatically. In this example we are showing a GKE regional external load balancer.

Example:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-gke-loadbalancer-example
spec:
  type: LoadBalancer
  loadBalancerClass: networking.gke.io/l4-regional-external
  selector:
    app: nginx
  ports:
  - name: tcp-port
    protocol: TCP
    port: 8080
    targetPort: 80
```


In cloud providers like AWS, Azure, or GCP, this typically creates a public IP or DNS name that routes traffic to your Pods.

## 6. Add MongoDB
Create a second deployment and expose it through a Service.

Example: mongo-deployment-and-service.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongo
        image: mongo:7
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_DATABASE
          value: demo
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongo
  ports:
  - protocol: TCP
    port: 27017
    targetPort: 27017
```

```bash
kubectl apply -f mongo-deployment-and-service.yaml
kubectl get deployment mongo-deployment
kubectl get svc mongo-service
kubectl get endpoints
```
## 7. Try to reach Mongo service from nginx pods
Execute a shell from a nginx pod and run a curl command to the mongo db service. 

```bash
kubectl get pods -l app=nginx
kubectl exec -it nginx-deployment-pod-name -- bash
curl mongodb-service:27017
```

## 8. Playing with Deployments
Update image for nginx and see the rollout history. 

```bash
kubectl describe pods -l app=nginx 
kubectl set image deployment/nginx-deployment nginx=nginx:1.25
kubectl rollout history deployment/nginx-deployment
kubectl rollout undo deployment/nginx-deployment
```

Scale the replicas for a deployment.

```bash
kubectl get pods -l app=nginx 
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods -l app=nginx 
```


## Summary
Pods run containers. Deployments manage Pods at scale and make updates, scaling, and self-healing easier. Services give Pods a stable way to communicate with each other inside the cluster.

