# K8s Skiff

I really liked the idea of [Skiff](https://github.com/basecamp/kamal-skiff/tree/main) but let's make it easily work for k8s.

Skiff promotes an easy way to build plane static sites powered by nginx
supporting partials through esi's. Keeping intact the old skool feel of ftp'ing a file and it's live.

## Installation

First create a git(hub) repo with the contents of your site.
Use [this template](https://github.com/mvanduijker/k8s-skiff-site-template) as a base.

Add helm repo

```console
helm repo add mvanduijker https://mvanduijker.github.io/mvanduijker-helm-charts
helm repo update
```

Example deployment to your k8s cluster with traefik as nginx controller

```console
helm upgrade --install --create-namespace -n hello-world hello-world mvanduijker/k8s-skiff \
  --set 'secret.git_url=https://github.com/mvanduijker/k8s-skiff-site-template' \
  --set 'config.host=hello-world.duyker.nl' \
  --set 'config.secretName=hello-world-k8s-skiff-tls' \
  --set "ingress.enabled=true" \
  --set 'ingress.className=traefik' \
  --set 'ingress.annotations.cert-manager\.io/cluster\-issuer=letsencrypt-prod'
```

## Configuration

Table of basic configuration. Want to know more print the values with

```bash
 helm show values mvanduijker/k8s-skiff
```

| Parameter              | Description                                         |  Default                                                 |
| ---------------------- | ---------------------------- | ----------------------------------------------- |
| config.update_interval | Interval to retrieve the website data from git repo |   10                                                     |
| config.git_branch      | The git branch to retrieve the website data from    |   main                                                   |
| secret.git_url         | The git url to retrieve the website data from       | <https://github.com/mvanduijker/k8s-skiff-site-template> |
