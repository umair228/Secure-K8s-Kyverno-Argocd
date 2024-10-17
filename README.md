# Enforce Automated Kubernetes Cluster Security Using Kyverno  and Argo CD

In this project we will learn how to enforce policies, governence and compliance on your kubernetes cluster. Whether your kubernetes cluster is on AWS, Azure, GCP or on-premises, this project will work without any additional changes.

To explain the project with examples, using this configuration you can

1. **Generate:** Create a default network policy whenever a namespace is created.
2. **Validate:** Block users from using `latest` tag in the deployment or pod resources.
3. **Mutate:** Attach pod security policy for a pod that is created without any pod security policy configuration.
4. **Verify Images:** Verify if the Images used in the pod resources are properly signed and verified images.

## Project Description

A **DevOps Engineer** will write the required **Kyverno Policy** custom resource and commits it to a Git repository. **Argo CD** which is pre configured with `auto-sync` to watch for resources in the git repo, deploys the Kyverno Policies on to the **Kubernetes** **cluster**.

![High-Level-Diagram](https://raw.githubusercontent.com/venugopalreddy1322/project-diagrams/main/Kyverno_project.png)

**Brief Explaination:** 
we write our **Kyverno policy** and push it to Github repository. Since **Argo CD** is configured to watch for changes in the Git repository and **sync** it with the K8s cluster, ArgoCD will deploy the necessary changes to our **namespace**.
## Why do we need Kyverno ?
### Problem Scenario
Imagine a scenario where we have a **node** with **10GB memory** and  **accidentally**,  deployed a pod with exactly 10GB memory. In that case no other pod would be deployed and all of them  would be in **pending state**.

### Solution
**Kyverno** is an **Advanced Admission Controller** where it evaluates, based on rules whether to deploy certain resoucres or not. In our case if we enforce a **Kyverno policy** that sets resources  limit to not cross  say 5GB, if we deployed a pod with 10GB Memory it would **reject** our request.

## The Heart of Kyverno
**Kyverno** is an open-source policy engine designed for Kubernetes.

A **Kyverno policy** is a collection of rules. Each rule consists of a match declaration, an _optional_ exclude declaration, and one of a validate, mutate, generate, or verifyImages declaration. Each **rule** can contain only a single validate, mutate, generate, or verifyImages child declaration.



![Kyverno-Policy-Structure](https://github.com/satya19977/K8S-Security-With-Kyverno-and-ArgoCD/assets/108000447/a8b4632d-3748-416b-8338-d49e79434aab)

**Policies** can be defined as **cluster-wide resources** (using the kind ClusterPolicy) or **namespaced resources** (using the kind Policy.) As expected, namespaced policies will only apply to resources within the namespace in which they are defined while cluster-wide policies are applied to matching resources across all namespaces. Otherwise, there is no difference between the two types.

# Installation

To setup this project you need to install **Argo CD controller** and **Kyverno controller**, Assuming you have Kubernetes installed.

Installation of both Kyverno and Argo CD are pretty straight forward as both of them support **Helm charts** and also provide a consolidated
installation **yaml files**.

## Kyverno

There are two easy ways to install kyverno:

1. Using Helm
2. Using the kubernetes manifest files

### Using Helm

Add **helm repo** for **Kyverno**.

```
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
```

**Option 1:** Install **Kyverno** in `HA mode`

```
 helm install kyverno kyverno/kyverno -n kyverno --create-namespace --set replicaCount=3
```

**OR**

Install **Kyverno** in `Standalone mode`

```
helm install kyverno kyverno/kyverno -n kyverno --create-namespace --set replicaCount=1
```
### Kyverno Policies
We **created** and **enforced** two Kyverno Policies:

1. **pod-requests-and-limits:** This policy makes sure that each Deployment is created with resource limits.
2. **memory-limit:** This policy ensures that memory limts in a pod don't cross a certain value.

Example YAML file where memory specified is greater than the limit:
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod-exceed-limits
  name: pod-exceed-limits
spec:
  containers:
  - image: nginx
    name: pod-exceed-limits
    resources: 
      requests:
        memory: "4Gi" # Should not exceed 2GB
        cpu: "250m"
      limits:
        memory: "5Gi"
        cpu: "500m"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```



### Argo CD
There are three ways to install **Argo CD**,

1. `kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/install.yaml`
2. Helm Charts, Follow the [link](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd#installing-the-chart).
3. Using the Argo CD Operator, Follow the [link](https://argocd-operator.readthedocs.io/en/latest/install/olm/).

### Components of ArgoCD
1. **Repo Server:** This is used to watch the state of the version control system like Github.
2. **Application Server:** This watches the state of K8s Cluster.
3. **API Server:** It is useful for interacting with ArgoCD.
4. **Dex:** Provides authentication.
5. **Redis:** Used for caching.

## Usage 
### Case 1:  Auto-Sync
Our Cluster has no pods deployed in the default namespace.

We upload a deloyment config,`deployment.yaml`in our git repo.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  selector:
    matchLabels:
      app: nginx-app
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: ng
        image: nginx
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"

# Seperate file for service
---
apiVersion: v1
kind: Service
metadata:
  name: nginxpp-service
spec:
  selector:
    app: nginx-app 
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
```
We now, proceed to build **Argo CD**:

1. **Source** - Our Github link and the path is our `folder name` where yaml files are stored.

2. **Destination** - Where our K8s cluster is running.

_Note - Self Heal and pruning are disabled by default_

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-test
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/umair228/Secure-K8s-Kyverno-Argocd
    targetRevision: HEAD
    path: argo-kyverno
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    automated:
     selfHeal: true
     prune: true  
```
**Argo CD** uses git as the single source of truth and synced our cluster with Git repo.

![Screenshot#2)](https://github.com/satya19977/K8S-Security-With-Kyverno-and-ArgoCD/assets/108000447/85245cb4-ec3d-439e-834b-608dbbfaad04)


As Evident, we have all the resources defined in our Git.
### Case 2: Self-Healing

If this is enabled, any change made directly in the cluster isn't enforced.

Let's increase the number of replicas directly in the cluster and observer what happens

![Screenshot#3](https://github.com/satya19977/K8S-Security-With-Kyverno-and-ArgoCD/assets/108000447/d22bcac5-7709-430b-9517-58b346d700ec)

![Screenshot#4](https://github.com/satya19977/K8S-Security-With-Kyverno-and-ArgoCD/assets/108000447/6ea53525-1574-4b57-a19a-e5969330f10b)

Even though, we **manually** made change to our cluster but since **self-heal** is **enabled**, Argo CD will not push those changes.

### Case 3: Enable Prune

Since prune is enabled any change in the Git will also reflect in our cluster.
![Screenshot#5](https://github.com/satya19977/K8S-Security-With-Kyverno-and-ArgoCD/assets/108000447/8cbe81d3-3290-48ba-923b-5fa598baafcf)

Changing the name of the deployment from `nginx-app` to `deployment-nginx` has caused the previous deployment  to be deleted and the deployment with the new name was created.


![Screenshot (1517)](https://github.com/satya19977/K8S-Security-With-Kyverno-and-ArgoCD/assets/108000447/47a96227-5d56-40fc-9c7c-03f713ae0169)

### Case 4: Deployment with no limits & resources
Let's deploy a pod with no limits and resources and see how Kyverno reacts.

This is the `yaml file` in our Github repo:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-nginx
spec:
  selector:
    matchLabels:
      app: nginx-app
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: ng
        image: nginx
        ports:
        - containerPort: 8080
```
We see that the pod has been **deployed** because in our **Kyverno policy** we  set `ValidateFailureAction `to `Audit` and not enforce. So that's why our pod has been deployed but the logs give us a **warning** to **enforce resource limits** in our pod.

![Screenshot (1521)](https://github.com/satya19977/K8S-Security-With-Kyverno-and-ArgoCD/assets/108000447/585a5be8-7e9c-41b1-ac9a-ceeaa1a7d061)