# Part 1 - Tekton Pipelines
This repo is an example for Tekton Piplines, it intended to bridge the gap of the Tekton documentation and the Tekton newbie.

The repo has 2 folders Part-1 and Part-2.

Part-1 deals with the Tekton Pipelines.
over ```Part-1-pipelines/simple-example``` it shows a very simple examnple.
It would be better for newbie to start here.

After you understand all the concepts you can continue to the Real example that is composed of all the resources here(discarding the simple-example)
```
├── README.md
├── configmap
│   └── aws-ecr-docker-config.json
├── pipeline
│   ├── pipeline.yaml
│   └── pipelinerun.yaml
├── resources
│   ├── git-resource.yaml
│   └── image-resource.yaml
├── secrets
│   ├── aws-cred.yaml
│   └── git-ssh-secret-file.yaml
├── serviceaccounts
│   └── serviceaccount.yaml
└── tasks
    ├── task-build-docker-image-from-git-source.yaml
    └── task-git-source.yaml
```    
    
    
# Tekton Getting started Image Build Pipeline with Private Github Repo

This is an example of defining the proper resources for the Tekton Pipelines.

### Steps:
- #### Step 1 - Create Cluster
use Kind/ Minikube for your localhost 

- #### Step 2 - Install TektonCD Pipelines
Follow [this instructions](https://github.com/tektoncd/pipeline/blob/main/docs/install.md#installing-tekton-pipelines-on-kubernetes)

- #### Step 3 - Create AWS Credentials

These should be the minimum needed credentials

We will create a secret named aws-creds, which will contain the aws config and the aws credentials.
These are all exist in your path:
```~/.aws```

```
kubectl create secret generic aws-creds --from-file=config=/Users/<username>/.aws/config --from-file=credentials=/Users/<username>/.aws/credentials-ecr
```
- #### Step 4 - Create Git SSH Credentials
We will need these credentials for the pull of the Git repo to be used as the build context.

#### Create SSH-Secret git credentials for tekton

The creation of a yaml file can be done via the ```--dry-run -o yaml```.
Note, we should add the proper annotation for the ```ssh-auth``` and the ```tekton.dev/git-0```


```
kubectl create secret generic ssh-key-secret --from-file=ssh-privatekey=/Users/<username>/.ssh/id_rsa  --from-file=known_hosts=/Users/<username>/.ssh/known_hosts --from-file=config=/Users/<username>/.ssh/config  --dry-run=client  -o yaml > git-ssh-secret-file.yaml
```

adding the annotation of the ssh git secret
```
type: kubernetes.io/ssh-auth
``` 
adding the annotaion of the tekton git for github.com
```
annotations:
    tekton.dev/git-0: github.com # Described below
```
see [here for annotation](https://github.com/tektoncd/pipeline/blob/main/docs/auth.md#configuring-ssh-auth-authentication-for-git)


getting this kind of file :

```
apiVersion: v1
type: kubernetes.io/ssh-auth
data:
  config: xxx=
  known_hosts: xxx=
  ssh-privatekey: xxx=
kind: Secret
metadata:
  annotations:
    tekton.dev/git-0: github.com # Described below
  creationTimestamp: null
  name: ssh-key-secret

```

**These annotation are crucial for the work of the git-clone task in accessing private repo for git fetch.**

- #### Step 5 - Create Git Input PipelineResources
  We should create the Git PipelineResource as Input Resource.
  The Git PipelineResource would pull the Git Respository using the ssh-auth secrets for the git credentiasl

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: fetch-git-repo
spec:
  type: git
  params:
    - name: revision
      value: test-kaniko
    - name: url
      value: git@github.com:yossicohn/go-api-skeleton.git
      #note: since I will be using the SSH-auth the url is used as ssh url format. 
      #configure: change if you want to build something else, perhaps from your own local git repository.

```

  - #### Step 6 - Create Image Output PipelineResources
  We should create Image PipelineResource as Output PipelineResource.
  This Resource would be used to define the output image created by the kaniko build.

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: go-app-image
spec:
  type: image
  params:
    - name: url
      value: <account-id>.dkr.ecr.us-east-1.amazonaws.com/app-go:v2
  #configure: replace with where the image should go: perhaps your local registry or Dockerhub with a secret and configured service account

```

  - #### Step 7 - Create ServiceAccount with secrets
  To enable the mount of the ssh-auth secret, we can use the ServiceAccount binded with the Secret

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: git-serviceaccount
secrets:
  - name: ssh-key-secret

```
 - #### Step 8 - Create Config value for the AWS ECR Credentials
  AWS ECR Credentials is using the .aws credentials, with ecr config.

```
{
    "credsStore": "ecr-login"
}
```
This tels the Kaniko to use the AWS credentials for the Image Registry.

This would be done via:
```
kubectl create configmap docker-config --from-file=config.json=configmap/aws-ecr-docker-config.json
```
meaning use the content of the file aws-ecr-docker-config.json and map it with a key config.json.



 - #### Step 9 - Build The Tekton Task

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-docker-image-from-git-source
spec:
  volumes:
    <!-- - name: kaniko-secret
      secret:
        secretName: regcred
        items:
        - key: .dockerconfigjson
          path: config.json -->
    - name: aws-creds
      secret:
        secretName: aws-creds
    - name: docker-configmap
      configMap:
        name: docker-config
  
  workspaces:
  - name: shared-data
  params:
    - name: pathToDockerFile
      type: string
      description: The path to the dockerfile to build
      default: $(resources.inputs.docker-source.path)/Dockerfile
    - name: pathToContext
      type: string
      description: |
        The build context used by Kaniko
        (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
      default: $(resources.inputs.docker-source.path)
  resources:
    inputs:
      - name: docker-source
        type: git
        targetPath: shared-data
    outputs:
      - name: builtImage
        type: image
        targetPath: shared-data
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:latest
      # image: alpine
      # image: amazon/aws-cli
      # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
      env:
        - name: "DOCKER_CONFIG"
          value: "/kaniko/.docker/"
      # command:
      #   - /bin/sh
      # args: ['-c', 'sleep 1h']
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(params.pathToDockerFile)
        - --destination=$(resources.outputs.builtImage.url)
        - --context=$(params.pathToContext)
        - --cache=true
        - --cache-repo=<account-id>.dkr.ecr.us-east-1.amazonaws.com/app-go
        - --cleanup
      
      volumeMounts:
        - name:  aws-creds
          mountPath:  /tekton/home/.aws
        - name: docker-configmap
          mountPath: /kaniko/.docker/
          
```
#### Task Notes:
**Mounting Secrets to Volume**
```
    - name: aws-creds
      secret:
        secretName: aws-creds
    - name: docker-configmap
      configMap:
        name: docker-config
```
we mount the aws credentials to volume aws-creds
we mount the config docker-config to volume docker-configmap

The Volume reference is done in the Task Spec, aboce the Task Steps.

Volume Mounting is done in the STask Steps
```
volumeMounts:
  - name:  aws-creds
    mountPath:  /tekton/home/.aws
  - name: docker-configmap
    mountPath: /kaniko/.docker/

```

Note, the Tekton user is ```/tekton/home``` hence the mounting of the aws-creds secrets is at ```/tekton/home/.aws```

The Docker config is set for Kaniko at the folder ```/kaniko/.docker/```, and we use the env variable DOCKER_CONFIG to update 
```
 env:
  - name: "DOCKER_CONFIG"
    value: "/kaniko/.docker/"
```

### Using DockerHub Registry
To use DockerHub Registry all you need is to change the Image OipelineResource to point on the DocxkerHub Registry URL.
And change the authentication.

Create authentication via k8s secret of type ```docker-registry```
```
kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=<username> --docker-password=<password> --docker-email=<user@email.com>
```
what is created is the resource:

```
apiVersion: v1
data:
  .dockerconfigjson: xxx=
kind: Secret
metadata:
  creationTimestamp: null
  name: regcred
type: kubernetes.io/dockerconfigjson
```
Now all we need to do is to add it to mount it for kaniko over /kaniko/.docker/config.js.

This is done first by mounting the secret to volume:

```
volumes:
    - name: kaniko-secret
      secret:
        secretName: regcred
        items:
        - key: .dockerconfigjson
          path: config.json
```

Then mount the volume to path :

```
      volumeMounts:
        - name: docker-configmap
          mountPath: /kaniko/.docker/
```



Note, regarding the tasks.
We could use git-clone from the catalog instead of ```PipelineResource```.
There are several ways to do the same thing.
In this case I used the ```PipelineResource``` and also used an additional task to define the resulted Image tag.
In this case getting the SHA1 of the last git commit.
We could also define a different strategy and this task could be the place for it.
More over we example a result usage in the pipeline.


## Tekton

### Install Tekton resources
[install from here](https://github.com/tektoncd/pipeline/blob/main/docs/install.md#installing-tekton-pipelines-on-kubernetes)

```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml

```


### Install Tekton CLI tkn

[install from here](https://github.com/tektoncd/cli)

### Install Tekton dashboard

[Install from here](https://github.com/tektoncd/dashboard/blob/main/docs/install.md)
```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/tekton-dashboard-release.yaml
```

Enter the Dashboard via:

```
kubectl proxy
http://localhost:8001/api/v1/namespaces/tekton-pipelines/services/tekton-dashboard:http/proxy/
```


Running the example

defining the Task
```
kubectl apply -f build-docker-image-from-git-source.yaml
```

running the task via Taskrun
```
kubectl apply -f taskrun.yaml
```
Note, 
The taskrun in this ecam ple include the service account and the definition for the workspace volume


### Using the TKN CLI
get the tasks
```
tkn tasks list
```

delete tasks
```
tkn tasks delete <taskname>
```

Get the Taskrun
```
tkn taskrun list
```

delete taskrun
```
tkn taskrun delete <taskrun name>
```
Get the taskriun logs

```
tkn taskrun logs -f <taskrun name>
```
