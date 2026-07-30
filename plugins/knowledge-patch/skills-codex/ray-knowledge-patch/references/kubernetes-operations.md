# KubeRay operations

## RayCluster bootstrap and head-service access

Installing the `kuberay/ray-cluster` chart creates a RayCluster custom
resource. The KubeRay operator responds by creating head and worker Pods.

Select the cluster's Pods with `ray.io/cluster=<name>`. The generated head
service exposes the Dashboard and Ray Jobs endpoint on port 8265.

```sh
helm install raycluster kuberay/ray-cluster --version 1.6.0
kubectl get pods --selector=ray.io/cluster=raycluster-kuberay
kubectl port-forward service/raycluster-kuberay-head-svc 8265:8265
ray job submit --address http://localhost:8265 -- python app.py
```

## RayJob cluster and submission configuration

A RayJob can obtain a cluster in either of two ways:

- embed `rayClusterSpec` so KubeRay creates a RayCluster; or
- use `clusterSelector` to target an existing RayCluster.

KubeRay submits `entrypoint` after the cluster is ready. `runtimeEnvYAML` is a
multiline YAML string. The `jobId`, `metadata`, and entrypoint CPU, GPU, and
custom-resource fields map to Ray Jobs submission options.

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

| Mode | Submission behavior | Constraints |
| --- | --- | --- |
| `K8sJobMode` | Creates a submitter Kubernetes Job; this is the default | Supports `submitterPodTemplate` |
| `HTTPMode` | The operator submits directly | — |
| `InteractiveMode` | Waits for user submission | Alpha |
| `SidecarMode` | Injects the submitter into the head Pod | Head restart policy must be `Never`; no `clusterSelector`, `submitterPodTemplate`, or `submitterConfig` |

A `submitterPodTemplate` applies only to `K8sJobMode`.

KubeRay injects two environment variables into the submitter:

- `RAY_DASHBOARD_ADDRESS` is
  `$HEAD_SERVICE:$DASHBOARD_PORT`; and
- `RAY_JOB_SUBMISSION_ID` comes from `RayJob.Status.JobId`.

## Retry scopes and deadlines

Top-level `backoffLimit`, added in KubeRay 1.2.0, defaults to zero. Each retry
at this scope creates a new RayCluster.

`submitterConfig.backoffLimit` instead retries the submitter Kubernetes Job
and defaults to two.

The lifecycle deadlines are independent:

- `preRunningDeadlineSeconds` fails a RayJob with
  `PreRunningDeadlineExceeded` if deployment does not reach `Running`. Zero
  disables this deadline.
- `activeDeadlineSeconds` bounds the time to reach `Complete` or `Failed` and
  reports `DeadlineExceeded`.

Job success and deployment completion are represented separately:
`status.jobStatus: SUCCEEDED` and
`status.jobDeploymentStatus: Complete`.

## Cleanup and suspension

`shutdownAfterJobFinishes` defaults to false.
`ttlSecondsAfterFinished` applies only when shutdown is enabled.

When shutdown is enabled, setting the operator environment variable
`DELETE_RAYJOB_CR_AFTER_JOB_FINISHES=true` also deletes the RayJob custom
resource and all resources created by that RayJob.

KubeRay 1.6.0 provides beta `deletionStrategy` behind the
`RayJobDeletionPolicy` feature gate. A rule can trigger any of these actions
for `SUCCEEDED` or `FAILED`, optionally after a per-rule TTL:

- `DeleteWorkers`;
- `DeleteCluster`;
- `DeleteSelf`; or
- `DeleteNone`.

This permits staged cleanup. Rules-based cleanup is incompatible with
`shutdownAfterJobFinishes` and the global `ttlSecondsAfterFinished`.
The older `onSuccess` and `onFailure` form is deprecated.

Setting `suspend: true` deletes both the RayCluster and the submitter.
Do not change it manually when Kueue controls RayJob scheduling.

## RayService readiness and endpoints

RayService manages both an underlying RayCluster and Ray Serve applications.
Use `serveConfigV2` for multi-application Serve configuration.

After Serve has endpoints, RayService:

- reports a `Ready=True` condition;
- creates a RayService head service exposing Dashboard access on port 8265;
  and
- creates a Serve service exposing application HTTP traffic on port 8000.

```sh
kubectl get rayservice
kubectl describe rayservices.ray.io rayservice-sample
kubectl port-forward svc/rayservice-sample-head-svc 8265:8265
```
