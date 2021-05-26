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

#### Using the TKN CLI
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
å