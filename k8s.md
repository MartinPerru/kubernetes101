# Kubernetes Demo
## 8. Playing with Deployments

## 8.1 Update MongoDB Deployment
Create a `movies` database during the first MongoDB startup.

Example: mongo-deployment-service.yaml
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
          value: movies
        volumeMounts:
        - name: mongo-init
          mountPath: /docker-entrypoint-initdb.d
      volumes:
      - name: mongo-init
        configMap:
          name: mongo-init
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
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-init
data:
  init.js: |
    db = db.getSiblingDB("movies");
    db.movies.insertMany([
      { title: "The Matrix", year: 1999, genres: ["Action", "Science Fiction"] },
      { title: "Spirited Away", year: 2001, genres: ["Animation", "Fantasy"] },
      { title: "The Grand Budapest Hotel", year: 2014, genres: ["Comedy", "Drama"] }
    ]);
```

The official MongoDB image runs `.js` files mounted in `/docker-entrypoint-initdb.d` only when the database directory is empty. The ConfigMap above therefore creates the `movies` database and inserts three documents on the first startup.

```bash
kubectl apply -f mongo-deployment-service.yaml
kubectl get deployment mongodb-deployment
kubectl get svc mongodb-service
kubectl get endpoints
```

Verify the seeded documents:

```bash
kubectl exec deployment/mongodb-deployment -- mongosh movies --quiet --eval 'db.movies.find().forEach(printjson)'
```

To run the initialization again, delete the MongoDB Pod and its data first. This demo has no persistent volume, so deleting the Pod removes its temporary data and the replacement Pod runs `init.js` again:

```bash
kubectl delete pod -l app=mongo
kubectl rollout status deployment/mongodb-deployment
```
## 8.2 Update nginx deployment
Nginx does not understand MongoDB's protocol, so the Deployment uses a sidecar container to query MongoDB. The sidecar gets the connection details from `mongo-client-settings`, writes the movie results to a shared volume, and Nginx serves the generated page.

```bash
kubectl apply -f nginx-deployment.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx
```

Open the Nginx page locally:

```bash
kubectl port-forward service/nginx-service 8080:80
```

Then visit http://localhost:8080. The page is generated from the documents in the `movies` collection.

Inspect the sidecar logs for MongoDB connection or query errors:

```bash
kubectl logs deployment/nginx-deployment -c movie-page-refresh
```

The sidecar refreshes the page every 30 seconds. To change the connection settings, edit the `mongo-client-settings` ConfigMap and restart the Deployment:

```bash
kubectl edit configmap mongo-client-settings
kubectl rollout restart deployment/nginx-deployment
``` 


