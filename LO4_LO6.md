# LO4 - LO6
## LO4

### Create Deployment and export YAML
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deploy web -o yaml > web.yaml
kubectl get pods -l app=web

### DockerHub authenticated pulls
kubectl create secret docker-registry dockerhub \
 --docker-server=docker.io \
 --docker-username=USER \
 --docker-password=PASS \
 --docker-email=email@example.com
kubectl patch sa default -p '{"imagePullSecrets":[{"name":"dockerhub"}]}'
kubectl get secret dockerhub -o jsonpath='{.type}'; echo

### Scale two ways
kubectl scale deploy web --replicas=5
kubectl get rs -l app=web
kubectl get deploy web -o yaml > web.yaml
gedit web.yaml &
kubectl apply -f web.yaml
kubectl get rs -l app=web

### History and rollback
kubectl annotate deploy/web kubernetes.io/change-cause="updated nginx" --overwrite
kubectl rollout history deploy/web
kubectl rollout undo deploy/web
kubectl rollout status deploy/web

### RollingUpdate maxSurge/maxUnavailable
kubectl patch deploy web -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
kubectl get deploy web -o jsonpath='{.spec.strategy}'; echo

### Recreate strategy
kubectl patch deploy web -p '{"spec":{"strategy":{"type":"Recreate"}}}'
kubectl get deploy web -o jsonpath='{.spec.strategy.type}'; echo

### CPU/memory requests and limits
kubectl set resources deploy web \
 --requests=cpu=100m,memory=128Mi \
 --limits=cpu=500m,memory=256Mi
kubectl describe pod -l app=web

### revisionHistoryLimit
kubectl patch deploy web -p '{"spec":{"revisionHistoryLimit":3}}'
kubectl get deploy web -o jsonpath='{.spec.revisionHistoryLimit}'; echo

### Bad image tag
kubectl set image deploy/web nginx=nginx:doesnotexist
kubectl rollout status deploy/web
kubectl get pods
kubectl get rs -l app=web
kubectl rollout undo deploy/web

### List pods by selector
kubectl get pods -l app=web
kubectl get rs -l app=web
kubectl get deploy web -o jsonpath='{.spec.selector.matchLabels}'; echo

### Expose Deployment
kubectl expose deploy web --port=80 --target-port=80
kubectl get svc web
kubectl get svc web -o yaml

### Sidecar container
cat > web-sidecar.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
- name: web-sidecar
spec:
replicas: 1
selector:
- matchLabels: {app: web-sidecar}
template:
- metadata:
- - labels: {app: web-sidecar}
spec:
- volumes:
- - name: shared
- - emptyDir: {}
- containers:
- - name: nginx
- - image: nginx:1.25
- - ports: [{containerPort: 80}]
- - volumeMounts: [{name: shared, mountPath: /usr/share/nginx/html}]
- - name: sidecar
- - image: busybox
- - command: ["sh","-c","while true; do date > /data/index.html; sleep 10; done"]
- - volumeMounts: [{name: shared, mountPath: /data}]
EOF
kubectl apply -f web-sidecar.yaml
kubectl get pods -l app=web-sidecar

### httpd Deployment with nodeSelector
cat > httpd.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
- name: httpd
spec:
- replicas: 2
- selector:
- - matchLabels: {app: httpd}
- template:
- - metadata:
- - labels: {app: httpd}
- - spec:
- - nodeSelector:
- - disktype: ssd
- - containers:
- - name: httpd
- - image: httpd:2.4
- - ports:
- - name: http
- - containerPort: 80
EOF
kubectl apply -f httpd.yaml
kubectl get pods
kubectl describe pod -l app=httpd
### If Pending, fix with:
kubectl label node minikube disktype=ssd

###  Bare Pod
kubectl run busy --image=busybox --restart=Never -- sleep 3600
kubectl get pod busy
kubectl delete pod busy

### StatefulSet + headless Service
cat > redis.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
- name: redis
spec:
- clusterIP: None
- selector: {app: redis}
- ports:
- - name: redis
- port: 6379
apiVersion: apps/v1
kind: StatefulSet
metadata:
- name: redis
spec:
- serviceName: redis
- replicas: 3
- selector:
- matchLabels: {app: redis}
- template:
- metadata:
- labels: {app: redis}
- spec:
- containers:
- - name: redis
- image: redis:7
- ports:
- - name: redis
- containerPort: 6379
EOF
kubectl apply -f redis.yaml
kubectl get pods -l app=redis
kubectl run tmp --rm -it --image=busybox -- nslookup redis-0.redis.default.svc.cluster.local

### DaemonSet
cat > agent.yaml <<'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
- name: agent
spec:
- selector:
- matchLabels: {app: agent}
- template:
- metadata:
- labels: {app: agent}
- spec:
- containers:
- - name: agent
- image: busybox
- command: ["sh","-c","while true; do sleep 3600; done"]
EOF
kubectl apply -f agent.yaml
kubectl get ds
kubectl get pods -l app=agent -o wide

### Job
kubectl create job calc --image=busybox -- sh -c 'echo $((6*7))'
kubectl get jobs
kubectl logs job/calc

### CronJob
kubectl create cronjob date --image=busybox --schedule="* * * * *" -- date
kubectl get cronjob
kubectl get jobs
kubectl patch cronjob date -p '{"spec":{"suspend":true}}'

### Manual Job from CronJob
kubectl create job manual-date --from=cronjob/date
kubectl get job manual-date
kubectl logs job/manual-date

### StatefulSet ordering
kubectl scale statefulset redis --replicas=3
kubectl get pods -l app=redis -w
kubectl scale statefulset redis --replicas=1
kubectl get pods -l app=redis -w

### Two containers share emptyDir
cat > shared.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
- name: shared-demo
spec:
- volumes:
- - name: shared
- emptyDir: {}
- containers:
- - name: writer
- image: busybox
- command: ["sh","-c","echo hello > /data/msg; sleep 3600"]
- volumeMounts: [{name: shared, mountPath: /data}]
- - name: reader
- image: busybox
- command: ["sh","-c","sleep 3600"]
- volumeMounts: [{name: shared, mountPath: /data}]
EOF
kubectl apply -f shared.yaml
kubectl exec shared-demo -c reader -- cat /data/msg

### List storage
kubectl get sc
kubectl get pv
kubectl get pvc

### ConfigMap as volume
kubectl create configmap appcfg --from-literal=color=blue --from-literal=mode=dark
cat > cm-vol.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
- name: cm-demo
spec:
- volumes:
- - name: cfg
- - configMap: {name: appcfg}
- containers:
- - name: app
- image: busybox
- command: ["sh","-c","sleep 3600"]
- volumeMounts: [{name: cfg, mountPath: /etc/cfg}]
EOF
kubectl apply -f cm-vol.yaml
kubectl exec cm-demo -- ls /etc/cfg
kubectl exec cm-demo -- cat /etc/cfg/color

### ConfigMap subPath
cat > cm-subpath.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
- name: cm-subpath
spec:
-  volumes:
- - name: cfg
- configMap: {name: appcfg}
- containers:
- - name: app
- image: busybox
- command: ["sh","-c","sleep 3600"]
- volumeMounts:
- - name: cfg
- mountPath: /etc/app/color.conf
- subPath: color
EOF
kubectl apply -f cm-subpath.yaml
kubectl exec cm-subpath -- cat /etc/app/color.conf

### Secret as volume
kubectl create secret generic mysecret --from-literal=username=admin --from-literal=password=s3cret
cat > secret-vol.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
- name: secret-demo
spec:
- volumes:
- - name: sec
- secret:
- secretName: mysecret
- defaultMode: 0400
- containers:
- - name: app
- image: busybox
- command: ["sh","-c","sleep 3600"]
- volumeMounts:
- - name: sec
- mountPath: /etc/sec
- readOnly: true
EOF
kubectl apply -f secret-vol.yaml
kubectl exec secret-demo -- ls -l /etc/sec
kubectl exec secret-demo -- cat /etc/sec/password

### StatefulSet PVC per replica
cat > redis-pvc.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
 name: redis-pvc
spec:
 clusterIP: None
 selector: {app: redis-pvc}
 ports: [{port: 6379}]
apiVersion: apps/v1
kind: StatefulSet
metadata:
- name: redis-pvc
spec:
- serviceName: redis-pvc
- replicas: 3
- selector:
- matchLabels: {app: redis-pvc}
- template:
- metadata:
- labels: {app: redis-pvc}
- spec:
- containers:
- - name: redis
- image: redis:7
- volumeMounts: [{name: data, mountPath: /data}]
- volumeClaimTemplates:
- - metadata:
- name: data
- spec:
- accessModes: ["ReadWriteOnce"]
- resources:
- requests:
- storage: 1Gi
EOF
kubectl apply -f redis-pvc.yaml
kubectl get pvc
kubectl delete statefulset redis-pvc
kubectl get pvc

### initContainer
cat > init.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
- name: init-demo
spec:
- volumes:
- - name: data
- emptyDir: {}
- initContainers:
- - name: seed
- image: busybox
- command: ["sh","-c","echo seeded > /data/file"]
- volumeMounts: [{name: data, mountPath: /data}]
- containers:
- - name: app
- image: busybox
- command: ["sh","-c","cat /data/file; sleep 3600"]
- volumeMounts: [{name: data, mountPath: /data}]
EOF
kubectl apply -f init.yaml
kubectl logs init-demo -c app

### readOnly volume mount
cat > readonly.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
- name: readonly-demo
spec:
- volumes:
- - name: data
- emptyDir: {}
- containers:
- - name: app
- image: busybox
- command: ["sh","-c","sleep 3600"]
- volumeMounts:
- - name: data
- mountPath: /data
- readOnly: true
EOF
kubectl apply -f readonly.yaml
kubectl exec readonly-demo -- sh -c 'echo x > /data/file' 

### emptyDir sizeLimit
cat > sizelimit.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
- name: sizelimit-demo
spec:
- volumes:
- - name: scratch
- emptyDir:
- sizeLimit: 100Mi
- containers:
- - name: app
- image: busybox
- command: ["sh","-c","sleep 3600"]
- volumeMounts: [{name: scratch, mountPath: /scratch}]
EOF
kubectl apply -f sizelimit.yaml
kubectl describe pod sizelimit-demo

### Secret from literals
kubectl create secret generic dbcreds --from-literal=username=admin --from-literal=password=s3cret
kubectl get secret dbcreds -o yaml
kubectl get secret dbcreds -o jsonpath='{.data.password}' | base64 -d; echo

### Secret from files
echo cert > cert.pem
echo key > key.pem
kubectl create secret generic tlsfiles --from-file=cert.pem --from-file=key.pem
kubectl get secret tlsfiles -o yaml

### TLS Secret
kubectl create secret tls mytls --cert=cert.pem --key=key.pem
kubectl get secret mytls -o yaml

### Secret volume vs env
env:
- name: PASSWORD
- valueFrom:
- secretKeyRef:
- name: dbcreds
- key: password
kubectl exec secret-demo -- cat /etc/sec/password

### envFrom secretRef
cat > envfrom.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
- name: envfrom-demo
spec:
- containers:
- - name: app
- image: busybox
- command: ["sh","-c","sleep 3600"]
- envFrom:
- - secretRef:
- name: dbcreds
EOF
kubectl apply -f envfrom.yaml
kubectl exec envfrom-demo -- env

### imagePullSecrets
kubectl create secret docker-registry regcred \
 --docker-server=docker.io \
 --docker-username=USER \
 --docker-password=PASS
spec:
 imagePullSecrets:
 - name: regcred

 ### Mount only one Secret key
kubectl create secret generic multi --from-literal=a=1 --from-literal=b=2 --from-literal=c=3
cat > secret-one.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
- name: secret-one
spec:
- volumes:
- - name: sec
- secret:
- secretName: multi
- items:
- - key: b
- path: only-b.txt
- containers:
- - name: app
- image: busybox
- command: ["sh","-c","sleep 3600"]
- volumeMounts: [{name: sec, mountPath: /etc/sec}]
EOF
kubectl apply -f secret-one.yaml
kubectl exec secret-one -- ls /etc/sec
kubectl exec secret-one -- cat /etc/sec/only-b.txt

### ClusterIP DNS
kubectl get deploy web || kubectl create deployment web --image=nginx --replicas=2
kubectl expose deploy web --port=80
kubectl run tmp --rm -it --image=busybox -- nslookup web.default.svc.cluster.local

### NodePort
kubectl delete svc web --ignore-not-found
kubectl expose deploy web --type=NodePort --port=80
kubectl get svc web
minikube service web --url
curl $(minikube ip):$(kubectl get svc web -o jsonpath='{.spec.ports[0].nodePort}')

### LoadBalancer
### Terminal 1:
kubectl delete svc web --ignore-not-found
kubectl expose deploy web --type=LoadBalancer --port=80
kubectl get svc web
### Terminal 2:
minikube tunnel
### Terminal 1:
kubectl get svc web

### Headless Service DNS
kubectl apply -f redis.yaml
kubectl get svc redis
kubectl run tmp --rm -it --image=busybox -- nslookup redis.default.svc.cluster.local

### Endpoints / EndpointSlice
kubectl get endpoints web
kubectl get endpointslices -l kubernetes.io/service-name=web
kubectl describe endpoints web

### Cross-namespace FQDN
kubectl create ns other
kubectl run tmp -n other --rm -it --image=busybox -- wget -qO- web.default.svc.cluster.local
kubectl run tmp -n other --rm -it --image=busybox -- nslookup web

### Port-forward Service vs Pod
kubectl port-forward svc/web 8080:80
### new terminal:
curl localhost:8080
### pod version:
POD=$(kubectl get pod -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward pod/$POD 8080:80

### port / targetPort / nodePort
kubectl get svc web -o yaml

### Debug pod connectivity
kubectl run tmp --rm -it --image=busybox -- sh
### inside pod:
wget -qO- web:80
nc -zv web 80
nslookup web
exit

### Two Deployments, one Service
cat > shop.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: {name: shop-v1}
spec:
- replicas: 2
- selector: {matchLabels: {app: shop, ver: v1}}
- template:
- metadata: {labels: {app: shop, ver: v1}}
- spec:
- containers:
- - name: echo
- image: hashicorp/http-echo
- args: ["-text=from-v1"]
- ports: [{containerPort: 5678}]
apiVersion: apps/v1
kind: Deployment
metadata: {name: shop-v2}
spec:
- replicas: 2
- selector: {matchLabels: {app: shop, ver: v2}}
- template:
- metadata: {labels: {app: shop, ver: v2}}
- spec:
- containers:
- - name: echo
- image: hashicorp/http-echo
- args: ["-text=from-v2"]
- ports: [{containerPort: 5678}]
apiVersion: v1
kind: Service
metadata: {name: shop}
spec:
- selector: {app: shop}
- ports:
- - port: 80
- targetPort: 5678
EOF
kubectl apply -f shop.yaml
kubectl get endpoints shop
kubectl run tmp --rm -it --image=busybox -- sh -c 'for i in 1 2 3 4 5; do wget -qO- shop; done'

### Diagnose Service failure
kubectl run tmp --rm -it --image=busybox -- nslookup web
kubectl get endpoints web
kubectl get svc web -o yaml
kubectl get pods --show-labels
kubectl describe pod -l app=web

### NodePort verify
kubectl expose deploy web --type=NodePort --port=80 --name=web-np
kubectl get svc web-np
minikube service web-np --url

### HTTP livenessProbe
kubectl patch deploy web --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/livenessProbe","value":{"httpGetkubectl describe pod -l app=web

### Redis readinessProbe
### Add inside redis container:
readinessProbe:
 tcpSocket:
 port: 6379
 periodSeconds: 5
### Then:
kubectl apply -f redis.yaml
kubectl describe pod -l app=redis

### startupProbe
### Add inside container:
startupProbe:
 httpGet:
 path: /
 port: 80
 failureThreshold: 30
 periodSeconds: 5
### Then:
kubectl apply -f web.yaml
kubectl describe pod -l app=web

### No probes
Say: Without probes, Kubernetes only knows whether the container process is running. It does not know whether the application is

### If you want a fresh namespace
kubectl delete all --all
kubectl delete pvc --all 
kubectl delete cm --all
kubectl delete secret
dockerhub regcred dbcreds mysecret multi tlsfiles mytls --ignore-not-found

## LO5

### Q: podman run -d --name web -p 80:8080 docker.io/library/nginx (nginx listens on 80; the site isn't
reachable on the host port you expected — what's reversed?)
FIX
podman run -d --name web -p 80:80 docker.io/library/nginx
curl localhost:80
WHY `-p host:container`. The original maps host 80 → container 8080, but nginx listens on 80
inside, so nothing answers. Map to the right container port.

### Q: podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql (the container exits during init — inspect podman logs.)
FIX
podman run -d --name db -e MYSQL_ROOT_PASSWORD=secret docker.io/library/mysql
podman logs db # init proceeds
WHY `-e MYSQL_ROOT_PASSWORD` with no value passes it empty; MySQL's entrypoint aborts
because it has no root password. Give it a value (or `-e MYSQL_ALLOW_EMPTY_PASSWORD=1`
/ `MYSQL_RANDOM_ROOT_PASSWORD=1`).

### Q: podman run -d --name app --network host -p 8080:80 docker.io/library/nginx (why is the -p flag effectively ignored here?)
FIX
podman run -d --name app -p 8080:80 docker.io/library/nginx
curl localhost:8080
WHY `--network host` shares the host's network namespace, so the container's ports are the host's
ports. There's no separate namespace to map, so `-p` is ignored (podman prints a warning).

### Q: podman run -d --name c1 docker.io/library/busybox (the container goes straight to Exited (0) — why, and how do you keep it running?)
FIX
podman run -d --name c1 docker.io/library/busybox sleep 1d # or: sleep infinity
podman ps
WHY busybox's default command runs and immediately returns, so the container exits `0`. A
container lives only as long as its PID 1 process — give it a long-running command.

### Q: podman run --rm -d --name job docker.io/library/alpine echo hello (then podman logs job fails — explain the --rm + detached gotcha.)
FIX
podman run -d --name job docker.io/library/alpine echo hello
podman logs job
# or see output live without detaching:
podman run --rm docker.io/library/alpine echo hello
WHY `--rm` deletes the container the moment it exits. A detached `echo` finishes instantly, so the
container (and its logs) are gone before `podman logs` runs.

### Q: podman run -d -p 8080:80 --memory 8m docker.io/library/mysql (the database never becomes healthy — what limit is the problem?)
FIX
podman run -d -p 8080:3306 --memory 1g -e MYSQL_ROOT_PASSWORD=secret docker.io/library/mysql
WHY 8 MB is far below MySQL's footprint; it's OOM-killed during initialization and never becomes
healthy. Raise `--memory` (≥512m, 1g is safe). (Also note MySQL needs a root password and
listens on 3306.)

### Q: two alpine containers, podman exec a ping b — name resolution fails on the default network; why, and how do you fix it?
FIX
podman network create mynet
podman run -d --name a --network mynet docker.io/library/alpine sleep 1d
podman run -d --name b --network mynet docker.io/library/alpine sleep 1d
podman exec a ping -c1 b
WHY Podman's default (rootless) network provides no inter-container DNS. A user-defined
network enables name-based resolution between attached containers

### Q: podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx (files aren't visible / SELinux denies access — what mount flag is missing?)
FIX
podman run -d --name web -v ./html:/usr/share/nginx/html:Z docker.io/library/nginx
### (use an absolute path if a relative one isn't resolved)
WHY On SELinux hosts the bind mount needs a relabel flag: `:Z` (private label for this container) or
`:z` (shared). Without it SELinux blocks the container from reading the host files.

### Q: FROM debian:12 / RUN apt-get install -y nginx (the build fails to find the package — what's missing before install?)
FIX
FROM debian:12
RUN apt-get update && apt-get install -y nginx
WHY The base image ships no package index; you must `apt-get update` first (same `RUN`) so
apt can find `nginx`.

### Q: ... CMD python app.py (the app can't be stopped cleanly — shell vs exec form?)
FIX
CMD ["python", "app.py"]
WHY Shell form `CMD python app.py` runs as `/bin/sh -c "…"`, so /bin/sh is PID 1 and doesn't
forward `SIGTERM` to python → the app can't shut down gracefully. Exec form (JSON array)
makes the app PID 1 and receives signals directly

### Q: FROM node:20 / COPY . /app / WORKDIR /app / RUN npm install (every source change triggers a full npm install — reorder for layer caching.)
FIX
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
WHY Copy the dependency manifests before `npm install` so that layer is cached and only re-runs
when `package*.json` changes. Copying all source first invalidates the install layer on every edit.

### Q: FROM alpine:3.20 / ENV PATH=/app/bin / RUN apk add --no-cache curl (curl/sh aren't found — what did setting PATH break?)
FIX
FROM alpine:3.20
ENV PATH="/app/bin:${PATH}"
RUN apk add --no-cache curl
WHY `ENV PATH=/app/bin` overwrites PATH entirely, dropping `/usr/bin:/bin:/usr/sbin:/sbin`, so
system binaries (sh, curl, apk) can't be found. Append to the existing PATH instead of replacing it.

### Q: FROM ubuntu:24.04 / EXPOSE 8080 / CMD ["python3","-m","http.server","3000"] — you map -p 8080:8080 but nothing answers; what mismatch is there, and what does EXPOSE actually do?
FIX — make the listening port match what you publish:
FROM ubuntu:24.04
EXPOSE 8080
CMD ["python3", "-m", "http.server", "8080"]
podman run -d -p 8080:8080 <image> # now answers
### (alternatively keep 3000 and map -p 8080:3000)
WHY The server listens on 3000 but you published 8080→8080, so nothing is behind 8080.
`EXPOSE` is documentation/metadata only — it does not publish or open any port; `-p` does the
actual publishing, and the app must listen on the targeted container port.

### Q: FROM debian:12 / RUN apt-get update && apt-get install -y build-essential (the image is huge — how do you cut the size in the same layer?)
FIX
FROM debian:12
RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential \
 && rm -rf /var/lib/apt/lists/*
WHY Clean the apt cache in the same `RUN` layer (and skip recommended extras). Removing it
in a later layer wouldn't shrink the image, because each layer is additive.

### Q: FROM python:3.11 / USER appuser / COPY requirements.txt … / RUN pip install … (permission denied — what's wrong with the USER/COPY ordering and ownership?)
FIX
FROM python:3.11
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
RUN useradd -m appuser && chown -R appuser /app
USER appuser
COPY . .
WHY Switching to a non-root `USER` before copying/installing means files are root-owned and the
user can't write to system paths → permission denied. Do privileged work (install) first, set
ownership, then drop to `USER appuser` last.

### Q: FROM golang:1.22 … RUN go build … CMD ["/src/app"] (the final image is ~1 GB — how
would a multi-stage build fix it?)
FIX
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app .
FROM gcr.io/distroless/base-debian12
COPY --from=build /app /app
CMD ["/app"]
WHY A multi-stage build compiles in the heavy `golang` image but copies only the resulting
binary into a tiny runtime base (distroless/alpine/scratch), dropping the ~1 GB toolchain from the
final image.

### Q: A pod is stuck Pending. Diagnose with kubectl describe pod and identify the reason from the events.
DO
kubectl describe pod <pod> | sed -n '/Events/,$p'
Read the event and act: `Insufficient cpu/memory` → lower `resources.requests` or add a node;
`didn't match node selector/affinity` → label the node / fix selector; `untolerated taint` → add a
toleration; `unbound PersistentVolumeClaim` → create the PVC.

### Q: Remove a couple of spaces from a deployment and try to deploy it. Identify the cause in the error messages and fix it.
DO
kubectl apply -f broken.yaml
### error: did not find expected key / mapping values are not allowed in this context
kubectl apply -f broken.yaml --dry-run=client # validate without applying
FIX/WHY YAML is indentation-sensitive; a misplaced space breaks the mapping structure. Restore
consistent 2-space indentation under the right parent key, then re-apply.

### Q: A pod is in ImagePullBackOff. Diagnose the cause and fix it.
DO
kubectl describe pod <pod> | grep -A6 Events # shows the exact pull error
FIX/WHY Causes: image typo, missing/wrong tag, or a private registry without a pull secret.
Correct the image/tag, or `kubectl create secret docker-registry …` + `imagePullSecrets`, then re-deploy.

### Q: A pod is in CrashLoopBackOff. Use kubectl logs --previous to read the last crash and fix the root cause.
DO
kubectl logs <pod> --previous
kubectl describe pod <pod> | grep -A3 'Last State'
FIX/WHY The container starts then exits repeatedly; `--previous` shows the last crashed
instance's output (bad command, missing env/config, failed dependency). BackOff is the kubelet's
increasing restart delay. Fix the root cause and redeploy.

### Q: A pod's last state shows OOMKilled. Confirm it, then fix it.
DO
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'; echo kubectl describe pod <pod> | grep -i oom
FIX
kubectl set resources deployment web --limits=memory=512Mi
WHY The container exceeded its memory limit and was killed. Raise `limits.memory` or fix the
app's memory usage.

### Q: A Deployment's pods never become Ready. Trace it to a failing readiness probe and correct it.
DO
kubectl describe pod <pod> | grep -A4 Readiness # shows probe failures
kubectl get pod <pod> -o jsonpath='{.status.conditions}'; echo
FIX/WHY A failing readiness probe keeps the pod out of endpoints (`READY 0/1`). Correct the
probe's `path`/`port`/`initialDelaySeconds`, or fix the app endpoint it checks.

### Q: A Service returns nothing. Diagnose a Service/pod label mismatch using kubectl get endpoints.
DO
kubectl get endpoints <svc> # empty = no matching/ready pods
kubectl get svc <svc> -o jsonpath='{.spec.selector}'; echo
kubectl get pods --show-labels
FIX/WHY Empty endpoints mean the Service selector doesn't match any pod labels (or pods aren't
Ready). Align the selector with the pod labels (or fix the labels).

### Q: spec.selector.matchLabels doesn't match spec.template.metadata.labels — explain the error and fix it.
DO
kubectl apply -f bad.yaml
### error: Deployment.apps "x" is invalid: spec.template.metadata.labels: Invalid value ...
### `selector` does not match template `labels`
FIX/WHY A Deployment's `selector.matchLabels` must be a subset of the pod template labels.
Make `template.metadata.labels` include every key/value in the selector.

### Q: resources.requests exceed any node's capacity — explain why the pod is Pending and fix it.
DO
kubectl describe pod <pod> # 0/N nodes available: Insufficient cpu/memory
kubectl describe nodes | grep -A5 Allocatable
FIX/WHY No node can satisfy the request, so the scheduler leaves the pod Pending. Lower
`resources.requests` to fit a node, or add a larger node.

### Q: A pod mounts a PVC that doesn't exist — diagnose the FailedMount/Pending and create the PVC.
DO
kubectl describe pod <pod> # persistentvolumeclaim "data" not found
FIX
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: data}
spec:
 accessModes: [ReadWriteOnce]
 resources: {requests: {storage: 1Gi}}
kubectl apply -f pvc.yaml
WHY A pod referencing a missing PVC can't mount its volume and stays Pending. Create the PVC
(a default StorageClass dynamically provisions the PV).

### Q: An ubuntu container with no long-running command shows Completed/CrashLoopBackOff.
Explain why and add a proper command.
FIX
 command: ["sleep", "infinity"]
WHY The `ubuntu` image has no foreground process, so the container exits immediately →
`Completed` (with `restartPolicy: Never`) or `CrashLoopBackOff` (with `Always`). Give it a
long-running command (or the real app).

### Q: Use kubectl get events --sort-by=.lastTimestamp to triage everything that happened to a pod over time.
DO
kubectl get events --sort-by=.lastTimestamp
kubectl get events --sort-by=.lastTimestamp --field-selector involvedObject.name=<pod>
WHY Sorted events give a chronological timeline (scheduling, pulls, probe failures, OOM, restarts) for root-cause triage.

### Q: Use kubectl exec -it to get a shell in a pod and debug DNS with nslookup and cat/etc/resolv.conf.
DO
kubectl exec -it <pod> -- sh
cat /etc/resolv.conf # nameserver = CoreDNS, search domains
nslookup kubernetes.default
nslookup <svc>
exit
WHY Confirms the pod's DNS config (CoreDNS server + search domains) and whether service
names resolve.

### Q: Use an ephemeral debug container to troubleshoot a no-shell/distroless container.
DO
kubectl debug -it <pod> --image=busybox --target=<container> -- sh
WHY Distroless/no-shell containers have no `sh` to exec into. `kubectl debug` attaches an
ephemeral container that shares the target's process/network namespaces, giving you tools without modifying the pod.

### Q: Spin up a throwaway kubectl run tmp pod to test connectivity to a Service from inside the cluster.
DO
kubectl run tmp --rm -it --image=busybox -- sh
wget -qO- <svc>:80
nc -zv <svc> 80
exit
WHY `--rm -it` gives a disposable in-cluster client to verify Service reachability and DNS, auto-deleted on exit.

### Q: A node shows NotReady. List what you'd check and inspect it on minikube.
DO
kubectl get nodes
kubectl describe node <node> | grep -A8 Conditions # Ready / MemoryPressure / DiskPressure
minikube logs | tail
minikube ssh -- 'systemctl status kubelet' # kubelet health
WHY A NotReady node usually means the kubelet is down, disk/memory pressure, or a broken CNI/network plugin. The node conditions and kubelet logs point to which.

### Q: A pod is stuck Terminating. Explain finalizers and the grace period, then force-delete it and
state the risks.
DO
kubectl get pod <pod> -o jsonpath='{.metadata.finalizers}'; echo
kubectl delete pod <pod> --grace-period=0 --force
WHY Finalizers block deletion until their controller completes cleanup; the grace period is the window between SIGTERM and SIGKILL. Force-deleting removes the API object immediately, but the container may still be running on the node → risk of an orphaned process and, for stateful apps, data corruption / split-brain

### Q: Wrong apiVersion/kind pairing (e.g. apps/v1 + Pod) — explain the validation error and correct it.
DO
kubectl apply -f bad.yaml
### error: no matches for kind "Pod" in version "apps/v1"
FIX/WHY A Pod is `apiVersion: v1`; Deployment/ReplicaSet/StatefulSet/DaemonSet are `apps/v1`. Match the kind to its correct API group/version.

### Q: A pod fails with CreateContainerConfigError because a configMapKeyRef key is missing. Diagnose and fix.
DO
kubectl describe pod <pod> # couldn't find key <X> in ConfigMap <cm>
FIX/WHY The pod references a ConfigMap key that doesn't exist. Add the key to the ConfigMap (`kubectl edit cm <cm>`) or correct the `configMapKeyRef.key` in the pod spec.

### Q: A pod can't mount a Secret that lives in a different namespace. Explain namespace scoping and fix it.
DO / FIX
### copy the secret into the pod's namespace:
kubectl get secret -n <src-ns> -o yaml \
 | sed 's/namespace: <src-ns>/namespace: <pod-ns>/' \
 | kubectl apply -n <pod-ns> -f -
WHY Secrets are namespace-scoped; a pod can only reference Secrets in its own namespace. There's no cross-namespace mount — the Secret must exist alongside the pod.

### Q: A rollout is stuck with ProgressDeadlineExceeded. Find the cause and remediate.
DO
kubectl rollout status deployment/web
kubectl describe deployment web | grep -A3 Conditions # ProgressDeadlineExceeded
kubectl get pods -l app=web # find the failing new pods
kubectl describe pod <new-pod>
FIX/WHY The new ReplicaSet didn't become available within `progressDeadlineSeconds` — usually a bad image, failing probe, or insufficient resources on the new pods. Fix that root cause (rollout resumes automatically) or `kubectl rollout undo deployment/web`.

###  Q: Catch indentation/field typos before applying with --dry-run=server / --validate=true, then fix them.
DO
kubectl apply -f f.yaml --dry-run=server
### e.g. error: unknown field "spec.template.spec.containers[0].imagePullpolicy"
FIX/WHY Server-side dry-run validates against the real schema and catches typos like `imagePullpolicy` (correct: imagePullPolicy) and misnested fields before anything is applied. Correct the field name/nesting and re-apply.

### Q: An app can't reach its database Service. Verify in order and identify the broken link.
DO (in order)
kubectl get pod <app> # 1 Running?
kubectl get svc db # 2 Service exists?
kubectl get endpoints db # 3 endpoints populated?
kubectl exec <app> -- nslookup db # 4 DNS resolves?
kubectl get svc db -o jsonpath='{.spec.ports}'; echo # 5 correct port?
WHY Walk pod → Service → endpoints → DNS → port; the first failing check is the broken link (e.g. empty endpoints → selector/label mismatch; NXDOMAIN → DNS; wrong port → connection refused).

### Q: Read a container's state/lastState and restartCount with kubectl get pod -o jsonpath to find the cause of repeated restarts.
DO
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}'; echo
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'; echo
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'; echWHY A high `restartCount` plus `lastState.terminated.reason` (e.g. OOMKilled, Error) and the exit code pinpoint why the container keeps restarting.

### Q: Compare OpenShift Route to Ingress.
ANSWER Both expose HTTP(S) services outside the cluster. Ingress is the Kubernetes-native, portable resource: you write Ingress rules, but you must install a separate ingress controller (nginx, Traefik, HAProxy) for them to work. Route is OpenShift's built-in object, backed by the platform's integrated HAProxy router — no controller to install, it auto-generates a hostname, and it offers simple TLS modes (edge / passthrough / re-encrypt) out of the box. OpenShift supports both and reconciles Ingress objects into Routes. In short: Route is simpler and native to OpenShift but non-portable; Ingress is the standard, portable choice that needs a controller.

## LO6

### Q: Analyze the networking model of Kubernetes versus Docker's network.
ANSWER Docker's default model is host-centric: each host has a bridge network with NAT,
containers get private IPs behind the host, you publish ports manually with `-p`, and DNS-based name resolution only works on user-defined networks. Kubernetes instead mandates a flat, cluster-wide network: every pod gets its own routable IP and any pod can reach any pod across nodes without NAT, implemented by CNI plugins (Calico, Flannel, Cilium). On top of that, Services provide stable virtual IPs and load balancing, and CoreDNS gives service discovery by name. So Docker networking is simple and local; Kubernetes networking is an abstraction designed for multi-node scale, discovery, and self-healing.

### Q: Evaluate Kubernetes storage abstraction (CSI, StorageClasses, dynamic provisioning) against Podman's storage plugins. Which is more flexible for stateful workloads, and why?
ANSWER Kubernetes abstracts storage through CSI drivers, StorageClasses, and dynamic
provisioning: a pod's PVC requests storage and a PV is provisioned on demand from whatever
backend the class points at (cloud disks, NFS, Ceph), and the volume follows the pod when it's rescheduled to another node. Podman's volumes/plugins are essentially host-local storage. For stateful workloads Kubernetes is far more flexible, because it provides portable, dynamic, multi-node, lifecycle-managed storage with access modes and reclaim policies — exactly what databases and other stateful services need — whereas Podman storage is tied to a single host.

### Q: Evaluate the operator pattern (CRDs + controllers) for managing stateful services versus using only k8s manifests, in terms of control and effort.
ANSWER Plain manifests describe desired state but leave day-2 operations (backups, failover,
version upgrades, scaling a clustered database safely) to humans or external scripts. An Operator encodes that operational knowledge into a custom controller that continuously reconciles a CRD, automating those tasks. The trade-off is effort vs control: operators require building or adopting and maintaining a controller (higher upfront cost and a learning curve), but they give far more automated control and dramatically reduce ongoing operational toil for complex stateful services. For simple/stateless apps, manifests are enough; for production databases, an operator usually pays off.

### Q: Assess the developer experience of plain Kubernetes (kubectl + manifests) versus OpenShift's Source-to-Image (S2I) and developer console. What does each optimize for?
ANSWER Plain Kubernetes optimizes for flexibility and control: you author Dockerfiles and YAML manifests and manage everything explicitly, which is powerful but has a steep learning curve and a lot of boilerplate. OpenShift's S2I and developer console optimize for developer velocity: a developer can point S2I at source code and get an image built and deployed without writing a Dockerfile or manifests, all through a guided web UI. In short, kubectl+manifests give maximum control at the cost of effort; S2I+console trade some control for speed, guardrails, and a gentler onboarding for developers.

### Q: Critically evaluate migrating a workload from OpenShift to Kubernetes.
ANSWER Migrating off OpenShift means giving up its integrated extras and rebuilding equivalents: Routes become Ingress (+ an ingress controller), DeploymentConfig becomes Deployment, S2I/BuildConfig become an external CI pipeline, you lose the integrated registry, the stricter SCC security defaults must be reproduced with PodSecurity/admission controllers, and you must stand up your own monitoring/logging/console. The migration effort is therefore substantial. The upside is portability, no vendor coupling, and lower licensing cost. Critically, it's a trade of integrated, supported tooling for flexibility and self-managed effort — worth it if portability/cost matter more than turnkey tooling, and risky if the team relies heavily on OpenShift's built-ins.

### Q: Evaluate running CI/CD build agents inside Kubernetes versus on dedicated VMs, considering isolation, autoscaling, and noisy-neighbor effects.
ANSWER Running agents (Jenkins agents, GitLab runners, Tekton) in Kubernetes gives
ephemeral, autoscaling capacity — pods spin up per build and disappear after — with good
bin-packing and lower idle cost. The downsides are weaker isolation (containers share the node kernel) and noisy-neighbor risk where a heavy build starves others, which you mitigate with resource requests/limits, quotas, and node pools. Dedicated VMs give strong isolation and predictable performance but waste resources when idle and scale slowly/manually. Net:
Kubernetes wins on elasticity and cost-efficiency; VMs win when you need hard isolation (e.g. untrusted or security-sensitive builds)

### Q: How does Kubernetes integrate with cloud environments and dynamic capacity provisioning?
ANSWER Kubernetes integrates with clouds through the cloud-controller-manager, which
provisions cloud LoadBalancers and attaches cloud block storage to PVCs via CSI. For dynamic capacity it scales at two levels: the Horizontal Pod Autoscaler (and VPA) scales workloads by metrics, while the Cluster Autoscaler (or Karpenter on AWS) adds and removes nodes automatically based on unschedulable-pod pressure. Combined with managed services
(EKS/GKE/AKS), this lets a cluster grow and shrink its infrastructure on demand, matching capacity to load.

### Q: Critically evaluate the default security and hardening of OpenShift versus vanilla Kubernetes. Why do enterprises in regulated industries often choose OpenShift?
ANSWER OpenShift is secure-by-default: Security Context Constraints (SCC) force containers
to run as non-root unless explicitly granted otherwise, it ships built-in OAuth/RBAC, an integrated registry, image provenance/scanning, and Red Hat-signed, supported images. Vanilla Kubernetes is permissive by default — you must add PodSecurity admission, RBAC policies, NetworkPolicies, and scanning yourself, and mistakes are easy. Regulated industries (finance, healthcare, government) often choose OpenShift because it provides a certified, vendor-supported, hardened baseline with a clear audit and compliance story out of the box, reducing both risk and the effort of proving controls to auditors.

### Q: Critically evaluate the learning curve and required skill set of OpenShift versus plain Kubernetes. Does OpenShift's added abstraction help or hinder a team new to containers?
ANSWER Plain Kubernetes already has a steep learning curve (manifests, networking, RBAC,
controllers, storage). OpenShift layers its own concepts on top — SCC, Routes,
BuildConfig/ImageStream, the `oc` CLI — so there's more to learn for deep operations. However, its web console and S2I lower the barrier for the common case: a newcomer can build and deploy an app without mastering Dockerfiles or YAML. So for a team new to containers, the abstraction helps initial productivity and onboarding, but can hinder later when they must understand the OpenShift-specific machinery to debug or operate at depth. Net: helpful early, with added complexity to master eventually.

### Q: Compare the resource footprint of full Kubernetes versus k3s for a small on-premise deployment. When is full Kubernetes overkill?
ANSWER Full Kubernetes runs a heavyweight control plane — etcd plus multiple components
— needing more CPU/RAM and ongoing operational care, and is built for large-scale HA. k3s
packages the whole thing into a single small binary (<100 MB), can use SQLite instead of etcd, strips legacy/in-tree cloud drivers, and runs comfortably on a Raspberry Pi — while exposing the same kubectl/API. For a small on-premise/edge/single-team deployment, full Kubernetes is overkill: its footprint and operational overhead aren't justified when k3s delivers the same developer-facing API at a fraction of the cost. Reach for full Kubernetes when you need large-scale HA, many nodes, or deep cloud integration.

### Q: Defend whether a small team should use Docker Compose on a single host or move straight to Kubernetes, considering complexity, scaling needs, and resilience.
ANSWER Docker Compose on a single host is trivial to learn and operate and is well suited to development and small, stable single-host production — but it offers no multi-node scheduling, no self-healing across hosts, no autoscaling, and is a single point of failure. Kubernetes provides resilience, horizontal scaling, and rolling updates, at the cost of significant complexity and operational overhead. For a small team with modest, predictable load and no high-availability requirement, Compose is the defensible choice — it ships product fastest with least overhead. The team should move to Kubernetes only once it genuinely needs HA, elastic scaling, or multi-host resilience, at which point Compose's limits become the bottleneck.

### Q: Analyze vendor lock-in across Kubernetes (open source / CNCF) and OpenShift (Red Hat). How does the choice affect long-term flexibility?
ANSWER Vanilla Kubernetes is CNCF-governed and open, with a portable API that runs across
clouds and distributions, so lock-in is low and long-term flexibility is high. OpenShift is Red Hat's opinionated distribution: it adds value (support, integrated tooling, hardening) but also Red-Hat-specific constructs (Routes, DeploymentConfig, SCC, subscriptions) and licensing cost, which create some lock-in. The choice is a trade-off: OpenShift buys convenience and vendor accountability now, at the price of reduced portability later; plain Kubernetes preserves flexibility but you assemble and support the platform yourself.

###  Q: Defend a recommendation of OpenShift over vanilla Kubernetes for a company that wants an integrated, vendor-supported platform with built-in developer and operations tooling.
ANSWER For a company that explicitly wants an integrated, supported platform, OpenShift is
the right call: it delivers one vendor-supported product with built-in CI/CD (S2I/Tekton pipelines), an integrated registry, monitoring and logging, a developer console, hardened
secure-by-default settings, and enterprise SLAs. That means far less integration work than
assembling these from separate open-source projects, faster developer onboarding, and a single accountable vendor for support and security patches. The trade-offs — licensing cost and some lock-in — are exactly what this company has chosen to accept in exchange for turnkey, supported tooling, so the recommendation is well justified.

### Q: Compare storage provisioning between Kubernetes and regular VM workloads.
ANSWER With VM workloads, storage is provisioned manually and statically: an admin attaches
disks/LUNs to a VM, and that storage is tied to the VM's lifecycle. Kubernetes makes storage declarative and dynamic: a developer's PVC requests capacity from a StorageClass, a CSI driver provisions a PV automatically, and the volume follows the pod when it's rescheduled to another node. So Kubernetes decouples the storage request from the provisioning and offers self-service, portability, and automated lifecycle management, whereas VM storage is admin-driven, host-bound, and manual.

### Q: A startup with two engineers must ship a containerized product quickly and cheaply.
Recommend a container solution and justify your choice on cost and operational overhead.
ANSWER Recommend a managed PaaS / managed container service (or Docker Compose /
k3s if self-hosting), not self-managed full Kubernetes. With only two engineers, the priority is shipping product, not running a control plane. A managed platform (Cloud Run, Render, Fly, App Runner, or a small managed cluster) removes cluster upgrades, patching, and HA of the control plane, so the team pays mainly for what they run and spends near-zero time on ops. Full self-managed Kubernetes would impose operational overhead that dwarfs a two-person team and slow delivery — the opposite of "quick and cheap." If they self-host, Compose or k3s on one host keeps cost and overhead minimal.

### Q: Compare Docker and Podman in terms of architecture (daemon vs daemonless) and rootless security. Why might a security-conscious team prefer Podman?
ANSWER Docker uses a central daemon (dockerd) that traditionally runs as root, which is a
single point of failure and a broad attack surface — the Docker socket effectively grants root. Podman is daemonless: it forks containers directly as child processes (via conmon/runc) with no long-running privileged daemon, and it runs rootless by default, mapping the container root to an unprivileged host user. It's CLI-compatible with Docker and integrates well with systemd and SELinux. A security-conscious team prefers Podman because removing the root daemon and running rootless shrinks the attack surface and improves isolation, with no privileged background process to compromise

### Q: Compare the day-2 operational overhead (cluster upgrades, scaling nodes, backups) of self-managed Kubernetes versus a managed Kubernetes service.
ANSWER Self-managed Kubernetes puts all day-2 work on you: control-plane and node
upgrades, etcd backups and restore testing, node scaling, OS/security patching, and ensuring control-plane HA — significant, specialized, ongoing effort. A managed service
(EKS/GKE/AKS/ARO) offloads the control plane: the provider runs, upgrades, and keeps it
available, and often automates node scaling and backups, leaving you to manage mainly your
workloads and worker nodes. The trade-off is some loss of control and a service fee in exchange for dramatically lower operational overhead — usually the better choice unless you have strong reasons (compliance, cost at scale, customization) to self-manage.

### Q: A media-streaming company expects spiky, unpredictable global traffic. Recommend a containerized solution. Elaborate your choice.
ANSWER Recommend managed Kubernetes with autoscaling, deployed across multiple
regions, fronted by a CDN. Streaming traffic that is spiky and global needs elastic, automatic scaling: the Horizontal Pod Autoscaler scales pods to demand and the Cluster
Autoscaler/Karpenter adds nodes when needed and removes them when traffic falls, controlling cost. Kubernetes' self-healing and rolling updates keep the service available during spikes and deploys, multi-region placement reduces latency and adds resilience, and a CDN offloads the heavy media delivery. A managed control plane keeps the ops burden manageable while the platform absorbs unpredictable load — exactly the elasticity this scenario demands.

### Q: Defend whether a company that has outgrown Docker Swarm (or a single Compose host)
should migrate to Kubernetes — weigh the migration effort against the long-term benefits.
ANSWER If the company has genuinely outgrown Swarm/Compose, migrating to Kubernetes is
justified. The effort is real: rewriting Compose/Swarm definitions into Kubernetes manifests, learning the platform, and rebuilding CI/CD, networking, and storage. But the long-term benefits are substantial and durable: a vast ecosystem (Helm, operators, service mesh, observability), autoscaling, self-healing, multi-cloud portability, and by far the largest community and talent pool. Swarm is effectively in maintenance mode and Compose can't scale across hosts, so staying put caps growth. When scaling and resilience needs are real and lasting, the one-time migration cost pays off; if needs are modest, it may not be worth it yet.

### Q: Would you choose Kubernetes as a solution for a microservices architecture? Focus on orchestration, resilience and ecosystem.
ANSWER Yes. Kubernetes is the de facto platform for microservices. For orchestration it gives each service independent deployment, scaling, and rolling updates, with service discovery and load balancing built in. For resilience it self-heals (restarts failed pods, reschedules off dead nodes), supports health probes, and enables zero-downtime rollouts and rollbacks. Its ecosystem is unmatched — Helm for packaging, service meshes (Istio/Linkerd) for traffic management and mTLS, and a rich observability stack — all of which microservices benefit from. The only caveat is complexity: for a mere handful of services it can be overkill, but for a real microservices architecture the orchestration, resilience, and ecosystem make Kubernetes the strong choice.

### Q: Would you choose Kubernetes for enterprise-grade applications? Focus on scalability, community support and feature richness.
ANSWER Yes. For enterprise-grade applications Kubernetes offers proven scalability —
horizontal pod and cluster autoscaling to thousands of nodes/pods — so it grows with demand. Community support is its biggest asset: CNCF governance, the largest contributor and vendor ecosystem, abundant documentation, and a deep talent pool, which de-risks long-term adoption. Its feature richness — RBAC, namespaces and quotas for multi-tenancy, rolling updates, autoscaling, and extensibility via CRDs/operators — covers enterprise needs, and managed offerings add vendor support and SLAs. Together these make Kubernetes a robust, future-proof default for enterprise workloads, provided the organization invests in the requisite operational skills.

### Q: Would you choose OpenShift as a solution for a microservices architecture? Focus on orchestration, resilience and ecosystem.
ANSWER Yes, especially for enterprises that want microservices plus integrated tooling.
OpenShift is Kubernetes underneath, so it inherits the same orchestration (independent scaling, rolling updates, discovery) and resilience (self-healing, health checks, zero-downtime deploys). On top it adds an ecosystem tuned for microservices: built-in CI/CD pipelines (Tekton/S2I), OpenShift Service Mesh (Istio), an integrated registry, and a developer console — reducing the assembly work of wiring these together yourself. The trade-offs are licensing cost and some vendor lock-in. For an organization that values an integrated, supported microservices platform over DIY flexibility, OpenShift is a strong choice.

### Q: Would you choose OpenShift for enterprise-grade applications? Focus on scalability community support and feature richness.
ANSWER Yes, when the enterprise values vendor support and an integrated, hardened
platform. OpenShift delivers Kubernetes' scalability with added enterprise features:
secure-by-default SCC, integrated CI/CD, registry, monitoring/logging, a developer console,
and Red Hat enterprise SLAs and certified, signed images. "Community support" here comes in
two forms — the broad Kubernetes/CNCF community beneath it and Red Hat's commercial support
and lifecycle guarantees — which is exactly what regulated, support-dependent enterprises want. The feature richness reduces integration effort and strengthens the compliance/audit story. The cost is licensing and some lock-in, but for enterprises that prioritize supportability, security, and turnkey tooling, OpenShift is well justified.

Docker Swarm (prof flagged). Docker's built-in orchestrator: `docker swarm init`, services
(`docker service create`), stacks (deploy a Compose file), an overlay network for multi-host, Raft consensus among managers, and a built-in routing mesh + secrets. Pros: trivial setup, Compose-native, gentle learning curve, fine for small clusters. Cons: small ecosystem, limited autoscaling/extensibility, and it's largely in maintenance mode industry-wide. vs Kubernetes: Swarm = simplicity at small scale; Kubernetes = power, ecosystem, and resilience at higher complexity.

k3s (prof flagged). A CNCF-certified, lightweight Kubernetes (Rancher/SUSE) in a single <100 MB binary bundling the API server, controller-manager, scheduler, kubelet and containerd. It defaults to SQLite (or embedded etcd for HA), has server and agent roles, and bundles Traefik ingress, a service-load-balancer, and local-path storage. Same kubectl/API as upstream Kubernetes. Ideal for edge, IoT, dev, and small on-prem; peers are k0s and microk8s. Choose it when full Kubernetes is overkill (small footprint, low ops) and full Kubernetes when you need large-scale HA and deep cloud integration.

Container runtimes & OCI. Kubernetes talks to runtimes via the CRI; the dockershim was
removed in v1.24, so clusters use containerd or CRI-O (OpenShift's default) directly, both built on runc. Docker images still run because everything follows OCI image/runtime specs — that standardization is what keeps images portable across Docker, Podman, containerd, and CRI-O.

Other handy talking points. Helm (templated charts) vs Kustomize (overlays) vs raw manifests for packaging; GitOps (ArgoCD/Flux) reconciling the cluster to Git for auditability and easy rollback; service mesh for mTLS/traffic control at many-service scale; HPA (pods) vs Cluster Autoscaler (nodes); etcd as the single source of truth that must be backed up; RBAC, NetworkPolicy (default-allow → zero-trust), and Pod Security Standards (privileged/baseline/restricted; OpenShift uses SCC); and when not to use Kubernetes — a single app, tiny team, or stable load is better served by Compose, a PaaS, or serverless.
