# OpenShift GitOps Sample


## Table of Contents
- [Overview](#overview)
- [Cohesity](#cohesity)
- [Prometheus](#prometheus)
- [Argo CD](#argo-cd)


## Overview

The included manifests can be used as examples, but please adjust them according to your environment.

### Environment
- OpenShift Version: 4.14.0 (Kubernetes v1.27.6)
- CNPG Operator: 1.23.6
- CNPG PostgreSQL Images tested:
  - cloudnative-pg/postgresql:15.8
  - cloudnative-pg/postgresql:16.2
  - cloudnative-pg/postgresql:16.3
- Storage: LVM Storage Operator (lvms-operator.v4.14.12)
- Base OS:
  - Bastion: Red Hat Enterprise Linux 8.10 (Ootpa)
  - Cluster Nodes: Red Hat Enterprise Linux CoreOS (RHCOS) 4.14

### Namespace / Parameters / Affinity / Taints / Tolerations / and others
- Adapt these settings to match the target guest environment.

### Secrets
Always manage secrets as **SealedSecrets** when storing them in Git.  
Example:
  ```bash
  kubeseal --format yaml < secret.yaml > secret-sealed.yaml
  ```
### Custom Images
The Dockerfiles used for building the custom CNPG images are maintained in a separate repository:  
[cnpg-docker](https://github.com/yyuutta/cnpg-docker)

### Packages
The built images are published on GitHub Packages.  
You can find them here:  
[GitHub Packages - yyuutta](https://github.com/yyuutta?tab=packages)


## Cohesity

> ⚠️ This section assumes that the basic Cohesity setup has already been completed.

### SSL Certificate

- **Obtain / Verify certificate**
  1. SSH into Cohesity
  2. Check current SSL certificate  
     ```bash
     iris_cli cluster list-ssl-cert-details
     ```
  3. Verify expiration date  
     ```bash
     openssl s_client -connect <your-cohesity-endpoint>:3000 -CAfile <your-route>/ca.crt
     ```

- **Renew certificate**
  1. Generate a new certificate  
     ```bash
     openssl req -x509 -nodes -days 365 \
       -newkey rsa:2048 \
       -keyout /<your-route>/key.pem \
       -out /<your-route>/cert.pem \
       -subj "/O=Cohesity Inc/CN=<your-cohesity-endpoint>"
     ```
  2. SSH into Cohesity  
  3. Update certificate  
     ```bash
     iris_cli cluster update-ssl-certificate ssl-certificate=<your-route>/cert.pem ssl-cert-private-key=<your-route>/key.pem
     ```
  4. Restart services  
     ```bash
     iris_cli cluster restart-services service-names=iris
     iris_cli cluster restart-services service-names=bridge
     ```
  5. Confirm updated certificate  
     ```bash
     iris_cli cluster list-ssl-cert-details
     ```
- **Apply to OpenShift Secret**
  ```bash
  oc create secret generic <your-cohesity-ca> \
    --from-file=ca.crt=<your-route>/ca.crt \
    -n <namespace> \
    --dry-run=client -o yaml | oc apply -f -
  ```

- **Connection check**
  ```bash
  openssl s_client -connect <your-cohesity-endpoint>:3000 -CAfile <your-route>/ca.crt
  ```

### AWS CLI (for backup verification)

- **Install**
  If not installed, follow AWS official instructions.

- **Set Cohesity credentials**
  ```bash
  export AWS_ACCESS_KEY_ID=<YOUR-ACCESS-KEY-ID>
  export AWS_SECRET_ACCESS_KEY=<YOUR-SECRET-ACCESS-KEY>
  ```

- **Check backup contents**
  ```bash
  aws s3 ls s3://<your-bucket>/<your-cluster-name>/ \
    --recursive \
    --human-readable \
    --summarize \
    --no-verify-ssl \
    --endpoint-url https://<your-cohesity-endpoint>:3000
  ```


## Prometheus

- **Enable PodMonitor**  
  Make sure that `enablePodMonitor` is set to `true` in the CNPG cluster manifest:  
  ```yaml
  monitoring:
    enablePodMonitor: true
  ```

- **Check metrics endpoint from PostgreSQL Pod (Exporter)**  
  You need to prepare a Pod where `curl` can be executed beforehand.
  ```bash
  oc get pods -o wide
  oc exec -ti <pod-name> -- curl -s <Cluster-IP>:9187/metrics | grep "<cnpg-metric-name>"
  ```

- **Prometheus API query (via port-forward)**  
  - In one terminal:  
    ```bash
    oc -n openshift-user-workload-monitoring port-forward svc/prometheus-operated 9090:9090
    ```
  - In another terminal:  
    ```bash
    curl "http://localhost:9090/api/v1/query?query=<cnpg-metric-name>"
    ```


## Argo CD

Argo CD is used for GitOps-based deployment.  
In this setup, Argo CD has been installed on OpenShift and exposed via a Route.

- **Access URL:**  
  [https://argocd-server-argocd.apps.your-apps-domain/](https://argocd-server-argocd.apps.your-apps-domain/)

- **Login:**  
  Use the `admin` account. The initial password is stored in the `argocd-secret` in the `argocd` namespace.

- **Repository registration:**  
  From the Argo CD web UI, navigate to:  
  `Settings → Repositories → Connect Repo using HTTPS`  
  Enter your GitHub repository URL and credentials.

- **Application creation:**  
  From the Argo CD web UI, navigate to:  
  `Applications → + NEW APP`  
  Provide the application name, project, repository, revision, path, destination cluster, and namespace.  
  Sync policy can remain **Manual** for the initial setup.

After configuration, you can push manifests to the Git repository, and Argo CD will detect changes.  
Click **SYNC** in the UI to apply updates to the OpenShift cluster.

