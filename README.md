# k8s_learning

After learning about containerization, I saw too many opportunities of working with Kubernetes pass by - enough to concentrate my time on this orchestrator. My learning process is based on small projects and its documentation (one may confirm by checking my other repos).

Learning Kubernetes is not easy. To build knowledge, I started this demo project based on [TechWorld with Nana's crash course] (https://www.youtube.com/watch?v=s_o8dwzRlu4), of a web app with MongoDB on K8s (using Minikube).

### 1. First step: the ConfigMap

At this point, I already had Docker, Kubernetes and Minikube installed, apart from having git already set (infrastructure as code good practice). 

Kubernetes' uses .yaml files to create and manage entities of its system (referred on documentation as "objects"). Such files usually have four required fields to set values: 
- `apiVersion`: version of the Kubernetes API used upon creation of the object
- `kind`: kind of object to be created (pod, service etc)
- `metadata`: data used to uniquely identify the object (a `name` string, `UID` etc)
- `spec`: state desired for the object (its nested fields vary depending on object)

There is a fifth field named `status`, which describes the current state of the object, but this is managed by the tool itself. It compares the current state with what is desired on `spec` and, if there is any difference, the orchestrator works on provisioning resources automatically.

The first file created was `mongo-config.yaml`, a ConfigMap, which does has `data` (or `binaryData`) fields instead of `spec`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-config #this value is arbitrary
data:
  mongo-url: mongo-service #UTF-8 string with the service name of the MongoDB node, which will be its endpoint
```