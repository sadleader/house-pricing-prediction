# The house price prediction app

### Requirements:

Dataset:

| House Price in $1000s (Y) | Square Feed (X) |
|---------------------------|-----------------|
| 245                       | 1400            |
| 312                       | 1600            |
| 279                       | 170             |
| 308                       | 1875            |
| 199                       | 1100            |
| 219                       | 1550            |
| 405                       | 2350            |
| 324                       | 2450            |
| 319                       | 1425            |
| 255                       | 1700            |

Tasks:

 - Create a Kubernetes pod (Application, Database, Webserver)  **(Webserver pod done)**
 - Store the data in a database container and make it available to your app container **(open with config questions)**
 - Create a ML module in Python with the ability to predict house prices (Y), based on the square feets as input parameter (X). Train your model based on the dataset. **(done with mocked data)**
 - Create a simple HTTP REST-API on top of your ML module that takes X as parameter for the request and responds with prediction Y **(done)**
 - Create a simple HTTP REST-API on top of your database module that allows passing a simple "SELECT" statement **(open)**
 - Augment your containers and serve the application and the database query with HTTPS via Nginx **(open)**
 - Replicate the app container and set a loadbalancer **(done)**
 - Monitor app & DB container (live & readiness) **(done with questions)**
 - Add another DB container that allows to query for Z (simulate and ingest Z!) **(open)**
 - Do a rolling upgrade of the application container. **(done)**
 - Create sharding of functions for querying X, Y and Z from the database **(open)**
 - Commit all code & results to Git using Gitflow **(done)**

### Steps:

##### ML module and REST-API (Done)
1. Create a mlModel(linear regression) to predict house prices based on mocked data
2. Build a REST-API to make predictions
3. Containerized the web-service and deploy with K8s

TODO: connect to the Database module to replace the mocked data


##### Database module 
1. Create a mySql database container with init data
2. Serve the DB via K8s
3. Expand the module with a REST-API that allows passing "SELECT" statement (?)

TODO: serve DB via K8s

##### K8s config
1. Add livenes & readiness probs
2. Scale the deployment with replicaSet
3. Rolling Updates, Canary Deployments & Blue-Green Deployments



### Run the app:

```
cd Build
docker build -t sadleader/house-pricing-server .
docker login
docker push sadleader/house-pricing-server
```

##### to run the app as single container
```
docker run -p 5000:5000 sadleader/house-pricing-server
```
 - open `http://localhost:5000` in the browser, you will see the welcome message
 - change the url to `http://localhost:5000/predicte-by-size/1200`, you will see the prediction (in $1000)


##### to create pod and expose service for the app
```
cd ../Deploy
source <(kubectl completion bash)
kubectl create -f app.pod.yaml
kubectl create -f app.service.yaml
```

check the pods and services
```
kubectl get pods
kubectl get services
kubectl describe service house-pricing-server
```
To access the service within the cluster,
notice that there's an entrypoint from the service description,
open `http://<ENTRYPOINT>` in the browser

#### the liveness & readiness probes (?? with some questions)
1. the liveness probes was added to the Deployment/app.pod.yaml
```
livenessProbe:
  httpGet:
    path: /healthz                                                                                                                                                            
    port: 5000
```
2. the health check endpoint was added to the app.py
```
@app.route('/healthz')
def healthz():
    return Response("Is Healthy", status=200, mimetype='application/json')
```
3. the readiness probes:

The only difference is that you use the readinessProbe field instead of the livenessProbe field.

[see: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/]

the above livenessProbes got the error 172.17.0.4:5000/healthz connecton refused.
the pod will be running and users can visit 172.17.0.4:5000, but soon failed, then repeat.

#### the replicaSet

created a new deployment file app.deployment.yaml, with a replicas setting:
```
replicas: 3
```
run
```
kubectl create -f app.deployment.yaml
kubectl create -f app.service.yaml
kubectl get pods
```
you will see 3 pods are running



#### the rolling update
make some change in the app.py and tag the image with a new tag 'sadleader/house-pricing-server:v2'.

So that 
 - in V1, http://localhost:5000/ says:
`Welcome to the house pricing prediction app!`
 - in V2, http://localhost;5000/ says:
 `Welcome to the house pricing prediction app!V2!!!`

show current deployments by `kubectl get deployments`
edit the deployment and change the image version form `V1` to `V1` by
```
kubectl edit deployment house-pricing-server
```
save and quit the editing mode, it will starts the rolling upgrade.

run `kubectl get replicaSet` to see the new replicaSet

run `kubectl rollout history deployment house-pricing-server` to view the new entry in the rollout history

run `kubectl rollout pause deployment house-pricing-server` to pause the update

run `kubectl rollout status deployment house-pricing-server` to view the current status of the update.

run `kubectl rollout resume deployment house-pricing-server` to resume the update.

after success, you should be able to see the new welcome message.

run `kubectl rollout undo deployment house-pricing-server` to rollback.
