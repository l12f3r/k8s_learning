# MongoDB + web app on Kubernetes ☸︎

After learning about containerization, I saw too many opportunities of working with Kubernetes pass by - enough to concentrate my time on this orchestrator. My learning process is based on small projects and its documentation (one may confirm by [checking my other repos](https://github.com/l12f3r?tab=repositories)).

Learning Kubernetes is not easy. To build knowledge, I started this demo project based on [TechWorld with Nana's crash course](https://www.youtube.com/watch?v=s_o8dwzRlu4), of a web app with MongoDB on K8s (using Minikube).

### 1. First step: the ConfigMap

At this point, I already had Docker, Kubernetes and Minikube installed, apart from having git already set (infrastructure as code good practice). 

Kubernetes uses .yaml files to create and manage entities of its system (referred on documentation as "objects"). Such files usually have four required fields to set values: 
- `apiVersion`: version of the Kubernetes API used upon creation of the object;
- `kind`: kind of object to be created (pod, service _etc_);
- `metadata`: data used to uniquely identify the object (a `name` string, `UID` _etc_);
- `spec`: state desired for the object (its nested fields vary depending on object).

There is a fifth field named `status`, which describes the current state of the object, but this is managed by the tool itself. It compares the current state with what is desired on `spec` and, if there is any difference, the orchestrator works on provisioning resources automatically.

The first file created was `mongo-config.yaml`, a **ConfigMap**, which has `data` (or `binaryData`) fields instead of `spec`; it expects an UTF-8 string as value with the service name of the MongoDB node (that will be its endpoint).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-config #this value is arbitrary
data:
  mongo-url: mongo-service
```

### 2. Secret 🔒

A **Secrets** file is similar to a **ConfigMap**, but for sensitive data (passwords, tokens _etc_). 

⚠️ By default, its contents are stored unencrypted on the master node's `etcd`, which means that anyone accessing it would be able to read and edit the content stored. Make sure to enable encryption at rest and configure RBAC rules to restrict who would be reading and writing what on the file.

For learning purposes, I created an encoded password using the `echo -n mongouser | base64` and `echo -n mongopassword | base64` commands on terminal, just like Nana teaches on the video, but this is not good practice.

I created a `mongo-secret.yaml` file, with the following fields properly filled:
- `kind`: set as Secret;
- `type`: types of built-in configurations that may be applicable depending on scenario (ServiceAccount tokens, serialized docker files, SSH credentials _etc_). The `Opaque` value is default for arbitrary used-defined data.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret #this value is arbitrary
type: Opaque
data:
  mongo-user: bW9uZ291c2Vy
  mongo-password: bW9uZ29wYXNzd29yZA==
```

### 3. Deployment and Service in one file

Following the tutorial, I created a new file named `mongo.yaml` where the **Deployment** and **Service** entities for the MongoDB node will be declared together. 

The values inserted were based on [Dockerhub's official MongoDB image](https://hub.docker.com/_/mongo), using the same 5.0 tag from the tutorial. And the creativity on arbitrary names are a result of the creative mind of yours truly.

Since that this is a study scenario, **Deployment** is used because it works as a blueprint for ephemeral pods; in real-life scenarios, databases must be configured as **StatefulSet** instead. 

The `spec` field on **Deployment** has a few specific subfields that are required:
- `spec.replicas`: the desired amount of pods to be created;
- `spec.selector`: similar to labels, `selector` work as a flag to determine which pod belong to which deployment - in this example, all pods matching labels with the deployment's `metadata.labels` field will follow this deployment's structure. Values are arbitrary;
- `spec.template`: contains the configuration of a pod, with its own `spec` and `metadata` fields. Multiple pods can be listed under this subfield (by adding extra `spec.template.spec.containers` reps) and everything is applicable to its listed pod, within this deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-deployment
  labels:
    app: mongo
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
      - name: mongodbcdefu
        image: mongo:5.0
        ports:
        - containerPort: 27017
```

**Service** works as "a declaration of how pods communicate regardless of status". Think of a route table coupled to a DNS server: useful because it allows pods to communicate via labels, removing the concern of trying to connect to a potential revived pod with a new IP address (there are better definitions on the web, believe me).

Upon defining `metadata.name`, the value set under `data` on the `mongo-config.yaml` **ConfigMap** file must be matched, in order to work as an endpoint to connect. 

Its `spec.selector` field/value must also be matched to the **Deployment** one, for the same connection reasons. Also under `spec`, `spec.ports` is a list that has the following subfields to define:
- `spec.ports.protocol`: Protocol used for this item list;
- `spec.ports.port`: Port used to access the **Service**;
- `spec.ports.targetPort`: Port used to access the pods within the **Deployment** (that is, its `spec.template.spec.containers.containerPort` field).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  selector:
    app: mongo
  ports:
    - protocol: TCP
      port: 27017 #same value inserted as good practice; this is an arbitrary value, usually 80
      targetPort: 27017
```

I also created a `webapp.yaml` file with a very similar structure from `mongo.yaml`, which will represent the webapp deployment with values properly set - [the container image used is the same as Nana's](https://hub.docker.com/r/nanajanashia/k8s-demo-app), some simple NodeJS.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
  labels:
    app: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nanajanashia/k8s-demo-app:v1.0
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
```

🔒 The final configuration would be passing **Secret** and **ConfigMap** data to MongoDB and webapp **Deployment** files. Avoiding hardcoding is always good practice.

The database expects specific environment variables for username and password (`MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD`, respectively), which are automatically set when a database is provisioned. This will be set on the `spec.template.spec.containers.env` field of the `mongo.yaml` file, fetching data from the **Secret** file:

```yaml
# mongo.yaml
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-user
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-password
```

Upon starting, the webapp needs to connect to the database. That means it'll expect the database endpoint set on the **ConfigMap** (as `DB_URL`), apart from username and password from **Secret** (as `USER_NAME` and `USER_PWD`), as environment variables like the `mongo.yaml` file.

```yaml
# webapp.yaml
        env:
        - name: USER_NAME
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-user
        - name: USER_PWD
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: mongo-password
        - name: DB_URL
          valueFrom:
            configMapKeyRef:
              name: mongo-config
              key: mongo-url
```

Now, the webapp must be accessible from a browser. In order to do so, the webapp's **Service** must be configured as external. A few changes must be applied under the `spec` field: 
- `spec.type`: internal services use `ClusterIP` as value; externals use `NodePort`;
- `spec.ports.nodePort`: static port that will expose the **Service** to the Internet on each node's IP. Must be between the 30000-32767 range.

```yaml
# webapp.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort # line added
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 30420 # line added
```

### 4. Deploying on Minikube

Now, all files must be deployed one by one: **Secret** and **ConfigMap** first. On the terminal with the Minikube cluster running, run the following commands in order:
1. `kubectl apply -f mongo-config.yaml`, to load the **ConfigMap**;
2. `kubectl apply -f mongo-secret.yaml`, to load the **Secret**;
3. `kubectl apply -f mongo.yaml`, to create the database; and
4. `kubectl apply -f webapp.yaml` to create the webapp.

⚠️ At this point, I got an error message (`The connection to the server <server-name:port> was refused - did you specify the right host or port?`) because `kubectl` was pointing to a stale minikube-vm (probably something with Docker Desktop, who knows), which led to have a Control Plane running within minikube, but with `apiserver` and `kubelet` both stopped, apart from a misconfigured `kubeconfig` file. I fixed it by updating minikube's context (`minikube update-context`) and restarting it.

After running all previous commands, run `kubectl get all` to view everything running so far. The results must be similar to this:

```bash
⠠⠵ kubectl get all
NAME                                     READY   STATUS    RESTARTS   AGE
pod/mongo-deployment-64bb45c874-ncpnw    1/1     Running   0          3m37s
pod/webapp-deployment-649d7fb885-ctnld   1/1     Running   0          3m25s

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP          2d1h
service/mongo-service    ClusterIP   10.107.66.168   <none>        27017/TCP        3m37s
service/webapp-service   NodePort    10.99.181.183   <none>        3000:30420/TCP   2m4s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mongo-deployment    1/1     1            1           3m37s
deployment.apps/webapp-deployment   1/1     1            1           3m25s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/mongo-deployment-64bb45c874    1         1         1       3m37s
replicaset.apps/webapp-deployment-649d7fb885   1         1         1       3m25s
```

As you can see, all pods, services and deployments are listed properly. To view the unlisted secrets and configmaps, use `kubectl get secret` or `kubectl get configmap`.

### 5. Acessing on the browser

The final step consists in viewing the results of the implementation on the browser. For that, enter `minikube service webapp-service --url` and click the URL provided.

### 6. Thank you

- [Nana Janashia](https://www.youtube.com/c/TechWorldwithNana), for providing such complex knowledge in a very didatic way.