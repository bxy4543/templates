# Template File Example

![FastGPT Page](docs/images/fastgpt.png)

[简体中文](example_zh.md)

Taking FastGPT as an example here to demonstrate how to create a template. This example assumes that you already have a certain understanding of Kubernetes resource files, and will only explains some parameters unique to the template. The template file is mainly divided into two parts.

![structure](docs/images/structure-black.png#gh-dark-mode-only)![structure](docs/images/structure-white.png#gh-light-mode-only)

## Part One: `Metadata CR`

```yaml
apiVersion: template.app.sealos.io/v1beta1
kind: Template
metadata: 
  name: ${{ defaults.app_name }}
spec:
  title: 'FastGpt'                         
  url: 'https://fastgpt.run/'                         
  github: 'https://github.com/labring/FastGPT'        
  author: 'sealos'                                     
  description: 'Fast GPT allows you to use your own openai API KEY to quickly call the openai interface, currently integrating Gpt35, Gpt4 and embedding. You can build your own knowledge base.'    
  readme: 'https://raw.githubusercontent.com/labring/FastGPT/main/README.md'
  icon: 'https://avatars.githubusercontent.com/u/50446880?s=96&v=4'
  template_type: inline
  defaults:
    app_name:
      type: string
      value: fastgpt-${{ random(8) }}
    app_host:
      type: string
      value: ${{ random(8) }}
  inputs:
    root_passowrd:
      description: 'Set root password. login: username: root, password: root_passowrd'
      type: string
      default: ${{ SEALOS_NAMESPACE }}
      required: true
    openai_key:
      description: 'openai api key'
      type: string
      default: ''
      required: true
    database_type:
      description: 'type of database'
      required: false
      type: choice
      default: 'mysql'
      options:
        - sqlite
        - mysql
```

As demo shows, the metatda CR is a regular Kubernetes custom resource type, and the following table lists the fields that need to be filled in.

| Code            | Description                                                  |
| :---------------| :----------------------------------------------------------- |
| `template_type` | `inline` indicates this is an inline mode template, all yaml files are integrated in one file. |
| `defaults`      | Define default values to be filled into the resource file, such as application name (app_name), domain name (app_host), etc. |
| `inputs`        | Define some parameters needed by the user when deploying the application, such as Email, API-KEY, etc. If none, this item can be omitted |

### Explain: `Variables`

Any characters enclosed in `${{ }}` are variables, and the variables are divided into these types: 
1. `SEALOS_` pre-defined system built-in variables with upper case letters, such as `${{ SEALOS_NAMESPACE }}`, are variables provided by Sealos itself. For all currently supported system variables, please refer to [System Variables](#built-in-system-variables-and-functions).
2. `functions()` functions, such as `${{ random(8) }}`, are functions provided by Sealos itself. For all currently supported functions, please refer to [Functions](#built-in-system-functions).
2. `defaults` is a list of names and values that are filled in by parsing, usually used for random values.
3. `inputs` filled in by the user when deploying the application, inputs will be rendered into frontend form.

### Explain: `Defaults`
`spec.defaults` is a map of names, types and values that are filled as defaults when parsing the template.

| Name    | Description |
| :-------| :---------- |
| `type`  | `string` or `number` stands for the type of the variable, the only difference is that the string type will be quoted when rendered, and the number type will not. |
| `value` | The value of the variable, if the value is a function, it will be rendered. |

### Explain: `Inputs`
`spec.defaults` is a map of defined objects that are parsed and shown as form inputs for user to react.

| Name    | Description |
| :-------| :---------- |
| `description` | The description of the input. Will be rendered as input placeholder. |
| `default`     | The default value of the input. |
| `required`    | Whether the input is required. |
| `type`        | Must be one of `string` \| `number` \| `choice` \| `boolean` |
| `options`?    | When type is `choice`, set list of available options. |

Inputs as demoed above will be rendered as form inputs in the frontend:

<table>
<tr>
<td> Template </td> <td> View </td>
</tr>
<tr>
<td width="50%"> 

```yaml
inputs:
  root_passowrd:
    description: 'Set root password. login: username: root, password: root_passowrd'
    type: string
    default: ''
    required: true
  openai_key:
    description: 'openai api key'
    type: string
    default: ''
    required: true
```

</td> 
<td> 

![render inputs](docs/images/render-inputs.png) 

</td>
</tr>
</table>

### Built-in system variables and functions

#### Built in system variables
- `${{ SEALOS_NAMESPACE }}` The namespace where Sealos user is deployed.
- `${{ SEALOS_CLOUD_DOMAIN }}` The domain suffix of the Sealos cluster.
- `${{ SEALOS_CERT_SECRET_NAME }}` The name of the secret that Sealos uses to store the tls certificate.

#### Built in system functions
- `${{ random(length) }}` generate a random string of length `length`.
- TODO: `${{ if() }}` `${{ endif() }}` Conditional rendering.

## Part Two: `Application Resource File(s)`

This part generally consists of a group of types of resources: 
- Applications `Deployment`, `StatefulSet`, `Service`
- External Access, `Ingress`
- Underlying Requirements, `Database`, `Object Storage`

Each resources can repeat any times without order.

### Explain: `Applications`

Applications is list of as many as `Deployment`, `StatefulSet`, `Service` Or/And `Job`, `Secret`, `ConfigMap`, `Custom Resource` as you want.

<details>

<summary>Demo</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${{ defaults.app_name }}
  annotations:
    originImageName: c121914yu/fast-gpt:latest
    deploy.cloud.sealos.io/minReplicas: '1'
    deploy.cloud.sealos.io/maxReplicas: '1'
  labels:
    cloud.sealos.io/deploy-on-sealos: ${{ defaults.app_name }}
    cloud.sealos.io/app-deploy-manager: ${{ defaults.app_name }}
    app: ${{ defaults.app_name }}
spec:
  replicas: 1
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      app: ${{ defaults.app_name }}
  template:
    metadata:
      labels:
        app: ${{ defaults.app_name }}
    spec:
      containers:
        - name: ${{ defaults.app_name }}
          image: c121914yu/fast-gpt:latest
          env:
            - name: MONGO_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ${{ defaults.app_name }}-mongo-conn-credential
                  key: password
            - name: PG_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ${{ defaults.app_name }}-pg-conn-credential
                  key: password    
            - name: ONEAPI_URL
              value: ${{ defaults.app_name }}-key.${{ SEALOS_NAMESPACE }}.svc.cluster.local:3000/v1
            - name: ONEAPI_KEY
              value: sk-xxxxxx
            - name: DB_MAX_LINK
              value: 5
            - name: MY_MAIL
              value: ${{ inputs.mail }}
            - name: MAILE_CODE
              value: ${{ inputs.mail_code }}
            - name: TOKEN_KEY
              value: fastgpttokenkey
            - name: ROOT_KEY
              value: rootkey
            - name: MONGODB_URI
              value: >-
                mongodb://root:$(MONGO_PASSWORD)@${{ defaults.app_name }}-mongo-mongo.${{ SEALOS_NAMESPACE }}.svc:27017
            - name: MONGODB_NAME
              value: fastgpt
            - name: PG_HOST
              value: ${{ defaults.app_name }}-pg-pg.${{ SEALOS_NAMESPACE }}.svc
            - name: PG_USER
              value: postgres
            - name: PG_PORT
              value: '5432'
            - name: PG_DB_NAME
              value: postgres
          resources:
            requests:
              cpu: 100m
              memory: 102Mi
            limits:
              cpu: 1000m
              memory: 1024Mi
          command: []
          args: []
          ports:
            - containerPort: 3000
          imagePullPolicy: Always
          volumeMounts: []
      volumes: []

---
apiVersion: v1
kind: Service
metadata:
  name: ${{ defaults.app_name }}
  labels:
    cloud.sealos.io/deploy-on-sealos: ${{ defaults.app_name }}
    cloud.sealos.io/app-deploy-manager: ${{ defaults.app_name }}
spec:
  ports:
    - port: 3000
  selector:
    app: ${{ defaults.app_name }}
```

</details>

Frequently changes with the following fields:

| Code                         | Description                                                  |
| :--------------------------- | :----------------------------------------------------------- |
| `metadata.annotations`<br/>`metadata.labels` | Change to match launchpad's requirements, like `originImageName`, `minReplicas`, `maxReplicas`. |
| `spec.containers[].image` | Change to your Docker image. |
| `spec.containers[].env` | Configure environment variables for the container            |
| `spec.containers[].ports.containerPort` | Change to the port corresponding to your Docker image        |
| `${{ defaults.app_name }}` | You can use the `${{ defaults.xxxx }}`\|`${{ inputs.xxxx }}` variables to set parameters defined in the `Template CR`. |

### Explain: `External Accesses`

If the application needs to be accessed externally, you need to add the following code:

<details>

<summary>Demo</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${{ defaults.app_name }}
  labels:
    cloud.sealos.io/deploy-on-sealos: ${{ defaults.app_name }}
    cloud.sealos.io/app-deploy-manager: ${{ defaults.app_name }}
    cloud.sealos.io/app-deploy-manager-domain: ${{ defaults.app_host }}
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-body-size: 32m
    nginx.ingress.kubernetes.io/server-snippet: |
      client_header_buffer_size 64k;
      large_client_header_buffers 4 128k;
    nginx.ingress.kubernetes.io/ssl-redirect: 'false'
    nginx.ingress.kubernetes.io/backend-protocol: HTTP
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/client-body-buffer-size: 64k
    nginx.ingress.kubernetes.io/proxy-buffer-size: 64k
    nginx.ingress.kubernetes.io/configuration-snippet: |
      if ($request_uri ~* \.(js|css|gif|jpe?g|png)) {
        expires 30d;
        add_header Cache-Control "public";
      }
spec:
  rules:
    - host: ${{ defaults.app_host }}.${{ SEALOS_CLOUD_DOMAIN }}
      http:
        paths:
          - pathType: Prefix
            path: /()(.*)
            backend:
              service:
                name: ${{ defaults.app_name }}
                port:
                  number: 3000
  tls:
    - hosts:
        - ${{ defaults.app_host }}.${{ SEALOS_CLOUD_DOMAIN }}
      secretName: ${{ SEALOS_CERT_SECRET_NAME }}
```

</details>

Please note that the `host` field needs to be randomly set for security purpose. You can use `${{ random(8) }}` set to `defaults.app_host` and use `${{ defaults.app_host }}` then.

### Explain: `Underlying Requirements`

Almost all applications needs underlying requirements, such as `database`, `cache`, `object storage` etc. You can add the following code to deploy some underlying requirements we provided:

#### `Database`

We using [`kubeblocks`](https://kubeblocks.io/) to provide database resources support. You can directly use the following code to deploy a database:

<details>

<summary>MongoDB</summary>

```yaml
apiVersion: apps.kubeblocks.io/v1alpha1
kind: Cluster
metadata:
  finalizers:
    - cluster.kubeblocks.io/finalizer
  labels:
    clusterdefinition.kubeblocks.io/name: mongodb
    clusterversion.kubeblocks.io/name: mongodb-5.0.14
    sealos-db-provider-cr: ${{ defaults.app_name }}-mongo
  annotations: {}
  name: ${{ defaults.app_name }}-mongo
  generation: 1
spec:
  affinity:
    nodeLabels: {}
    podAntiAffinity: Preferred
    tenancy: SharedNode
    topologyKeys: []
  clusterDefinitionRef: mongodb
  clusterVersionRef: mongodb-5.0.14
  componentSpecs:
    - componentDefRef: mongodb
      monitor: true
      name: mongodb
      replicas: 1
      resources:
        limits:
          cpu: 1000m
          memory: 1024Mi
        requests:
          cpu: 100m
          memory: 102Mi
      serviceAccountName: ${{ defaults.app_name }}-mongo
      volumeClaimTemplates:
        - name: data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 3Gi
            storageClassName: openebs-backup  
  terminationPolicy: Delete
  tolerations: []


---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-mongo
    app.kubernetes.io/instance: ${{ defaults.app_name }}-mongo
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-mongo

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-mongo
    app.kubernetes.io/instance: ${{ defaults.app_name }}-mongo
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-mongo
rules:
  - apiGroups:
      - ''
    resources:
      - events
    verbs:
      - create

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-mongo
    app.kubernetes.io/instance: ${{ defaults.app_name }}-mongo
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-mongo
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ${{ defaults.app_name }}-mongo
subjects:
  - kind: ServiceAccount
    name: ${{ defaults.app_name }}-mongo
    namespace: ${{ SEALOS_NAMESPACE}}
```

</details>

<details>

<summary>PostgreSQL</summary>

```yaml
apiVersion: apps.kubeblocks.io/v1alpha1
kind: Cluster
metadata:
  finalizers:
    - cluster.kubeblocks.io/finalizer
  labels:
    clusterdefinition.kubeblocks.io/name: postgresql
    clusterversion.kubeblocks.io/name: postgresql-14.8.0
    sealos-db-provider-cr: ${{ defaults.app_name }}-pg
  annotations: {}
  name: ${{ defaults.app_name }}-pg
spec:
  affinity:
    nodeLabels: {}
    podAntiAffinity: Preferred
    tenancy: SharedNode
    topologyKeys: []
  clusterDefinitionRef: postgresql
  clusterVersionRef: postgresql-14.8.0
  componentSpecs:
    - componentDefRef: postgresql
      monitor: true
      name: postgresql
      replicas: 1
      resources:
        limits:
          cpu: 1000m
          memory: 1024Mi
        requests:
          cpu: 100m
          memory: 102Mi
      serviceAccountName: ${{ defaults.app_name }}-pg
      switchPolicy:
        type: Noop
      volumeClaimTemplates:
        - name: data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 5Gi
            storageClassName: openebs-backup
  terminationPolicy: Delete
  tolerations: []

---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-pg
    app.kubernetes.io/instance: ${{ defaults.app_name }}-pg
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-pg

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-pg
    app.kubernetes.io/instance: ${{ defaults.app_name }}-pg
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-pg
rules:
  - apiGroups:
      - ''
    resources:
      - events
    verbs:
      - create
  - apiGroups:
      - ''
    resources:
      - configmaps
    verbs:
      - create
      - get
      - list
      - patch
      - update
      - watch
      - delete
  - apiGroups:
      - ''
    resources:
      - endpoints
    verbs:
      - create
      - get
      - list
      - patch
      - update
      - watch
      - delete
  - apiGroups:
      - ''
    resources:
      - pods
    verbs:
      - get
      - list
      - patch
      - update
      - watch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-pg
    app.kubernetes.io/instance: ${{ defaults.app_name }}-pg
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-pg
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ${{ defaults.app_name }}-pg
subjects:
  - kind: ServiceAccount
    name: ${{ defaults.app_name }}-pg
    namespace: ${{ SEALOS_NAMESPACE }}
```

</details>

<details>

<summary>MySQL</summary>

```yaml
apiVersion: apps.kubeblocks.io/v1alpha1
kind: Cluster
metadata:
  finalizers:
    - cluster.kubeblocks.io/finalizer
  labels:
    clusterdefinition.kubeblocks.io/name: apecloud-mysql
    clusterversion.kubeblocks.io/name: ac-mysql-8.0.30
    sealos-db-provider-cr: ${{ defaults.app_name }}-mysql
  annotations: {}
  name: ${{ defaults.app_name }}-mysql
spec:
  affinity:
    nodeLabels: {}
    podAntiAffinity: Preferred
    tenancy: SharedNode
    topologyKeys: []
  clusterDefinitionRef: apecloud-mysql
  clusterVersionRef: ac-mysql-8.0.30
  componentSpecs:
    - componentDefRef: mysql
      monitor: true
      name: mysql
      replicas: 1
      resources:
        limits:
          cpu: 1000m
          memory: 1024Mi
        requests:
          cpu: 100m
          memory: 102Mi
      serviceAccountName: ${{ defaults.app_name }}-mysql
      volumeClaimTemplates:
        - name: data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 3Gi
            storageClassName: openebs-backup
  terminationPolicy: Delete
  tolerations: []
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-mysql
    app.kubernetes.io/instance: ${{ defaults.app_name }}-mysql
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-mysql

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-mysql
    app.kubernetes.io/instance: ${{ defaults.app_name }}-mysql
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-mysql
rules:
  - apiGroups:
      - ''
    resources:
      - events
    verbs:
      - create

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-mysql
    app.kubernetes.io/instance: ${{ defaults.app_name }}-mysql
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-mysql
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ${{ defaults.app_name }}-mysql
subjects:
  - kind: ServiceAccount
    name: ${{ defaults.app_name }}-mysql
    namespace: ${{ SEALOS_NAMESPACE }}

```

</details>

<details>

<summary>Redis</summary>

```yaml
apiVersion: apps.kubeblocks.io/v1alpha1
kind: Cluster
metadata:
  finalizers:
    - cluster.kubeblocks.io/finalizer
  labels:
    clusterdefinition.kubeblocks.io/name: redis
    clusterversion.kubeblocks.io/name: redis-7.0.6
    sealos-db-provider-cr: ${{ defaults.app_name }}-redis
  annotations: {}
  name: ${{ defaults.app_name }}-redis
spec:
  affinity:
    nodeLabels: {}
    podAntiAffinity: Preferred
    tenancy: SharedNode
    topologyKeys: []
  clusterDefinitionRef: redis
  clusterVersionRef: redis-7.0.6
  componentSpecs:
    - componentDefRef: redis
      monitor: true
      name: redis
      replicas: 1
      resources:
        limits:
          cpu: 1000m
          memory: 1024Mi
        requests:
          cpu: 100m
          memory: 102Mi
      serviceAccountName: ${{ defaults.app_name }}-redis
      switchPolicy:
        type: Noop
      volumeClaimTemplates:
        - name: data
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 3Gi
            storageClassName: openebs-backup
    - componentDefRef: redis-sentinel
      monitor: true
      name: redis-sentinel
      replicas: 1
      resources:
        limits:
          cpu: 100m
          memory: 100Mi
        requests:
          cpu: 100m
          memory: 100Mi
      serviceAccountName: ${{ defaults.app_name }}-redis
  terminationPolicy: Delete
  tolerations: []
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-redis
    app.kubernetes.io/instance: ${{ defaults.app_name }}-redis
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-redis

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-redis
    app.kubernetes.io/instance: ${{ defaults.app_name }}-redis
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-redis
rules:
  - apiGroups:
      - ''
    resources:
      - events
    verbs:
      - create

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    sealos-db-provider-cr: ${{ defaults.app_name }}-redis
    app.kubernetes.io/instance: ${{ defaults.app_name }}-redis
    app.kubernetes.io/managed-by: kbcli
  name: ${{ defaults.app_name }}-redis
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ${{ defaults.app_name }}-redis
subjects:
  - kind: ServiceAccount
    name: ${{ defaults.app_name }}-redis
    namespace: ${{ SEALOS_NAMESPACE }}
```

</details>


When deploy databases, you only need to concern about the resources used by the database:

| Code        | Description             |
| ----------- | ----------------------- |
| `replicas`  | Number of instances     |
| `resources` | Allocate CPU and memory |
| `storage`   | Volume size             |

#### How to access database for applications

The database username/password is set into one secret for future usage. It can be added to the environment variables through the following code. After adding, you can read the MONGODB password in the container through $(MONGO_PASSWORD). 

```yaml
...
spec:
  containers:
    - name: ${{ defaults.app_name }}
      ...
      env:
        - name: MONGO_PASSWORD
          valueFrom:
            secretKeyRef:
              name: ${{ defaults.app_name }}-mongo-conn-credential
              key: password
...
```

#### TODO: `Minio`