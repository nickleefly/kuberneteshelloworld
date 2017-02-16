# A simple Kubernetes Frontend-Backend Service on Google Cloud Platform

Sample application that deploys a trivial [Kubernetes service][1] to connect a frontend system (show) with a backend (backend).

This is merely to demonstrate Kubernetes service discovery in [Google Container Engine (GKE)][2], nothing more and is based off the guestbook service example.

Parts of this sample code is from the default kubernetes 'environment-guide' example and is a direct copy from my github page:

https://github.com/nickleefly/kuberneteshelloworld

### Build your images, and push to docker hub

    git clone https://github.com/nickleefly/kuberneteshelloworld.git
    cd kuberneteshelloworld/container/backend
    docker build -t env-backend .
    docker tag env-backend nickleefly/env-backend:latest
    cd kuberneteshelloworld/container/show
    docker build -t env-show .
    docker tag env-show nickleefly/env-show:latest
    docker images
    docker push nickleefly/env-backend
    docker push nickleefly/env-show

Also taken from kubernetes [environment guide][3] Example

HTTP requests to the frontend handler causes a service lookup for the backend and retrieves some data from the backend.

user → loadbalancer → frontend → lookup backend service → make backend API call → return data to fronend → return web page to user showing some data from the backend.

#### Frontend/backend Services

The frontend consists of a golang program that listens for http requests on port :8080. It is wrapped in the container

[nickleefly/container/env-show][4]

The frontend service is configured to startup 2 replicas inside a Loaldbalanced Kubernetes service:

Frontend

Replication Controller

```
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: show-rc
  labels:
    type: show-type
spec:
  replicas: 3
  template:
    metadata:
      labels:
        type: show-type
    spec:
      containers:
      - name: show-container
        image: gcr.io/google-samples/env-show:1.1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          protocol: TCP
        env:
        - name: USER_VAR
          value: important information
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```


Frontend Service Definition

```
---
apiVersion: v1
kind: Service
metadata:
  name: show-srv
  labels:
    type: show-type
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    type: show-type
```


#### Backend

The backend consists of another golang program which for http requests on port :5000. It is wrapped in the container

[nickleefly/container/env-backend][5]

Backend uses the advertises the label _be-type_

Replication Controller


```
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: backend-rc
  labels:
    type: backend-type
spec:
  replicas: 3
  template:
    metadata:
      labels:
        type: backend-type
    spec:
      containers:
      - name: backend-container
        image: gcr.io/google-samples/env-backend:1.1
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
          protocol: TCP
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```

Backend Servce Definition


```
---
apiVersion: v1
kind: Service
metadata:
  name: show-srv
  labels:
    type: show-type
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    type: show-type
```

### Discovery

Once a request hits any pod running the frontend service, the front end attempts to discover how to connect to the backend service. This is done in two ways: using environment variables or (preferably), by DNS SRV lookups.
Each node in the cluster runs a local [DNS][6] server.

Also see [Kubernetes networking][7]

Environment Variables


    backendHost := os.Getenv("BACKEND_SRV_SERVICE_HOST")
    backendPort := os.Getenv("BACKEND_SRV_SERVICE_PORT")
    backendRsp, backendErr := http.Get(fmt.Sprintf(
      "http://%v:%v/",
      backendHost,
      backendPort))
    if backendErr == nil {
      defer backendRsp.Body.Close()
    }

DNS SRV


    cname, rec, err := net.LookupSRV("be", "tcp", "be-srv.default.svc.cluster.local")
        if err != nil {
            http.Error(resp, err.Error(), http.StatusInternalServerError)
        }
        fmt.Fprintf(resp, "SRV CNAME: %v\n", cname)
        for i := range rec {
            fmt.Fprintf(resp, "SRV Records: %v \n", rec[i])
            DNSbackendHost = rec[i].Target
            DNSbackendPort = strconv.Itoa(int(rec[i].Port))
        }

### Create Test Cluster

Create the cluster with two nodes in us-central1-a using either gcloud or the Cloud Console


    gcloud config set compute/zone us-central1-a
    gcloud container clusters create cluster-1 --num-nodes 3


    gcloud compute instances list
    NAME                                                  ZONE           MACHINE_TYPE               PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
    gke-cluster-1-default-pool-1e4517ed-44z4              us-central1-a  n1-standard-1                           10.128.0.29  104.154.236.204  RUNNING
    gke-cluster-1-default-pool-1e4517ed-g970              us-central1-a  n1-standard-1                           10.128.0.8   35.184.46.102    RUNNING
    gke-cluster-1-default-pool-1e4517ed-p8m1              us-central1-a  n1-standard-1                           10.128.0.24  104.197.175.224  RUNNING

### Create Replication Controllers and Services

Run thefollowing to create the frontend/backend replication controllers and services.


    kubectl create -f backend-rc.yaml
    kubectl create -f backend-srv.yaml
    kubectl create -f show-rc.yaml
    kubectl create -f show-srv.yaml

#### List the nodes


    kubectl get no
    NAME                                       STATUS    AGE
    gke-cluster-1-default-pool-1e4517ed-44z4   Ready     11m
    gke-cluster-1-default-pool-1e4517ed-g970   Ready     11m
    gke-cluster-1-default-pool-1e4517ed-p8m1   Ready     11m


#### List the pods


    kubectl get po
    NAME               READY     STATUS    RESTARTS   AGE
    backend-rc-33mm5   1/1       Running   0          2m
    backend-rc-9hlm5   1/1       Running   0          2m
    backend-rc-knf85   1/1       Running   0          2m
    show-rc-lhvff      1/1       Running   0          1m
    show-rc-qjk95      1/1       Running   0          1m
    show-rc-qnm6w      1/1       Running   0          1m

#### List the replication controllers


    kubectl get rc
    NAME         DESIRED   CURRENT   READY     AGE
    backend-rc   3         3         3         3m
    show-rc      3         3         3         2m


#### List the services


    kubectl get svc
    NAME          CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
    backend-srv   10.3.252.149   <none>           5000/TCP       4m
    kubernetes    10.3.240.1     <none>           443/TCP        14m
    show-srv      10.3.248.167   130.211.198.39   80:30720/TCP   3m


The above shows the service IP addresses. Since we're running this on GKE, it will also create a provider-specific commands to generate a public IP, loadbalancer, filewall rules.

The public IP assigned to our Frontend Loadblancer is: 130.211.198.39

### Test the GKE cluster

The frontend service is available 130.211.198.39:80 so an invocation shows:


    curl http://130.211.198.39/
    Pod Name: show-rc-qnm6w
    Pod Namespace: default
    USER_VAR: important information

    Kubenertes environment variables
    BACKEND_SRV_SERVICE_HOST = 10.3.252.149
    BACKEND_SRV_SERVICE_PORT = 5000
    KUBERNETES_SERVICE_HOST = 10.3.240.1
    KUBERNETES_SERVICE_PORT = 443
    SHOW_SRV_SERVICE_HOST = 10.3.248.167
    SHOW_SRV_SERVICE_PORT = 80

    Found backend ip: 10.3.252.149 port: 5000
    Response from backend
    Backend Container
    Backend Pod Name: backend-rc-knf85
    Backend Namespace: default

The above output is from the frontend and shows the backend discovery by both environment variables and DNS SRV. The response from the backend is just the part BACKEND Response.

The output shows the frontend discovered the backend using envionment variables IP address/port values:10.167.252.252:5000

The output also shows the DNS SRV request contained the host and port to connect to from the frontend: be-srv.default.svc.cluster.local. port: 5000

### Call Flow

The following shows the call flow between the the frontend and backend.

For a detailed description of the proxy and VIPs, see [Kubernetes Services][8]

![Diagram](/images/docs/hello-container-engine-cluster.jpg)

[available draw link](https://goo.gl/iZG7Ul)
### Automatic configuration on GKE

The firwall, network configuration created automatically


Cleanup

    kubectl delete rc,service -l type=show-type
    kubectl delete rc,service -l type=backend-type
    gcloud container clusters delete cluster-1

### Extending the sample

If you want to extend the sample, the easiest way is to build the sample an push it to your public dockerhub area where kubernetes can download it.

Remember to rename the image section to whatever you taged to it.

You can, of course use [Google Container Registry][9]

[1]: https://github.com/kubernetes/kubernetes.github.io/blob/master/docs/user-guide/services/index.md
[2]: https://cloud.google.com/container-engine/docs/services/
[3]: https://github.com/kubernetes/kubernetes.github.io/tree/master/docs/user-guide/environment-guide
[4]: https://hub.docker.com/r/nickleefly/env-show/
[5]: https://hub.docker.com/r/nickleefly/env-backend/
[6]: https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns
[7]: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/networking.md
[8]: http://kubernetes.io/v1.0/docs/user-guide/services.html#ips-and-vips
[9]: https://cloud.google.com/container-registry/
