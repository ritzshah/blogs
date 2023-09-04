<!DOCTYPE html>
<html>
<head>
	<title>Simple demonstration on how to pass information between tasks in Tekton/OpenShift Pipelines</title>
	<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/prism/1.24.1/themes/prism.min.css" integrity="sha512-9ZVv7w4z5QmOJyYv7j5yBjKz5J6QzQ4f5J5fz3K8vJ1Xc6fKgJ7qfQvJj9J7GfZ4JzJ1IbXvz1DwKz4Z+Jg3/A==" crossorigin="anonymous" />
	<style>
		body {
			font-family: Arial, sans-serif;
			margin: 0;
			padding: 0;
		}
		h1, h2, h3 {
			margin-top: 2rem;
			margin-bottom: 1rem;
		}
		pre {
			background-color: #f5f5f5;
			padding: 1rem;
			overflow-x: auto;
		}
		code {
			font-family: Consolas, monospace;
			font-size: 0.9rem;
		}
	</style>
</head>
<body>
	<h1>Simple demonstration on how to pass information between tasks in Tekton/OpenShift Pipelines</h1>
	<p>This document simplifies how you can pass information from one task to other task in a tekton pipeline. I has all the steps needed including yaml files for a test use case demonstrating simple way of passing information from one task to another in a tekton pipeline. It starts right from basis like creating a project and persistent volume creation (PVC) before you start creation of task and pipeline. Finally it gives you yaml files for respective task runs and pipeline run. You can extend this example for your use case.</p>
	<p>This example is tested with Red Hat OpenShift Pipelines version 1.70 and higher, running on OpenShift Version 4.10 and higher. Ensure that Red Hat OpenShift Pipelines operator is installed.</p>
	<h2>Create a Test Project</h2>
	<pre><code>kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: test</code></pre>
	<p>Before you create tasks and pipeline, ensure that you have a PVC associated with the project where you are creating tasks and pipeline.</p>
	<h2>Create a PVC Named Test</h2>
	<pre><code>kind: PersistentVolumeClaim
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
  volumeMode: Filesystem</code></pre>
	<p>In the first task, add a workspace like this in spec. First, have a workspace name 'source' in this case and use that in the 2nd task. Capture what you need to pass in data as results and redirect that information to a file for eg. 'ee_data.json' which you call in 2nd task.</p>
	<pre><code>apiVersion: tekton.dev/v1beta1
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
        printf "%s" ${AC_EE_ID}</code></pre>
	<p>In the next task, task 2, you can reference 'ee_data.json' as shown below:</p>
	<pre><code>apiVersion: tekton.dev/v1beta1
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
        printf "%s" ${AC_EE_ID}</code></pre>
	<p>When you run task1 and task2 in a pipeline, both should print the same output from both the tasks.</p>
	<h2>Create a Pipeline</h2>
	<pre><code>apiVersion: tekton.dev/v1beta1
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
            workspace: source</code></pre>
	<p>When you run the pipeline, both the tasks will show the same output as shown below:</p>
	<pre>STEP-TASK1 This is the output from task 1
STEP-TASK2 This is the output from task 1</pre>
	<h2>Task Runs and Pipeline Run</h2>
	<pre><code>apiVersion: tekton.dev/v1beta1
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
        claimName: test</code></pre>
	<p>This simple example provides you with a basic understanding of how you can pass output from one task to another, as input and you can extend this for your use case.</p>
</body>
</html>
