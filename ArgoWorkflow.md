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

### Answer of Hands On on Slide P32

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
