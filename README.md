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

The first file created was `mongo-config.yaml`, a **ConfigMap**, which has `data` (or `binaryData`) fields instead of `spec`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-config #this value is arbitrary
data:
  mongo-url: mongo-service #UTF-8 string with the service name of the MongoDB node, which will be its endpoint
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