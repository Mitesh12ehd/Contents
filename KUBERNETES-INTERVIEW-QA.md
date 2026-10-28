# Kubernetes, Helm, and Helmfile interview Q&A

For a ~3 year professional targeting development and DevOps. Two kinds of questions are mixed in on purpose:

1. Knowledge questions. Can you explain the object model and the tools.
2. Lived-work questions. Interviewers use these to decide if you have actually operated workloads, or only completed a tutorial.

Answers are written the way you should speak: specific, with failure modes, not a list of product names.

How they judge "you actually worked on Kubernetes":

- You debug from Events, previous logs, and probes, not from "the pod is failing".
- You talk about a rollout that got stuck, a probe that killed healthy pods, a ConfigMap that did not reload, or a Helm upgrade that left a failed history entry.
- You know the difference between what you typed (`kubectl`, `helm`, `helmfile`) and what the cluster did (ReplicaSet, endpoints, kubelet).
- You mention namespaces, RBAC, and who can do what, without being asked twice.
- You can describe one incident with a timeline. Memorized architecture diagrams do not substitute for that.

If you have not done the thing in the answer, do not fake a story. Say you have done it in a lab, then describe the mechanism correctly. Interviewers catch invented incidents faster than missing a definition.

---

## 1. Core Kubernetes knowledge

### What is Kubernetes, in one working definition?

A control plane that keeps declared objects (Deployments, Services, Jobs, and so on) matching cluster state. You submit YAML. Controllers reconcile. Kubelets run containers. It is not "a place to run Docker". It is a reconciliation loop plus a scheduler plus a network model.

### What problem does it solve that a VM or a single Docker host does not?

Desired state, rescheduling when a node dies, rolling updates, service discovery, resource isolation across many apps. It does not magically give you zero downtime, secrets hygiene, or good application health checks. Those are still your job.

### Control plane vs worker node. What runs where?

Control plane: API server, etcd, scheduler, controller manager. Optionally a cloud controller manager.

Worker: kubelet, kube-proxy (or an equivalent dataplane), container runtime (containerd is the common one now). Your Pods run here.

If they ask "where does kubectl talk", the answer is the API server. Never the kubelet directly in normal use.

### What is etcd's role? What happens if etcd is unhealthy?

etcd is the cluster's source of truth for object state. If etcd is slow or down, the API server cannot persist or read reliably. Existing Pods often keep running (kubelet already has spec), but you cannot schedule, scale, or deploy. That distinction is a lived-experience signal: cluster "down" for operators is not the same as apps already running.

### Pod vs container vs Deployment vs ReplicaSet

A Pod is the smallest deployable unit: one or more containers sharing network namespace and volumes.

A ReplicaSet keeps N copies of a Pod template.

A Deployment owns ReplicaSets and implements rolling updates and rollbacks. You almost never create a ReplicaSet by hand.

If they ask "why not put one container per Deployment and skip Pods", the answer is sidecars, shared volumes, and localhost communication (service mesh proxy, log shipper, cloud SQL proxy).

### Why would you put two containers in one Pod?

They must share lifecycle, disk, or localhost. Examples: nginx + app on a shared volume, Envoy sidecar, a vault agent that writes a secret file. If they can scale independently, they should not share a Pod.

### What is a Service? ClusterIP vs NodePort vs LoadBalancer vs ExternalName vs headless

Service is a stable virtual IP and DNS name plus a set of Endpoints/EndpointSlices selected by labels.

- ClusterIP: internal only. Default. This is what most app-to-app traffic uses.
- NodePort: opens the same port on every node. Used for labs or as the backing of some LoadBalancers. Awkward in production.
- LoadBalancer: cloud provider allocates an external LB that targets node ports or, with some CNIs, Pods directly.
- ExternalName: DNS CNAME. No proxy.
- Headless (`clusterIP: None`): DNS returns Pod IPs. Used for StatefulSets and client-side discovery.

Lived detail: a Service with no matching Pods still exists; DNS resolves; connections fail. Check Endpoints, not only the Service YAML.

### How does DNS work inside the cluster?

CoreDNS (usually). A Service named `api` in namespace `payments` is `api.payments.svc.cluster.local`. Same-namespace apps can use `api`. Pods get `pod-ip.namespace.pod.cluster.local` in some setups.

If app A in `frontend` cannot reach `api`, first question is whether they used a short name that only works in the same namespace.

### Ingress vs Service vs Gateway API

Service is L4 (mostly) in-cluster.

Ingress is an L7 HTTP/HTTPS object. Nothing happens until an Ingress controller (nginx, Traefik, ALB, etc.) is installed. People who only did tutorials forget this and say "we applied Ingress and it just worked".

Gateway API is the newer, more expressive model (Gateway, HTTPRoute). Same idea: a controller must implement it.

Lived detail: TLS can terminate at the load balancer, at the Ingress controller, or in the Pod. You should know which one your last project used and where certificates lived.

### Labels vs annotations

Labels are for selection: Services, Deployments, NetworkPolicies, kubectl queries.

Annotations are non-identifying metadata: checksums for rolling when a ConfigMap changes, Ingress controller settings, build IDs.

You do not select on annotations.

### ConfigMap and Secret. How does a Pod consume them?

Env vars, or mounted files. Secrets are base64 in etcd by default, not encrypted at rest unless you turned that on. Anyone with get secrets in that namespace can read them.

Lived details interviewers wait for:

- Changing a ConfigMap does not restart Pods. Processes that only read env at boot keep old values. File mounts may update depending on kubelet, but many apps do not re-read the file.
- Common pattern: annotate the Deployment with a checksum of the ConfigMap so a change triggers a rollout (`checksum/config: {{ include ... | sha256sum }}` in Helm).
- `kubectl create secret generic` from a file vs sealing secrets in git. If you committed raw Secrets, say so and say you would not do it again.

### Requests vs limits. What is OOMKilled vs CPU throttling?

Requests: scheduler and (for CPU) the baseline. Also used by QoS class.

Limits: hard cap. Memory limit exceeded -> OOMKilled (exit 137). CPU limit exceeded -> throttling, not a kill.

Lived details:

- A Pod with no requests can be packed onto a busy node and then starve.
- Setting CPU limit equal to request is a common production choice to avoid noisy-neighbor throttling surprises; some teams omit CPU limits on purpose. Know which you did and why.
- `kubectl describe pod` shows Last State OOMKilled. `kubectl top pod` is useless if metrics-server is missing.

QoS: Guaranteed (requests=limits for all containers), Burstable, BestEffort. Eviction under node memory pressure hits BestEffort first.

### Liveness vs readiness vs startup probes

Readiness: if it fails, the Pod is removed from Service Endpoints. Traffic stops. Pod is not killed.

Liveness: if it fails, kubelet kills and restarts the container.

Startup: disables the other probes until the app has started. Used for slow JVMs.

Lived trap: a liveness probe that hits the database. DB blip -> all Pods fail liveness -> all restart -> worse outage. Liveness should mean "this process is deadlocked", not "dependencies are healthy". Dependencies belong in readiness, or in the app's own 503.

Another trap: readiness too strict during startup without a startupProbe, so rolling update never adds Endpoints and the deploy looks hung.

### Rolling update. maxSurge and maxUnavailable

Deployment strategy RollingUpdate. maxSurge: extra Pods above desired. maxUnavailable: how many can be down.

Default often allows surge. If the cluster is tight on CPU, surge Pods pend, old Pods stay, rollout looks stuck.

Readiness must pass on new Pods or the Deployment never completes. `kubectl rollout status` waits on that.

`progressDeadlineSeconds` exceeded -> Deployment "timed out". People who have done this have used `kubectl rollout history` and `kubectl rollout undo`.

### Recreate vs RollingUpdate vs blue/green vs canary

Recreate: kill all, then start new. Downtime. Used when two versions cannot share a resource (an old hostPort, a non-HA singleton).

RollingUpdate: built in.

Blue/green: two complete ReplicaSets or two Deployments, switch Service or Ingress. Helm can do this with extra objects; Kubernetes does not have a native blue/green object.

Canary: send a fraction of traffic to a new version (Ingress weights, service mesh, Flagger, Argo Rollouts). A second Deployment with a subset of Pods plus a weighted Ingress is the homemade version.

### StatefulSet vs Deployment vs DaemonSet vs Job vs CronJob

Deployment: stateless, interchangeable Pods.

StatefulSet: stable identity (`web-0`, `web-1`), stable storage via volumeClaimTemplates, ordered start/stop. Databases, Kafka, Zookeeper. You still need an operator or a lot of care; a StatefulSet is not "Postgres is production-ready".

DaemonSet: one Pod per node (or per matching node). Logs agents, node exporters, CNI.

Job: run to completion. CronJob: schedule Jobs.

Lived: deleting a StatefulSet does not always delete PVCs. That surprises people during teardown.

### PersistentVolume, PVC, StorageClass

StorageClass: how to provision (cloud disk type, reclaim policy, binding mode).

PVC: the claim the Pod uses.

PV: the actual volume.

WaitForFirstConsumer vs Immediate: Immediate can provision a disk in zone A while the Pod schedules in zone B. People who have fought pending PVCs mention this.

Lived: PVC Pending -> describe PVC and StorageClass, check provisioner logs, check zone, check quota. Not "restart the Pod".

### HPA. Why did it not scale?

HorizontalPodAutoscaler needs metrics (metrics-server for CPU/memory, or Prometheus adapter for custom metrics).

Common real failures:

- metrics-server not installed
- container has no CPU request, so utilization is unknown
- minReplicas = maxReplicas
- app is I/O bound, CPU never rises
- cooldown / stabilization window makes it look "broken"

### PDB (PodDisruptionBudget)

Limits how many Pods of a set can be down during voluntary disruption (drain, cluster upgrade, some evictions). Not a protection against node crash.

Lived: a PDB of `minAvailable: 1` with `replicas: 1` blocks drains. Cluster upgrades stall. You have to temporarily lower the PDB or scale up.

### Init containers and hooks

Init containers run to completion before app containers. Used to wait for a dependency, render config, or migrate (migrations in init are controversial; a Job is often safer).

Helm hooks (`pre-install`, `pre-upgrade`, `post-install`) create extra Jobs/Pods tied to the release lifecycle. Interviewers ask this in the Helm section because hooks that fail leave releases in a bad state.

### NetworkPolicy. Default behavior without one

Without NetworkPolicies (or with a CNI that does not enforce them), all Pods can talk to all Pods.

A NetworkPolicy is deny-by-default for the Pods it selects, then you allow ingress/egress. Policies are additive.

Lived: you applied a policy in namespace A and broke DNS because you forgot to allow egress to kube-system CoreDNS (port 53). That story is a strong signal.

### RBAC: Role, ClusterRole, RoleBinding, ClusterRoleBinding, ServiceAccount

ServiceAccount: identity for Pods.

Role: permissions in one namespace. ClusterRole: cluster-wide or for namespaced resources across namespaces.

Binding attaches a role to a user, group, or ServiceAccount.

Lived: a CI runner using a ServiceAccount that can `*` in kube-system is a finding, not a flex. Least privilege: get/list/watch on the objects it needs, create on Jobs in one namespace.

`kubectl auth can-i` is the command that shows you have used RBAC for real.

### Namespaces. What they isolate and what they do not

Isolate: names of most objects, RBAC scope, ResourceQuotas, NetworkPolicies (as a boundary you can write policies around).

Do not isolate: nodes, cluster DNS zone in the sense that names are still unique per namespace not globally secret, and not a security boundary by themselves if anyone can create Pods that mount hostPath.

### Taints, tolerations, affinity

Taint on node: Pods stay off unless they tolerate it. Used for control-plane nodes, GPU nodes, dedicated pools.

Affinity: prefer or require certain nodes or co-location.

Lived: after a node upgrade, Pods pending because the new pool has a taint `workload=jobs:NoSchedule` and the Deployment was never given a toleration.

### ResourceQuota and LimitRange

Quota: namespace cap on CPU, memory, object counts.

LimitRange: default requests/limits if the Pod omitted them. This is why a Pod "mysteriously" got a 100m CPU request in some clusters.

---

## 2. kubectl and debugging (this is where they catch you)

Interviewers often skip theory and say: "A Pod is CrashLoopBackOff. What do you do?" Talk through this order, and name the commands.

### Walk me through debugging a broken Pod

1. `kubectl get pod -n <ns> -o wide` — status, restarts, node.
2. `kubectl describe pod <name> -n <ns>` — Events at the bottom. ImagePullBackOff, FailedScheduling, probe failures, FailedMount show up here first.
3. `kubectl logs <pod> -n <ns> -c <container>` and `kubectl logs --previous` if it already restarted.
4. `kubectl get events -n <ns> --sort-by=.lastTimestamp`
5. If it is Running but not serving: check Endpoints `kubectl get endpoints`, readiness, Service selector vs Pod labels.
6. If Pending: describe; look at scheduler (insufficient CPU, affinity, PVC, taints).
7. If node-level: `kubectl get nodes`, `describe node`, disk pressure, not-ready kubelet.

Weak answer: "I would check the logs." Strong answer: describe Events first, previous logs, then whether the Service has Endpoints.

### CrashLoopBackOff vs ImagePullBackOff vs Error vs Pending vs OOMKilled

CrashLoopBackOff: container started and exited. Logs and `--previous` matter. Backoff delay increases.

ImagePullBackOff: cannot pull. Wrong tag, private registry, missing imagePullSecrets, rate limit on Docker Hub.

Pending: not scheduled. Resources, PVC, taints, affinity.

OOMKilled: last state in describe. Raise limit or fix the leak; do not only "restart".

RunContainerError / CreateContainerConfigError: often a missing Secret/ConfigMap key, or a bad command.

### A Service exists but the app cannot connect

Check: selector labels match Pod labels; Pods are Ready; Endpoints object is not empty; you used the right namespace DNS; NetworkPolicy; the app listens on 0.0.0.0 not 127.0.0.1; containerPort vs Service port vs targetPort mismatch.

Lived: `targetPort` named wrong after a Helm values change. `kubectl get endpoints` showed nothing or the wrong port.

### How do you exec, copy files, port-forward?

`kubectl exec -it <pod> -c <container> -- /bin/sh`

`kubectl cp` works but is clumsy; for real debugging many teams use ephemeral debug containers: `kubectl debug`.

`kubectl port-forward svc/api 8080:80` is for a laptop, not a production access pattern.

If you have never used `kubectl debug` or a similar ephemeral container, say so. Using exec into a distroless image and failing is a real story.

### How do you see why a node is NotReady?

`kubectl describe node`. Look at KubeletNotReady, memory/disk pressure, taints, and whether Pods are evicted. SSH only if that is your operations model; many managed clusters you cannot SSH. On EKS/GKE/AKS you check the node group's status and kubelet logs via the cloud.

### How do you drain a node? What can block it?

`kubectl cordon <node>` then `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` (know why each flag is dangerous).

Blocked by: PDBs, bare Pods not owned by a controller, local data.

Lived: drain hung for 30 minutes; `kubectl get pdb -A` showed the culprit.

### How do you inspect RBAC for your current user?

`kubectl auth can-i list pods --namespace x`

`kubectl auth can-i --list`

If you work with several clusters: `kubectl config get-contexts`, `use-context`. People who have not worked on more than minikube fumble this.

### Useful get/describe habits

`kubectl get deploy,po,svc,ing -n x`

`kubectl get pod -n x -l app=api -o yaml`

`kubectl explain deployment.spec.strategy` — this command is a strong signal you read the API instead of Stack Overflow only.

---

## 3. Deployments, releases, and day-2 operations

### What happens during `kubectl apply -f deployment.yaml`?

API server admits the object (schema, RBAC, admission webhooks, quotas). etcd stores it. Deployment controller creates or updates a ReplicaSet. ReplicaSet creates Pods. Scheduler binds each Pod to a node. Kubelet pulls images and starts containers. EndpointSlices update when readiness passes.

Mention admission webhooks if you have used them (OPA/Gatekeeper, Kyverno, sidecar injectors). If you have not, say you know they can reject or mutate.

### Why is `:latest` a problem?

The kubelet may not pull again if it thinks it has the image (`imagePullPolicy` IfNotPresent vs Always). You cannot roll back by tag. Two nodes can run different bits with the same name. Use immutable tags or digests.

### How do you roll back?

`kubectl rollout undo deployment/api`

`kubectl rollout history deployment/api`

Helm: `helm rollback <release> <revision>`

Lived: undo is not enough if a ConfigMap changed in place and you did not version it; old ReplicaSet still mounts new ConfigMap. This is why checksum annotations exist.

### Zero downtime. What actually has to be true?

More than one replica; readiness probes that match real traffic readiness; preStop sleep or terminationGracePeriodSeconds long enough for the load balancer to stop sending; app handles SIGTERM; no single-replica in-memory sessions; migrations are expand/contract if schema changes.

`preStop` hook with a few seconds of sleep is a common nginx/Ingress trick so endpoints de-register before the process dies.

### Config change vs image change

Image change: new ReplicaSet, rolling update.

ConfigMap env change: no rollout unless you trigger one.

If they ask how you ship config, talk about Helm values per environment, or a GitOps repo, not "we kubectl edit in prod". kubectl edit happens, but it is an incident story, not a process.

### Jobs and migrations

A Helm pre-upgrade hook Job that runs `migrate`. Failure blocks the release if `atomic` / hook deletion policy is set.

Danger: two replicas starting migrate at once without a lock. Lived teams use one Job, or a leader, or migrate in CI against a dedicated step.

### Multi-container node issues: disk, inodes, image GC

Node disk pressure evicts Pods. ImagePull storms fill disk. People who ran clusters have seen `eviction manager` in kubelet events.

---

## 4. Helm knowledge

### What is Helm?

A package manager for Kubernetes manifests. A chart is a template plus default values. A release is a chart instance in a cluster (name + namespace + revision history). Helm 3 stores release secrets in the namespace; there is no Tiller. If someone still explains Tiller as current, they have not used Helm recently.

### Chart structure

```
mychart/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
    _helpers.tpl
  charts/          # bundled subcharts
  .helmignore
```

`templates/_helpers.tpl` defines named templates (`mychart.fullname`, labels). Interviewers sometimes ask you to read a helper and say what the resource name will be.

### `helm template` vs `helm install` vs `helm upgrade` vs `helm upgrade --install`

`template`: render locally, no cluster. You should use this in CI to review diffs.

`install`: first release.

`upgrade`: next revision.

`upgrade --install` (or `helmfile` apply): create or update. This is what pipelines use.

Always mention `--dry-run` and `helm diff` (plugin) if you have used them. `helm diff` is a lived-work signal.

### What is `values.yaml` vs `-f values-prod.yaml` vs `--set`

Layering: chart defaults, then files in order, then `--set`. Rightmost wins.

`--set` in CI is brittle (commas, nested keys). Files in git are better. `--set-string` vs `--set` matters for `"false"` vs boolean.

Lived: someone `--set replicaCount=2` and wondered why prod values file still showed 3 in git. The cluster was not git. GitOps vs Helm-in-pipeline is a follow-up they may ask.

### `helm get values`, `helm history`, `helm rollback`

`helm list -n <ns>` — releases.

`helm get values <release> -n <ns>` — what was actually applied (including `--set`).

`helm get manifest`

`helm history`

`helm rollback <release> <rev>`

If a release is in status `failed`, you often cannot upgrade until you rollback or `helm rollback` / `helm upgrade` with `--force` (force is dangerous). `--atomic` on upgrade: wait, and rollback automatically on failure.

Lived: a failed hook left status failed; next pipeline apply kept failing; fix was rollback then repair the hook Job.

### Helm hooks. Failure modes

`helm.sh/hook: pre-install,pre-upgrade`

`hook-weight` for order.

`hook-delete-policy`: before-hook-creation, hook-succeeded.

A migrate Job as a hook that never completes (image pull, DB network policy) blocks the release. `kubectl get job` in that namespace during install is a real debugging step.

### Look up functions and why `index` vs `dig`

`{{ .Values.image.tag }}` blows up if `image` is nil depending on usage.

`{{ .Values.image.tag | default "latest" }}`

People who have written charts talk about `required`, `fail`, and not using `latest`.

### `nindent`, `toYaml`, and the whitespace bugs

Helm templates are text. Wrong indent breaks YAML. `toYaml | nindent 4` is the usual pattern for nested blocks. This is a "have you written a chart" question. If you only consume charts, say that honestly and talk as a consumer.

### Subcharts and global values

Parent `values.yaml`:

```yaml
subchart:
  replicaCount: 2
global:
  imageRegistry: example.com
```

Subchart values are namespaced under the subchart name. `global` is the exception. Umbrella charts vs helmfile: see helmfile section. Lived teams often prefer helmfile over giant umbrella charts because isolation and diffs are cleaner.

### `helm dependency update`

Reads Chart.yaml `dependencies`, fills `charts/` and Chart.lock. CI should vendor or run this with a locked file. Broken CI because Chart.lock was not committed is a real incident.

### Helm and CRDs

CRDs in `crds/` install first but are not upgraded by design in older Helm behavior. Upgrading CRDs is a known sharp edge. Operators (cert-manager, prometheus-operator) document this. Mentioning it is a strong signal.

### Libraries and common charts

`bitnami/common`, company library charts. You override values, you rarely fork unless you must.

### How do you keep secrets out of Helm values in git?

helm-secrets plugin + SOPS/age, Sealed Secrets, External Secrets Operator, or inject at deploy from a vault. Plain `values-prod.yaml` with passwords is a fail. If your company did that, describe the risk and the better pattern.

### `lookup` function

Templates can query the cluster during install. Makes `helm template` in CI incomplete (no cluster). Many teams forbid `lookup` for that reason.

---

## 5. Helmfile knowledge

Helmfile is not Helm. It is a declarative wrapper to run Helm across many releases, environments, and value files. If the job uses it, they will ask this because many resumes say "Helm" and mean "I ran helm install once".

### What problem does helmfile solve?

A product is rarely one chart. You have ingress-nginx, cert-manager, an app, a worker, maybe Redis. Helmfile lists them, their chart sources, versions, namespaces, and which value files apply in `dev` vs `prod`. One `helmfile apply` converges all of it. Selectors let you apply one release.

Compared to an umbrella chart: releases are independent Helm releases (separate history, rollback, values). That is usually what you want.

### Typical `helmfile.yaml` pieces

- `repositories:` chart repos
- `helmDefaults:` wait, timeout, createNamespace, atomic, historyMax
- `environments:` per-env values and kubeContext
- `releases:` name, namespace, chart, version, values, secrets, needs, condition, labels, installed

Example shape:

```yaml
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami

helmDefaults:
  wait: true
  timeout: 300
  createNamespace: true
  atomic: true

environments:
  dev:
    values:
      - env/dev.yaml
  prod:
    values:
      - env/prod.yaml

releases:
  - name: api
    namespace: shop
    chart: ./charts/api
    version: 1.4.0
    values:
      - values/api.yaml.gotmpl
    needs:
      - shop/postgres
    labels:
      app: api
```

### `helmfile diff` vs `lint` vs `apply` vs `sync` vs `destroy`

`lint`: YAML/template checks.

`diff`: requires helm-diff plugin. Shows what would change. CI should run this on PRs.

`apply`: applies only releases that differ (plus needed deps). Preferred day to day.

`sync`: makes the cluster match the file more strictly; can be more aggressive about the full set.

`destroy`: uninstalls releases helmfile knows.

Lived: people who have used it say `helmfile -e prod diff` in PR, then `helmfile -e prod apply` in the deploy job. They mention the helm-diff plugin as a dependency of the runner image.

### Selectors

`helmfile -e prod -l app=api apply`

`helmfile -l name=ingress-nginx apply`

This is how you do not bounce the whole platform to ship one service. If you cannot explain selectors, it sounds like you never operated a multi-release file.

### Environments and `{{ .Environment.Values }}`

`helmfile -e prod` loads that environment's values. Templates in `*.gotmpl` can use environment name and values. This is how replica counts and hostnames change per env without duplicating the release list.

Lived pitfall: forgetting `-e` and applying defaults to prod, or applying with the wrong kubeContext. `helmDefaults` / environment `kubeContext` exists to prevent that. Ask for confirmation that the context is production before apply.

### `needs:` vs Helm hooks vs umbrella charts

`needs` is helmfile-level ordering: install postgres before api. It is not the same as Kubernetes readiness. Postgres can be "installed" while still crashing. Apps still need retry and probes.

### `installed: false`

Keeps the release in the file but uninstalls/omits it. Used to turn off a component in an environment without deleting it from git. Lived feature; tutorials rarely mention it.

### `condition` and `installed`

Skip a release based on environment values. Same idea.

### Secrets in helmfile

`secrets:` list of SOPS-encrypted files, needs helm-secrets. `helmfile` decrypts at apply time. Interviewers ask where the SOPS key lives (KMS, age key in CI). If you do not know, do not invent: say the encrypted files lived in git and CI had the key.

### `helmfile.d/`

Split files instead of one giant yaml. Helmfile concatenates. Used when many teams own many releases.

### `--skip-deps` and chart caches

CI performance and reproducibility. `helmfile repos` then apply. Flaky public Helm repos are a real outage; vendoring charts or using an internal registry is the mature answer.

### What helmfile does not do

It does not replace Kubernetes, GitOps controllers, or a mesh. Argo CD can even consume helmfile in some setups, but classically helmfile is imperative from CI: the pipeline has cluster credentials and applies. Be ready to compare:

- helmfile from GitHub Actions: simple, credentials in CI, drift if someone kubectl-ed
- Argo CD / Flux: cluster pulls git, better drift detection

If you used helmfile in CI, say how you detected drift (`diff` on a cron, or you did not). Honesty here scores well.

### Common helmfile errors that signal experience

- helm-diff plugin missing in CI image
- release name collision across files
- `needs` pointing at `namespace/name` incorrectly
- values file path wrong so defaults shipped to prod
- CRDs not ready; first apply of prometheus-operator failed; second apply worked
- timeout too low on `wait: true`; apply failed but cluster was still converging

---

## 6. Questions meant to test that you actually worked on it

These are the ones that decide the interview. For each: what they ask, a weak answer, a strong answer. Do not memorize the strong answer as a script. Replace names with your real cluster, chart, and incident.

### "Tell me about a Kubernetes issue you owned end to end."

Weak: "Pods were restarting and I restarted them" or a textbook probe definition with no timeline.

Strong: one incident. Symptom (error rate, not "pods unhealthy"). What you looked at (Events, previous logs, HPA, recent helmfile apply). Root cause (bad liveness hitting Redis, missing imagePullSecret after registry change, PDB blocking drain, values file with wrong tag). Fix. Prevention (checksum annotation, pipeline diff, better probe). Approximate timeline.

If you have no production incident, use a lab but label it as a lab, and still use the same debugging order.

### "How do you deploy to production today?"

Weak: "We use Kubernetes and Helm."

Strong: git branch -> CI build image digest -> PR with helmfile/helm values tag change -> `helmfile -e prod diff` -> approval -> `helmfile -e prod apply` (or Argo sync) -> `kubectl rollout status` / helm wait -> dashboard on error rate. Mention namespace, release name, who has credentials, how rollback is done (`helm rollback` or revert git).

### "A rollout is stuck at 2/3 updated. What do you look at?"

Weak: "Wait, or delete the Pods."

Strong: `kubectl rollout status`, `kubectl get rs`, new Pods not Ready, describe new Pods (probes, image pull, crash), Events, PDB, insufficient resources for maxSurge, `progressDeadlineSeconds`. Check if readiness path depends on a migration that has not run.

### "How do you inject configuration and how do you ship a config change?"

Weak: "ConfigMaps."

Strong: ConfigMap or values rendered by Helm. Mounted file vs env. Config change does not restart; we add a checksum annotation in the chart so upgrades roll Pods. Sensitive values from SOPS/ExternalSecrets, not in git. We do not kubectl edit in prod except emergencies, and then we write it back to git.

### "How do you handle secrets?"

Weak: "Kubernetes Secrets."

Strong: Secrets objects exist, but they are not encryption. We keep encrypted files in git (SOPS + KMS) or a vault, sync with External Secrets. RBAC: app ServiceAccount can get only its secret. imagePullSecrets for the registry. We rotated X because it leaked in logs / was in an old values file.

### "App is slow in the cluster but fine on a laptop. What now?"

Weak: "Scale the Deployment."

Strong: define slow (p99, CPU, I/O). `kubectl top`, throttling from CPU limits, noisy neighbor, node disk, N+1 queries still in the app, DNS, downstream latency, missing connection pool, readiness not related. Check if CPU limits are throttling. Check if the Service is hitting other-AZ traffic. Scaling can hide a query problem.

### "We need to take a node out. Walk me through it."

Weak: "Delete the node."

Strong: cordon, drain with awareness of DaemonSets and PDBs, wait for Pods elsewhere, confirm Endpoints, then terminate the instance if it is a cloud node group. If drain blocks, find PDB or singleton. Do not drain all nodes in a zone at once.

### "How do you run migrations?"

Weak: "Init container."

Strong: a Helm hook Job or a CI step with network access to the DB, one active migrate, expand/contract for compatibility with old Pods still running during the roll. Init container migrate on every Pod is a race. We learned that when two replicas ran migrate together.

### "How do you expose this to the internet?"

Weak: "NodePort."

Strong: Ingress or Gateway in front of ClusterIP. TLS at the Ingress or at the cloud LB. Source IP / PROXY protocol if we needed it. WAF if the company had one. Internal apps stay ClusterIP. We do not give every Service a LoadBalancer because of cost and of open security groups.

### "How do you restrict traffic between namespaces?"

Weak: "Namespaces isolate traffic."

Strong: they do not. NetworkPolicy (or mesh). Default deny in the namespace, allow from ingress controller and from specific app labels, allow DNS to kube-system. We broke DNS the first time and had to allow port 53.

### "How do you know a deploy is good?"

Weak: "Pods are Running."

Strong: rollout complete, new ReplicaSet Ready, Endpoints point at new Pods, smoke test, p99 and error rate on the dashboard for N minutes, logs have no crash loop. Running is not Ready. Ready is not "the business transaction works".

### "Show me the kubectl commands you use every day."

They want muscle memory, not a complete list:

```
kubectl get po,deploy,svc,ing -n shop
kubectl describe pod ...
kubectl logs ... --previous
kubectl get events -n shop --sort-by=.lastTimestamp
kubectl get endpoints ...
kubectl auth can-i ...
kubectl rollout status deploy/api -n shop
kubectl exec -it ... -- sh
```

Helm/helmfile:

```
helm list -n shop
helm history api -n shop
helm get values api -n shop
helmfile -e prod -l app=api diff
helmfile -e prod -l app=api apply
```

If you hesitate on `--previous`, `endpoints`, or `helm history`, they will assume limited exposure.

### "What does this YAML fragment do?" (they paste a probe or a Service)

Read it out loud. Call out: named port vs number, `targetPort`, `failureThreshold`, `initialDelaySeconds`. If `livenessProbe.httpGet.path` is `/health` and that path hits the DB, say the operational risk.

### "Have you written a Helm chart or only consumed one?"

Answer truthfully. Consumer: you override values, bump versions, read `NOTES.txt`, know `helm get values`. Author: helpers, checksum annotations, hooks, `lookup` avoided, subchart globals, `required` on values. Overclaiming chart authorship is easy to test with a five-line template.

### "Helm upgrade failed in CI. What now?"

Strong: look at the Job/hook Pod, `helm status`, `helm history` (failed revision), `kubectl describe` the new ReplicaSet. If `--atomic`, it should have rolled back; confirm that it did. Do not keep pressing apply. Fix the rendered manifest (`helmfile diff` / `helm template`) locally.

### "Why helmfile instead of a bunch of helm commands in a shell script?"

Strong: declarative list, environments, selectors, needs, one diff in PR, values layering, repos pinned. Scripts drift; the file is the inventory of what should be in the cluster.

### "If you change a value in helmfile, what exactly changes in the cluster?"

Strong: only the Helm release whose rendered manifests changed, unless I apply without selectors. Helm computes a three-way merge against the previous release (Helm 3). Objects you hand-edited in the cluster may be overwritten. Objects you added manually are not deleted unless you use extra tools. Drift is real.

### "How do you pin chart versions?"

Strong: `version:` on the release in helmfile, Chart.lock for local charts' dependencies. We do not use `version: "*"` in prod.

### "How do you handle multiple environments?"

Strong: same chart, different environment values (`env/dev.yaml`, `env/prod.yaml`), different kubeContext, different replica counts, hostnames, and resources. Maybe `installed: false` for a debug tool in prod. We do not copy the whole helmfile per environment.

### "What Kubernetes version are you on? What did you have to change on upgrade?"

If you cannot name a version family (1.28, 1.29, …) and one deprecated API (`policy/v1beta1` PDB, `batch/v1beta1` CronJob in older upgrades, Ingress `networking.k8s.io/v1`), they doubt production work. Look up what you actually ran and remember it.

### "How do you run cron jobs in the cluster?"

CronJob -> Job -> Pod. Concurrency policy (`Forbid` vs `Allow`). StartingDeadlineSeconds. A Job that never finishes fills the namespace; you set history limits. Missed schedules after control plane downtime. Lived: a CronJob timezone (`spec.timeZone`) surprise.

### "How do you debug CrashLoop on a distroless image?"

Cannot exec a shell. Logs, `--previous`, copy a debug image with `kubectl debug`, or add a temporary sidecar. Raising log level via values and rolling. This answer separates operators from tutorial users.

### "What happens when you delete a namespace?"

All namespaced objects go. Finalizers can stick the namespace in Terminating (a CRD webhook gone, a stuck PV). Lived people have had to edit finalizers carefully. It is dangerous; say you treat it as last resort.

### "Do you use GitOps?"

If yes: Argo/Flux, app-of-apps or helmfile generator, auto-sync or manual, what happens on a bad sync. If no: helmfile from CI, and the weakness is drift. Either is fine. Pretending to use Argo without being able to explain sync waves is not.

---

## 7. Scenario questions (work them on paper)

### Scenario A: ImagePullBackOff after a pipeline change

Likely: tag does not exist, digest wrong, registry credentials expired, imagePullSecrets not in the new namespace, Docker Hub rate limit.

Commands: describe Pod (Events show 401/404), try to pull from a debug Pod if policy allows, check ServiceAccount imagePullSecrets.

### Scenario B: 503 from Ingress after deploy

New Pods not Ready; Service selector no longer matches labels (Helm fullname change); Ingress path wrong; probes failing; NetworkPolicy; TLS secret missing so some controllers drop the route.

Commands: get ing, describe ing, get endpoints, describe new Pods, curl from a debug Pod to ClusterIP.

### Scenario C: HPA will not scale above 1

No resources.requests; metrics-server missing; HPA in another namespace; custom metric not exported; maxReplicas=1.

Commands: `kubectl get hpa`, `kubectl describe hpa`, `kubectl top pod`, check metrics-server in kube-system.

### Scenario D: helmfile apply timed out, `atomic: true`

Helm rolled back. Read history. The wait condition was readiness. Why not ready: migrate hook, probe, missing secret. Increase timeout only after you know it was still pulling a 1GB image, not as the first fix.

### Scenario E: One replica of a StatefulSet is Pending

PVC in a zone with no nodes; node selector; last volume still attached (cloud); resource request too big. describe Pod and PVC. Do not delete PVC until you know you can lose the disk.

### Scenario F: After adding NetworkPolicy, app cannot talk to API and cannot resolve DNS

Allow egress 53 to kube-system; allow egress to the API Pods' labels and port; allow ingress on the API from the client's labels. Test with a debug Pod and `nslookup kubernetes.default`.

---

## 8. Development-leaning questions (if the role is not pure DevOps)

You still need these. Interviewers for backend roles use them to see if you wrote apps that survive Kubernetes.

### What must the application implement?

Listen on 0.0.0.0. Read config from env or files. Structured logs to stdout. Handle SIGTERM (stop taking work, finish in-flight, exit 0). Health on a cheap path. Timeouts on outbound calls. Do not assume a local disk is durable unless a PVC is mounted.

### Why did connections drop every deploy?

No graceful shutdown; `terminationGracePeriodSeconds` too low; no preStop; kube-proxy/Ingress still routing; long requests (WebSockets, file uploads).

### Should the app know it is on Kubernetes?

Downward API for node name, Pod IP, namespace. In-cluster ServiceAccount token for talking to the API (only if it must). Most business apps should only know DNS names of Services.

### How do you pass the DB URL?

Secret or ExternalSecret -> env. Not a baked-in image. Connection pooling sized for replicas * pool size vs database max_connections. This is a classic "you actually ran it" question: 5 replicas * 100 connections = database refused.

---

## 9. Short-answer drill (keep these tight in an interview)

What restarts a container? kubelet, because of liveness failure or native process exit, not the Deployment controller directly.

What creates Pods? ReplicaSet (for Deployments), not the Deployment itself.

Does a Service load balance to not-Ready Pods? No, unless you set `publishNotReadyAddresses` (headless/StatefulSet cases).

Who allocates ClusterIP? the API / control plane, from the service CIDR.

Can two Deployments share one Service? Yes if labels match. That is a homemade canary.

What stores Helm 3 release state? Secrets (or configmaps if configured) in the release namespace.

Does helmfile require Helm? Yes. It shells out to helm. Also typically helm-diff.

Helm 2 vs 3 in one line? No Tiller; releases are per-namespace; three-way strategic merge.

What is a container runtime today? containerd commonly; Docker-shim was removed. `docker` on a node is not how kubelet runs Pods anymore.

---

## 10. How to practice so your answers sound real

1. Take the Phase 4 assignment in ROADMAP.md and actually break it: bad probe, wrong selector, missing secret, failed helm hook.
2. Write a one-page incident for each break: commands, output, fix.
3. Author a small chart with a checksum annotation and a pre-upgrade Job. Install, fail the Job on purpose, read `helm history`, rollback.
4. Write a helmfile with two releases and two environments. Use `-l` and `needs`. Run `diff` then `apply`.
5. Before interviews, speak three stories out loud (2 minutes each): a stuck rollout, a probe/outage, a Helm/helmfile failure.

Stories beat definitions. Definitions still matter; they are section 1–5. The offer is usually decided in section 6.
