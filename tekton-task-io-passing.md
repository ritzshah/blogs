
This example is tested with Red Hat OpenShift Pipelines version 1.70 and higher, running on OpenShift Version 4.10 and higher. Ensure that Red Hat OpenShift Pipelines operator is installed.
Create a Test Project

yaml
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: test

Before you create tasks and pipeline, ensure that you have a PVC associated with the project where you are creating tasks and pipeline.
Create a PVC Named Test

yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test
  namespace: test
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: gp2
  volumeMode: Filesystem

In the first task, add a workspace like this in spec. First, have a workspace name 'source' in this case and use that in the 2nd task. Capture what you need to pass in data as results and redirect that information to a file for eg. 'ee_data.json' which you call in 2nd task.
yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: task1
spec:
  description: >-
    Add execution environment to automation controller
  workspaces:
    - name: source
      results:
        - name: data
          description: ID to be passed to next task
  steps:
    - name: task1
      image: quay.io/rshah/jq
      workingDir: $(workspaces.source.path)
      resources: {}
      script: |
        #!/usr/bin/env bash
        data="This is the output from task 1"
        printf "%s" "${data}" > ee_data.json
        AC_EE_ID=$(cat ee_data.json)
        printf "%s" ${AC_EE_ID}

In the next task, task 2, you can reference 'ee_data.json' as shown below:
yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: task2
spec:
  workspaces:
    - name: source
  steps:
    - name: task2
      image: quay.io/rshah/jq
      workingDir: $(workspaces.source.path)
      resources: {}
      script: |
        #!/usr/bin/env bash
        AC_EE_ID=$(cat ee_data.json)
        printf "%s" ${AC_EE_ID}

When you run task1 and task2 in a pipeline, both should print the same output from both the tasks.
Create a Pipeline

yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: "value_pass_pipeline"
spec:
  workspaces:
    - name: source
  params:
    - description: Verify the TLS on the registry endpoint (for push/pull to a non-TLS registry)
      name: TLSVERIFY
      type: string
      default: "false"
    - description: Dummy parameter for task1
      name: task1
      type: string
      default: "task1"
    - description: Dummy parameter for task2
      name: task2
      type: string
      default: "task2"
  tasks:
    - name: task1
      taskRef:
        kind: Task
        name: task1
      params:
        workspaces:
          - name: source
            workspace: source
    - name: task2
      taskRef:
        kind: Task
        name: task2
      params:
        runAfter:
          - task1
        workspaces:
          - name: source
            workspace: source

When you run the pipeline, both the tasks will show the same output as shown below:
STEP-TASK1 This is the output from task 1
STEP-TASK2 This is the output from task 1

Task Runs and Pipeline Run

yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: test-0ij91k-task1
  namespace: test
spec:
  resources: {}
  serviceAccountName: pipeline
  taskRef:
    kind: Task
    name: task1
  timeout: 59m59.989014151s
  workspaces:
    - name: source
      persistentVolumeClaim:
        claimName: test
---
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: test-0ij91k-task2
  namespace: test
spec:
  resources: {}
  serviceAccountName: pipeline
  taskRef:
    kind: Task
    name: task2
  timeout: 59m59.989014151s
  workspaces:
    - name: source
      persistentVolumeClaim:
        claimName: test
---
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: test-0ij91k
  namespace: test
spec:
  pipelineRef:
    name: test
  serviceAccountName: pipeline
  timeout: 1h0m0s
  workspaces:
    - name: source
      persistentVolumeClaim:
        claimName: test

This simple example provides you with a basic understanding of how you can pass output from one task to another, as input and you can extend this for your use case.
