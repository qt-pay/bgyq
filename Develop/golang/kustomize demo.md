### kustomize

### 基础概念

base中存放不经常改变yaml对象

每个 `base` 或 `overlays` 中都必须要有一个 `kustomization.yaml`

Kustomize 允许用户以一个应用描述文件 （YAML 文件）为基础（Base YAML），然后通过 Overlay 的方式生成最终部署应用所需的描述文件。

 `kustomize build dir/` 都可以使用 `kubectl apply -k dir/` 实现，但是需要 `v14.0` 版以上的 `kubectl`



### kustomization.yaml:控制生成KRM

The kustomization file is a YAML specification of a Kubernetes Resource Model ([KRM](https://github.com/kubernetes/design-proposals-archive/blob/main/architecture/resource-management.md)) object called a *Kustomization*. A kustomization describes how to generate or transform other KRM objects.

 `kustomization.yaml` file is basically four lists:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- {pathOrUrl}
- ...

generators:
- {pathOrUrl}
- ...

transformers:
- {pathOrUrl}
- ...

validators:
- {pathOrUrl}
- ...
```

In all cases the `{pathOrUrl}` list entry can specify

- a file system path to a YAML *file* containing one or more KRM objects, or
- a *directory* (local or in a remote git repo) that contains a `kustomization.yaml` file.

### kustomize patch scope

#### patch all scope: 

This can be pretty useful if, for example, you want to do a new deployment even if the docker image specified in that deployment hasn't changed.

Running `kustomize build .` now would keep that line as-is.

如下，命令会导致该目录下所有的对象的annotation都会追加：deployment，service，ingress等

you *could* add an annotation programmatically, like this:

```bash
kustomize edit add annotation example.com/git_commit:$CI_COMMIT_SHORT_SHA
```

But then all your resources would be affected, mearning that your service and ingress would also be redeployed in that example. In some cases, you really want to scope your changes.

```bash
## 目录结构
$ tree
.
|-- base
|-- deployment.yaml
`-- kustomization.yaml

1 directory, 2 files

$ cat kustomization.yaml
resources:
- deployment.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

## 声明环境变量
$ export CI_COMMIT_SHORT_SHA="a"
$  kustomize edit add annotation example.com/git_commit:$CI_COMMIT_SHORT_SHA      
$ cat kustomization.yaml
resources:
- deployment.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
commonAnnotations:
  example.com/git_commit: a


## kustomize build生成 yaml
$ kustomize build ./testyaml/
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    example.com/git_commit: a
  labels:
    app: MYAPP
  name: MYAPP
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: MYAPP
  template:
    metadata:
      annotations:
        example.com/git_commit: a
    ...

```



#### patch only deployment scope

Each variable defined here must have a name and references to let Kustomize know where it's supposed to get that value. In that example, I'm using a `configMap`, which is often the best option to store configuration.

```bash
# kustomization.yaml
$ cat kustomization.yaml
resources:
- deployment.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configMapGenerator:
- behavior: create
  envs:
  - environment-properties.env
  name: environment-variables

configurations:
- env-var-transformer.yaml

vars:
- fieldref:
    fieldPath: data.CI_COMMIT_SHORT_SHA
  name: CI_COMMIT_SHORT_SHA
  objref:
    apiVersion: v1
    kind: ConfigMap
    name: environment-variables
generatorOptions:
  disableNameSuffixHash: true
  
## 记录要修改哪些变量 
$ cat environment-properties.env
CI_COMMIT_SHORT_SHA=789

## 指定修改范围
$ cat env-var-transformer.yaml
varReference:
- kind: Deployment
  path: spec/template/metadata/annotations



```

Finally, in our CI job, we can build that file:

```bash
echo CI_COMMIT_SHORT_SHA=$CI_COMMIT_SHORT_SHA > environment-properties.env
# And then, you just have to apply your changes:
kustomize build . |kubectl apply -f -
```

Now running `kustomize build .` would result in this :-)

```yaml
apiVersion: v1
data:
  CI_COMMIT_SHORT_SHA: "123456"
kind: ConfigMap
metadata:
  name: environment-variables
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    annotations:
      example.com/git-commit: "123456"
```

Tada!

### patch type

- patchesStrategicMerge：将所有的文件解析为Strategic Merge Patch

  ```yaml
  bases:
  - ../add
  patchesStrategicMerge:
  - remove-svc.yaml
  - remove-field.yaml
  ```

- patchesJSON6902：将所有的文件解析为Json Path

- patches：它会自动检测是patchesStrategicMerge还是patchesJSON6902 在使用这三个patch的时候，按道理来说是应该区别对待 ，但是第三个可以自动检测，我在生产环境中都是使用第三个patches，让它自动解析格式进行，不知道是否有坑，目前没有遇到过

### JsonPatches6902

There is an [RFC6902](https://tools.ietf.org/html/rfc6902) standard defines how to apply JSON patches. 

According to the standard, there are 6 types of operations that can be performed on an object: `add`, `remove`, `replace`, `move`, `copy`, `test`. We will take a look at the first three since they have a more applicable sense to our use case. From a syntactical point of view, each operation must have `op` member indicating one of the mentioned types, e.g. `add`:

```json
{ "op": "add" }
```

In addition to that, operations must include a `path` key. The value of the path is a location within the JSON document need to be modified and standardized by [RFC6901](https://tools.ietf.org/html/rfc6901):

```json
{ "op": "add", "path": "/a/b" }
```

The other key-value pairs of operation depend on the particular operation being performed.

`path`可以是文件，记录要替换的json数据。

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../base


patches:
- path: endpoint_patch.json
  target:
    version: v1
    kind: Endpoints

## endpoint_patch.json
[
  {"op": "replace",
    "path": "/subsets/0/addresses/0/ip",
    "value": "{{ VARS.K8S_VIP }}" }
]

```



### patch渲染Demo

首先，上传一个vm list清单（txt or excel），里面包含了各个组件的名称和IP。

然后通过前端js脚本解析vm list，生成一个全局的配置信息`global.yaml`or `global.json`

再通过js读取global config，渲染对应的kustomize yaml？？？

#### 问题场景

1. 大量变量需要人工配置管理
2. 部分变量值无法提前预知

#### 解决方案

基本思路:

1. 新建"全局变量表"，统一管理所有环境变量
2. 引入新的"渲染描述文件"，声明项目中需要进行替换文件及其使用的占位符

```yaml
## 渲染文件参考，即取哪个变量去哪个文件渲染
target:                                                 # 文件清单
    - file: test-base/overlays/coredns-cm.yaml          # 目标文件1
      vars:                                             # 占位符列表
        - AUTOPS_UCO_K8S_VIP: 127.0.0.1    			    # 占位符
        - AUTOPS_UCA_K8S_VIP: 127.0.0.1
        - AUTOPS_REGION: default

```

0.0

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - coredns-cm.yaml

## ?
patchesStrategicMerge:
  - exporter-deployment.yaml

#替换指定endpoint中IP address
patchesJson6902:
  - target:
      version: v1
      kind: Endpoints
      name: kafka-service
    ## 对应kafka service ip list
    patch: |-
      - op: replace
        path: /subsets/0/addresses/0/ip
        ## 这个VARS是什么
        ## {{ VARS. }}是占位符，会被后续js渲染替换
        value: "{{ VARS.KAFKA_IP.0 }}"
      - op: replace
        path: /subsets/0/addresses/1/ip
        value: "{{ VARS.KAFKA_IP.1 }}"
      - op: replace
        path: /subsets/0/addresses/2/ip
	    value: "{{ VARS.KAFKA_IP.2 }}"

## 渲染规则
generatorOptions:
  disableNameSuffixHash: true

secretGenerator:
# mysql secret
  - name: mysql-secret
    namespace: omc
    envs:
      - mysql-secret.env

configMapGenerator:
- name: example-configmap-1
  files:
  - application.properties
  
-------  

## kafka service yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-service
  namespace: default
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: kafka-service
    port: 9092
    protocol: TCP
    targetPort: 9092
  sessionAffinity: None
---
apiVersion: v1
kind: Endpoints
metadata:
  name: kafka-service
  namespace: default
subsets:
- addresses:
## 这里就是随便给个string？
  - ip: KAFKA_IP_1
  - ip: KAFKA_IP_2
  - ip: KAFKA_IP_3
  ports:
  - name: kafka-service
    port: 9092
    protocol: TCP


```





### 转载 kustomize 101

#### Your base

To start with Kustomize, you need to have your original yaml files describing any resources you want to deploy into your cluster. Those files will be stored for this example in the folder `./k8s/base/`.

Those files will NEVER (EVER) be touched, we will just apply customization above them to create new resources definitions

> **Note**: You can build base templates (e.g. for dev environment) at any point in time using the command kubectl apply -f ./k8s/base/.

In this example, we will work with a `service` and a `deployment` resources:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
    - name: http
      port: 8080
  selector:
    app: sl-demo-app
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  selector:
    matchLabels:
      app: sl-demo-app
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - name: app
        image: foo/bar:latest
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
```

We wil add a new file inside this folder, named `kustomization.yaml` :

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - service.yaml
  - deployment.yaml
```

This file will be the central point of your base and it describes the resources you use. Those resources are the path to the files relatively to the current file.

> **Note**: This `kustomization.yaml` file could lead to errors when running `kubectl apply -f ./k8s/base/`, you can either run it with the parameter `--validate=false` or simply not running the command against the whole folder

To apply your base template to your cluster, you just have to execute the following command:

```
$ kubectl apply -k k8s/base
```

To see what will be applied in your cluster, we will mainly use in this article the command `kustomize build` instead of `kubectl apply -k`.

The result of `kustomize build k8s/base` command will be the following, which is for now only the two files previously seen, concatenated:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  selector:
    matchLabels:
      app: sl-demo-app
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

#### **K**ustomization

Now, we want to kustomize our app for a specific case, for example, for our `prod` environement. In each step, we will see how to enhance our base with some modification.

The main goal of this article is not to cover the whole set of functionnalities of Kustomize but to be a standard example to show you the phiplosophy behind this tool.

First of all, we will create the folder `k8s/overlays/prod` with a `kustomization.yaml` inside it.

The `k8s/overlays/prod/kustomization.yaml` has the following content:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base
```

If we build it, we will see the same result as before when building the base.

```sh
$ kustomize build k8s/overlays/prod
```

This will output the following `yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  selector:
    matchLabels:
      app: sl-demo-app
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

We are now ready to apply kustomization for our `prod` env

#### Define Env variables for our deployment

In our `base`, we didn’t define any env variable. We will now add those `env` variables above our base. To do so, it’s very simple, we just have to create the chunk of yaml we would like to apply above our base and referece it inside the `kustomization.yaml`.

This file `custom-env.yaml` containing env variables will look like this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  template:
    spec:
      containers:
        - name: app # (1)
          env:
            - name: CUSTOM_ENV_VARIABLE
              value: Value defined by Kustomize ❤️
```

> **Note**: The `name` **(1)** key here is very important and allow Kustomize to find the right container which need to be modified.

You can see this `yaml` file isn’t valid by itself but it describes only the addition we would like to do on our previous base.

We just have to add this file to a specific entry in the `k8s/overlays/prod/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- custom-env.yaml
```

If we build this one, we will have the following result:

```sh
$ kustomize build k8s/overlays/prod
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  selector:
    matchLabels:
      app: sl-demo-app
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - env:
        - name: CUSTOM_ENV_VARIABLE # (1)
          value: Value defined by Kustomize ❤️
        image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

You can see our `env` block has been applied above our base and now the `CUSTOM_ENV_VARIABLE` (1) will be defined inside our deployment.yaml.

#### Change the number of replica

Like in our previous example, we will extend our base to define variables not already defined

> **Note**: You can also override some variables already present in your base files.

Here, we would like to add information about the number of replica. Like before, a chunk or `yaml` with just the extra info needed for defining replica will be enought:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  replicas: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
```

And like before, we add it to the list of `patchesStrategicMerge` in the `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- custom-env.yaml
- replica-and-rollout-strategy.yaml
```

The result of the command `kustomize build k8s/overlays/prod` give us the following result

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: sl-demo-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - env:
        - name: CUSTOM_ENV_VARIABLE
          value: Value defined by Kustomize ❤️
        image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

And you can see the replica number and rollingUpdate strategy have been applied above our base.

#### Use a secret define through command line

One of the things we often do is to set some variables as secret from command-line. In our case, we are doing this directly from our Gitlab-CI on [Gitlab.com](https://gitlab.com/).

But you can do this from anywhere else, the main purpose here is to define Kubernetes Secret without putting them inside Git 😱.

To do so, `kustomize` has a sub-command to edit a `kustomization.yaml` and create a secret for you. You just have to use it in your deployment like if it already exists.

```sh
$ cd k8s/overlays/prod
$ kustomize edit add secret sl-demo-app --from-literal=db-password=12345
```

These commands will modify your `kustomization.yaml` and add a `SecretGenerator` inside it.

> **Note**: You can also use secret comming from properties file (with `--from-file=file/path`) or from env file (with `--from-env-file=env/path.env`)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- custom-env.yaml
- replica-and-rollout-strategy.yaml

secretGenerator:
- literals:
  - db-password=12345
  name: sl-demo-app
  type: Opaque
```

If you run the `kustomize build k8s/overlays/prod` from the root folder of the example project, you will have the following output

```yaml
apiVersion: v1
data:
  db-password: MTIzNDU=
kind: Secret
metadata:
  name: sl-demo-app-6ft88t2625
type: Opaque
---
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: sl-demo-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - env:
        - name: CUSTOM_ENV_VARIABLE
          value: Value defined by Kustomize ❤️
        image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

> **Note**: The secret name is `sl-demo-app-6ft88t2625` instead of `sl-demo-app`, it’s normal and this is made to trigger a rolling update of the deployment if secrets content is changed.

If we want to use this secret from our deployment, we just have, like before, to add a new layer definition which uses the secret.

For example, this file will mount the `db-password` value as environement variables

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  template:
    spec:
      containers:
      - name: app
        env:
        - name: "DB_PASSWORD"
          valueFrom:
            secretKeyRef:
              name: sl-demo-app
              key: db.password
```

And, like before, we add this to the `k8s/overlays/prod/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- custom-env.yaml
- replica-and-rollout-strategy.yaml
- database-secret.yaml

secretGenerator:
- literals:
  - db-password=12345
  name: sl-demo-app
  type: Opaque
```

If we build the whole `prod` files, we now have

```yaml
apiVersion: v1
data:
  db-password: MTIzNDU=
kind: Secret
metadata:
  name: sl-demo-app-6ft88t2625
type: Opaque
---
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: sl-demo-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              key: db.password
              name: sl-demo-app-6ft88t2625 # (1)
        -  name: CUSTOM_ENV_VARIABLE
          value: Value defined by Kustomize ❤️
        image: foo/bar:latest
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

You can see the `secretKeyRef.name` used is automatically modified to follow the name defined by Kustomize (1)

> **Note**: Don’t forget, the command to put the secret inside the `kustomization.yaml` file should be made only from safe env and should not be commited.

The same logic exists with ConfigMap with hash at the end to allow redeployement of your app if ConfigMap changes.

#### Change the image of a deployment

Like for secret, there is a custom directive to allow changing of image or tag directly from the command line. This is very useful if you need to deploy the image previously tagged by your continuous build system.

To do that, you can use the following command:

```sh
$ cd k8s/overlays/prod
$ TAG_VERSION=3.4.5 # (1)
$ kustomize edit add secret sl-demo-app --from-literal=db-password=12345 # To create my required secret
$ kustomize edit set image foo/bar=foo/bar:$TAG_VERSION
```

> **Note**: the `TAG_VERSION` here is usualy defined by your CI/CD system

The `k8s/overlays/prod/kustomization.yaml` will be modified with those values:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- custom-env.yaml
- replica-and-rollout-strategy.yaml
- database-secret.yaml

secretGenerator:
- literals:
  - db-password=12345
  name: sl-demo-app
  type: Opaque

images:
- name: foo/bar
  newName: foo/bar
  newTag: 3.4.5
```

And if we build it, with the `kustomize build k8s/overlays/prod/` we have the following result:

```yaml
apiVersion: v1
data:
  db-password: MTIzNDU=
kind: Secret
metadata:
  name: sl-demo-app-6ft88t2625
type: Opaque
---
apiVersion: v1
kind: Service
metadata:
  name: sl-demo-app
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: sl-demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sl-demo-app
spec:
  replicas: 10
  selector:
    matchLabels:
      app: sl-demo-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: sl-demo-app
    spec:
      containers:
      - env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              key: db.password
              name: sl-demo-app-6ft88t2625
        - name: CUSTOM_ENV_VARIABLE
          value: Value defined by Kustomize ❤️
        image: foo/bar:3.4.5 # (1)
        name: app
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

You see the first `container.image` of the deployment have been modified to be run with the version `3.4.5` (1).