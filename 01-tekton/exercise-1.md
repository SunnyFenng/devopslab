# Tekton Hands-on Lab - from source code to production 

## 实验目标
- 了解Tekton pipelline的基本概念
- 创建一个pipeline来build和部署一个Knatvie的应用
- 执行一个pipeline并查看状态

## 基本概念

- PipelineResource 定义了输入 (例如一个 git repository) 或输出 (例如 docker image)供pipeline来使用.
- PipelineRun 定义了一个pipeline的执行. 它引用PipelineResources做为输入输出，引用Pipeline来执行，
- Pipeline 定义了一系列的Tasks.
- Task 定义了一系列的build steps例如编译代码, 执行测试, 构建和部署images。

![alt text](https://github.com/IBM/tekton-tutorial/blob/master/doc/source/images/crd.png)


## 实验准备   
### 1. 登录Online command Line tool   
1. 用浏览器打开https://workshop.shell.cloud.ibm.com/   
2. 点击右上角的Login按钮，使用您自己的ibm account登录。
3. 在右上角的下拉菜单中选择IBM。
4. 点击右上角的命令行图标，根据提示输入passcode: ikslab

### 2. 登录Kubecluster

### 3. 确保您已完成[安装Istio和Knative](https://github.com/daisy-ycguo/devopslab/blob/master/00-install/istio-knative-install.md)   

### 4. 确保您已完成[Tekton安装](https://github.com/daisy-ycguo/devopslab/blob/master/00-install/tekton-install.md)  

### 5. 安装container-registry CLI plug-in
`ibmcloud plugin install container-registry`

## 实验步骤
### 1. Fork tekton-tutorial 项目到您自己的repo并且clone到local workstation
1. 登录您的github account。如果您没有github account, 请先注册一个账号，可参考： https://github.com/daisy-ycguo/devopslab/blob/master/00-install/apply-github-account.md
2. 在浏览器中打开https://github.com/QiuJieLi/tekton-tutorial 点击Fork。
3. Fork成功后会跳转到您自己的github账户下的tekton-tutorial repo。点击"Clone or download",拷贝url。
4. 在您的本地目录克隆以上的repository    
`git clone https://github.com/<your git account>/tekton-tutorial.git`

### 2. 创建一个Task来build一个image并push到您的container registry。 
这个Task的文件在tekton/tasks/source-to-image.yaml。这个Task构建一个docker image并把它push到一个registry。   
```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: source-to-image
spec:
  inputs:
    resources:
      - name: git-source
        type: git
    params:
      - name: pathToContext
        description: The path to the build context, used by Kaniko - within the workspace
        default: .
      - name: pathToDockerFile
        description: The path to the dockerfile to build (relative to the context)
        default: Dockerfile
      - name: imageUrl
        description: Url of image repository
      - name: imageTag
        description: Tag to apply to the built image
        default: "latest"
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor
      command:
        - /kaniko/executor
      args:
        - --dockerfile=${inputs.params.pathToDockerFile}
        - --destination=${inputs.params.imageUrl}:${inputs.params.imageTag}
- --context=/workspace/git-source/${inputs.params.pathToContext}
```
说明：
- 一个Task可以包含一个或多个`Steps`。每个step定义了一个image用来执行这个step. 这个Task的步骤中使用了[kaniko](https://github.com/GoogleContainerTools/kaniko)项目来build source为一个docker image并把它push到一个registry。      
- 这个Task需要一个git类型的input resource,来定义souce的位置。这个git souce将被clone到本地的/workspace/git-source目录下。在Task中这个resource只是一个引用。后面我们将创建一个PipelineResources来定义真正的resouce资源。
- Task还使用了input parameters。这样做的好处是可以重用Task。   
- 后面我们会看到task是如何获得认证来puhs image到repository的。  

下面创建这个Task。   
`kubectl apply -f tekton/tasks/source-to-image.yaml`

### 3. 创建另一个Task来将image部署到Kubernetes cluster。   
这个Task的文件在tekton/tasks/deploy-using-kubectl.yaml。   
说明：   
这个Task有两个步骤。    
- 第一，在container里通过执行sed命令更新yaml文件来部署第1步时通过source-to-image Task创建出来image。   
- 第二，使用Lachlan Evenson的k8s-kubectl container image执行kubectl命令来apply上一步的yaml文件。   
后面我们会看到这个task是如何获得认证来apply这个yaml文件中的resouce的。   
```
....
  steps:
    - name: update-yaml
      image: alpine
      command: ["sed"]
      args:
        - "-i"
        - "-e"
        - "s;__IMAGE__;${inputs.params.imageUrl}:${inputs.params.imageTag};g"
        - "/workspace/git-source/${inputs.params.pathToYamlFile}"
    - name: run-kubectl
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "/workspace/git-source/${inputs.params.pathToYamlFile}"
```

下面创建这个Task。   
`kubectl apply -f tekton/tasks/deploy-using-kubectl.yaml`

### 4. 创建一个Pipeline来组合以上两个Task。   
这个Pipeline文件在tekton/pipeline/build-and-deploy-pipeline.yaml。
```
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: build-and-deploy-pipeline
spec:
  resources:
    - name: git-source
      type: git
  params:
    - name: pathToContext
      description: The path to the build context, used by Kaniko - within the workspace
      default: src
    - name: pathToYamlFile
      description: The path to the yaml file to deploy within the git source
    - name: imageUrl
      description: Url of image repository
    - name: imageTag
      description: Tag to apply to the built image
  tasks:
  - name: source-to-image
    taskRef:
      name: source-to-image
    params:
      - name: pathToContext
        value: "${params.pathToContext}"
      - name: imageUrl
        value: "${params.imageUrl}"
      - name: imageTag
        value: "${params.imageTag}"
    resources:
      inputs:
        - name: git-source
          resource: git-source
  - name: deploy-to-cluster
    taskRef:
      name: deploy-using-kubectl
    runAfter:
      - source-to-image
    params:
      - name: pathToYamlFile
        value:  "${params.pathToYamlFile}"
      - name: imageUrl
        value: "${params.imageUrl}"
      - name: imageTag
        value: "${params.imageTag}"
    resources:
      inputs:
        - name: git-source
resource: git-source
```    
说明：    
- Pipeline列出了需要执行的task，以及input output resources。    
- Pipeline还定义了每个task需要的input parameters。Task的input可以以多种方式进行定义，通过pipeline里的input parameter定义，或者直接设置，也可以使用task中的default值。在这个pipeline里，source-to-image task中的pathToContext parameter被暴露成为一个parameter 'pathToContext'，而source-to-image task中pathToDockerFile则使用task中的default值。      
- Task之间的顺序用runAfter关键字来定义。在这个例子中，deploy-using-kubectl task需要在source-to-image task之后执行。    

下面创建这个Pipeline。    
`kubectl apply -f tekton/pipeline/build-and-deploy-pipeline.yaml`

### 5. 创建PipelineRun和PipelineResources   
下面我们创建一个PipelineRun来指定input resource和parameters，并执行这个pipeline。     
#### 5.1 修改PipelineRun文件，替换`<REGISTRY>/<NAMESPACE>`为具体的值。PipelineRun文件：tekton/run/picalc-pipeline-run.yaml。      
1. 登录UI https://cloud.ibm.com/login，切换到**您自己的**ibm account(很重要!不要使用IBM account)。
2. 打开 https://cloud.ibm.com/iam/apikeys 页面， 点击“Create an IBM Cloud API key”按钮。
3. 输入一个名字，点击"Create"按钮。
4. Download API key，打开文件获取apikey。
5. 使用刚刚得到的API key登录 ibmcloud。
`ibmcloud login --apikey <YOURAPIKEY>`   
6. 登录您的私人container registry   
`ibmcloud cr login`   
7. 列出您的`namespace`   
`ibmcloud cr namespaces`   
8. 如果您还没有一个namespace,创建一个。   
`ibmcloud cr namespace-add <yourspacename>`
9. 执行以下命令获得registry，在以下例子中registry为us.icr.io。      
```
$ ibmcloud cr region
You are targeting region 'us-south', the registry is 'us.icr.io'.
```   
10. 将文件中的`<REGISTRY>`和`<NAMESPACE>`用以上的值代替。
```
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  generateName: picalc-pr-
spec:
  pipelineRef:
    name: build-and-deploy-pipeline
  resources:
    - name: git-source
      resourceRef:
        name: picalc-git
  params:
    - name: pathToYamlFile
      value: "knative/picalc.yaml"
    - name: imageUrl
      value: <REGISTRY>/<NAMESPACE>/picalc
    - name: imageTag
      value: "1.0"
  trigger:
    type: manual
  serviceAccount: pipeline-account
```
说明：
- PipelineRun没有一个固定的名字，每次执行的的时候会使用generateName的内容生成一个名字，例如‘picalc-pr-4jrtd’。这样做的好处是可以多次执行PipelineRun。   
- PipelineRun要执行的Pipeline由pipelineRef指定。   
- Pipeline暴露出来的parameters被指定了具体的值。   
- 关于Pipeline需要的resources，我们之后会定义一个名为picalc-git的PipelineResources。   
- 关于pipeline执行时所需要的认证信息，我们后面将会创建一个名为pipeline-account的service account。    

#### 5.2 创建Tekton PipelineResource
PipelineResource指向一个git source。这git source是一个计算圆周率的go程序。它包含了一个Dockerfile来测试，编译代码，build image。    
```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: picalc-git
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
value: https://github.com/IBM/tekton-tutorial
```
创建Pipelineresource。   
`kubectl apply -f tekton/resources/picalc-git.yaml`   

#### 5.3 创建service account
Service account让pipeline可以访问被保护的资源-您私人的container registry。在创建service account之前，我们先要创建一个secret,它含了对您的container registry进行操作所需要的认证信息。   
```
kubectl create secret docker-registry ibm-cr-push-secret --docker-server=<REGISTRY> --docker-username=iamapikey --docker-password=<YOURAPIKEY> --docker-email=me@here.com
``` 
其中`<YOURAPIKEY>`和`<REGISTRY>`的值，在实验步骤5.1中已经获得。      
现在可以创建service account了。Service account的文件在这里tekton/pipeline-account.yaml。   
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-account
secrets:
- name: ibm-cr-push-secret

---

apiVersion: v1
kind: Secret
metadata:
  name: kube-api-secret
  annotations:
    kubernetes.io/service-account.name: pipeline-account
type: kubernetes.io/service-account-token

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pipeline-role
rules:
- apiGroups: ["serving.knative.dev"]
  resources: ["services"]
  verbs: ["get", "create", "update", "patch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pipeline-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pipeline-role
subjects:
- kind: ServiceAccount
name: pipeline-account
```
`kubectl apply -f tekton/pipeline-account.yaml`   

说明：
这个yaml创建了以下资源：   
- 一个名为pipeline-account的ServiceAccount。在之前PipelineRun的定义中我们引用了这个serviceAccount。这个serviceAccount引用了我们之前创建的名为ibm-cr-push-secret的secret。这样就让pipeline获得了向你私人的container registry push image的认证。   
- 一个名为kube-api-secret的Secret,包含了用来访问Kubernetes API的认证信息信息，使得pipeline可以适用kubectl去操作您的kube cluster。   
- 一个名为pipeline-role的Role和一个名为pipeline-role-binding的RoleBinding，提供给pipeline基于resource的访问控制权限来创建和修改Knative services。  

下面为pipeline-account添加imagePullSecrets    
`kubectl patch sa pipeline-account -p '"imagePullSecrets": [{"name": "ibm-cr-push-secret" }]'`

### 6. 执行Pipeline  
1. 执行这个pipeline run。        
`kubectl create -f tekton/run/picalc-pipeline-run.yaml`   
PipelineRun没有一个固定的名字，每次执行的的时候会使用generateName的内容生成一个名字。kubectl会返回一个新生成的PipelineRun resource名字。   
`pipelinerun.tekton.dev/picalc-pr-rqzgp created`   
2. 检查taskruns的状态,直到的状态都是SUCCEEDED。
```
$ k get tr
NAME                                      SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
picalc-pr-wwcwh-deploy-to-cluster-d9tdj   True        Succeeded   9m19s       9m8s
picalc-pr-wwcwh-source-to-image-628bt     True        Succeeded   10m         9m19s
```
3. 如果看到以上结果，我们就可以查看部署好的Knative service了。READY状态为True说明部署成功了。    
```
$ kubectl get ksvc picalc
NAME     URL                                                             LATESTCREATED   LATESTREADY    READY   REASON
picalc   http://picalc-default.cdl-performance-3c3cell.us-south.containers.appdomain.cloud   picalc-zgqkq    picalc-zgqkq   True
```   
4. 最后，访问应用。
```
$ curl http://picalc-default.capacity-demo.us-south.containers.appdomain.cloud?iterations=20000000
3.1415926036
```

### 问题诊断
如果以上步骤遇到问题，尝试下面的方法进行诊断：   
1. 检查task run的状态。   
```
kubectl get taskrun
```
2. 如果有taskrun失败了，检查该taskrun的describe之后的‘Message’。   
```
kubectl describe taskrun picalc-pr-kpxbr-source-to-image-hbbw4
...
Message
"step-build-and-push" exited with code 1 (image: "gcr.io/kaniko-project/executor@sha256:9c40a04cf1bc9d886f7f000e0b7fa5300c31c89e2ad001e97eeeecdce9f07a29"); for logs run: kubectl -n default logs picalc-pr-7gr8f-source-to-image-sn7r8-pod-02e5da -c step-build-and-push
```
3. 按照提示检查pod的log获取失败的详细信息。   
```
$ kubectl -n default logs picalc-pr-fwdz7-source-to-image-6tvck-pod-9a97c6 -c step-build-and-push
error checking push permissions -- make sure you entered the correct tag name, and that you are authenticated correctly, and try again: checking push permission for "us.icr.io/sophy/picalc:1.0": DENIED: You have exceeded your storage quota. Delete one or more images, or review your storage quota and pricing plan. For more information, see https://ibm.biz/BdjFwL; [map[Action:pull Class: Name:sophy/picalc Type:repository] map[Action:push Class: Name:sophy/picalc Type:repository]]
```

4. 检查pipeline的状态
```
kubectl get pipelinerun
```
最后的Events应该是‘Succeeded’状态
```
kubectl describe pipelinerun picalc-pr-xxxx
...
Normal   Succeeded          5m43s                  pipeline-controller  All Tasks have completed executing
```

5. 检查service的状态
```
$ kubectl get ksvc
NAME     URL                                                                       LATESTCREATED   LATESTREADY    READY   REASON
picalc   http://picalc-default.capacity-demo.us-south.containers.appdomain.cloud   picalc-2d5z8    picalc-2d5z8   True
```

6. 如果READY不为True,describe ksvc查看Message提供的错误信息。
```
$ kubectl describe ksvc picalc
```