<h1>User Guide</h1>

Learn how to work with the Postgres Operator in a Kubernetes (K8s) environment.

## Create a manifest for a new PostgreSQL cluster

Make sure you have [set up](quickstart.md) the operator. Then you can create a
new Postgres cluster by applying manifest like this [minimal example](../manifests/minimal-postgres-manifest.yaml):

```yaml
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: acid-minimal-cluster
spec:
  teamId: "acid"
  volume:
    size: 1Gi
  numberOfInstances: 2
  users:
    # database owner
    zalando:
    - superuser
    - createdb

    # role for application foo
    foo_user: # or 'foo_user: []'

  #databases: name->owner
  databases:
    foo: zalando
  postgresql:
    version: "11"
```

Once you cloned the Postgres Operator [repository](https://github.com/zalando/postgres-operator)
you can find this example also in the manifests folder:

```bash
kubectl create -f manifests/minimal-postgres-manifest.yaml
```

Make sure, the `spec` section of the manifest contains at least a `teamId`, the
`numberOfInstances` and the `postgresql` object with the `version` specified.
The minimum volume size to run the `postgresql` resource on Elastic Block
Storage (EBS) is `1Gi`.

Note, that the name of the cluster must start with the `teamId` and `-`. At
Zalando we use team IDs (nicknames) to lower the chance of duplicate cluster
names and colliding entities. The team ID would also be used to query an API to
get all members of a team and create [database roles](#teams-api-roles) for
them.

## Watch pods being created

```bash
kubectl get pods -w --show-labels
```

## Connect to PostgreSQL

With a `port-forward` on one of the database pods (e.g. the master) you can
connect to the PostgreSQL database. Use labels to filter for the master pod of
our test cluster.

```bash
# get name of master pod of acid-minimal-cluster
export PGMASTER=$(kubectl get pods -o jsonpath={.items..metadata.name} -l application=spilo,version=acid-minimal-cluster,spilo-role=master)

# set up port forward
kubectl port-forward $PGMASTER 6432:5432
```

Open another CLI and connect to the database. Use the generated secret of the
`postgres` robot user to connect to our `acid-minimal-cluster` master running
in Minikube. As non-encrypted connections are rejected by default set the SSL
mode to require:

```bash
export PGPASSWORD=$(kubectl get secret postgres.acid-minimal-cluster.credentials -o 'jsonpath={.data.password}' | base64 -d)
export PGSSLMODE=require
psql -U postgres -p 6432
```

## Defining database roles in the operator

Postgres Operator allows defining roles to be created in the resulting database
cluster. It covers three use-cases:

* `manifest roles`: create application roles specific to the cluster described
in the manifest.
* `infrastructure roles`: create application roles that should be automatically
created on every cluster managed by the operator.
* `teams API roles`: automatically create users for every member of the team
owning the database cluster.

In the next sections, we will cover those use cases in more details.

### Manifest roles

Manifest roles are defined directly in the cluster manifest. See
[minimal postgres manifest](../manifests/minimal-postgres-manifest.yaml)
for an example of `zalando` role, defined with `superuser` and `createdb` flags.

Manifest roles are defined as a dictionary, with a role name as a key and a
list of role options as a value. For a role without any options it is best to
supply the empty list `[]`. It is also possible to leave this field empty as in
our example manifests. In certain cases such empty field may be missing later
removed by K8s [due to the `null` value it gets](https://kubernetes.io/docs/concepts/overview/object-management-kubectl/declarative-config/#how-apply-calculates-differences-and-merges-changes)
(`foobar_user:` is equivalent to `foobar_user: null`).

The operator accepts the following options:  `superuser`, `inherit`, `login`,
`nologin`, `createrole`, `createdb`, `replication`, `bypassrls`.

By default, manifest roles are login roles (aka users), unless `nologin` is
specified explicitly.

The operator automatically generates a password for each manifest role and
places it in the secret named
`{username}.{team}-{clustername}.credentials.postgresql.acid.zalan.do` in the
same namespace as the cluster. This way, the application running in the
K8s cluster and connecting to Postgres can obtain the password right from the
secret, without ever sharing it outside of the cluster.

At the moment it is not possible to define membership of the manifest role in
other roles.

### Infrastructure roles

An infrastructure role is a role that should be present on every PostgreSQL
cluster managed by the operator. An example of such a role is a monitoring
user. There are two ways to define them:

* With the infrastructure roles secret only
* With both the the secret and the infrastructure role ConfigMap.

#### Infrastructure roles secret

The infrastructure roles secret is specified by the `infrastructure_roles_secret_name`
parameter. The role definition looks like this (values are base64 encoded):

```yaml
user1: ZGJ1c2Vy
password1: c2VjcmV0
inrole1: b3BlcmF0b3I=
```

The block above describes the infrastructure role 'dbuser' with password
'secret' that is a member of the 'operator' role. For the following definitions
one must increase the index, i.e. the next role will be defined as 'user2' and
so on. The resulting role will automatically be a login role.

Note that with definitions that solely use the infrastructure roles secret
there is no way to specify role options (like superuser or nologin) or role
memberships. This is where the ConfigMap comes into play.

#### Secret plus ConfigMap

A [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
allows for defining more details regarding the infrastructure roles. Therefore,
one should use the new style that specifies infrastructure roles using both the
secret and a ConfigMap. The ConfigMap must have the same name as the secret.
The secret should contain an entry with 'rolename:rolepassword' for each role.

```yaml
dbuser: c2VjcmV0
```

And the role description for that user should be specified in the ConfigMap.

```yaml
data:
  dbuser: |
    inrole: [operator, admin]  # following roles will be assigned to the new user
    user_flags:
    - createdb
    db_parameters:  # db parameters, applied for this particular user
      log_statement: all
```

One can allow membership in multiple roles via the `inrole` array parameter,
define role flags via the `user_flags` list and supply per-role options through
the `db_parameters` dictionary. All those parameters are optional.

Both definitions can be mixed in the infrastructure role secret, as long as
your new-style definition can be clearly distinguished from the old-style one
(for instance, do not name new-style roles `userN`).

Since an infrastructure role is created uniformly on all clusters managed by
the operator, it makes no sense to define it without the password. Such
definitions will be ignored with a prior warning.

See [infrastructure roles secret](../manifests/infrastructure-roles.yaml)
and [infrastructure roles configmap](../manifests/infrastructure-roles-configmap.yaml)
for the examples.

### Teams API roles

These roles are meant for database activity of human users. It's possible to
configure the operator to automatically create database roles for lets say all
employees of one team. They are not listed in the manifest and there are no K8s
secrets created for them. Instead they would use an OAuth2 token to connect. To
get all members of the team the operator queries a defined API endpoint that
returns usernames. A minimal Teams API should work like this:

```
/.../<teamname> -> ["name","anothername"]
```

A ["fake" Teams API](../manifests/fake-teams-api.yaml) deployment is provided
in the manifests folder to set up a basic API around whatever services is used
for user management. The Teams API's URL is set in the operator's
[configuration](reference/operator_parameters.md#automatic-creation-of-human-users-in-the-database)
and `enable_teams_api` must be set to `true`. There are more settings available
to choose superusers, group roles, [PAM configuration](https://github.com/CyberDem0n/pam-oauth2)
etc. An OAuth2 token can be passed to the Teams API via a secret. The name for
this secret is configurable with the `oauth_token_secret_name` parameter.

## Resource definition

The compute resources to be used for the Postgres containers in the pods can be
specified in the postgresql cluster manifest.

```yaml
spec:
  resources:
    requests:
      cpu: 10m
      memory: 100Mi
    limits:
      cpu: 300m
      memory: 300Mi
```

The minimum limit to properly run the `postgresql` resource is `256m` for `cpu`
and `256Mi` for `memory`. If a lower value is set in the manifest the operator
will cancel ADD or UPDATE events on this resource with an error. If no
resources are defined in the manifest the operator will obtain the configured
[default requests](reference/operator_parameters.md#kubernetes-resource-requests).

## Use taints and tolerations for dedicated PostgreSQL nodes

To ensure Postgres pods are running on nodes without any other application pods,
you can use [taints and tolerations](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)
and configure the required toleration in the manifest.

```yaml
spec:
  tolerations:
  - key: postgres
    operator: Exists
    effect: NoSchedule
```

## How to clone an existing PostgreSQL cluster

You can spin up a new cluster as a clone of the existing one, using a clone
section in the spec. There are two options here:

* Clone directly from a source cluster using `pg_basebackup`
* Clone from an S3 bucket

### Clone directly

```yaml
spec:
  clone:
    cluster: "acid-batman"
```

Here `cluster` is a name of a source cluster that is going to be cloned. The
cluster to clone is assumed to be running and the clone procedure invokes
`pg_basebackup` from it. The operator will setup the cluster to be cloned to
connect to the service of the source cluster by name (if the cluster is called
test, then the connection string will look like host=test port=5432), which
means that you can clone only from clusters within the same namespace.

### Clone from S3

```yaml
spec:
  clone:
    uid: "efd12e58-5786-11e8-b5a7-06148230260c"
    cluster: "acid-batman"
    timestamp: "2017-12-19T12:40:33+01:00"
```

Here `cluster` is a name of a source cluster that is going to be cloned. A new
cluster will be cloned from S3, using the latest backup before the `timestamp`.
In this case, `uid` field is also mandatory - operator will use it to find a
correct key inside an S3 bucket. You can find this field in the metadata of the
source cluster:

```yaml
apiVersion: acid.zalan.do/v1
kind: postgresql
metadata:
  name: acid-test-cluster
  uid: efd12e58-5786-11e8-b5a7-06148230260c
```

Note that timezone is required for `timestamp`. Otherwise, offset is relative
to UTC, see [RFC 3339 section 5.6) 3339 section 5.6](https://www.ietf.org/rfc/rfc3339.txt).

For non AWS S3 following settings can be set to support cloning from other S3
implementations:

```yaml
spec:
  clone:
    uid: "efd12e58-5786-11e8-b5a7-06148230260c"
    cluster: "acid-batman"
    timestamp: "2017-12-19T12:40:33+01:00"
    s3_endpoint: https://s3.acme.org
    s3_access_key_id: 0123456789abcdef0123456789abcdef
    s3_secret_access_key: 0123456789abcdef0123456789abcdef
    s3_force_path_style: true
```

## Setting up a standby cluster

Standby clusters are like normal cluster but they are streaming from a remote
cluster. As the first version of this feature, the only scenario covered by
operator is to stream from a WAL archive of the master. Following the more
popular infrastructure of using Amazon's S3 buckets, it is mentioned as
`s3_wal_path` here. To start a cluster as standby add the following `standby`
section in the YAML file:

```yaml
spec:
  standby:
    s3_wal_path: "s3 bucket path to the master"
```

Things to note:

- An empty string in the `s3_wal_path` field of the standby cluster will result
  in an error and no statefulset will be created.
- Only one pod can be deployed for stand-by cluster.
- To manually promote the standby_cluster, use `patronictl` and remove config
  entry.
- There is no way to transform a non-standby cluster to a standby cluster
  through the operator. Adding the standby section to the manifest of a running
  Postgres cluster will have no effect. However, it can be done through Patroni
  by adding the [standby_cluster](https://github.com/zalando/patroni/blob/bd2c54581abb42a7d3a3da551edf0b8732eefd27/docs/replica_bootstrap.rst#standby-cluster)
  section using `patronictl edit-config`. Note that the transformed standby
  cluster will not be doing any streaming. It will be in standby mode and allow
  read-only transactions only.

## Sidecar Support

Each cluster can specify arbitrary sidecars to run. These containers could be
used for log aggregation, monitoring, backups or other tasks. A sidecar can be
specified like this:

```yaml
spec:
  sidecars:
    - name: "container-name"
      image: "company/image:tag"
      resources:
        limits:
          cpu: 500m
          memory: 500Mi
        requests:
          cpu: 100m
          memory: 100Mi
      env:
        - name: "ENV_VAR_NAME"
          value: "any-k8s-env-things"
```

In addition to any environment variables you specify, the following environment
variables are always passed to sidecars:

  - `POD_NAME` - field reference to `metadata.name`
  - `POD_NAMESPACE` - field reference to `metadata.namespace`
  - `POSTGRES_USER` - the superuser that can be used to connect to the database
  - `POSTGRES_PASSWORD` - the password for the superuser

The PostgreSQL volume is shared with sidecars and is mounted at
`/home/postgres/pgdata`.

**Note**: The operator will not create a cluster if sidecar containers are
specified but globally disabled in the configuration. The `enable_sidecars`
option must be set to `true`.

## InitContainers Support

Each cluster can specify arbitrary init containers to run. These containers can
be used to run custom actions before any normal and sidecar containers start.
An init container can be specified like this:

```yaml
spec:
  initContainers:
    - name: "container-name"
      image: "company/image:tag"
      env:
        - name: "ENV_VAR_NAME"
          value: "any-k8s-env-things"
```

`initContainers` accepts full `v1.Container` definition.

**Note**: The operator will not create a cluster if `initContainers` are
specified but globally disabled in the configuration. The
`enable_init_containers` option must be set to `true`.

## Increase volume size

Postgres operator supports statefulset volume resize if you're using the
operator on top of AWS. For that you need to change the size field of the
volume description in the cluster manifest and apply the change:

```yaml
spec:
  volume:
    size: 5Gi # new volume size
```

The operator compares the new value of the size field with the previous one and
acts on differences.

You can only enlarge the volume with the process described above, shrinking is
not supported and will emit a warning. After this update all the new volumes in
the statefulset are allocated according to the new size. To enlarge persistent
volumes attached to the running pods, the operator performs the following
actions:

* call AWS API to change the volume size

* connect to pod using `kubectl exec` and resize filesystem with `resize2fs`.

Fist step has a limitation, AWS rate-limits this operation to no more than once
every 6 hours. Note, that if the statefulset is scaled down before resizing the
new size is only applied to the volumes attached to the running pods. The
size of volumes that correspond to the previously running pods is not changed.

## Logical backups

You can enable logical backups from the cluster manifest by adding the following
parameter in the spec section:

```yaml
spec:
  enableLogicalBackup: true
```

The operator will create and sync a K8s cron job to do periodic logical backups
of this particular Postgres cluster. Due to the [limitation of K8s cron jobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#cron-job-limitations)
it is highly advisable to set up additional monitoring for this feature; such
monitoring is outside the scope of operator responsibilities. See
[configuration reference](reference/cluster_manifest.md) and
[administrator documentation](administrator.md) for details on how backups are
executed.

## TLS configuration

By default self-signed certificates are generated when the postgres container
starts up. But it's also possible to provide your own secrets like so:

Upload the cert as a kubernetes secret:
```sh
kubectl create secret tls pg-tls \
  --key pg-tls.key \
  --cert pg-tls.crt
```

Or with a CA:
```sh
kubectl create secret generic pg-tls \
  --from-file=tls.crt=server.crt \
  --from-file=tls.key=server.key \
  --from-file=ca.crt=ca.crt
```

Configure the postgres resource with the TLS secret:

```yaml
apiVersion: "acid.zalan.do/v1"
kind: postgresql

metadata:
  name: acid-test-cluster
spec:
  tls:
    secretName: "pg-tls"
```

Before applying these changes, the operator must also be configured with the
`spilo_fsgroup` set to the GID matching the postgres user group. If the value
is not provided, the cluster will default to `103` which is the GID from the
default spilo image.
