---
layout: post
title: Kubernetes Makes No Sense Until You Need It
comments: true
---

I've spent the last several years building a Kubernetes development platform used daily by upwards of 100 engineers. Before that, I built an operator, a cluster portal, and helped migrate a monolithic server into containers. I've seen what K8s looks like in production.

I also know what it feels like when you read the docs for the first time and nothing sticks. Pod. ReplicaSet. Deployment. StatefulSet. The docs are thorough. They clearly tell you what each one is. But they read like a glossary. You memorize it, close the tab, and forget everything by Monday. My eyes glazed over and stayed that way for a while.

It took me a long time to internalize any of it. Every concept in Kubernetes exists because someone hit a real problem and needed a solution. But until you hit that problem yourself, YAGNI. So let's just build something and see what breaks.

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

The in-memory slice just grows. Every event appended, never evicted. After a few weeks, the process eats enough memory that the OS kills it. Nobody notices until morning. All the events are gone.

Swap the in-memory slice for SQLite. Now events survive restarts. But the process is still dying, and needs to be restarted.  

You write a systemd unit to restart it automatically. You add a cron job that curls the health endpoint every minute. It works. This was just how you ran services. Systemd, cron, maybe a shell script that checks the PID file. The industry ran on this stuff for a long time and it was fine until it wasn't.

## Containers

The systemd setup keeps it running. But now you're iterating on the app. You fixed the OOM issue with SQLite. You want to add filtering. You want to add a retention policy. Every change means building on the server, restarting the service, checking the logs. If you want to test on a clean machine, you're provisioning a VM and installing dependencies from scratch.

Containers speed all of this up. You describe the build and runtime environment in a Dockerfile, and the output is a single image that runs anywhere Docker runs. Build it on your laptop, ship it to any server, same result. The development loop goes from minutes to seconds.

```dockerfile
# Build stage: compile the Go binary
FROM golang:1.22-alpine AS build
WORKDIR /app
COPY . .
RUN go build -o collector .

# Run stage: just the binary, nothing else
FROM alpine:3.19
COPY --from=build /app/collector /collector
EXPOSE 8080
CMD ["/collector"]
```

Build the image. Run it with `docker run --restart=unless-stopped`. Now if the process dies, Docker brings it back automatically. And deploying to a new server is just `docker pull` and `docker run`. No provisioning VMs, no installing Go, no configuring systemd. Same image, same behavior, everywhere.

Life is good again. But it's still one process on one machine.

## It grows up

Your event collector needs a real database now. SQLite was fine for a single process, but you're thinking ahead. You add Postgres. You add Redis for caching recent events. Now you have three things to run: the collector, Postgres, and Redis. Running `docker run` three times with the right flags and port mappings gets old fast.

Docker Compose handles this cleanly. One YAML file, all three services, one command to bring it all up.

```yaml
services:
  collector:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: events
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

`docker compose up` and everything starts in the right order. This is genuinely great for development and small deployments. A lot of teams run Compose in production for a long time and it works fine. It's one of those tools that's easy to underestimate.

But Compose runs on a single machine. If your event collector gets popular and one machine isn't enough, Compose can't help you.

## It gets popular

Another team hears about your event collector and starts sending their deploy webhooks to it. Then the QA team hooks it up to their test runner. Traffic doubles. Response times go up. The single machine can't keep up.

So you SSH into a second server. Pull the images. Run the containers. Put nginx in front of both as a load balancer. Traffic splits. Things speed up.

Then you add a third for redundancy. Now you're SSH-ing into three machines, running docker commands, checking if things are up, updating images one at a time. You write a bash script to automate it. The script breaks when a server gets a new IP after a reboot. You write a better script. Someone changes the port and forgets to update it. You need to roll back a bad deploy and realize your script doesn't know how.

This is where every team ends up: managing containers across machines with scripts that were written in a hurry and are only updated when something breaks. The scripts always rot. At some point you start searching for something better. Every blog post, every conference talk, every "how to run containers at scale" thread points at the same thing. You don't know what Kubernetes is yet, but it keeps showing up.

## What Kubernetes actually is

Before we get into the specifics, here's the mental model that makes everything else click.

Kubernetes is a database that fires events when things change.

You write a YAML file describing what you want and post it to the Kubernetes API. The API server validates it and stores it in etcd (a distributed key-value store). And that's basically it.

All the magic happens in controllers — Go programs watching etcd for changes. When something changes, the relevant controller wakes up and diffs two things: the desired state you wrote in YAML, and the state of the processes running in the cluster right now. If there's a discrepancy (you asked for 3 replicas but only 2 are running) it fixes it. Then it goes back to sleep.

It's a thermostat. You set it to 72°F. It checks the room. If it's 68°F, it turns on the heat. When it hits 72°F, it stops. It just knows what the temperature should be, measures what it actually is, and "reconciles" the difference. That's what Kubernetes calls it: a reconcile loop.

That's the whole system. You declare the desired state. Controllers reconcile.

Every single concept in the rest of this post is just a different controller with a different job. Once that clicks, new K8s concepts stop being intimidating. They're just new reconcile loops.

## The Pod

So let's use it. A Pod is the smallest unit you can run in Kubernetes. For now, think of it as one container.

```yaml
apiVersion: v1          # Which version of the K8s API to use
kind: Pod               # What type of resource this is
metadata:
  name: collector       # A name to identify it
spec:
  containers:           # List of containers in this Pod (usually just one)
    - name: collector
      image: collector:latest
      ports:
        - containerPort: 8080
```

This is the shape of every Kubernetes resource you'll ever write. An API version, a kind, a name, and a spec describing what you want. Deployments, Services, Ingresses, custom resources, they all look like this. The `kind` changes, the `spec` changes, but the structure is always the same.

You apply it with `kubectl apply -f pod.yaml`. Kubernetes reads the YAML, stores it in etcd, and a controller notices the new entry. It finds a node with capacity and starts the container. Your event collector is running.

If the container crashes, the controller on that node notices the gap (desired: running, actual: not running) and restarts it. Same self-healing you had with Docker, but now it's managed by a system that knows about your whole cluster, not just one machine.

## The server dies

Your event collector is humming along on node-2. Then node-2 loses power. The Pod is gone. Unlike a container crash, there's no controller at the node level to restart it because the node itself is dead.

Nobody recreates the Pod on another node. It's just gone. You find out an hour later when the QA team asks why their webhook endpoint is down.

The problem with Pods is that they're tied to a single node. If the node dies, the Pod dies with it. Nothing in the cluster knows it should be recreated somewhere else.

## ReplicaSets: "I want three of these, always"

A ReplicaSet is a cluster-level controller. Its job is simple: maintain a count. You say "I want 3 copies of this Pod." The ReplicaSet watches the cluster. If there are only 2, it creates another one. If a node goes down and takes a Pod with it, the ReplicaSet sees the gap and schedules a replacement on a healthy node.

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
          image: collector:latest
          ports:
            - containerPort: 8080
```

Now you have three copies of your event collector running across different nodes. Desired state: 3. Actual state: 3. If one dies, desired: 3, actual: 2, reconcile: create one more. The same cruise control pattern.

No more SSH-ing into three servers. No more bash scripts that break when an IP changes. You wrote a YAML file that says "three copies, always" and Kubernetes handles the rest.

## Pushing a new version

Your event collector has a bug. Events with empty sources are getting stored instead of rejected. You fix it, build `collector:v2`, and push the image.

Now what? You have three Pods running v1. You need to get them onto v2 without dropping events. If you just delete all three and create new ones, there's a gap where nothing is running and the QA team's webhooks fail.

A ReplicaSet doesn't know how to do this. It maintains a count. If you change the image, it kills all the old Pods and creates new ones at the same time. Hard cut. Downtime.

## Deployments: rolling updates

A Deployment wraps a ReplicaSet and adds a rollout strategy. It creates a *new* ReplicaSet with the v2 image, scales it up one Pod at a time, and scales the old ReplicaSet down as the new Pods become healthy. At any point during the rollout, some Pods are running v1 and some are running v2, but the total count stays at 3. No gap. No downtime.

```yaml
apiVersion: apps/v1
kind: Deployment
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
          image: collector:v2
          ports:
            - containerPort: 8080
```

If v2 turns out to be worse than the bug it fixed, `kubectl rollout undo deployment/collector` rolls everything back to v1. The Deployment keeps the old ReplicaSet around so it can switch back.

This is why almost nobody creates ReplicaSets directly. You create Deployments. The Deployment manages the ReplicaSet. The ReplicaSet manages the Pods. Controllers all the way down, each watching one layer and reconciling it.

In practice, you'll use `kubectl rollout status deployment/collector` to watch the rollout, and `kubectl rollout undo deployment/collector` when v2 sets something on fire.

## Services: finding your Pods

You have three Pods. Each has its own IP address. Those IPs change every time a Pod dies and gets recreated. You can't hardcode them.

A Service gives your set of Pods a stable DNS name and IP. Traffic to the Service gets load-balanced across all the Pods behind it.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: collector
spec:
  selector:
    app: collector
  ports:
    - port: 80
      targetPort: 8080
```

Now anything inside the cluster can reach your app at `http://collector` or `http://collector.default.svc.cluster.local`.

Notice the `selector`. The Service doesn't know about specific Pods. It doesn't keep a list of IPs. It just says "find everything with the label `app: collector` and route traffic to it." It works like a name tag at a conference. You walk into the room and look for everyone wearing a tag that says "collector." You don't know their names. You don't know how long they've been here. People come and go all day. Doesn't matter. If they're wearing the tag, they get the next request.

Pods come and go. They crash. They get rescheduled to different nodes with different IPs. The Service doesn't care. It re-evaluates the label selector constantly and routes to whatever matches right now.

This is where it starts feeling like a real system.

## The state problem: StatefulSets

Everything so far has been stateless. Your app doesn't care which Pod is which. Kill one, replace it, nobody notices. Deployments are built for this. Pods are interchangeable.

But not everything is stateless. Some workloads need stable identity. A message broker like NATS needs stable identities to form a cluster and elect leaders for stream replication. A distributed cache like Redis Cluster needs stable network addresses so nodes can find each other after restarts. A custom data store needs to keep its data on disk even when the Pod gets rescheduled.

A Deployment can't give you this. It treats Pods as disposable. If a Pod dies, the replacement gets a new name, a new IP, and starts fresh. For stateless apps, that's fine. For anything that writes to disk or needs peers to recognize it, that's a problem.

A StatefulSet solves this. Each Pod gets a stable name: `myapp-0`, `myapp-1`, `myapp-2`. They always start in order and stop in reverse. Each Pod gets its own persistent volume that follows it across restarts. If `myapp-0` crashes and gets rescheduled to a different node, it remounts the same disk and comes back as `myapp-0`. Peers can still find it.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mystore
spec:
  serviceName: mystore
  replicas: 3
  selector:
    matchLabels:
      app: mystore
  template:
    metadata:
      labels:
        app: mystore
    spec:
      containers:
        - name: mystore
          image: mystore:latest
          ports:
            - containerPort: 9000
          volumeMounts:
            - name: data
              mountPath: /var/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

In practice, you'll see StatefulSets used for things like message brokers, search indexes, and custom data stores that were built before Kubernetes existed and need to be lifted into containers without rewriting them. I've worked on teams that run a custom Go-based data store this way. It was originally a standalone service. Wrapping it in a StatefulSet let us run it in the cluster alongside everything else without changing how it stores data on disk.

For databases specifically, most teams use a database operator (like the Postgres operator) that manages the StatefulSet for you through a CRD. You declare "I want a Postgres cluster with 3 replicas and 10Gi of storage" and the operator handles the StatefulSet, replication, failover, and backups. You get the benefits of running the database in your cluster without manually managing StatefulSet YAML. It's the same declarative pattern: you describe what you want, a controller makes it happen.

## Namespaces: keeping your stuff separate

Your cluster now has your app, a database, a monitoring stack, and the stuff some other team deployed. Everything is in the `default` namespace and it's getting messy.

Namespaces are how you organize. Think of them as folders for your cluster resources.

```bash
kubectl create namespace production
kubectl create namespace staging
```

Deploy the same app to both:

```bash
kubectl apply -f deployment.yaml -n production
kubectl apply -f deployment.yaml -n staging
```

Each namespace gets its own Services, Pods, ConfigMaps, and Secrets. `hello.production` and `hello.staging` are completely separate. You can set resource quotas per namespace so staging can't eat all the CPU. You can set RBAC rules so the intern can't accidentally delete production.

I've worked on clusters running dozens of services from different teams. Without namespaces, that would be chaos. With them, each team owns their space.

## Gateway: letting the outside world in

Your app has a Service, but it's only reachable inside the cluster. Time to expose it. The quickest way is to change the Service type to `LoadBalancer`, which gives your Service a public IP. Done.

But now you're building more things. You add a dashboard. A notification service. An auth service. Each one needs to be reachable from outside the cluster, and each LoadBalancer Service gets its own public IP. You're managing three, four, five separate entry points. Each one costs money if you're in the cloud. Each one needs its own DNS record and TLS certificate. It doesn't scale.

What you actually want is one front door that routes traffic to the right Service based on the hostname or path. That's what the Gateway API does.

A Gateway is the entry point itself. Think of it as the load balancer that accepts traffic from the outside world:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

An HTTPRoute defines where that traffic goes:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: collector-route
spec:
  parentRefs:
    - name: main-gateway
  hostnames:
    - collector.example.com
  rules:
    - backendRefs:
        - name: collector
          port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: dashboard-route
spec:
  parentRefs:
    - name: main-gateway
  hostnames:
    - dashboard.example.com
  rules:
    - backendRefs:
        - name: dashboard
          port: 80
```

`collector.example.com` goes to the collector Service. `dashboard.example.com` goes to the dashboard. Add the auth service tomorrow and it's just another HTTPRoute.

The separation is clean: platform teams own the Gateway (the infrastructure), application teams own their HTTPRoutes (the routing rules). Same pattern as everything else. You declare what you want. A controller reconciles it into a working load balancer.

## ConfigMaps and Secrets: configuration that lives outside your image

Hardcoding config into your container image is a mistake you make once. Different environments need different database URLs, feature flags, and API keys. You don't want to rebuild your image every time you change a log level.

If you've read the [Twelve-Factor App](https://12factor.net/config), you already know the answer: store config in the environment. Kubernetes gives you two primitives for this.

**ConfigMaps** hold non-sensitive configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: collector-config
data:
  DB_HOST: "postgres.default.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "events"
  LOG_LEVEL: "info"
```

**Secrets** hold sensitive values. Your collector needs to connect to Postgres:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: collector-secrets
type: Opaque
stringData:
  DB_PASSWORD: "your-password-here"
```

Here's something that surprises people: Kubernetes Secrets are just base64-encoded. Not encrypted. (`stringData` lets you write plain values; Kubernetes encodes them before storing.) Anyone with access to the cluster can decode them. They're "secrets" in the sense that Kubernetes treats them differently from ConfigMaps (they're stored separately, they don't show up in `kubectl describe`, they can be encrypted at rest if you configure it), but out of the box, base64 is not security. It's encoding.

In practice, most teams use an external secrets operator that syncs secrets from a real vault (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault) into Kubernetes Secret objects. Your app never knows the difference. It just reads the Secret the normal way. But the actual secret value lives in a proper secrets manager with encryption, audit logging, and access control. The operator watches the vault for changes and reconciles them into the cluster. Same pattern again.

Both ConfigMaps and Secrets can be injected into your container two ways: as environment variables or as files mounted on disk. For simple key-value pairs, environment variables are the most common choice. You wire them up in the Deployment:

```yaml
containers:
  - name: collector
    image: collector:v2
    envFrom:
      - configMapRef:
          name: collector-config    # injects DB_HOST, DB_PORT, DB_NAME, LOG_LEVEL
    env:
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: collector-secrets
            key: DB_PASSWORD        # pulled individually from the Secret
```

`envFrom` pulls every key from the ConfigMap in one shot. The password comes from the Secret individually with `secretKeyRef` — you could use `envFrom` on a Secret too, but being explicit about which keys you take from a Secret is safer. Your app just reads `os.Getenv("DB_HOST")` and it works the same way it would outside of Kubernetes.

The point is separation. Your image is immutable. Your config is not. They deploy independently.

## Liveness and readiness: how Kubernetes decides to kill your app

Remember when your app crashed at 2am? Kubernetes restarts crashed containers. But what about containers that are running but broken? Deadlocked. Stuck in a loop returning 500s. Out of memory but not quite dead enough for the OS to kill.

From the outside, the container is "running." Kubernetes has no reason to touch it. Traffic keeps flowing to it. Users keep getting errors.

Liveness probes fix this. You give Kubernetes an endpoint to check. If it fails, Kubernetes kills the container and makes a new one.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

If `/healthz` stops responding, Kubernetes kills the Pod and starts a replacement. The reconcile loop sees desired state (healthy Pod running) vs actual state (Pod failing health checks) and takes corrective action.

Readiness probes are different. If `/readyz` fails, Kubernetes doesn't kill the Pod. It just stops sending traffic to it. The Pod stays alive but gets pulled out of the Service's rotation. This is how zero-downtime deployments work: new Pods only receive traffic after they pass their readiness check. Old Pods keep serving until the new ones are ready.

Without these probes, you're flying blind. You'll push a bad version, Kubernetes will happily route traffic to it, and you won't know until someone complains.

Which brings up the question: how do you know if the new version is actually working?

## Observability: seeing what your cluster is doing

Your app is running across three Pods on two nodes. You deployed v2 an hour ago. Is it healthy? Is it slower than v1? Is it using more memory?

You can't SSH into a Pod and tail a log file. Pods are ephemeral. The one you want to debug might already be gone, replaced by a new one on a different node.

The QA team reports that event ingestion feels slow since the v2 deploy. You check the liveness probe. It's passing. The Pods are healthy. But "healthy" and "fast" aren't the same thing. You have no idea how many events are being processed, how long requests are taking, or whether error rates changed after the deploy. The app is running. You just can't see what it's doing.

What would you even want to know? How many events are being ingested per second. How long each request takes. How many are failing. Whether memory usage crept up after the deploy. These are metrics, and your app is the only thing that can produce them.

Add a `/metrics` endpoint to your collector using the Prometheus client library. It exposes those numbers as key-value pairs in a specific text format over HTTP:

```
GET /metrics HTTP/1.1
Host: collector:8080

HTTP/1.1 200 OK
Content-Type: text/plain

collector_events_total{source="qa-runner"} 48291        # 48k events from QA
collector_events_total{source="deploy-hooks"} 12037     # 12k from deploy hooks
collector_request_duration_seconds_sum 4823.4           # total time spent handling requests
collector_request_duration_seconds_count 60328          # total requests handled
collector_errors_total 142                              # 142 failures
process_resident_memory_bytes 94371840                  # ~90MB of RAM
```

These numbers look static. A single scrape just tells you "60,328 requests have been handled, total." Not very useful on its own. The trick is scraping repeatedly.

Prometheus runs inside your cluster. But here's the part that ties back to the mental model: the modern, idiomatic way to tell Prometheus what to scrape is through the Prometheus Operator—which introduces its own CRDs, `ServiceMonitor` and `PodMonitor`. These are custom resources reconciled by a controller, just like everything else in Kubernetes.

A `ServiceMonitor` looks like this:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: collector
spec:
  selector:
    matchLabels:
      app: collector
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

Just like a Deployment manages Pods declaratively, a `ServiceMonitor` tells the Prometheus controller which endpoints to watch. The Prometheus controller sees the new `ServiceMonitor`, reconciles it, and starts scraping your collector's `/metrics` endpoint. You write YAML describing what you want scraped. A controller makes it happen. Same pattern again.

Just like a Service doesn't keep a list of Pod IPs, Prometheus doesn't keep a list of scrape targets. It discovers endpoints through `ServiceMonitor` selectors dynamically. Pods come and go. Doesn't matter. Every 15 seconds (by default), it scrapes the path and port from the spec and stores the result with a timestamp. Now you have a time series. If `collector_request_duration_seconds_count` was 60,328 at 3:46pm and 60,391 at 3:47pm, that's 63 requests in one minute. If `collector_request_duration_seconds_sum` went from 4,823 to 4,831, that's 8 seconds of total request time across 63 requests, so about 127ms average latency. One number is a snapshot. Two numbers over time tell a story.

Grafana connects to Prometheus and turns those time series into dashboards. You deployed v2 at 3:47pm and the latency spike is obvious.

That spike is v2. You didn't have to guess. You didn't have to wait for someone to complain. You can see it.

And yes, you can set up Alertmanager to page you at 2am when error rates cross a threshold. How? You guessed it. More YAML, another controller.

This is where canary deployments become possible. Roll out v2 to one Pod. Watch the metrics. If error rates stay flat and latency looks normal, the canary survived. Roll it out to more Pods. If the metrics go red, roll back before most users ever notice. Without observability, every deployment is a leap of faith. With it, you can make decisions based on data.

For logs, tools like Loki or Elasticsearch aggregate logs from every container in the cluster into one searchable place. Combined with metrics and the health probes from the last section, you have a complete picture: Is it running? Is it healthy? Is it performing well? What happened when it wasn't?

You're not waiting for someone to report a problem anymore. You find it first.

## And now you have a platform

Look at what just happened. You started with a Go binary on a server. You hit a series of real problems, and each one had a Kubernetes answer:

- App crashes? **Containers and Pods** restart it.
- Node dies? **ReplicaSets** replace the Pod elsewhere.
- Need to update without downtime? **Deployments** do rolling updates.
- Can't find your Pods? **Services** give them stable DNS names.
- Need stable identity and persistent storage? **StatefulSets**.
- Multiple teams sharing a cluster? **Namespaces**.
- External traffic needs to reach internal services? **Gateway API**.
- Different config per environment? **ConfigMaps and Secrets**.
- Running but broken? **Liveness and readiness probes** catch it.
- Need to see what's happening? **Prometheus, Grafana, Loki**.

And every single one follows the same pattern. You write YAML describing the desired state. A controller watches for changes. The reconcile loop makes reality match the description. That's it. Once you internalize that mental model, new concepts stop being intimidating. They're just new controllers with new jobs.

I used to watch the TGI Kubernetes streams where Joe Beda would demo things like running a Roblox server on K8s and explain the architecture and history along the way. What struck me was how intentional the design is. Every abstraction exists because someone hit a real operational problem and needed a clean answer. It's not overcomplicated for the sake of it. It's layered so you only use what you need.

This is exactly how real Kubernetes platforms get built. Not from a grand architecture diagram. From a series of problems and solutions. You deploy something. It breaks in a new way. You fix it. You add the next layer.

That's been my experience. I've been solving the same problem for years: make it easier for engineers to run and ship software. The tools changed. The scale changed. But the problem never did. Each layer I added grew out of solving the next version of it.

If you're learning Kubernetes, don't memorize definitions. Deploy something. Break it. Fix it. That's how platforms get built.

