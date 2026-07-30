# Serving and recovery

## Pipelining deployment responses

A deployment method call returns a `DeploymentResponse` immediately. Await it
when the local caller needs the resolved value. Otherwise, pass it directly
to another `DeploymentHandle` call so composed deployments can forward an
intermediate result without materializing it locally.

```python
summary = self.summarizer.remote(text)
translation = await self.translator.translate.remote(summary)
```

## Application and replica failures

An application exception:

- produces an HTTP 500 response with traceback information; and
- does not kill the replica.

Serve replaces a failed replica actor. It also restarts failed proxies and
the controller.

## Controller recovery boundaries

The controller restores durable state such as routing policies and deployment
configuration from the GCS. Transient connections and internal request queues
are not restored.

HTTP, gRPC, and deployment-handle requests can continue while the Serve
controller is down. Autoscaling pauses during the outage. After the controller
recovers, autoscaling resumes without the metrics collected before the
failure.

Recovery from an entire Ray cluster failure requires cluster recovery at the
KubeRay layer; Serve's controller recovery does not provide that boundary.

## REST management

Every node in a Ray cluster provides a Serve REST API server. Each server can
connect to the Serve instance and process management requests. The Serve CLI
is available alongside these per-node REST management endpoints.
