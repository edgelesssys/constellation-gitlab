# General
Deploy the GitLab helm chart in a Constellation cluster on Azure.

# Prerequisites
The chart assumes the following setup:
- An existing Constellation cluster (v2.3.0 or higher) on Azure
- You manage your domain through [GoDaddy](https://www.godaddy.com)

To use either AWS or GCP with this chart, you need to deploy another CSI driver instead of the one installed as part of this Helm chart.

To use another domain provider rather than GoDaddy, you need to modify the deployment of the external-dns service. Read further to find out how you can achieve this.

In case encounter any issues with this deployment, feel free to open an issue.

# Architecture
While the original [GitLab helm chart](https://docs.gitlab.com/charts/) itself can deploy most of the required components except from external-dns and the CSI driver by itself, we use a split approach where we deploy the *infrastructure* components separately from the *application* components.
The idea here is that the infrastructure components usually are already available in an existing cluster, making it redundant to deploy them again.

The `infra` target deploys the following components:
- Constellation Azure Disk CSI driver
- cert-manager
- external-dns
- NGINX Ingress Controller

The external-dns service is preconfigured to use [GoDaddy](https://www.godaddy.com) by default.
You can change this by modifying the [deployment.yaml](gitlab/charts/external-dns/templates/deployment.yaml) config file
and adjusting any values in [values.yaml](gitlab/values.yaml).

The `app` target by default deploys the following components:
- cert-manager TLS Issuer (Let's Encrypt)
- GitLab
  - Gitaly
  - Exporter
  - Sidekiq
  - Shell
  - Toolbox
  - Webservice
- PostgreSQL
- Minio
- Redis

Additional components such as e.g. KAS, Pages, GitLab Runner (CI/CD) and more can be activated optionally.
Take a look at [values.yaml](gitlab/values.yaml) to see what components can be enabled.

Note that GitLab does not recommend using this setup for production deployments.
For more information, take a look at the [official recommendations](https://docs.gitlab.com/charts/installation/index.html) for production deployments of GitLab on Kubernetes.

# Deploy
```bash
export GODADDY_API_KEY=<your creds here>
export GODADDY_SECRET_KEY=<your creds here>
export TARGET_DOMAIN=<your domain, e.g. example.com>
export TLS_ISSUER_EMAIL=<your e-mail address>
helm dependency update ./gitlab
helm upgrade gitlab-infra ./gitlab --install --namespace default --set infra.enabled=true --set apiKey=$GODADDY_API_KEY --set secretKey=$GODADDY_SECRET_KEY
helm upgrade gitlab-app ./gitlab --install --timeout 600s --set app.enabled=true --set gitlab.global.hosts.domain=$TARGET_DOMAIN --set gitlab.certmanager-issuer.email=$TLS_ISSUER_EMAIL --namespace gitlab --create-namespace
```

The GitLab Helm chart automatically sets up DNS, Ingress and TLS certificates.

After deployment, you need to wait approximately 5-10 minutes for all components to set up completely.

To reach your GitLab instance afterwards, go to `https://gitlab.<your-domain>`.
For example, if you defined `example.com` as `TARGET_DOMAIN`, GitLab will be deployed under `gitlab.example.com`.

This usually also works with subdomains which do not exist yet.
For example, for a staging environment, you can define:
```bash
export TARGET_DOMAIN=staging.example.com
```

This will automatically deploy and setup GitLab under: `https://gitlab.staging.example.com`

# Other Demos
In case you are interested about other example deployments for Constellation, you can take a look at our [How to set up "always encrypted" Rocket.Chat🚀 on Kubernetes](https://dev.to/flxflx/rocketchat-constellation-most-secure-chat-server-ever--50oa) guide.

You can find the repository [here](https://github.com/edgelesssys/constellation-rocketchat).
If you used this guide before, you can skip the `rocket-infra` deployment step, as it sets up the same components as `gitlab-infra` does. 
