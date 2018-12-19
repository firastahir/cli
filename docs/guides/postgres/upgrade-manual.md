---
title: PostgreSQL Quickstart
menu:
  docs_0.9.0:
    identifier: pg-quickstart-quickstart
    name: Overview
    parent: pg-quickstart-postgres
    weight: 10
menu_name: docs_0.9.0
section_menu_id: guides
---
> New to KubeDB? Please start [here](/docs/concepts/README.md).

# KubeDB Upgrade Manual

This tutorial will show you how to upgrade KubeDB from previous version to 0.9.0.

## Before You Begin

At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [minikube](https://github.com/kubernetes/minikube).

Now, install KubeDB 0.8.0 cli on your workstation and KubeDB operator in your cluster following the steps [here](https://kubedb.com/docs/0.8.0/setup/install/).

## Previous operator and sample database

In this tutorial we are using helm to install kubedb 0.8.0 release. But, user can install kubedb operator from script too.
Follow [Install instructions](https://github.com/kubedb/project/issues/262) to install kubedb-operator 0.8.0.

```console
$ helm ls
NAME               REVISION    UPDATED                     STATUS      CHART           APP VERSION    NAMESPACE
kubedb-operator    1           Wed Dec 19 15:42:37 2018    DEPLOYED    kubedb-0.8.0    0.8.0          default  
```

Also a sample Postgres database (with scheduled backups) to examine successful upgrade. Read the guide [here](https://kubedb.com/docs/0.8.0/guides/postgres/snapshot/scheduled_backup/#create-postgres-with-backupschedule) to learn about scheduled backup in details.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: Postgres
metadata:
  name: scheduled-pg
  namespace: demo
spec:
  version: "9.6"
  replicas: 3
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  backupSchedule:
    cronExpression: "@every 1m"
    storageSecretName: gcs-secret
    gcs:
      bucket: kubedb
```

Now create secret and deploy database.

```console
$ kubectl create ns demo
namespace/demo created

$ kubectl create secret -n demo generic gcs-secret \
    --from-file=./GOOGLE_PROJECT_ID \
    --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
secret/gcs-secret created

$ kubectl create -f scheduled-pg.yaml
postgres.kubedb.com/scheduled-pg created
```

See running scheduled snapshots,

```console
NAME                           DATABASE          STATUS      AGE
scheduled-pg-20181219-094347   pg/scheduled-pg   Succeeded   5m
scheduled-pg-20181219-094447   pg/scheduled-pg   Succeeded   4m
scheduled-pg-20181219-094547   pg/scheduled-pg   Succeeded   3m
scheduled-pg-20181219-094647   pg/scheduled-pg   Succeeded   2m
scheduled-pg-20181219-094747   pg/scheduled-pg   Succeeded   1m
scheduled-pg-20181219-094847   pg/scheduled-pg   Succeeded   29s
```

## Upgrade kubedb-operator

For helm, `upgrade` command works fine.

```console
$ helm upgrade --install kubedb-operator appscode/kubedb --version 0.9.0
$ helm install appscode/kubedb-catalog --name kubedb-catalog --version 0.9.0 --namespace default

$ helm ls
NAME               REVISION    UPDATED                     STATUS      CHART                   APP VERSION    NAMESPACE
kubedb-catalog     1           Wed Dec 19 15:50:40 2018    DEPLOYED    kubedb-catalog-0.9.0    0.9.0          default  
kubedb-operator    2           Wed Dec 19 15:49:55 2018    DEPLOYED    kubedb-0.9.0            0.9.0          default  
```

For Bash script installation, uninstall first, then install again with 0.9.0 script. See [0.9.0 installation guide](https://kubedb.com/docs/0.9.0/setup/install/).

## Stale CRD objects

At this state, the operator is skipping this `scheduled-pg` Postgres. Because, Postgres version `9.6` is deprecated in `kubedb 0.9.0`.
The scheduled snapshot also in paused state.

```concole
$ kubectl get snap -n demo
NAME                           DATABASENAME   STATUS      AGE
scheduled-pg-20181219-094347   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094447   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094547   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094647   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094747   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094847   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094947   scheduled-pg   Succeeded   1h
```

Some [other fields](https://github.com/kubedb/apimachinery/blob/6319e29148b40f1ac9a7ea312394754e83feba8e/apis/kubedb/v1alpha1/postgres_types.go#L90-L127) also got deprecated and added. The good thing is, kubedb operator will handle those changes in it's mutating webhook. (So, always try to run kubedb with webhooks enabled). But, user has to update the db-version on it's own.

## Upgrade CRD objects

Note that, Once the DB version is updated, kubedb-operator will update the statefulsets too. This update strategy can be modified by `spec.updateStrategy`. Read [here](https://kubedb.com/docs/0.9.0/concepts/databases/postgres/#spec-updatestrategy) for details.

Now, Before updating CRD, find Available PostgresVersion.

```console
$ kubectl get postgresversions
NAME       VERSION   DB_IMAGE                   DEPRECATED   AGE
10.2       10.2      kubedb/postgres:10.2       true         1h
10.2-v1    10.2      kubedb/postgres:10.2-v2                 1h
9.6        9.6       kubedb/postgres:9.6        true         1h
9.6-v1     9.6       kubedb/postgres:9.6-v2                  1h
9.6.7      9.6.7     kubedb/postgres:9.6.7      true         1h
9.6.7-v1   9.6.7     kubedb/postgres:9.6.7-v2                1h
```

Notice the `DEPRECATED` column. Here, `true` means that this PostgresVersion is deprecated for current KubeDB version. KubeDB will not work for deprecated PostgresVersion. To know more about what is `PostgresVersion` crd and why there is `10.2` and `10.2-v1` variation, please visit [here](/docs/concepts/catalog/postgres.md).

Now, Update the CRD and set `Spec.version` to `9.6-v1`.

```console
kubectl edit pg -n demo scheduled-pg
```

See the changed object (defaulted).

```yaml
$ kubectl get pg -n demo scheduled-pg -o yaml
apiVersion: kubedb.com/v1alpha1
kind: Postgres
metadata:
  creationTimestamp: "2018-12-19T09:25:52Z"
  finalizers:
  - kubedb.com
  generation: 5
  name: scheduled-pg
  namespace: demo
  resourceVersion: "27252"
  selfLink: /apis/kubedb.com/v1alpha1/namespaces/demo/postgreses/scheduled-pg
  uid: 144abd0f-0370-11e9-8ff4-080027860845
spec:
  backupSchedule:
    cronExpression: '@every 1m'
    gcs:
      bucket: kubedb
    podTemplate:
      controller: {}
      metadata: {}
      spec:
        resources: {}
    storageSecretName: gcs-secret
  databaseSecret:
    secretName: scheduled-pg-auth
  podTemplate:
    controller: {}
    metadata: {}
    spec:
      resources: {}
  replicas: 3
  serviceTemplate:
    metadata: {}
    spec: {}
  storage:
    accessModes:
    - ReadWriteOnce
    dataSource: null
    resources:
      requests:
        storage: 1Gi
    storageClassName: standard
  storageType: Durable
  terminationPolicy: Pause
  updateStrategy:
    type: RollingUpdate
  version: 9.6-v1
status:
  observedGeneration: 5$4214054550087021099
  phase: Running
```

Watch for pod updates in `RollingUpdate` UpdateStrategy.

```console
$ kubectl get po -n demo -w
NAMESPACE     NAME     READY   STATUS    RESTARTS   AGE
...
demo   scheduled-pg-2   1/1   Terminating   0     107m
demo   scheduled-pg-2   0/1   Terminating   0     107m
demo   scheduled-pg-2   0/1   Terminating   0     107m
demo   scheduled-pg-2   0/1   Terminating   0     107m
demo   scheduled-pg-2   0/1   Pending   0     0s
demo   scheduled-pg-2   0/1   Pending   0     0s
demo   scheduled-pg-2   0/1   ContainerCreating   0     0s
demo   scheduled-pg-2   0/1   ContainerCreating   0     19s
demo   scheduled-pg-2   1/1   Running   0     19s
demo   scheduled-pg-1   1/1   Terminating   0     108m
demo   scheduled-pg-1   0/1   Terminating   0     108m
demo   scheduled-pg-1   0/1   Terminating   0     108m
demo   scheduled-pg-1   0/1   Terminating   0     108m
demo   scheduled-pg-1   0/1   Pending   0     0s
demo   scheduled-pg-1   0/1   Pending   0     0s
demo   scheduled-pg-1   0/1   ContainerCreating   0     0s
demo   scheduled-pg-1   0/1   ContainerCreating   0     1s
demo   scheduled-pg-1   1/1   Running   0     2s
demo   scheduled-pg-0   1/1   Terminating   0     108m
demo   scheduled-pg-0   0/1   Terminating   0     109m
demo   scheduled-pg-0   0/1   Terminating   0     109m
demo   scheduled-pg-0   0/1   Terminating   0     109m
demo   scheduled-pg-0   0/1   Pending   0     0s
demo   scheduled-pg-0   0/1   Pending   0     0s
demo   scheduled-pg-0   0/1   ContainerCreating   0     0s
demo   scheduled-pg-0   0/1   ContainerCreating   0     1s
demo   scheduled-pg-0   1/1   Running   0     2s
```

Watch for statefulset states.

```console
$ kubectl get statefulset -n demo -w
NAME           READY   AGE
scheduled-pg   3/3     105m
scheduled-pg   3/3   107m
scheduled-pg   3/3   107m
scheduled-pg   2/3   107m
scheduled-pg   2/3   107m
scheduled-pg   3/3   108m
scheduled-pg   2/3   108m
scheduled-pg   2/3   108m
scheduled-pg   3/3   108m
scheduled-pg   2/3   109m
scheduled-pg   2/3   109m
scheduled-pg   3/3   109m
```

The scheduled snapshot is in working state now.

```console
$ kubectl get snap -n demo
NAME                           DATABASENAME   STATUS      AGE
scheduled-pg-20181219-094347   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094447   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094547   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094647   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094747   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094847   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-094947   scheduled-pg   Succeeded   1h
scheduled-pg-20181219-111319   scheduled-pg   Succeeded   2m
scheduled-pg-20181219-111419   scheduled-pg   Succeeded   1m
scheduled-pg-20181219-111519   scheduled-pg   Running     4s
```

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```console
kubectl patch -n demo pg/scheduled-pg -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo pg/scheduled-pg

kubectl delete ns demo
```

## Next Steps

- Learn about [custom PostgresVersions](/docs/guides/postgres/custom-versions/setup.md).
- Want to setup PostgreSQL cluster? Check how to [configure Highly Available PostgreSQL Cluster](/docs/guides/postgres/clustering/ha_cluster.md)
- Detail concepts of [Postgres object](/docs/concepts/databases/postgres.md).
- Detail concepts of [Snapshot object](/docs/concepts/snapshot.md).
- Want to hack on KubeDB? Check our [contribution guidelines](/docs/CONTRIBUTING.md).