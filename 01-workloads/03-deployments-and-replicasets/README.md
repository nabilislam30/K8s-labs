# Deployments and ReplicaSets

This module documents six hands-on Kubernetes labs covering Deployment lifecycle management, rollouts and rollbacks, resource governance, labels and selectors, ReplicaSet behaviour, and deployment strategies.

The command records below are based on the terminal session. Where the observed result differed from the lab's expected result, the difference is called out explicitly.

## Prerequisites

- A working Kubernetes cluster
- `kubectl` configured for the cluster
- Permission to create cluster and namespace-scoped resources

Confirm that the nodes are ready:

```bash
kubectl wait --for=condition=Ready nodes --all --timeout=300s
```

## Lab 1: Create and Scale a Deployment

### 1. Create the Deployment

Create an nginx Deployment with three replicas:

```bash
kubectl create deployment nginx --image=nginx:1.21 --replicas=3
```

The first two attempts contained pasted terminal control characters (`^[[200~` and `~`) and Bash reported `kubectl: command not found`. Retyping the command normally succeeded:

```text
deployment.apps/nginx created
```

Inspect the Deployment and its Pods:

```bash
kubectl get deployments
kubectl get pods -l app=nginx
kubectl get replicasets
```

The Deployment initially showed `0/3` available while its three Pods were being created. Its ReplicaSet was then visible as `nginx-649ffc778`, with three desired replicas.

### 2. Inspect the ReplicaSet

```bash
kubectl describe rs nginx-649ffc778
```

The description confirmed that:

- the ReplicaSet was controlled by `Deployment/nginx`;
- its desired and current replica counts were three;
- it created three nginx Pods.

### 3. Scale Imperatively

```bash
kubectl scale deployment nginx --replicas=5
kubectl get pods -l app=nginx -w
kubectl get deployment nginx
```

The Deployment reached five ready replicas.

### 4. Scale Declaratively

Export the live manifest and change `spec.replicas` from `5` to `2`:

```bash
kubectl get deployment nginx -o yaml > nginx-deployment.yaml
sed -i 's/replicas: 5/replicas: 2/' nginx-deployment.yaml
kubectl apply -f nginx-deployment.yaml
```

Because the resource was originally created imperatively, `kubectl apply` warned that the last-applied-configuration annotation was missing and added it automatically. The Deployment was configured successfully and scaled down to two Pods.

Verify the final state and scaling events:

```bash
kubectl get pods -l app=nginx
kubectl get rs -l app=nginx
kubectl describe deployment nginx
kubectl get deployment nginx -o jsonpath='{.status.replicas}'
```

Observed final replica count: `2`.

## Lab 2: Rolling Updates and Rollbacks

### 1. Create a Versioned Deployment

```bash
kubectl create deployment webapp --image=nginx:1.21 --replicas=4
kubectl get deployment webapp
kubectl get pods -l app=webapp
```

Four Pods were created; some were briefly in `ContainerCreating` while the Deployment became ready.

### 2. Perform a Rolling Update

```bash
kubectl set image deployment/webapp nginx=nginx:1.22
kubectl rollout status deployment/webapp
```

The rollout completed successfully. Kubernetes created a new ReplicaSet and scaled the old one to zero:

```bash
kubectl get rs -l app=webapp
```

```text
webapp-65c7fd944d   4   4   4
webapp-69978cccf   0   0   0
```

### 3. Inspect Revision History

```bash
kubectl rollout history deployment/webapp
kubectl rollout history deployment/webapp --revision=2
```

Revision 2 used `nginx:1.22`.

### 4. Test a Failed Rollout

```bash
kubectl set image deployment/webapp nginx=nginx:broken-tag
kubectl rollout status deployment/webapp --timeout=30s
kubectl get pods -l app=webapp
```

The new image could not be pulled. The new Pods entered `ErrImagePull` or `ImagePullBackOff`, while existing Pods remained available.

Roll back to the previous revision:

```bash
kubectl rollout undo deployment/webapp
kubectl rollout status deployment/webapp
```

The rollback completed successfully.

### 5. Roll Back to a Specific Revision

```bash
kubectl rollout history deployment/webapp
kubectl rollout undo deployment/webapp --to-revision=1
kubectl rollout status deployment/webapp
kubectl get deployment webapp -o jsonpath='{.spec.template.spec.containers[0].image}'
```

The rollback command succeeded. The image-verification output was not captured clearly in the screenshots, so no exact value is asserted here.

### 6. Pause and Resume a Rollout

```bash
kubectl set image deployment/webapp nginx=nginx:1.23
kubectl rollout pause deployment/webapp
kubectl get pods -l app=webapp
kubectl rollout resume deployment/webapp
kubectl rollout status deployment/webapp
```

The paused state temporarily retained a mixture of revisions. After resuming, the rollout completed.

> **Verification note:** `kubectl rollout history deployment/webapp | grep -c REVISION` returned `1`. That command counts the table header containing `REVISION`; it does not count the number of stored revisions.

## Lab 3: Resource Governance

### 1. Create a Namespace and ResourceQuota

```bash
kubectl create namespace limited
```

Apply a namespace quota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: limited
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "10"
    limits.memory: 16Gi
    pods: "10"
```

```bash
kubectl apply -f resource-quota.yaml
kubectl get resourcequota -n limited
kubectl describe resourcequota compute-quota -n limited
```

Initial quota usage was zero.

### 2. Create a LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: limited
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      min:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: "2"
        memory: 4Gi
```

```bash
kubectl apply -f limit-range.yaml
kubectl get limitrange -n limited
kubectl describe limitrange default-limits -n limited
```

### 3. Deploy a Resource-Constrained Application

The `resource-app` Deployment used two replicas with the following resources per container:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

```bash
kubectl apply -f resource-app.yaml
kubectl get deployment resource-app -n limited
kubectl get pods -n limited
```

### 4. Test the Quota

```bash
kubectl scale deployment resource-app -n limited --replicas=7
```

The attempted scale was rejected as exceeding the namespace quota. The Deployment was restored to two replicas:

```bash
kubectl scale deployment resource-app -n limited --replicas=2
```

### 5. Test Default Resources

Create a Pod without an explicit `resources` section:

```bash
kubectl run default-resources --image=nginx:1.21 -n limited
kubectl get pod default-resources -n limited -o jsonpath='{.spec.containers[0].resources}'
```

The LimitRange supplied:

| Type | CPU | Memory |
|---|---:|---:|
| Request | `100m` | `128Mi` |
| Limit | `500m` | `512Mi` |

### 6. Reject an Oversized Pod

A Pod requesting and limiting itself to `3` CPUs and `5Gi` memory was rejected because it exceeded the LimitRange maxima of `2` CPUs and `4Gi` memory.

### 7. Inspect Final Quota Usage

```bash
kubectl describe resourcequota compute-quota -n limited
```

Observed usage:

| Resource | Used | Hard |
|---|---:|---:|
| `limits.cpu` | `1500m` | `10` |
| `limits.memory` | `1536Mi` | `16Gi` |
| `pods` | `3` | `10` |
| `requests.cpu` | `500m` | `4` |
| `requests.memory` | `640Mi` | `8Gi` |

## Lab 4: Label Selectors and Filtering

### 1. Create Labelled Workloads

Three Deployments were created:

| Deployment | Replicas | Image | Labels |
|---|---:|---|---|
| `frontend` | 3 | `nginx:1.21` | `app=frontend,tier=web,env=prod` |
| `backend` | 2 | `nginx:1.21` | `app=backend,tier=api,env=prod` |
| `cache` | 1 | `redis:7` | `app=cache,tier=data,env=prod` |

Verify labels:

```bash
kubectl get pods --show-labels
```

### 2. Basic Selection

```bash
kubectl get pods -l app=frontend
kubectl get pods -l tier=web
kubectl get pods -L tier,env,app
```

The equality selectors returned the three frontend Pods.

### 3. Set-Based Selection

```bash
kubectl get pods -l 'tier in (web,api)'
kubectl get pods -l 'app!=cache'
kubectl get pods -l 'tier in (web,api),env=prod'
```

The combined selector returned the frontend and backend Pods. The inequality selector also matched unrelated pre-existing Pods whose `app` label was not `cache`; inequality does not imply that a particular application family was selected.

### 4. Existence Selection

```bash
kubectl get pods -l tier
kubectl get pods -l '!tier'
```

The first command selected Pods with any `tier` value; the second selected earlier workloads without a `tier` label.

### 5. Operate on Selected Resources

Scale every Deployment labelled `tier=api`:

```bash
kubectl get deployments -l tier=api
kubectl scale deployment -l tier=api --replicas=4
kubectl get pods -l tier=api
```

The backend reached four replicas.

Delete the frontend Pods and let their ReplicaSet recreate them:

```bash
kubectl delete pod -l app=frontend
kubectl get pods -l app=frontend
```

Three replacement Pods appeared immediately.

### 6. Add Version Labels

```bash
kubectl label deployment frontend version=v1.0
kubectl label deployment backend version=v2.0
kubectl label deployment cache version=v1.5
kubectl get deployments --show-labels | grep version=v1
kubectl get deployments -l tier=web,env=prod,version=v1.0
```

The version filter selected the production frontend Deployment.

### 7. Cross-Resource Selection

```bash
kubectl label deployment frontend team=platform
kubectl label deployment backend team=platform
kubectl label deployment cache team=data-eng
kubectl get all -l team=platform
```

Observed result: only the `frontend` and `backend` Deployments were listed. The command did **not** list their ReplicaSets or Pods because `kubectl label deployment ...` changed only Deployment metadata; it did not update the Pod-template labels.

To make the label propagate to future Pods, label the template as well, for example:

```bash
kubectl patch deployment frontend -p '{"spec":{"template":{"metadata":{"labels":{"team":"platform"}}}}}'
```

### 8. Namespace-Wide Environment Labels

```bash
kubectl create deployment staging-app --image=nginx:1.21 --replicas=2
kubectl label deployment staging-app env=staging tier=web
kubectl get deployments -l env=prod
kubectl get deployments -l env=staging
```

The production query returned `backend`, `cache`, and `frontend`; the staging query returned `staging-app`.

The text `env: prod|staging|dev` was later pasted into Bash and produced `command not found`. It is a label-pattern example, not a shell command.

Final checks were run back-to-back and printed as `webapi7`, meaning:

- frontend tier: `web`;
- backend tier: `api`;
- Pod count for `tier in (web,api)`: `7` (three frontend plus four backend).

## Lab 5: ReplicaSet Deep-Dive

### 1. Create a Standalone ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: standalone-rs
  labels:
    app: standalone
spec:
  replicas: 3
  selector:
    matchLabels:
      app: standalone
      version: v1
  template:
    metadata:
      labels:
        app: standalone
        version: v1
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
```

```bash
kubectl apply -f standalone-rs.yaml
kubectl get rs standalone-rs
kubectl get pods -l app=standalone
```

The ReplicaSet reached three desired, current, and ready replicas.

### 2. Test Self-Healing

```bash
POD_NAME=$(kubectl get pods -l app=standalone -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod "$POD_NAME"
kubectl get pods -l app=standalone -w
```

The deleted Pod was immediately replaced, preserving the desired count of three.

### 3. Change the Pod Template

```bash
kubectl patch rs standalone-rs -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.22"}]}}}}'
kubectl get pods -l app=standalone -o jsonpath='{.items[*].spec.containers[0].image}'
```

Existing Pods remained on `nginx:1.21`; changing a ReplicaSet template does not trigger a rollout. After one Pod was deleted, its replacement used `nginx:1.22`, leaving a mixed set of images.

### 4. Orphan the Pods

```bash
kubectl delete rs standalone-rs --cascade=orphan
kubectl get rs
kubectl get pods -l app=standalone
kubectl describe pod -l app=standalone | grep 'Controlled By'
```

The ReplicaSet was deleted, its three Pods remained running, and the controller check produced no output.

### 5. Attempt Adoption with a Deployment

A Deployment named `adopted` was created with three replicas, matching `app=standalone,version=v1`, and an `nginx:1.22` template.

```bash
kubectl apply -f adopted-deployment.yaml
kubectl get pods -l app=standalone
kubectl get rs
kubectl describe pod -l app=standalone | grep 'Controlled By'
```

**Observed result:** the Deployment created a new ReplicaSet and three new `adopted-*` Pods. The three original `standalone-rs-*` Pods remained orphaned. The controller output appeared only for the new managed Pods, so the expected adoption did not occur.

### 6. Scale the Deployment

```bash
kubectl scale deployment adopted --replicas=5
kubectl get pods -l app=standalone \
  -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

The managed Deployment reached five `nginx:1.22` Pods, but the three orphaned Pods still matched the same label selector. Therefore the query returned eight Pods in total, including old `nginx:1.21` Pods.

```bash
kubectl rollout restart deployment adopted
```

### 7. Verify Selector Immutability

```bash
kubectl get rs -l app=standalone -o name | head -1 | \
  xargs kubectl patch -p '{"spec":{"selector":{"matchLabels":{"app":"different"}}}}'
```

The patch failed. The output reported both selector/template mismatch validation errors and that the ReplicaSet selector field is immutable.

Final commands printed `58` without separators:

```bash
kubectl get deployment adopted -o jsonpath='{.spec.replicas}'
kubectl get pods -l app=standalone --no-headers | wc -l
```

This represents `5` desired Deployment replicas and `8` matching Pods—not the lab's expected `5` and `5`—because three orphaned Pods remained.

## Lab 6: Deployment Strategies Comparison

### 1. Create a RollingUpdate Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-app
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2
      maxSurge: 2
  selector:
    matchLabels:
      app: rolling-app
  template:
    metadata:
      labels:
        app: rolling-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f rolling-app.yaml
kubectl get deployment rolling-app
kubectl get pods -l app=rolling-app
```

The Deployment reached six ready replicas.

### 2. Perform a Rolling Update

```bash
kubectl set image deployment/rolling-app nginx=nginx:1.22
kubectl get pods -l app=rolling-app -w
kubectl get pods -l app=rolling-app \
  -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image,STATUS:.status.phase
```

The watch showed surge Pods being created and old Pods terminating in stages. The final table showed six running Pods on `nginx:1.22`.

### 3. Understand `maxUnavailable` and `maxSurge`

| Setting | Effect |
|---|---|
| Lower `maxUnavailable` | Preserves more capacity but may slow the rollout |
| Higher `maxUnavailable` | Allows faster replacement but reduces available capacity |
| Lower `maxSurge` | Uses less temporary capacity |
| Higher `maxSurge` | Can accelerate the rollout but requires spare capacity |

Example strategy fragments such as `maxUnavailable: 0` and `maxSurge: 1` were pasted directly into Bash during the session and produced `command not found`. They are YAML settings and must be placed in a manifest or applied with `kubectl patch`.

### 4. Create a Recreate Deployment

Create `recreate-app` with six replicas, image `nginx:1.21`, and:

```yaml
strategy:
  type: Recreate
```

```bash
kubectl apply -f recreate-app.yaml
kubectl get deployment recreate-app
kubectl get pods -l app=recreate-app
```

All six Pods became ready.

### 5. Perform a Recreate Update

The intended image was `nginx:1.22`, but the terminal command actually used:

```bash
kubectl set image deployment/recreate-app nginx=nginx:1.2222
```

Kubernetes accepted the image reference and began a Recreate rollout. Since that tag was a typo, the session did not demonstrate a successful `1.22` Recreate update.

### 6. Compare the Strategies

| Aspect | RollingUpdate | Recreate |
|---|---|---|
| Availability | Can preserve old ready Pods during an update | Removes old Pods before creating the new set |
| Old and new versions coexist | Temporarily | No |
| Temporary resource use | May be higher because of surge Pods | No surge Pods |
| Rollout style | Gradual | All-at-once replacement |
| Best fit | Highly available, backward-compatible services | Workloads that cannot run two versions concurrently |

Verify the configured types:

```bash
kubectl get deployment rolling-app -o jsonpath='{.spec.strategy.type}'
kubectl get deployment recreate-app -o jsonpath='{.spec.strategy.type}'
```

The two outputs appeared concatenated as `RollingUpdateRecreate` because the JSONPath commands did not add newlines.

### 7. Test Failure and Rollback Behaviour

RollingUpdate failure:

```bash
kubectl set image deployment/rolling-app nginx=nginx:broken
kubectl rollout status deployment/rolling-app --timeout=30s
kubectl get pods -l app=rolling-app
```

The rollout timed out. Existing Pods remained running while new Pods entered `ErrImagePull` and `ImagePullBackOff`, demonstrating the strategy's availability protection.

```bash
kubectl rollout undo deployment/rolling-app
```

The rollback succeeded. `kubectl` warned that the resource had previously been managed with `kubectl apply`, so the stored last-applied annotation would not be updated by the rollback.

Recreate failure:

```bash
kubectl set image deployment/recreate-app nginx=nginx:broken
kubectl get pods -l app=recreate-app
```

The captured output showed six new Pods in `ContainerCreating` shortly after the command and no old Pods. This confirms the availability gap, although the screenshot was taken before the new Pods reached the expected image-pull failure state.

```bash
kubectl rollout undo deployment/recreate-app
```

The rollback succeeded with the same last-applied-configuration warning.

Production-strategy YAML examples were also pasted into Bash and returned `command not found`; they were documentation fragments, not executable commands.

## Production Checklist

Before using these patterns in production:

- Define resource requests and limits for every container.
- Use ResourceQuotas and LimitRanges to establish namespace boundaries and defaults.
- Prefer RollingUpdate for services that require continuous availability.
- Use readiness probes so a rollout does not treat an unready application as available.
- Set `maxUnavailable: 0` only when the cluster has enough capacity for surge Pods.
- Use Recreate only when downtime is acceptable or simultaneous versions are unsafe.
- Apply labels consistently to both resource metadata and Pod templates.
- Keep rollback procedures documented and test them regularly.
- Monitor Deployment and ReplicaSet events during rollouts.
- Avoid deleting old ReplicaSets prematurely; they preserve rollback history.

## Key Concepts

- **Deployment:** declarative Pod updates, scaling, revision history, and rollout management.
- **ReplicaSet:** desired replica count and continuous Pod self-healing.
- **RollingUpdate:** gradual replacement controlled by `maxUnavailable` and `maxSurge`.
- **Recreate:** removes the old replica set before bringing up the replacement.
- **ResourceQuota:** namespace-level aggregate limits.
- **LimitRange:** per-container defaults, minima, and maxima.
- **Labels:** metadata used for organisation and resource selection.
- **Selectors:** equality, set-based, and existence expressions used to target resources.

## Completion Summary

The labs demonstrated:

1. Imperative and declarative Deployment scaling.
2. Rolling updates, revision inspection, failed rollouts, and rollbacks.
3. Namespace resource governance using quotas and default limits.
4. Label-based querying and operations across workloads.
5. ReplicaSet reconciliation, orphaning, non-rollout template changes, and immutable selectors.
6. The availability and capacity trade-offs between RollingUpdate and Recreate.
