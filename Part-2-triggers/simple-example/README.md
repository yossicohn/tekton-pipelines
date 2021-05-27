# Part 1 Tekton Pipeline - Simple Example

This is an example of defining the proper resources for the Tekton Pipelines.
In this example you will have a single task **add-task** used twice in the pipeline.
In the=is example you will get to know the different entities in a Pipeline, though the example does not show all types, still it is a relaxed intro.

## Running the example
To run the example follow the follwing steps in which we will refernce the flow and different entities.

  Note, this pipeline is taken from the Tekton examples for the concept of Pipeline **results**
  [see here](https://github.com/tektoncd/pipeline/blob/main/examples/v1beta1/pipelineruns/pipelinerun-results.yaml)

### Steps:

- #### Step 1 - Create Cluster
use Kind/ Minikube for your localhost 

- #### Step 2 - Install TektonCD Pipelines
  Follow [these instructions](https://github.com/tektoncd/pipeline/blob/main/docs/install.md#installing-tekton-pipelines-on-kubernetes)

- #### Step 3 - Install(define) the Pipeline & Pipelinerun


  What we define here is several things, we will go from Top to Bottom:
  - Piplinerun is the running of the Pipeline
    The pipelinerun would define the external resources used by the Pipeline e.g. parameters, secrets, workspaces (and volumes) and other PiplineResources.

```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: sum-three-pipeline-run
spec:
  pipelineRef:
    name: sum-three-pipeline
  params:
    - name: first
      value: "2"
    - name: second
      value: "10"
    - name: third
      value: "10"
  ```  
  Here we see that the Pipelinerun is running a pipeline named *sum-three-pipeline* and is supplying 3 parameters

  Pipeline would define the Tasks invloved in the Pipeline and all the usage of the resources by the tasks.

  ```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: sum-three-pipeline
spec:
  params:
    - name: first
      description: the first operand
    - name: second
      description: the second operand
    - name: third
      description: the third operand
  tasks:
    - name: first-add
      taskRef:
        name: add-task
      params:
        - name: first
          value: $(params.first)
        - name: second
          value: $(params.second)
    - name: second-add
      taskRef:
        name: add-task
      params:
        - name: first
          value: $(tasks.first-add.results.sum)
        - name: second
          value: $(params.third)
  results:
    - name: sum
      description: the sum of all three operands
      value: $(tasks.second-add.results.sum)
    - name: partial-sum
      description: the sum of first two operands
      value: $(tasks.first-add.results.sum)
    - name: all-sum
      description: the sum of everything
      value: $(tasks.second-add.results.sum)-$(tasks.first-add.results.sum)

  ```

Here we see that the Pipeline is composed of 2 Tasks *first-add* snd *second-add*.
These 2 Tasks are using the same defined task *add-task* but with different parameters.
More over we can see the defined results of the Pipline that are connected to the created results by the Tasks.
Note the way we write to results
```results.<param-name>.path``` 


and the way we reference results
```tasks.<task-name>results.<param-name>.path``` for example 
```
$(tasks.first-add.results.sum)
```

 - #### Step 4 - Install(define) the Task

The Task is the basic building block taht is composed of steps.
Each Task would be manifested by a POD while each step would be a container in the POD.
In this task we see that it gets 2 parameters and it writes a result sum ```echo -n $((${OP1}+${OP2})) | tee $(results.sum.path)```

This task is then used in the Pipeline via 2 tasks
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: add-task
spec:
  params:
    - name: first
      description: the first operand
    - name: second
      description: the second operand
  results:
    - name: sum
      description: the sum of the first and second operand
  steps:
    - name: add
      image: alpine
      env:
        - name: OP1
          value: $(params.first)
        - name: OP2
          value: $(params.second)
      command: ["/bin/sh", "-c"]
      args:
        - echo -n $((${OP1}+${OP2})) | tee $(results.sum.path);

```

 - #### Step 5 - running the Pipeline
  Note, when we run the *Pipeline* via *Pipelinerun*, each task is manifetsed via *taskrun*, which is a resource that supplies the resources to the *tasks* in the same way *pipelinerun* supplies resources to *pipeline*.

  To run the pipeline we need to install the resources to kubernetes:
  ```
  kubectl apply -f ./Part-1-pipelines/simple-example/pipeline.yaml
  ```
Now, to execute the Pipelinerun
```
Part-1-pipelines/simple-example/pipelinerun.yaml
```


### Install Tekton CLI tkn

[install from here](https://github.com/tektoncd/cli)

```
➜  tekton-pipelines git:(main) ✗ cd Part-1-pipelines/simple-example
➜  simple-example git:(main) ✗ kubectl apply -f pipeline.yaml
task.tekton.dev/add-task created
pipeline.tekton.dev/sum-three-pipeline created
➜  simple-example git:(main) ✗ tkn p ls
NAME                 AGE             LAST RUN   STARTED   DURATION   STATUS
sum-three-pipeline   6 seconds ago   ---        ---       ---        ---
➜  simple-example git:(main) ✗ kubectl apply -f pipelinerun.yaml
pipelinerun.tekton.dev/sum-three-pipeline-run created
➜  simple-example git:(main) ✗ tkn pr ls
NAME                     STARTED         DURATION   STATUS
sum-three-pipeline-run   8 seconds ago   ---        Running
➜  simple-example git:(main) ✗ tkn pr logs sum-three-pipeline-run
Pipeline still running ...
[first-add : add] 12

➜  simple-example git:(main) ✗ tkn pr ls
NAME                     STARTED          DURATION     STATUS
sum-three-pipeline-run   56 seconds ago   22 seconds   Succeeded
```

A very usefull tool is the describe, which shows the resoirces used and the flow
```
➜  simple-example git:(main) ✗ tkn p ls
NAME                 AGE              LAST RUN                 STARTED          DURATION     STATUS
sum-three-pipeline   21 minutes ago   sum-three-pipeline-run   21 minutes ago   22 seconds   Succeeded
➜  simple-example git:(main) ✗ tkn p describe sum-three-pipeline
Name:        sum-three-pipeline
Namespace:   default

📦 Resources

 No resources

⚓ Params

 NAME       TYPE     DESCRIPTION          DEFAULT VALUE
 ∙ first    string   the first operand    ---
 ∙ second   string   the second operand   ---
 ∙ third    string   the third operand    ---

📝 Results

 NAME            DESCRIPTION
 ∙ sum           the sum of all thre...
 ∙ partial-sum   the sum of first tw...
 ∙ all-sum       the sum of everythi...

📂 Workspaces

 No workspaces

🗒  Tasks

 NAME           TASKREF    RUNAFTER   TIMEOUT   CONDITIONS   PARAMS
 ∙ first-add    add-task              ---       ---          first: string, second: string
 ∙ second-add   add-task              ---       ---          first: , second: string

⛩  PipelineRuns

 NAME                       STARTED          DURATION     STATUS
 ∙ sum-three-pipeline-run   21 minutes ago   22 seconds   Succeeded
```


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

# Part 2 Tekton Triggers - Simple Example

The Tekton Triggers documentation is vast and you might get lost there.
This example would be very simple, so just describe the concept and the main entities.

There are 3 main entities in the Triigers flow:
1. TriggerTemplate
2. TriggerBinding
3. EventListener

These 3 Resources define the trigger, meaning:
1.  what is executed by the trigger - this is defined by the **TriggerTemplate**, which defines a template for the Pipeline created when the trigger is activated.
2.  What data is scraped from the Trigger Request call and how it is mapped to the Template, this is defined by the **TriggerBinding**
3.  What is the event listener created to be called as the trigger endpoint, this is done by the **EventListener**
   
   
   ---
   ---

Let's look on the **TriggerTemplate**
```
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerTemplate
metadata:
  name: sum-three-pipeline-template
  namespace: getting-started
spec:
  params:
  - name: first
    description: first.
  - name: second
    description: second.
  - name: third
    description: third.
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: sum-three-pipeline-run-
        namespace: getting-started
      spec:
        pipelineRef:
          name: sum-three-pipeline
        params:
        - name: first
          value: "1"
        - name: second
          value: "2"
        - name: third
          value: "3"
```

We can see that the spec defines 3 parameters (first, second, third) which will be used in the template.

The  ```resourcetemplates``` tag defines the template of the resource to be created, which in this case is ```PipelineRun``` resource.
The ```PipelineRun``` resource is referencing the existing pipeline resource.

---
**TriggerBinding**
```
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: sum-three-pipeline-binding
spec:
  params:
  - name: first
    value: "1"
  - name: second
    value: "2"
  - name: third
    value: "3"
```

This is a simple case of TriggerBinding, in a real triiger binding we would reference the request body and extract the data.
In this case we have constant values (1, 2, 3)

a real example for exatrction of data from the request would look like this:
```
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: pipeline-binding
spec:
  params:
  - name: gitrevision
    value: $(body.head_commit.id)
  - name: gitrepositoryurl
    value: $(body.repository.url)
  - name: contenttype
    value: $(header.Content-Type)
```
---
**EventListener**

```
apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: getting-started-listener
  namespace: getting-started
spec:
  serviceAccountName: tekton-triggers-example-sa
  triggers:
    - bindings:
      - ref: sum-three-pipeline-binding
      template:
        ref: sum-three-pipeline-template
```

The EventrListener stich the TriggerBinding and the TriggerTemplate together as a trigger.

Notice that the ```serviceAccount``` *tekton-triggers-example-sa* is used.
This Service account get's man y credentials (defined in the rbac.yaml)


## The credentials are needed for :
### EventListeners need to be able to fetch all namespaced resources
```
- apiGroups: ["triggers.tekton.dev"]
  resources: ["eventlisteners", "triggerbindings", "triggertemplates", "triggers"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
# configmaps is needed for updating logging config
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
```
### Permissions to create resources in associated TriggerTemplates

```
- apiGroups: ["tekton.dev"]
  resources: ["pipelineruns", "pipelineresources", "taskruns"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["serviceaccounts"]
  verbs: ["impersonate"]
- apiGroups: ["policy"]
  resources: ["podsecuritypolicies"]
  resourceNames: ["tekton-triggers"]
  verbs: ["use"]
```

 ### EventListeners need to be able to fetch any clustertriggerbindings
 ```
- apiGroups: ["triggers.tekton.dev"]
  resources: ["clustertriggerbindings", "clusterinterceptors"]
  verbs: ["get", "list", "watch"]
```

After we install all the new trigger resources, we can continue for exposing the trigger to the world.

```
kubectl apply -f Part-2-triggers/simple-example/rbac.yaml
kubectl apply -f Part-2-triggers/simple-example/trigger-template.yaml
kubectl apply -f Part-2-triggers/simple-example/triggerbinding.yaml
kubectl apply -f Part-2-triggers/simple-example/eventlistener.yaml
```
We now would see 2 new ClusterIP services:
```
tekton-pipelines git:(main) ✗ kubectl get svc -n getting-started
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
el-getting-started-listener   ClusterIP   172.20.153.238   <none>        8080/TCP   26h
event-display                 ClusterIP   172.20.141.11    <none>        8080/TCP   29h
```

and new pods
```
 tekton-pipelines git:(main) ✗ kubectl get pods -n getting-started
NAME                                                      READY   STATUS      RESTARTS   AGE
el-getting-started-listener-64586f7fcf-8v27q              1/1     Running     5          26h
event-display                                             1/1     Running     0          29h
```

Now we need to expose that to the world.
This can be done by NginxIngress or Kong or other Gateway.

I will use Kong, install via helm 

```
helm repo add kong https://charts.konghq.com
helm repo update
```

Create a namespace
```
kubectl create ns kong
```

Install helm chart
```
helm install kong kong/kong  --set ingressController.installCRDs=false  -nkong
```

Now, let;s define ingress resource

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: el-getting-started-listener-ingress
  annotations:
    kubernetes.io/ingress.class: kong

spec:
  rules:
  - host: trigger-example.com
    http:
      paths:
      - path: /*
        backend:
          serviceName: el-getting-started-listener
          servicePort: 8080
```

So, assuming I have a dns ```trigger-example.com```.
Any call from that dns now would be routed to the service ```el-getting-started-listener```



Now a simple curl should activate our trigger:
```
curl -i  -H 'Content-Type: application/json'  -d '{"nothing":"nothing"}'  http://trigger-example.com
```


```
tkn pr ls

 tekton-pipelines git:(main) ✗ tkn pr ls
NAME                           STARTED          DURATION     STATUS
sum-three-pipeline-run-kk8gv   1 minute ago     8 seconds    Succeeded
```

---
---

# Using the TKN CLI
get the tasks
```
tkn tasks list
or
tkn t ls 
```

delete tasks
```
tkn tasks delete <taskname>
tkn t delete <taskname> 
```

describe tasks
```
tkn tasks describe <taskname>
tkn t describe <taskname> 
```

tasks logs
```
tkn task logs -f <task name>
or
tkn t logs -f <task name>
```


Get the Taskrun
```
tkn taskrun list
or
tkn tr ls
```

delete taskrun
```
tkn taskrun delete <taskrun name>
or
tkn tr delete <taskrun name>
```

Get the Pipeline
```
tkn pipeline list
or
tkn p ls
```

delete Pipeline
```
tkn pipeline delete <Pipeline name>
or
tkn p delete <Pipeline name>  
```


Pipeline Logs
```
tkn pipeline logs -f <pipline name>
or
tkn p logs -f <pipline name>
```

describe pipline
```
tkn pipline describe <pipline>
tkn p describe <pipline> 
```



# Security for Web hook


## Create the Secret:
```
k create secret generic github-secret --from-literal=secretToken=<token> --dry-run=client -oyaml
```

which will give:
```
apiVersion: v1
data:
  secretToken: <base64 token>
kind: Secret
metadata:
  creationTimestamp: null
  name: github-secret
```

Now we need to patch oue ServiceAccount with the change:

```
kubectl patch serviceaccount tekton-triggers-example-sa  -p '{"secrets": [{"name": "github-secret"}]}'
```

Hey5
