# What is this
Working on official `getting started` and `training` content using microk8s.
I write the procedures and command histories so that I do not forget precise info. 

# Getting started

https://argoproj.github.io/argo-workflows/quick-start/ 

## Setup local k8s

I used microk8s for the local environment of kubernetes.
But I think there are many other options.

## Hello world

I was able to run the sample workflow by following the official website above.

Some notes not written in the official website:

It was necessary for operating argo CLI commands to add the following environment variables:

- ARGO_SERVER=localhost:2746

  - Also you have to forwarding Port 2746 to the port of argo server deployed on microk8s.

    - `microk8s kubectl -n argo port-forward deployment/argo-server 2746:2746`

- ARGO_INSECURE_SKIP_VERIFY=true

# Training

https://www.youtube.com/playlist?list=PLGHfqDpnXFXLHfeapfvtt9URtUF1geuBo 

## Workshop 101
The basic concepts and terminologies of Argo Workflow.
https://www.youtube.com/watch?v=XySJb-WmL3Q&list=PLGHfqDpnXFXLHfeapfvtt9URtUF1geuBo&index=2 

### Hands On: Step/DAG executor

#### Step executor
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: add-example-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: addStep
            template: add
            arguments:
              parameters:
                - name: "a"
                  value: "2"
                - name: "b"
                  value: "5"
        - - name: addStepFinal
            template: add
            arguments:
              parameters:
                - name: "a"
                  value: "{{steps.addStep.outputs.result}}"
                - name: "b"
                  value: "10"
        - - name: sayHello
            arguments:
                  parameters:
                    - name: "a"
                      value: "{{steps.addStepFinal.outputs.result}}"
            template: sayhello
            when: "{{steps.addStepFinal.outputs.result}} > 5"
    - name: add
      inputs:
        parameters:
          - name: "a"
          - name: "b"
      container:
        image: alpine:latest
        command: [sh, -c]
        args: ["echo $(( {{inputs.parameters.a}} + {{inputs.parameters.b}} ))"]
    - name: sayhello
      inputs:
          parameters:
            - name: "a"
      container:
        image: alpine:latest
        command: [sh, -c]
        args: ["echo 'Result is: ' $(( {{inputs.parameters.a}} ))"]
```

#### DAG Executor
About dag executor: https://argoproj.github.io/argo-workflows/walk-through/dag/ )
Note: In case of DAG Executor, it was necessary to fix the `when` condition calling sayHello template. I don’t know about this difference.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: add-dag-example-
spec:
  entrypoint: main
  templates:
    - name: main
      dag:
        tasks:
          - name: addTask
            template: add
            arguments:
              parameters:
                - name: "a"
                  value: "2"
                - name: "b"
                  value: "5"
          - name: addTaskFinal
            dependencies: [addTask]
            template: add
            arguments:
              parameters:
                - name: "a"
                  value: "{{tasks.addTask.outputs.result}}"
                - name: "b"
                  value: "10"
          - name: sayHello
            dependencies: [addTaskFinal]
            arguments:
                  parameters:
                    - name: "a"
                      value: "{{tasks.addTaskFinal.outputs.result}}"
            template: sayhello
            when: '{{tasks.addTaskFinal.outputs.result}} > 5'
    - name: add
      inputs:
        parameters:
          - name: "a"
          - name: "b"
      container:
        image: alpine:latest
        command: [sh, -c]
        args: ["echo $(( {{inputs.parameters.a}} + {{inputs.parameters.b}} ))"]
    - name: sayhello
      inputs:
          parameters:
            - name: "a"
      container:
        image: alpine:latest
        command: [sh, -c]
        args: ["echo 'Result is: ' $(( {{inputs.parameters.a}} ))"]
```

### Hands On: Artifact
references:
- Official minio setup procedure: https://argoproj.github.io/argo-workflows/configure-artifact-repository/#configuring-minio

#### Setup minio on microk8s
Note: this may be old method. 

```bash
$ microk8s enable dns
$ microk8s enable helm3
$ microk8s enable storage
$ microk8s helm3 repo add minio https://helm.min.io/ # official minio Helm charts
$ microk8s helm3 repo update
$ microk8s helm3 install argo-artifacts minio/minio --set service.type=LoadBalancer --set fullnameOverride=argo-artifacts
```

#### Create bucket on minio
First, you have to get keys neccesary for login.
```bash
$ ACCESS_KEY=$(microk8s kubectl get secret argo-artifacts --namespace default -o jsonpath="{.data.accesskey}" | base64 --decode)
$ SECRET_KEY=$(microk8s kubectl get secret argo-artifacts --namespace default -o jsonpath="{.data.secretkey}" | base64 --decode)
```
Then port-forwarding to argo-artifacts:9000
```bash
$ microk8s kubectl port-forward  svc/argo-artifacts 9000:9000
```
Open localhost:9000 in your browser.
You have to input ACCESS_KEY and SECRET_KEY here.
After you log in to minio, create bucket named `my-bucket`.

#### Add secret for access minio from workflow pods
First, you have to get base-64 encoded keys.
Be careful not to include newline character(-n option!).
```bash
$ echo $ACCESS_KEY -n | base64
$ echo $SECRET_KEY -n | base64
```
Then, create `minio.yml` as following.
```yml
apiVersion: v1
kind: Secret
metadata:
  name: my-argo-artifacts-cred
data:
  accesskey: <FILL IN YOUR base64 encoded ACCESS_KEY>
  secretkey: <FILL IN YOUR base64 encoded SECRET_KEY>
```
Apply this secret to your k8s cluster.
```bash
microk8s kubectl apply -f /path/to/minio.yml
```

#### Execute workflow sample: Output workflow (Slide P37)
Try creating new workflow as follow.
In this workflow, a txt file is uploded into artifact repository you created.
After running of the workflow, you can download file from minio UI (localhost:9000).
Note: This is a little bit modified from official training content
```yml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: output-artifact-s3-
spec:
  entrypoint: whalesay
  templates:
    - name: whalesay
      container:
        image: docker/whalesay:latest
        command: [sh, -c]
        args: ["cowsay hello world | tee /tmp/hello_world.txt"]
      outputs:
        artifacts:
          - name: message
            path: /tmp/hello_world.txt
            s3:
              bucket: my-bucket
              endpoint: argo-artifacts:9000
              insecure: true
              key: output/hello_world.txt
              accessKeySecret:
                name: my-argo-artifacts-cred
                key: accesskey
              secretKeySecret:
                name: my-argo-artifacts-cred
                key: secretkey
```
