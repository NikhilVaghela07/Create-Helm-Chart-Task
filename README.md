# Custom NGINX Helm Chart

This repository contains a Helm chart that deploys a custom NGINX page to
Minikube. The same chart can be installed more than once in the same
Kubernetes cluster. Each Helm release has its own Deployment, Service, and
ConfigMap, so the releases can be updated independently.

## What this chart demonstrates

- Changing the NGINX replica count from `values.yaml`
- Changing the NGINX image tag without editing the templates
- Replacing the default NGINX page with a configurable message
- Running `test` and `demo` copies at the same time
- Upgrading only one copy
- Reviewing Helm release history
- Removing the releases and verifying that their resources are gone

The chart is in `nginx-custom-chart/`

## Prerequisites

Install the following tools and make sure they are available in your `PATH`:

- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm 3](https://helm.sh/docs/intro/install/)

Start a local Kubernetes cluster:

```bash
minikube start
kubectl get nodes
```

The node should have the status `Ready`.

## Change the chart values

The main configurable values are in
[`nginx-custom-chart/values.yaml`](./nginx-custom-chart/values.yaml):

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: ""

service:
  type: ClusterIP
  port: 80

customMessage: "Hello from my custom Helm chart!"
```

The `configmap.yaml` template
creates an `index.html` file containing `customMessage`. The Deployment mounts
that ConfigMap at `/usr/share/nginx/html`, which replaces the default NGINX
page.

Values can be overridden at install or upgrade time with `--set`. Use
`--set-string` for messages so Helm always treats the message as text.

You can also keep release-specific settings in separate values files:

- `values-test.yaml`
- `values-demo.yaml`

The keys in these files must match the chart values, such as `replicaCount`,
`image.tag`, and `customMessage`.

## Validate and render the chart

Run these commands from the repository root:

```bash
helm lint ./nginx-custom-chart
```
```bash
helm template ./nginx-custom-chart
```

`helm lint` checks the chart for errors. `helm template` renders the
Kubernetes manifests locally without creating resources in the cluster.

## Install the test release

Install one copy with one replica:

```bash
helm install test ./nginx-custom-chart \
  --set replicaCount=1 \
  --set image.tag=1.27 \
  --set-string customMessage="This is the TEST application"
```

Or use the separate values file:

```bash
helm install test ./nginx-custom-chart --values ./values-test.yaml
```

Check the release and its resources:

```bash
helm status test
kubectl get pods,deployments,services
```

## Upgrade the test release

+ by using value-test.yaml file
```
helm upgrade test ./nginx-custom-chart --values ./values-test.yaml
```
+ By using direct command
```
helm upgrade demo ./nginx-custom-chart \
  --set replicaCount=3 \
  --set image.tag=1.27 \
  --set-string customMessage="The DEMO application was upgraded"
```

## Install the demo release

Install a second copy in the same namespace with two replicas:

```bash
helm install demo ./nginx-custom-chart \
  --set replicaCount=2 \
  --set image.tag=1.27 \
  --set-string customMessage="This is the DEMO application"
```

Or use the separate values file:

```bash
helm install demo ./nginx-custom-chart --values ./values-demo.yaml
```

Both releases can run at the same time because Helm includes the release name
in the generated resource names:

```
helm list
```
```bash
helm status demo
kubectl get pods,deployments,services
```

The cluster should contain one `test` pod and two `demo` pods.

## Open both applications

The chart uses a `ClusterIP` Service, so forward each Service to a different
local port. Run each command in its own terminal and leave it running while
you test the application:

```bash
kubectl port-forward service/test-nginx-custom-chart 8081:80
```

Open <http://localhost:8081>. The page should show:

```text
This is the TEST application
```

In another terminal:

```bash
kubectl port-forward service/demo-nginx-custom-chart 8082:80
```

Open <http://localhost:8082>. The page should show:

```text
This is the DEMO application
```

## Run the Helm tests

The `test-connection.yaml`
template defines a Helm test that checks whether each `ClusterIP` Service is
reachable from inside the cluster. The test uses the Service DNS name and the
`http://` scheme; it does not try to access the Service through the Minikube
node or a browser port-forward:

```bash
helm test test
helm test demo
```

Inspect test pods if a test fails:

```bash
kubectl get pods
kubectl logs <test-pod-name>
```

## Upgrade only the demo release

Change the demo replica count and message without changing the test release:

```bash
helm upgrade demo ./nginx-custom-chart \
  --set replicaCount=3 \
  --set image.tag=1.27 \
  --set-string customMessage="The DEMO application was upgraded"
```

The same upgrade can use the demo values file. Edit
`values-demo.yaml` first if you want to change the
replica count or message, then run:

```bash
helm upgrade demo ./nginx-custom-chart --values ./values-demo.yaml
```

Verify the upgrade:

```bash
kubectl get deployment demo-nginx-custom-chart
helm status demo
helm get values demo
```

Refresh <http://localhost:8082>. It should show the upgraded message. The test
release should still have one replica and its original message:

```bash
helm get values test
kubectl get deployment test-nginx-custom-chart
```

## Review Helm history

Helm records each install and upgrade as a release revision:

```bash
helm history demo
helm get manifest demo
helm get manifest demo --revision 1
```

The demo release should have at least two revisions: the original install and
the upgrade.

## Clean up

Remove both releases:

```bash
helm uninstall test
helm uninstall demo
```

Confirm that the chart resources are gone:

```bash
helm list
kubectl get pods,deployments,services
```

`helm list` should not show either release, and the `test-nginx-custom-chart`
and `demo-nginx-custom-chart` resources should no longer exist.
---