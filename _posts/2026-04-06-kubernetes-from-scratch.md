---
layout: post
title: Kubernetes Makes No Sense Until You Need It
comments: true
---

I've spent the last several years building a Kubernetes development platform used daily by upwards of 100 engineers. Before that, I built an operator, a cluster portal, and helped migrate a monolithic server into containers. I've seen what K8s looks like in production.

I also know what it feels like when you read the docs for the first time and nothing sticks. Pod. ReplicaSet. Deployment. StatefulSet. The docs tell you what each one is, but they read like a glossary. You memorize it, close the tab, and forget everything by Monday. My eyes glazed over and stayed that way for a while.

It took me a long time to internalize any of it. Every concept in Kubernetes exists because someone hit a real operational problem and needed a reusable answer. But until you hit that problem yourself, YAGNI. So let's build something and see what breaks.

## The app

You build a small Go service for your team. It's a simple event collector. Other services POST events to it, and you can GET recent events back. A few endpoints, an in-memory slice, nothing fancy.

```go
package main

import (
    "encoding/json"
    "net/http"
    "sync"
    "time"
)

type Event struct {
    Source string    `json:"source"`
    Action string    `json:"action"`
    Time   time.Time `json:"time"`
}

// Store everything in memory. What could go wrong?
var (
    events []Event
    mu     sync.Mutex
)

func main() {
    http.HandleFunc("/events", func(w http.ResponseWriter, r *http.Request) {
        mu.Lock()
        defer mu.Unlock()

        switch r.Method {
        case "POST":
            var e Event
            if err := json.NewDecoder(r.Body).Decode(&e); err != nil {
                http.Error(w, err.Error(), http.StatusBadRequest)
                return
            }
            e.Time = time.Now()
            events = append(events, e)
            w.WriteHeader(http.StatusCreated)
        case "GET":
            json.NewEncoder(w).Encode(events)
        default:
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        }
    })
    http.ListenAndServe(":8080", nil)
}
```

You build the binary. You run it on a server. Your team starts sending events to it. It works. Life is good.

## It crashes at 2am

The in-memory slice just grows. Every event is appended and never evicted. After a few weeks, the process eats enough memory that the OS kills it. Nobody notices until morning. All the events are gone.

That is really two problems. The application needs to stop retaining events forever, and the events you do retain need to survive a restart. You add a retention policy and move them into SQLite. Memory stays bounded. Recent events survive.

A week later, an unrelated bug crashes the process. The data is still there, but the service is not. You restart it by hand, then write a systemd unit with `Restart=on-failure`.

Systemd does not know whether the collector is correct or healthy. It only knows that the process exited. For one service on one machine, that may be all you need. The industry ran on systemd, cron, and shell scripts for a long time. Plenty of systems still should.

## Containers

Now you're iterating on the app. You add filtering, change the retention policy, and deploy a new binary. Every release means copying files, restarting the service, and hoping the machine still looks like the machine you tested on.

A container image gives the release a versioned, repeatable artifact. Build it once, ship that exact image to another machine, and start it the same way every time.

```dockerfile
FROM golang:1.26-alpine3.24 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -o /collector .

FROM alpine:3.24
RUN adduser -D -u 10001 collector
COPY --from=build /collector /collector
USER collector
EXPOSE 8080
ENTRYPOINT ["/collector"]
```

Run it with `docker run --restart=unless-stopped`. If the process exits, Docker starts the container again. Deploying to a new server is mostly `docker pull` and `docker run`. Same image, same entrypoint, fewer mystery differences between releases.

Life is good again. But everything still lives inside one process on one machine.

## It grows up

People want more than the most recent events. Another developer builds a small reporting service that queries them, groups them by source, and renders a dashboard.

SQLite was a good answer when one collector owned one local file. You could make two processes on the same server coordinate around that file, but now the database location dictates where both applications run. What they need is a database service both can reach over the network.

You move the events into Postgres. Now you have a collector, a reporting service, and a database to configure, connect, and start in the right order.

Docker Compose handles this cleanly. One YAML file, all three services, one command to bring them up.

```yaml
services:
  collector:
    build: .
    ports:
      - "8080:8080"
    environment:
      DB_HOST: postgres
      DB_PASSWORD: events
    depends_on:
      postgres:
        condition: service_healthy

  reports:
    build: ./reports
    ports:
      - "8081:8080"
    environment:
      DB_HOST: postgres
      DB_PASSWORD: events
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:18
    environment:
      POSTGRES_PASSWORD: events
    volumes:
      - pgdata:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

`docker compose up` and everything starts together. This is genuinely great for development and small deployments. A lot of systems never need more than this.

But Compose still has one view of one machine. Your problem is about to become a fleet problem.

## It gets popular

Another team hears about your event collector and starts sending deploy webhooks to it. Then QA hooks it up to their test runner. Traffic doubles. Response times go up. One collector cannot keep up.

The reporting service is fine, and there should still be one shared source of truth for events. You leave Postgres where it is and run only another copy of the collector on a second server. Both collectors connect to the same database. You put a load balancer in front of them, and traffic splits.

Then you add a third server for redundancy. Now you're checking whether each machine is alive, updating collector containers one at a time, and repairing the load balancer when an address changes. You write a script. The script gets bigger. A deploy fails halfway through and leaves different versions running on different machines. You discover that your rollback script is mostly a comment that says `TODO`.

This is the moment Kubernetes starts to make sense. Not because containers suddenly became complicated, but because the live system has behavior you can no longer keep in your head. You need somewhere to record what should be running, somewhere to record what is actually running, and software that keeps comparing the two.

## What Kubernetes actually is

You install Kubernetes across the three machines, turning them into a **cluster**. Kubernetes calls each machine a **node**.

In this simplified cluster, one node runs the **control plane**: the API server, scheduler, and controllers. The other two run your application workloads. Each worker runs a small agent called the **kubelet**, which manages containers on that machine.

Instead of connecting to each worker and starting containers yourself, you use the **API server** to tell the cluster which containers should be running. Kubernetes stores that request as a resource.

The smallest resource you can create to run those containers is called a **Pod**. When you create one, the **scheduler** looks at its requirements, finds a worker node with room, and assigns the Pod to that node.

The kubelet on that node sees the assignment, starts the requested containers, and reports what happened back to Kubernetes.

Kubernetes gets much of its behavior from specialized control loops called **controllers**. A controller watches the resource types it understands, compares what you asked for with what currently exists, and makes changes to close the gap. Many controller-managed resources expose a `spec` describing intent and a `status` describing what the controller observed, but not every resource has both and there is not a one-to-one controller behind every kind.

Think of a thermostat. You set a desired temperature. The thermostat observes the room and turns equipment on or off. A controller follows the same pattern: observe, compare, act, and repeat.

That is the general model. The first thing you create is simpler.

## The Pod

You only want to run the collector container. A Pod is the smallest workload resource that can do that. It can contain multiple tightly coupled containers, but most application Pods have one main container, so we'll start there.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: collector
  labels:
    app: collector
spec:
  containers:
    - name: collector
      image: registry.example.com/collector:v1
      ports:
        - name: http
          containerPort: 8080
      env:
        - name: DB_HOST
          value: postgres.internal.example.com
        - name: DB_PASSWORD
          value: not-a-real-password
```

That fake password is standing in for configuration you still need to handle properly. One problem at a time.

Apply the resource, then ask Kubernetes where the Pod landed:

```bash
kubectl apply -f pod.yaml
kubectl get pod collector -o wide
```

The API server accepts the object. The scheduler finds an eligible node and binds the Pod to it. The kubelet on that node sees the assignment, asks the container runtime to pull the image, and keeps the Pod's containers aligned with the Pod spec.

If the collector process exits, the kubelet restarts that container according to the Pod's restart policy. The Pod was not rescheduled and a controller did not create a new Pod. The same kubelet repaired the containers inside the same Pod.

That distinction sounds pedantic until the whole node disappears.

## The server dies

Your collector is running on `node-2`. Then `node-2` loses power. The kubelet is gone with it, so there is nobody on that node to restart the container.

The control plane eventually marks the node unhealthy and the Pod for deletion. But this is a bare Pod. Nothing owns it. Kubernetes knows that a Pod existed; it has no higher-level instruction saying another one must exist somewhere else.

You do not need to pull a power cable to see the same limitation:

```bash
kubectl delete pod collector
kubectl get pods
```

The Pod stays gone. Pods are scheduled once in their lifetime. A dead Pod is not picked up and moved to another node. To get replacement behavior, you need an object whose desired state is larger than one named Pod.

## ReplicaSets: "I want three of these, always"

A ReplicaSet contains a Pod template and a replica count. Its controller continually asks a simple question: how many Pods match, and how many should?

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: collector
spec:
  replicas: 3
  selector:
    matchLabels:
      app: collector
  template:
    metadata:
      labels:
        app: collector
    spec:
      containers:
        - name: collector
          image: registry.example.com/collector:v1
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: DB_HOST
              value: postgres.internal.example.com
            - name: DB_PASSWORD
              value: not-a-real-password
```

Apply it and watch:

```bash
kubectl apply -f replicaset.yaml
kubectl get pods --watch
```

The ReplicaSet controller creates three Pods from the template. Each new Pod becomes a request for the rest of Kubernetes. The scheduler chooses a node for it, and that node's kubelet starts its container. Each part does one job.

Delete one of the Pods in another terminal. A replacement appears:

```bash
kubectl delete pod <one-of-the-collector-pods>
```

Now desired replicas is 3 and actual replicas returns to 3. If a Pod disappears with a failed node, the ReplicaSet creates another. No SSH. No server list in a shell script.

Three replicas do not automatically mean three different nodes. The scheduler considers resources and its configured policies, but it is allowed to place multiple replicas together. When failure-domain spread matters, you add topology spread constraints or pod anti-affinity. Kubernetes makes placement controllable; it does not infer every availability requirement you forgot to state.

You normally do not create ReplicaSets directly. We did it here because it exposes exactly what the controller adds: the durable instruction that a certain number of matching Pods should exist.

## Pushing a new version

Your collector has a bug. Events with empty sources are being stored instead of rejected. You fix it, build `collector:v2`, and push the image.

Now what? You have three Pods running v1. A ReplicaSet knows how to maintain a count, not how to conduct a release. Changing its Pod template does not rewrite the Pods it already owns. Existing Pods keep running v1; only Pods created later use v2.

You can prove it by changing the image in `replicaset.yaml`, applying it, and inspecting the existing Pods. Nothing rolls forward.

You could delete Pods one at a time and wait for replacements. You could create a second ReplicaSet, scale it up, and scale the first one down. You could write a script that coordinates both and stops when the new version is broken.

You have just discovered another behavior worth giving to a controller.

## Deployments: controlled rollouts

A Deployment owns ReplicaSets and adds release history and rollout strategy. When its Pod template changes, the Deployment creates a new ReplicaSet for the new version, gradually scales it up, and scales old ReplicaSets down.

Delete the teaching ReplicaSet and replace it with the resource you would ordinarily use for this application:

```bash
kubectl delete replicaset collector
kubectl apply -f deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: collector
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: collector
  template:
    metadata:
      labels:
        app: collector
    spec:
      containers:
        - name: collector
          image: registry.example.com/collector:v1
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: DB_HOST
              value: postgres.internal.example.com
            - name: DB_PASSWORD
              value: not-a-real-password
```

Then update the image while watching the resources it manages:

```bash
kubectl get replicasets --watch
kubectl set image deployment/collector collector=registry.example.com/collector:v2
kubectl rollout status deployment/collector
```

A second ReplicaSet appears. The Deployment scales the new one up and the old one down. The Pods emerge from that behavior, and the ownership chain is now visible:

`Deployment → ReplicaSet → Pod → container`

With this strategy, the Deployment may run one extra Pod during the rollout and does not intentionally reduce the available count below three. That is more precise than saying a rolling update always keeps exactly three Pods or guarantees zero downtime. Availability depends on the rollout settings, spare capacity, and whether Kubernetes can tell when the application is actually ready.

If v2 is worse than the bug it fixed, `kubectl rollout undo deployment/collector` restores the previous Pod template, provided that revision is still in the Deployment's retained history.

## Services: finding your Pods

You have three Pods, but each has its own IP address. Those addresses change as Pods are replaced. Your existing load balancer should not have to watch the cluster and maintain its own backend list.

A Service gives the workload a stable virtual address inside the cluster. Its selector defines which Pods are eligible backends. Because this is your own three-machine cluster, a NodePort also gives the load balancer a stable port on every node.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: collector
  labels:
    app: collector
spec:
  type: NodePort
  selector:
    app: collector
  ports:
    - name: http
      port: 80
      targetPort: http
      nodePort: 30080
```

Anything in the same namespace can reach `http://collector`. Your existing load balancer can send traffic to port `30080` on the cluster nodes instead of tracking Pod addresses.

The Service itself is not secretly scanning packets for labels. The EndpointSlice controller watches Services and Pods and writes EndpointSlice objects containing the current backend addresses. A service proxy or networking implementation watches the Service and EndpointSlices and programs the data plane.

Watch that relationship:

```bash
kubectl get endpointslices -l kubernetes.io/service-name=collector --watch
```

Delete a collector Pod. Its old address disappears, the ReplicaSet creates a replacement, and the replacement's address appears. You declared a durable relationship with a selector. Several control loops keep turning that relationship into current routing state.

## The rollout that isn't ready

Version 3 adds a set of validation rules loaded from Postgres. The process begins listening on port 8080 immediately, but it needs another 20 seconds before it can serve real requests.

The Deployment sees a running container and continues the rollout. Without another signal, the Pod is considered ready, its address enters the Service's usable endpoints, and users receive errors during those 20 seconds. `maxUnavailable: 0` did exactly what you asked. The problem is that Kubernetes could not distinguish **started** from **ready**.

You add a `/readyz` endpoint that returns success only after the rules have loaded, then add a readiness probe to the Deployment's container:

```yaml
readinessProbe:
  httpGet:
    path: /readyz
    port: http
  periodSeconds: 5
  failureThreshold: 2
```

The kubelet performs the check. While it fails, the container keeps running but the Pod is not a ready backend for the Service. When it succeeds, the EndpointSlice becomes usable for traffic and the Deployment can count the Pod as available.

Readiness did not make the application healthy. It gave Kubernetes an application-specific fact it could not infer.

## Namespaces: giving names a scope

QA has been sending test events into the same collector as everyone else. Now they want an isolated staging environment where a broken rollout cannot pollute production data.

You could invent names such as `collector-production` and `collector-staging` for everything, but the cluster also contains the reporting service, monitoring, and another team's applications. Names and policies need a consistent scope.

Namespaces let `collector` exist independently in production and staging:

```bash
kubectl create namespace production
kubectl create namespace staging

kubectl apply -f deployment.yaml -n production
kubectl apply -f service.yaml -n production

kubectl apply -f deployment.yaml -n staging
kubectl apply -f service.yaml -n staging
```

Inside `production`, the Service is still simply `collector`. From another namespace, it can be addressed as `collector.production`; its full cluster DNS name is usually `collector.production.svc.cluster.local`.

Namespaces are often described as folders, which is useful until it isn't. They are API scopes, not hard security boundaries. Pods in different namespaces may still communicate unless NetworkPolicies and the network implementation enforce restrictions. RBAC controls who can do what. ResourceQuota and LimitRange control consumption. Admission policy controls what may be created.

A namespace gives those policies somewhere coherent to apply. It does not provide isolation merely by existing.

## ConfigMaps and Secrets: the environments diverge

Production and staging run the same collector image, but they should not use the same database address, credentials, or log level. The values copied into the Pod, ReplicaSet, and Deployment examples have become both repetitive and dangerous.

ConfigMaps hold non-confidential configuration. Here is the production version:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: collector-config
  namespace: production
data:
  DB_HOST: "postgres.internal.example.com"
  DB_NAME: "events"
  LOG_LEVEL: "info"
```

Secrets hold values Kubernetes intends to treat as sensitive. This example uses a fake password for illustration; committing a real Secret manifest to Git is still committing the secret.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: collector-secrets
  namespace: production
type: Opaque
stringData:
  DB_PASSWORD: "not-a-real-password"
```

The values in a Secret object's `data` field are base64-encoded. Base64 is not encryption. In a self-managed cluster, Secret data is stored unencrypted in etcd unless encryption at rest is configured. RBAC, encryption at rest, audit controls, and careful distribution are what protect the value.

Teams often integrate a cloud secrets manager or Vault. One pattern synchronizes external values into Kubernetes Secrets. Another mounts values through a CSI provider without creating a long-lived Secret object. The right choice depends on the threat model, but an external source does not magically make an ordinary Kubernetes Secret harmless once the value has been copied into it.

The Deployment can now refer to configuration by name:

```yaml
containers:
  - name: collector
    image: registry.example.com/collector:v3
    envFrom:
      - configMapRef:
          name: collector-config
    env:
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: collector-secrets
            key: DB_PASSWORD
```

Create different ConfigMaps and Secrets in `staging`, but deploy the same image. Image and configuration now have separate lifecycles.

That does not mean the running process changes whenever an environment-backed ConfigMap or Secret changes. Environment variables are fixed when the container starts, so existing Pods need to be replaced to receive new values. A mounted configuration volume behaves differently, and the application still needs to notice and reload the file.

## Do we need a StatefulSet?

You may have noticed that Postgres never moved into the cluster. That was intentional.

The collector Pods are interchangeable. Their durable state lives in Postgres, which can remain on a dedicated machine or move to a managed database service. Running the application on Kubernetes does not require moving every dependency into Kubernetes too.

So this system does not need a StatefulSet. That is the point.

A StatefulSet solves a more specific problem. Some distributed workloads need stable member identities, predictable ordinal names such as `database-0`, and a durable one-to-one relationship between each member and its own volume. A StatefulSet provides those building blocks. It does not invent replication, backups, failover, consistency, or safe upgrades for the software inside it.

If you later decide to operate a replicated Postgres cluster inside Kubernetes, you may install a database operator and create a higher-level resource such as a `PostgresCluster`. The operator can observe both Kubernetes objects and the live database, then create StatefulSets, Services, Secrets, and persistent volume claims underneath.

That is a better way to encounter a StatefulSet in this story: not as the thing you write because "database," but as one implementation resource that emerges from a controller managing stable database members. You can inspect it, follow its ownership, and understand why it exists from the behavior it supports.

## Gateway API: the load balancer becomes a config file

The collector API is exposed through NodePort `30080`. The reporting service moves into the cluster and gets another NodePort. Then another team needs to expose an application too.

Your external load balancer now contains a growing map of hostnames, node addresses, and ports. Application deploys require someone to edit infrastructure outside the cluster. You have rebuilt centralized HTTP routing, but its desired state lives in a load balancer console instead of beside the applications.

What you want is a shared front door that routes by hostname or path. Ingress filled that role for years. Gateway API is its successor and models the shared infrastructure and application routes as separate resources.

Gateway API is not implemented by the core Kubernetes binaries. You install its CRDs and a compatible implementation, and that implementation normally provides one or more GatewayClasses. Suppose the platform exposes a class named `internet`.

The platform team creates a Gateway in `gateway-system` and allows Routes from application namespaces to attach:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public
  namespace: gateway-system
spec:
  gatewayClassName: internet
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
```

The collector team creates its Route in `production`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: collector
  namespace: production
spec:
  parentRefs:
    - name: public
      namespace: gateway-system
  hostnames:
    - collector.example.com
  rules:
    - backendRefs:
        - name: collector
          port: 80
```

`collector.example.com` now routes to the `collector` Service in the Route's namespace. The reporting service and other teams can attach their own HTTPRoutes to the same Gateway without editing the collector's file.

The clean part is ownership. The platform team owns the GatewayClass and Gateway: how traffic enters the cluster. Application teams own HTTPRoutes: where their traffic goes. The implementation's controller reconciles those resources into whatever load balancer, proxy, or cloud infrastructure actually carries the traffic.

## Liveness: the process is alive but stuck

At 2am, the collector fails again. This time the process does not exit. A bug deadlocks its request handling, so the container remains running while `/events` hangs.

The readiness probe removes that replica from Service traffic, which protects users, but nothing repairs it. Docker's restart policy cannot help because the process is still alive. The kubelet needs a signal that this instance cannot recover without a restart.

You add a `/livez` endpoint and a liveness probe:

```yaml
livenessProbe:
  httpGet:
    path: /livez
    port: http
  periodSeconds: 10
  failureThreshold: 3
```

After repeated failures, the kubelet restarts the container in the same Pod. It does not delete the Pod and ask the Deployment for a new one.

Liveness is not "can every dependency answer me right now?" If `/livez` fails whenever Postgres has a bad minute, every collector may restart together and turn one outage into two. Liveness should detect a state the process cannot recover from without a restart. Readiness should answer whether this instance should receive traffic.

Kubernetes also has a startup probe. The collector starts quickly, so you do not need one. If a future version legitimately needs a long initialization, a startup probe can delay liveness and readiness checks until startup succeeds. Again, add it when the application demonstrates that problem, not because the field exists.

## Observability: seeing what the system is doing

The latest rollout is technically healthy, but users say the collector feels slower. The Pods are ready. None are restarting. The replica that first showed the problem has already been replaced.

`kubectl logs` is useful, but searching one container at a time cannot reconstruct what happened after a Pod disappears. Health probes only tell the kubelet whether to restart a container or route traffic to it. They do not tell you how many events are being processed, how long requests take, or whether error rates changed after the deploy.

Your application can expose those measurements as Prometheus metrics:

```text
collector_events_total{source="qa-runner"} 48291
collector_events_total{source="deploy-hooks"} 12037
collector_request_duration_seconds_sum 4823.4
collector_request_duration_seconds_count 60328
collector_errors_total 142
process_resident_memory_bytes 94371840
```

The cluster and runtime expose useful infrastructure metrics too, but only the application can explain its own work in domain terms. A CPU graph cannot tell you whether deploy webhooks are being rejected.

You install the Prometheus Operator because you want Prometheus itself configured through Kubernetes resources. The operator adds kinds such as `Prometheus`, `ServiceMonitor`, `PodMonitor`, and `PrometheusRule`. These are not built-in Kubernetes kinds; they are CRDs understood by the operator.

Because the collector Service has the label `app: collector` and a named port `http`, a ServiceMonitor in the same namespace can select it:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: collector
  namespace: production
spec:
  selector:
    matchLabels:
      app: collector
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

The Prometheus Operator selects ServiceMonitor objects according to the `Prometheus` resource configuration and generates Prometheus scrape configuration. Prometheus service discovery then follows the current Service endpoints. The exact labels and namespaces it discovers depend on how that Prometheus resource is configured; installing the CRD alone is not enough.

Every 15 seconds in this example, Prometheus records another sample. Counter values over time become rates. Histograms become latency distributions. Grafana turns those time series into dashboards. Alerting rules turn them into pages you hopefully do not receive at 2am.

Centralized logging gives you the other half of the story. Loki, Elasticsearch, or a cloud logging service can collect container output before an ephemeral Pod disappears. Tracing becomes useful when a request crosses several services and no single log explains where the time went.

## And now you can see a platform emerging

Look at what happened. You started with a Go binary on a server. Each new abstraction arrived after the live system exposed a new problem:

- The process exits? **systemd**, then the container runtime, restarts it.
- The node disappears? A **ReplicaSet** creates a replacement Pod and the scheduler places it.
- Need a controlled release? A **Deployment** coordinates old and new ReplicaSets.
- Backends keep changing? A **Service** and **EndpointSlices** provide stable discovery and current endpoints.
- The new process is running but cannot serve? A **readiness probe** keeps traffic away.
- Production and staging need the same names? **Namespaces** give those names a scope.
- Their settings and credentials differ? **ConfigMaps and Secrets** keep configuration outside the image.
- Several applications need one front door? **Gateway API** separates infrastructure from routes.
- A process is alive but cannot recover? A **liveness probe** tells the kubelet to restart its container.
- You need to know what happened? **Metrics, logs, and traces** preserve the evidence.

And what did not happen matters too. The collector did not need stable replica identities, so you did not create a StatefulSet. It did not start slowly, so you did not add a startup probe. Those resources make sense now, but they have not earned a place in the running system.

Not every resource has its own controller, and not every controller directly manages a process. The recurring pattern is more precise than that: durable API objects record intent and observation; specialized control loops react to them; each component changes the part of the world it owns; status and events tell the next loop what happened.

I used to watch the TGI Kubernetes streams where Joe Beda would demo things like running a Roblox server on K8s and explain the architecture and history along the way. What struck me was how intentional the design is. The abstractions are layered because the operational problems are layered. You do not need all of them on day one.

Kubernetes gives you mechanisms. A platform begins to emerge when you turn those mechanisms into a paved path: a standard Deployment, configuration conventions, safe rollout defaults, shared routing, policies, dashboards, alerts, and perhaps higher-level resources backed by your own controllers.

That is not built from a grand architecture diagram or by installing every operator you can find. You deploy something. It breaks in a new way. You decide whether the problem is recurring and worth encoding. Then you add the smallest resource, policy, or controller that can remember the answer.

That's been my experience. I've been solving the same problem for years: make it easier for engineers to run and ship software. The tools changed. The scale changed. But the problem never did. Each layer grew out of solving the next version of it.

If you're learning Kubernetes, don't memorize the glossary. Deploy something. Observe what it does. Break it. Fix it. That is how the abstractions become obvious, and how platforms actually get built.
