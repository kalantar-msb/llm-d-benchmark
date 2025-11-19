# Benchmarking with tektonc

Related folders are `tekton`, `tektonc` and `tektoncsample`

## Tekton Basics

A **Pipeline** is set of **Tasks**. Tasks run in parallel. The execution flow can be controlled implicitly (via one task consume a result of another) or explcitly with mechanisms like `runAfter`, `when` and `finally`.
A **Task** is a sequence of **Steps**. Steps run sequentially. The step can programmatically determine to execute or skip.

To execute a **Pipeline** create a **PipelineRun**, 
an object that identifies:
 - the Pipeline to execute and 
 - the values of any parameters

Tekton creates a **TaskRun** for each Task in the Pipeline.
A TaskRun is an object that identifies: 
 - the Task and 
 - the values of any parameters (passed from the PipelineRun)

The TaskRun is implemented by a Pod
Each Step is implemented by a Container in the Pod.

## Usage

### Requirements

1. HF token
2. s3 bucket and necessary keys for uploading results
3. tekton CLI (`tkn`)

### Setup

1. Create a namespace where the Tekton pipeline will execute.
    ```shell
    export $NAMESPACE=your_namespace
    ```
    ```shell
    kubectl create ns $NAMESPACE
    ```
    or
    ```shell
    oc new-project $NAMESPACE
    ```

    For convenience, set the current context:
    ```shell
    kubectl config set-context --current --namespace $NAMESPACE
    ```

2. Create a secret `hf-secret` containing your HuggingFace token in the namespace.
    ```shell
    kubectl create secret generic hf-secret \
        --namespace ${NAMESPACE} \
        --from-literal="HF_TOKEN=${HF_TOKEN}" \
        --dry-run=client -o yaml | kubectl apply -f -
    ```

3. Create a secret containing your s3 credentials `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

    ```shell
    kubectl create secret generic s3-secret \
        --namespace ${NAMESPACE} \
        --from-literal="AWS_ACCESS_KEY_ID=${HF_TOKEN}" \
        --from-literal="AWS_SECRET_ACCESS_KEY=${HF_TOKEN}" \
        --dry-run=client -o yaml | kubectl apply -f -
    ```

4. Give the tasks needed permissions
    ```shell
    envsubst '$NAMESPACE' < tekton/roles.yaml | kubectl apply -f -
    ```

    ```shell
    oc adm policy add-scc-to-user anyuid -z default -n $NAMESPACE
    ```

5. Create RWX PVC `model-pvc` (300Gi) and `data-pvc` (20Gi) for storing models and execution results, respectively. This PVC is shared between all tasks.  For example:
    ```shell
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
        name: ${PVC_NAME}
        namespace: ${NAMESPACE}
    spec:
        accessModes:
        - ReadWriteMany
        resources:
            requests:
                storage: ${PVC_SIZE}
        storageClassName: ocs-storagecluster-cephfs
        volumeMode: Filesystem
    EOF
    ```
5. Install `tkn` cli:

    ```shell
    brew install tektoncd-cli
    ```


### Running a pipeline

1. Deploy the steps and tasks:

    ```shell
    for step in tekton/steps/*.yaml; do
        kubectl apply -f tekton/steps/$step
    done
    for task in tekton/tasks/*.yaml; do
        kubectl apply -f tekton/tasks/$task
    done
    ```

2. Build and deploy the pipeline:

    ```shell
    python tektonc.py \
    -t tektoncsamples/prefix-caching/pipeline.yaml.j2 \
    -f tektoncsamples/prefix-caching/values.yaml \
    -o tektoncsamples/prefix-caching/pipeline.yaml

    kubectl apply -f tektoncsamples/prefix-caching/pipeline.yaml
    ```

3. Deploy the PipelineRun.

    Run the pipeline by deploying the PipelineRun:

    ```shell
    kubectl apply -f tektoncsamples/prefix-caching/pipelinerun.yaml
    ```

### Inspection

See the `PipelineRun` object created:

```shell
tkn pr list
```

See the `TaskRun` objects created:

```shell
tkn tr list
```

See the logs for a `TaskRun`:

```shell
tkn tr logs <taskrun_name> -f
```

Describe a `TaskRun`:

```shell
tkn tr describe <taskrun_name>
```

### Cleanup

Delete the `PipelineRun`: 

```shell
tkn pr delete <pipelinerun_name> -f
```
