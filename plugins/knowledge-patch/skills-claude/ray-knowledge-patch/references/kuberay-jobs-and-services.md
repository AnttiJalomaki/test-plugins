# KubeRay Jobs and Services

## RayCluster bootstrap and head-service access

Installing the `kuberay/ray-cluster` chart creates a `RayCluster` custom
resource. The KubeRay operator responds by creating the head and worker Pods.

```sh
helm install raycluster kuberay/ray-cluster --version 1.6.0
kubectl get pods --selector=ray.io/cluster=raycluster-kuberay
kubectl port-forward service/raycluster-kuberay-head-svc 8265:8265
ray job submit --address http://localhost:8265 -- python app.py
```

Select cluster Pods with `ray.io/cluster=<name>`. The generated head service
exposes the Dashboard and Ray Jobs endpoint on port 8265.

## RayJob cluster and submission model

A `RayJob` may embed `rayClusterSpec`, in which case KubeRay creates a cluster,
or use `clusterSelector` to target an existing `RayCluster`. The operator
submits `entrypoint` after the selected or created cluster becomes ready.

`runtimeEnvYAML` is a multiline YAML string. `jobId`, `metadata`, and the
entrypoint CPU, GPU, and custom-resource fields map to Ray Jobs submission
options.

```yaml
spec:
  rayClusterSpec: ...
  entrypoint: python /home/ray/app.py
  runtimeEnvYAML: |
    pip:
      - requests==2.26.0
    env_vars:
      KEY: "VALUE"
```

## Submission modes

`K8sJobMode` is the default and creates a submitter Kubernetes Job.
`HTTPMode` has the operator submit directly. Alpha `InteractiveMode` waits for
the user to submit. `SidecarMode` injects the submitter into the head Pod.

Sidecar mode requires the head Pod restart policy to be `Never` and does not
support:

- `clusterSelector`;
- `submitterPodTemplate`; or
- `submitterConfig`.

A `submitterPodTemplate` applies only to `K8sJobMode`. KubeRay injects
`RAY_DASHBOARD_ADDRESS` as `$HEAD_SERVICE:$DASHBOARD_PORT` and
`RAY_JOB_SUBMISSION_ID` from `RayJob.Status.JobId`.

## Retry scopes and deadlines

Top-level `backoffLimit`, added in KubeRay 1.2.0, defaults to zero. Each retry
at this level creates a new `RayCluster`. In contrast,
`submitterConfig.backoffLimit` retries the submitter Kubernetes Job and
defaults to two.

`preRunningDeadlineSeconds` fails a job with
`PreRunningDeadlineExceeded` if deployment does not reach `Running`; zero
disables this deadline. `activeDeadlineSeconds` limits the time to reach
`Complete` or `Failed` and reports `DeadlineExceeded`.

Successful completion has two separate status views:

```yaml
status:
  jobStatus: SUCCEEDED
  jobDeploymentStatus: Complete
```

## Cleanup policies

`shutdownAfterJobFinishes` defaults to false.
`ttlSecondsAfterFinished` applies only when shutdown is enabled.

When shutdown is enabled, setting the operator environment variable
`DELETE_RAYJOB_CR_AFTER_JOB_FINISHES=true` also deletes the `RayJob` custom
resource and all resources it created.

KubeRay 1.6.0 provides beta `deletionStrategy` behind the
`RayJobDeletionPolicy` feature gate. Each rule can react to `SUCCEEDED` or
`FAILED` with one of:

- `DeleteWorkers`;
- `DeleteCluster`;
- `DeleteSelf`; or
- `DeleteNone`.

An optional per-rule TTL enables staged cleanup. Rules-based cleanup is
incompatible with `shutdownAfterJobFinishes` and the global
`ttlSecondsAfterFinished`. The older `onSuccess` and `onFailure` style is
deprecated.

Setting `suspend: true` deletes both the `RayCluster` and the submitter. Avoid
changing it manually when Kueue controls RayJob scheduling.

## RayService readiness and endpoints

`RayService` manages an underlying `RayCluster` and Ray Serve applications.
`serveConfigV2` supports a multi-application Serve configuration.

After Serve exposes endpoints, the `RayService` reports a `Ready=True`
condition. It creates two services:

- a RayService head service for Dashboard access on port 8265; and
- a Serve service for application HTTP traffic on port 8000.

```sh
kubectl get rayservice
kubectl describe rayservices.ray.io rayservice-sample
kubectl port-forward svc/rayservice-sample-head-svc 8265:8265
```
