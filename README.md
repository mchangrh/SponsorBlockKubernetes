### Create cluster

```bash
terraform apply -auto-approve
```

Destroy:

```bash
terraform destroy -auto-approve
```


### Install nginx ingress

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f kubernetes-config/ingress/nginx-ingress-values.yaml
```

### Setup postgres


```bash
kubectl apply -f kubernetes-config/secrets/cluster1-backrest-repo-config-secret.yaml
kubectl apply -f kubernetes-config/postgres/operator.yaml

# Then wait a little bit for it to startup and install the CRDs

kubectl apply -f kubernetes-config/postgres
```

### Then use kustomize

Might have to run a second time to make sure cert works (will error if it fails)

```bash
kubectl apply -k kubernetes-config
```

# Postgres

```bash
kubectl apply -k kubernetes-config/kustomize/install/namespace
kubectl apply --server-side -k kubernetes-config/kustomize/install/default
```

To check on the status of your installation, you can run the following command:

```bash
kubectl -n postgres-operator get pods \
  --selector=postgres-operator.crunchydata.com/control-plane=postgres-operator \
  --field-selector=status.phase=Running
```

Then install postgres itself

```bash
kubectl apply -k kubernetes-config/kustomize/postgres
```


To check on the status of your installation, you can run the following command:

```bash
kubectl describe postgresclusters.postgres-operator.crunchydata.com pgsb
```

Then fix the affinity by running

```bash
kubectl edit deployments.apps cluster1-repl<number>
```

And adding the following to `spec.affinity`

```yaml
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: largerdb
                operator: In
                values:
                - "true"
```

And adding the following to `spec.affinity.podAntiAffinity` for only the repl ones

```yaml
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                - key: deployment-name
                  operator: In
                  values:
                  - cluster3
              topologyKey: "kubernetes.io/hostname"
```

# Install other configs

```bash
terraform apply -auto-approve

kubectl apply -f kubernetes-config/secrets
kubectl apply -f kubernetes-config/certs
kubectl apply -f kubernetes-config/*
```

## For setup from scratch

### Set default kustomize

```bash
KUBECONFIG=<full file path to kubeconfig.yaml>
```

### Prep Cert Manager

Pre-setup (already done)
```bash
cmctl x install --dry-run > cert-manager.custom.yaml
```

The cert manager yaml has already been created and is in kubernetes-config/certs


### Restoring database from backup

See https://www.percona.com/doc/kubernetes-operator-for-postgresql/standby.html

- Make new cluster with standby and old repoPath
- Copy over the secrets (script there doesn't work, just edit the file manually)
- Make sure s3 secrets are there for that cluster

Deleting clusters can be done with:

```bash
kubectl delete perconapgclusters.pg.percona.com clustername
```

When scaling down, make sure everything is still working before deleting volumes

### Recovering from wal failure causing out of date replicas

Check to see if backrest is out of space. If so, raise it in cr.yml and it will auto resize.

Then, follow the steps below to fix wal being out of date

#### If only one is failing

Raise storage size in backup pvc

Kill that one and delete the pvc, then recreate the pvc.

Then wait for the automated full backup to finish or use the method below

Then reinit that node

Then delete the old backups to save space

Then lower the storage claim? Is the only way to lower it making a new cluster based on the old one?

#### Option 1

start a full backup with the repo (see sample-backup.yaml)

Delete replica pods and pvc and restart them or use `patronictl reinit clusterX` and choose the specific pod.

#### Option 2

Untested

https://dba.stackexchange.com/questions/242722/archive-command-repeatly-failing-for-one-particular-file


```bash
kubectl edit configmap cluster1-pgha-config
```

Set archive_command = /bin/true

Delete replica pods and pvc and restart them or use `patronictl reinit clusterX` and choose the specific pod.

Restore archive_command to default

```
source /opt/crunchy/bin/postgres-ha/pgbackrest/pgbackrest-archive-push-local-s3.sh %p
```

Make sure to trigger a full backup after instead of incremental

then need to start a full backup with the repo (see sample-backup.yaml)

# Other useful things

### Database

Get password

```bash
kubectl get secret cluster1-pguser-secret -o go-template='{{.data.password | base64decode}}'
```

#### Notes

If the main switches to a replica, there might be an issue due to replica service and pgbouncer now pointing to the same pod, removing any load balancing. A solution would be to delete the replica pod to cause it to failover to the main one.