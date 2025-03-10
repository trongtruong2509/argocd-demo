# argocd-demo

Currently there are 2 approaches for migrating "parent helm" concept to ArgoCD.

| Approach | Description | Pros | Cons |
|----------|----------|----------|----------|
| 1    | Similar to parent helm concept   | Centralize all applications in one folder `apps`, reduce duplicate files for multiple clusters.   | More complex with additional Helm chart of apps and value files.   |
| 2    | Each cluster will have its own apps folder  | Easy to understand and implement  | Multiple app file for one service. Have to change all files when any changes from the helm chart.    |

## 1. `Approach 1` is similar to parent helm concept.

Folder structure
```bash
├── apps 
│   ├── Chart.yaml
│   └── templates
│       ├── minio.yaml
│       └── nginx-demo.yaml
├── clusters
│   └── minikube
│       ├── app-of-apps.yaml
│       ├── cluster-config.yaml
│       └── values
│           ├── minio-values.yaml
│           └── nginx-demo-values.yaml
└── helm-charts
    ├── namespaces
    └── nginx-demo
        ├── Chart.yaml
        └── templates
            ├── deployment.yaml
            └── namespaces.yaml
```

- Folder `apps` contains all applications of parent helm chart. Each ArgoCD Application will be disable by default as example below. Or user can set default is enable if they want. The enable variable for each application will be define in `clusters/<cluster_name>/cluster_config.yaml` file.
- Actually this `apps` is a Helm chart of applications with value is `clusters/<cluster_name>/cluster_config.yaml` file.

``` yaml
{{ if index .Values.apps "minio-app" | default false }}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: minio-app
  namespace: argocd  # Change if ArgoCD is installed in a different namespace
spec:
  project: default
  source:
    repoURL: https://operator.min.io
    chart: operator
    targetRevision: "{{ .Values.minio.version }}"  # Check for the latest version on Artifact Hub
    helm: # overwrite helm values here
      releaseName: minio-operator

  destination:
    server: https://kubernetes.default.svc  # Deploy to the in-cluster Kubernetes API
    namespace: "{{ .Values.namespace }}"  # MinIO will be installed in this namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
{{ end }}
```

- Secondly, in clusters folder, each cluster configs will be defined in each folder. In cluster folder, for instance like `minikube`, you will see `app-of-apps.yaml` file for App of Apps concept. Then user can define values for all applications in one place `clusters/<cluster_name>/cluster_config.yaml` or separete each application with its own value file.

- `helm-charts` folder just define custom helm chart from user if they want to restore helm charts in same place as Argocd repo.

## 2. `Approach 2`: Each cluster will have its own apps folder:
Folder structure
```bash
├── clusters
│   └── minikube
│       ├── app-of-apps.yaml
│       └── apps
│           ├── minio.yaml
│           └── nginx-demo.yaml
└── helm-charts
    ├── namespaces
    └── nginx-demo
        ├── Chart.yaml
        └── templates
            ├── deployment.yaml
            └── namespaces.yaml
```

- This approach is basic app-of-apps concept with a `app-of-apps.yaml` that point to a folder that contains all child apps.
- Each `app.yaml` file, for instance like `minio.yaml`, is declaration of ArgoCD Application with helm values inside the yaml as well.
- If user want to disable a application,  just simply remove `minio.yaml` file or comment its content.
- Example of `nginx-demo.yaml` with helm values

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-nginx-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/trongtruong2509/argocd-demo.git
    targetRevision: approach_2
    path: "helm-charts/nginx-demo"
    helm:
      values: | 
        replicaCount: 1
        image:
          repository: nginx
          tag: "latest"
          pullPolicy: Always
        dependency: []

        deployment:
          environment: "development"
        
  destination:
    server: https://kubernetes.default.svc
    namespace: "nginx-demo"
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 3. Create fist app for a cluster:
- Currently for this demo we have to create the first app for each cluster. Asume that user already setup argocd with credentials and k8s, using this argocd command to create an application:
``` bash
argocd app create <app_name> \
  --repo https://github.com/trongtruong2509/argocd-demo.git \
  --revision <branch_name> \
  --path <path_to_cluster> \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --sync-policy automated
```

For example:
- with app name is `minikube`
- branch_name is `approach_2`
- path_to_cluster is `clusters/minikube` to point to minikube cluster

```bash
argocd app create minikube \
  --repo https://github.com/trongtruong2509/argocd-demo.git \
  --revision approach_2 \
  --path clusters/minikube \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --sync-policy automated
```

You can go to the argo UI to check the app you just created.