# Kubernetes Controllers

This lab explores the Kubernetes controllers used to maintain different workload types. It covers ReplicaSets, DaemonSets, StatefulSets, Jobs and CronJobs.

## Objectives

- Maintain a desired number of identical Pods with a ReplicaSet.
- Run one Pod on every eligible node with a DaemonSet.
- Give stateful workloads stable identities and ordered lifecycle behaviour.
- Run finite and recurring workloads with Jobs and CronJobs.
- Verify controller behaviour through resource state, logs, events and cleanup.

## Environment verification

The cluster was checked before the exercises began:

```bash
kubectl wait --for=condition=Ready nodes --all --timeout=300s
```

Observed result:

```text
node/controlplane condition met
Cluster is ready. Let's explore controllers!
```

---

## Lab 1: ReplicaSets

A ReplicaSet maintains a specified number of identical Pod replicas. Deployments normally manage ReplicaSets, but creating one directly demonstrates the underlying controller.

### Task: Create an NGINX ReplicaSet

The following manifest requested three NGINX Pods:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80
```

It was applied with:

```bash
kubectl apply -f nginx-rs.yaml
```

Observed result:

```text
replicaset.apps/nginx-rs created
```

The following verification command was entered:

```bash
kubectl get rs
```

The supplied terminal capture does not include the resulting ReplicaSet table, so the creation response is recorded but the three ready replicas are not claimed as verified evidence.

---

## Lab 2: DaemonSets

A DaemonSet ensures that a copy of a Pod runs on every eligible node. Typical uses include log collection, monitoring agents, CNI components and storage drivers.

### Task 1: Create a DaemonSet

The logging DaemonSet was defined as:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: logging-agent
  labels:
    app: logging
spec:
  selector:
    matchLabels:
      app: logging
  template:
    metadata:
      labels:
        app: logging
    spec:
      containers:
        - name: logger
          image: busybox
          command:
            - sh
            - -c
            - "while true; do echo 'Collecting logs from \$(hostname)...'; sleep 30; done"
          resources:
            limits:
              memory: 64Mi
              cpu: 100m
```

```bash
kubectl apply -f logging-ds.yaml
```

Observed result:

```text
daemonset.apps/logging-agent created
```

### Task 2: Check DaemonSet status

```bash
kubectl get ds
kubectl get pods -l app=logging -o wide
```

Observed result:

```text
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
logging-agent   1         1         1       1            1           <none>

NAME                  READY   STATUS    RESTARTS   IP              NODE
logging-agent-vtsl2   1/1     Running   0          192.168.0.224   controlplane
```

The single-node cluster had one desired, current, ready and available DaemonSet Pod.

### Task 3: Restrict a DaemonSet with a node selector

The node name was retrieved and labelled:

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label node "$NODE" tier=frontend
```

Observed result:

```text
node/controlplane labeled
```

A second DaemonSet targeted nodes with the new label:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: frontend-agent
spec:
  selector:
    matchLabels:
      app: frontend-agent
  template:
    metadata:
      labels:
        app: frontend-agent
    spec:
      nodeSelector:
        tier: frontend
      containers:
        - name: agent
          image: busybox
          command: ["sh", "-c", "echo 'Running on frontend node'; sleep 3600"]
```

```bash
kubectl apply -f frontend-ds.yaml
kubectl get pods -l app=frontend-agent -o wide
```

Observed result:

```text
daemonset.apps/frontend-agent created

NAME                   READY   STATUS    RESTARTS   IP             NODE
frontend-agent-k729c   1/1     Running   0          192.168.0.55   controlplane
```

This verified that the Pod was scheduled onto the node matching `tier=frontend`.

### Task 4: View DaemonSet details

```bash
kubectl describe ds logging-agent
```

Relevant observed fields:

```text
Selector:                                      app=logging
Node-Selector:                                 <none>
Desired Number of Nodes Scheduled:             1
Current Number of Nodes Scheduled:             1
Number of Nodes Scheduled with Up-to-date Pods: 1
Number of Nodes Scheduled with Available Pods:  1
Pods Status: 1 Running / 0 Waiting / 0 Succeeded / 0 Failed
```

The Events section showed that the DaemonSet controller successfully created the logging Pod.

### Task 5: Check Pod logs

```bash
kubectl logs -l app=logging --tail=5
```

Observed output:

```text
Collecting logs from $(hostname)...
```

The intended purpose was to print the hostname of the node agent. However, `\$(hostname)` was escaped in the manifest, so the shell printed the expression literally. Removing the backslash would allow command substitution:

```yaml
command: ["sh", "-c", "while true; do echo \"Collecting logs from $(hostname)...\"; sleep 30; done"]
```

### Task 6: Perform a rolling update

```bash
kubectl set image ds/logging-agent logger=busybox:1.36
kubectl rollout status ds/logging-agent
kubectl get ds logging-agent -o jsonpath='{.spec.updateStrategy}'
echo ""
```

Observed result:

```text
daemonset.apps/logging-agent image updated
daemon set "logging-agent" successfully rolled out
{"rollingUpdate":{"maxSurge":0,"maxUnavailable":1},"type":"RollingUpdate"}
```

The DaemonSet used the default `RollingUpdate` strategy with a maximum of one unavailable Pod.

### Task 7: Compare with a Deployment

```bash
kubectl create deployment logger-deploy --image=busybox --replicas=3 -- \
  sh -c "while true; do echo 'Deploy pod'; sleep 30; done"

kubectl get ds,deploy
kubectl get pods -o wide
```

Observed result:

```text
deployment.apps/logger-deploy created

daemonset.apps/frontend-agent   1   1   1   1   1   tier=frontend
daemonset.apps/logging-agent    1   1   1   1   1   <none>
deployment.apps/logger-deploy   3/3   3   3
```

The single-node cluster ran one Pod for each eligible DaemonSet, whereas the Deployment maintained three replicas on the available node.

### Task 8: Clean up

```bash
kubectl delete ds logging-agent frontend-agent
kubectl delete deployment logger-deploy
kubectl label node "$NODE" tier-
kubectl get ds,deploy
```

Observed result:

```text
daemonset.apps "logging-agent" deleted
daemonset.apps "frontend-agent" deleted
deployment.apps "logger-deploy" deleted
node/controlplane unlabeled
No resources found in default namespace.
```

---

## Lab 3: StatefulSets

StatefulSets are designed for applications requiring stable Pod names, ordered deployment and scaling, stable network identity and, when configured, per-Pod persistent storage.

### Task 1: Create a headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web
  ports:
    - port: 80
      name: web
```

```bash
kubectl apply -f web-headless.yaml
kubectl get svc web-headless
```

Observed result:

```text
service/web-headless created

NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
web-headless   ClusterIP   None         <none>        80/TCP
```

The `None` ClusterIP confirmed that this was a headless Service.

### Task 2: Create a StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f web-statefulset.yaml
kubectl get pods -l app=web -w
```

Observed result:

```text
statefulset.apps/web created

NAME    READY   STATUS    RESTARTS
web-0   1/1     Running   0
web-1   1/1     Running   0
web-2   1/1     Running   0
```

The ordered names demonstrate the StatefulSet identity pattern. The Pods became available as `web-0`, `web-1` and `web-2`.

### Task 3: Verify stable Pod names

The task requested:

```bash
kubectl get pods -l app=web
kubectl delete pod web-1
kubectl get pods -l app=web
```

The initial Pod names were captured, but the supplied terminal output does not show the deletion and recreation of `web-1`. Stable recreation therefore remains an intended task rather than verified evidence in this session.

### Task 4: Test Pod DNS

```bash
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup web-0.web-headless
```

Observed result:

```text
Server:  10.96.0.10
Address: 10.96.0.10:53

** server can't find web-0.web-headless: NXDOMAIN
pod "dns-test" deleted
pod default/dns-test terminated (Error)
```

The DNS verification did not succeed. A suitable follow-up is to test the fully qualified name and inspect the Service endpoints:

```bash
kubectl get endpoints web-headless
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup web-0.web-headless.default.svc.cluster.local
```

### Task 5: Demonstrate ordered scaling

Scale up:

```bash
kubectl scale sts web --replicas=5
kubectl get pods -l app=web -w
```

Observed result:

```text
statefulset.apps/web scaled
web-0   1/1   Running
web-1   1/1   Running
web-2   1/1   Running
web-3   1/1   Running
web-4   1/1   Running
```

Scale down:

```bash
kubectl scale sts web --replicas=2
kubectl get pods -l app=web
```

Observed result:

```text
statefulset.apps/web scaled
web-0   1/1   Running
web-1   1/1   Running
```

The StatefulSet retained the lowest ordered identities after scaling down.

### Task 6: Review persistent-storage configuration

The exercise supplied this YAML as a reference example:

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

This was reference YAML and was not meant to be executed directly. It was pasted into Bash, which produced `command not found` messages for YAML keys such as `volumeClaimTemplates:`, `spec:` and `storage:`. This was a shell input error, not a Kubernetes or StatefulSet failure.

To use it, the block would need to be included under the StatefulSet specification in a YAML file and applied with `kubectl apply -f`.

### Task 7: Inspect and update the StatefulSet

```bash
kubectl get sts web -o jsonpath='{.spec.updateStrategy}'
echo ""
kubectl set image sts/web nginx=nginx:1.23
kubectl rollout status sts/web
```

Observed result:

```text
{"rollingUpdate":{"partition":0},"type":"RollingUpdate"}
statefulset.apps/web image updated
partitioned roll out complete: 2 new pods have been updated...
```

The two remaining Pods were updated successfully using the StatefulSet's rolling-update strategy.

### Task 8: Compare StatefulSet and Deployment naming

```bash
kubectl create deployment nginx-deploy --image=nginx --replicas=3
sleep 5

echo "Deployment pods:"
kubectl get pods -l app=nginx-deploy -o name

echo "StatefulSet pods:"
kubectl get pods -l app=web -o name
```

Observed result:

```text
Deployment pods:
pod/nginx-deploy-76c96b875b-22rph
pod/nginx-deploy-76c96b875b-262kv
pod/nginx-deploy-76c96b875b-fnbxd

StatefulSet pods:
pod/web-0
pod/web-1
```

Deployment Pods used generated suffixes, while StatefulSet Pods retained predictable ordinal names.

### Task 9: Clean up

```bash
kubectl delete sts web
kubectl delete svc web-headless
kubectl delete deployment nginx-deploy
kubectl get sts,svc,deploy 2>/dev/null | grep -E "web|nginx-deploy" || \
  echo "All cleaned up"
```

Observed result:

```text
statefulset.apps "web" deleted
service "web-headless" deleted
deployment.apps "nginx-deploy" deleted
All cleaned up
```

---

## Lab 4: Jobs and CronJobs

Jobs run finite workloads to completion. CronJobs create Jobs according to a recurring schedule.

### Task 1: Create a simple Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: countdown
spec:
  template:
    spec:
      containers:
        - name: counter
          image: busybox
          command: ["sh", "-c", "for i in 5 4 3 2 1; do echo $i; sleep 1; done; echo 'Liftoff!'"]
      restartPolicy: Never
```

```bash
kubectl apply -f countdown-job.yaml
kubectl get jobs -w
kubectl get jobs
```

Observed result:

```text
job.batch/countdown created
countdown   Complete   1/1   9s
```

### Task 2: View Job logs

```bash
kubectl logs job/countdown
```

Observed result:

```text
5
4
3
2
1
Liftoff!
```

This confirmed that the finite task completed successfully.

### Task 3: Inspect the completed Job Pod

```bash
kubectl get pods -l job-name=countdown
```

Observed result:

```text
NAME             READY   STATUS      RESTARTS
countdown-hc58n   0/1     Completed   0
```

The completed Pod remained available for inspection and log retrieval.

The task also requested `kubectl describe job countdown`; its resulting output was not included in the terminal evidence.

### Task 4: Create a parallel Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-work
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      containers:
        - name: worker
          image: busybox
          command: ["sh", "-c", "echo 'Working on task...'; sleep 5; echo 'Done!'"]
      restartPolicy: Never
```

```bash
kubectl apply -f parallel-job.yaml
kubectl get pods -l job-name=parallel-work -w
kubectl get job parallel-work
```

Observed evidence confirms that `parallel-work` was created and that the Pod watch command was started. The final `5/5` completion state was not visible in the supplied terminal capture.

### Task 5: Demonstrate Job failure handling

The failing Job used `backoffLimit: 3` and deliberately exited with status `1`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: failing-job
spec:
  backoffLimit: 3
  template:
    spec:
      containers:
        - name: fail
          image: busybox
          command: ["sh", "-c", "echo 'About to fail...'; exit 1"]
      restartPolicy: Never
```

Verification:

```bash
kubectl get pods -l job-name=failing-job -w
kubectl get job failing-job
kubectl describe job failing-job | tail -10
```

Observed result:

```text
NAME          STATUS   COMPLETIONS   DURATION
failing-job   Failed   0/1           81s

Warning  BackoffLimitExceeded  Job has reached the specified backoff limit
```

The controller created replacement Pods after failures and stopped once the configured backoff limit was reached.

### Task 6: Create a CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: busybox
              command: ["sh", "-c", "echo 'Hello from CronJob at $(date)'"]
          restartPolicy: Never
```

```bash
kubectl apply -f hello-cron.yaml
kubectl get cronjobs
```

Observed result:

```text
cronjob.batch/hello-cron created

NAME         SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE
hello-cron   */1 * * * *   False     0        <none>
```

The CronJob was configured to run every minute.

### Task 7: Watch the CronJob execute

```bash
echo "Waiting for CronJob to trigger..."
kubectl get jobs -w
```

The supplied capture shows that the watch ran for approximately 70 seconds. However, the visible output contains only the existing `countdown` and `failing-job` resources. A generated `hello-cron-...` Job was not captured, so successful execution is not claimed.

### Task 8: Inspect concurrency policy

The task requested:

```bash
kubectl get cronjob hello-cron -o jsonpath='{.spec.concurrencyPolicy}'
echo ""
```

No matching terminal result was supplied. Kubernetes defaults this field to `Allow`, but that value was not verified in the captured session.

### Task 9: Create a manual Job from the CronJob

The task requested:

```bash
kubectl create job manual-hello --from=cronjob/hello-cron
kubectl get jobs | grep manual-hello
kubectl logs job/manual-hello
```

No matching execution output was supplied, so this task is recorded as instructed but not verified.

### Task 10: Clean up

The task specified:

```bash
kubectl delete job countdown parallel-work failing-job manual-hello
kubectl delete cronjob hello-cron
kubectl delete pods --field-selector=status.phase=Succeeded
kubectl delete pods --field-selector=status.phase=Failed
kubectl get jobs,cronjobs,pods
```

The cleanup output was not included in the supplied terminal capture.

---

## Troubleshooting summary

| Issue | Cause | Resolution |
|---|---|---|
| DaemonSet logs printed `$(hostname)` | The dollar sign was escaped, preventing shell substitution | Remove the backslash before `$` in the container command |
| StatefulSet Pod DNS returned `NXDOMAIN` | The short lookup did not resolve in the test session | Inspect endpoints and retry with the fully qualified cluster DNS name |
| YAML keys returned `command not found` | Reference YAML was pasted directly into Bash | Put YAML inside a manifest file before using `kubectl apply -f` |
| Several tasks have no final output | The supplied terminal capture ended before verification | Rerun only the missing verification commands if complete evidence is required |

## Controller comparison

| Controller | Primary purpose | Pod identity | Execution model |
|---|---|---|---|
| ReplicaSet | Maintain a desired replica count | Interchangeable | Continuous |
| DaemonSet | Run one Pod per eligible node | Node-oriented | Continuous |
| StatefulSet | Manage ordered, stateful replicas | Stable ordinal names | Continuous |
| Job | Run work to completion | Temporary | One-off |
| CronJob | Schedule Jobs | Temporary | Recurring |

## What I learned

- Different workload patterns require different Kubernetes controllers.
- A ReplicaSet maintains a desired number of interchangeable Pods.
- A DaemonSet targets nodes rather than a fixed replica count.
- Node labels and selectors control where DaemonSet Pods can run.
- StatefulSets provide stable identities and ordered scaling.
- Headless Services support network identity for StatefulSet Pods.
- Jobs track finite work until the required completions are reached.
- `parallelism` controls concurrent Job Pods, while `completions` controls the total successful executions required.
- `backoffLimit` prevents a failing Job from retrying indefinitely.
- CronJobs create Jobs according to a schedule and support concurrency policies.
- Successful resource creation and successful behavioural verification are separate forms of evidence.