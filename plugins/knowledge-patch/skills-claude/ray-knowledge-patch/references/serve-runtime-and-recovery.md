# Ray Serve Runtime and Recovery

## Pipelining deployment-handle responses

A deployment method call returns a `DeploymentResponse` immediately. Await it
to obtain its value locally, or pass it directly into another
`DeploymentHandle` call. Passing the response allows a composed deployment to
forward an intermediate result without materializing it in the caller.

```python
summary = self.summarizer.remote(text)
translation = await self.translator.translate.remote(summary)
```

## Application and component failures

An application exception returns HTTP 500 with traceback information. It does
not kill the replica.

Serve handles infrastructure component failures as follows:

- a failed replica actor is replaced;
- failed proxies and the Serve controller are restarted; and
- controller state such as routing policies and deployment configuration is
  restored from the GCS.

Transient connections and internal request queues are lost during recovery.
Recovery from failure of the entire Ray cluster requires cluster recovery at
the KubeRay layer.

## Requests during controller failure

HTTP, gRPC, and deployment-handle requests can continue while the Serve
controller is unavailable. Autoscaling pauses until the controller recovers.
It then resumes without metrics collected before the failure, so immediate
post-recovery scaling decisions do not have that missing history.

## Per-node management endpoints

Every node in a Ray cluster runs a Serve REST API server. Each server can
connect to the Serve instance and handle management requests. These endpoints
exist alongside the Serve CLI; management access is therefore not limited to
a single controller-local REST server.
