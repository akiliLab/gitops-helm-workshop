---
title: Helm Releases
---

# Helm Releases

A chart release is described through a Kubernetes custom resource named **HelmRelease**.

A Helm release can refer a chart from:
* public or private Helm repositories over HTTPS
* public or private Git repositories over SSH

## Install NGINX

To expose applications outside of the cluster you'll be using the NGINX ingress controller. 
The controller will run inside the Linkerd mesh.

Create a namespace with linkerd injection enabled:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    fluxcd.io/ignore: "false"
    linkerd.io/inject: enabled
  name: ingress-nginx
```

Create a Helm release to install the NGINX ingress controller:

```yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: nginx-ingress
  namespace: ingress-nginx
  annotations:
    fluxcd.io/ignore: "false"
spec:
  releaseName: nginx-ingress
  chart:
    repository: https://kubernetes-charts.storage.googleapis.com/
    name: nginx-ingress
    version: 1.16.0
  values:
    controller:
      service:
        type: LoadBalancer
```

Apply changes:

```sh
git add . && \
git commit -m "install ingress" && \
git push origin master && \
fluxctl sync
```

Validate that the Helm operator has installed the release:

```sh
kubectl -n ingress-nginx get hr
```

Find the public IP of the ingress controller:

```sh
kubectl -n ingress-nginx get svc
```

## Install podinfo

[Podinfo](http://github.com/stefanprodan/podinfo) is tiny Go web application.
You'll be installing podinfo using a Helm chart stored in the git repository at `charts/podinfo`.

Create the `prod` namespace with linkerd injection enabled:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    fluxcd.io/ignore: "false"
    linkerd.io/inject: enabled
  name: prod
```

Create a Helm release to install the podinfo chart (replace `GHUSER` with your GitHub username):

```yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: podinfo
  namespace: prod
  annotations:
    fluxcd.io/ignore: "false"
spec:
  releaseName: podinfo
  chart:
    git: git@github.com:GHUSER/gitops-workshop
    ref: master
    path: cluster/charts/podinfo
  values:
    image:
      repository: stefanprodan/podinfo
      tag: 2.1.0
    service:
      enabled: true
      type: ClusterIP
    ingress:
      enabled: true
      annotations:
        kubernetes.io/ingress.class: "nginx"
        nginx.ingress.kubernetes.io/configuration-snippet: |
          proxy_set_header l5d-dst-override $service_name.$namespace.svc.cluster.local:9898;
          proxy_hide_header l5d-remote-ip;
          proxy_hide_header l5d-server-id;
      path: /
      hosts:
        - *
```

Apply changes:

```sh
git add . && \
git commit -m "install podinfo" && \
git push origin master && \
fluxctl sync
```

Validate that the Helm operator has installed podinfo:

```sh
kubectl -n prod get hr
```

Open your browser and navigate to `http://<LB-IP>/`, you should see podinfo v2.1.0 UI.

![v2.1.0](/podinfo-2.1.0.png)

## Automated upgrade

Flux can be used to automate container image updates in your cluster.
You can enable the automate image tag updates by annotating Helm release objects.
You can also control what tags should be considered for an
update by using glob, regex or semantic version expressions.

Edit the podinfo Helm release and enable Flux automated image updates:

```yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  annotations:
    fluxcd.io/automated: "true"
    fluxcd.io/tag.chart-image: semver:~2.1
```

Commit and push the changes to GitHub:

```sh
git add . && git commit -m "automate podinfo" && git push origin master
```

Sync the the changes on the cluster:

```sh
fluxctl sync
```

Validate that the Helm operator has upgraded podinfo:

```sh
kubectl -n prod get hr
```

Pull the changes made by Flux locally:

```sh
git pull origin master
```

Open your browser and navigate to `http://<LB-IP>/`, you should see podinfo v2.1.3 UI.

![v2.1.3](/podinfo-2.1.3.png)