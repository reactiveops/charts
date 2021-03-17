# Fairwinds Insights

[Fairwinds Insights](https://insights.fairwinds.com) Software to automate, monitor, and enforce Kubernetes best practices.

The self-hosted version of Fairwinds Insights is currently in alpha. The documentation is incomplete, and it is subject to breaking changes.

## Installation

```

We recommend installing Fairwinds Insights in its own namespace and with a simple release name:

```bash
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
# These options are for quickstart only. See documentation for hardening tips
helm install fairwinds-insights fairwinds-stable/fairwinds-insights \
  --namespace fairwinds-insights \
  --create-namespace=true \
  --set options.autogenerateKeys=true \
  --set options.allowHTTPCookies=true \
  --set postgresql.sslMode=disable
```

### Hardening
The default configuration will give you a working version of Fairwinds Insights.
But there are a few issues you'll want to solve before starting to use it seriously:
* Data: You'll want to set up durable storage for the database (Postgres) and files (S3 or Minio).
* Ingress: You'll want to host Insights behind a custom domain
* Sessions: Generate permanent session keys in order to preserve running sessions when Insights is updated.
* Email: In order to confirm email addresses and add new users, you'll need to set up an email provider.

Some of these things simply involve passing new data to the Helm chart. Others
may require the creation of new Kubernetes secrets.

For example, a hardened setup might look like this:

_values.yaml_
```yaml
fairwindsInsights:
  host: https://insights.example.com
  adminEmail: admin@example.com
ingress:
  enabled: true
  hostedZones:
    - insights.example.com
  annotations:
    kubernetes.io/ingress.class: "nginx-ingress"
email:
  strategy: smtp
  smtpHost: smtp.gmail.com
  smtpUsername: you@gmail.com
  smtpPort: "465"
postgresql:
  ephemeral: false
  passwordSecret: fwinsights-postgresql
  postgresqlUsername: your_username
  postgresqlDatabase: desired_database_name
  postgresqlHost: your-server.us-east-1.rds.amazonaws.com
  sslMode: require
  service:
    port: 5432
```

_secrets.yaml_
```yaml
apiVersion: v1
data:
    cookie_hash_key: TjZCbzAzMVJ0U1lXS2RPaE9sU0ZERDJJdXpTNFhLeG91VWFYdU9DcU9kTkpmenlFNWFsT29sajZ3VGpNbjNSSA==
    cookie_block_key: RDFNVVhWVzM4QllJd0Z5NXNIT1kxV1RrVWNweWRnbmM=
    session_auth_key: NTczV0o0NGFBeGtXVzduZjZHV25FdFZMb0s5TE5SSWU3bG00YkNtaE93bHZUVW1VSXZUYW9ya2UzdHE2eFZXSA==
    session_encryption_key: ME1LeFRPdTYwOVU0YlVvUGVEUUdQYjZnbTlyZUh4b2I=
    smtp_password: aGVsbG93b3JsZA==
    aws_access_key_id: aGVsbG93b3JsZA==
    aws_secret_access_key: aGVsbG93b3JsZA==
kind: Secret
metadata:
    name: fwinsights-secrets
type: Opaque
```

_postgres-secret.yaml_
```yaml
apiVersion: v1
data:
    postgresql-password: aGVsbG93b3JsZA==
kind: Secret
metadata:
    name: fwinsights-postgresql
type: Opaque
```

See below for additional details and options.

## Session keys
In the default install, we autogenerate session and cookie keys each time the application is
installed or upgraded, which could cause disruption for users. Instead, you should generate
these tokens once and save them in a Kubernetes secret.

> Most secrets (with the exception of your PostgreSQL password) must stored in a secret
> named `fwinsights-secrets`

We'll need the following keys:
* cookie_hash_key (64 chars)
* cookie_block_key (32 chars)
* session_auth_key (64 chars)
* session_encryption_key (32 chars)

An easy way to generate these keys is:
```bash
cookie_hash_key=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 64 | base64 -w 0)
cookie_block_key=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 32 | base64 -w 0)
session_auth_key=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 64 | base64 -w 0)
session_encryption_key=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 32 | base64 -w 0)
```

You should use these values to populate _secrets.yaml_:
```yaml
apiVersion: v1
data:
    cookie_hash_key: TjZCbzAzMVJ0U1lXS2RPaE9sU0ZERDJJdXpTNFhLeG91VWFYdU9DcU9kTkpmenlFNWFsT29sajZ3VGpNbjNSSA==
    cookie_block_key: RDFNVVhWVzM4QllJd0Z5NXNIT1kxV1RrVWNweWRnbmM=
    session_auth_key: NTczV0o0NGFBeGtXVzduZjZHV25FdFZMb0s5TE5SSWU3bG00YkNtaE93bHZUVW1VSXZUYW9ya2UzdHE2eFZXSA==
    session_encryption_key: ME1LeFRPdTYwOVU0YlVvUGVEUUdQYjZnbTlyZUh4b2I=
kind: Secret
metadata:
    name: fwinsights-secrets
type: Opaque
```

```bash
kubectl apply -f secrets.yaml -n fwinsights
```
Consider using
[SOPS](https://github.com/mozilla/sops)
to encrypt your secrets file and store it in your git repository.

## Database
Fairwinds Insights requires a Postgres database to store backend data.

### Ephemeral Postgres
By default, Insights will install Postgres via
[its helm chart](https://github.com/helm/charts/tree/master/stable/postgresql).
We don't recommend running databases in Kubernetes, due to the possibility of lost data.
If you use this option, keep in mind you'll be responsible for maintaining
and backing up that database.

```yaml
postgresql:
  ephemeral: true
  postgresqlPassword: fwinsightstmp
  postgresqlUsername: postgres
  postgresqlDatabase: fairwinds_insights
  service:
    port: 5432
  persistence:
    enabled: false
  replication:
    enabled: false
```

### BYO Postgres
If you'd like to use your own Postgres instance (e.g. on Amazon RDS),
you'll need to point the Insights chart to your database:

_values.yaml_
```yaml
postgresql:
  ephemeral: false
  passwordSecret: fwinsights-postgresql
  postgresqlUsername: your_username
  postgresqlDatabase: desired_database_name
  postgresqlHost: your-server.us-east-1.rds.amazonaws.com
  sslMode: require
  service:
    port: 5432
```
The password for your postgres database should be in a secret stored in Kubernetes,
using the key `postgresql-password`. Make sure the `name` matches `passwordSecret` above.

This example creates a secret with the text `helloworld`:
```bash
echo -n "helloworld" | base64
# aGVsbG93b3JsZA==
```

_postgres-secret.yaml_
```yaml
apiVersion: v1
data:
    postgresql-password: aGVsbG93b3JsZA==
kind: Secret
metadata:
    name: fwinsights-postgresql
type: Opaque
```
```bash
kubectl apply -f postgres-secret.yaml -n fwinsights
```

## File Storage
When clusters report back to Fairwinds Insights (e.g. with a list of Trivy scans, or Polaris results),
Insights stores that data as a JSON file. In order to use Insights, you'll need a place to store those
files. Currently we support two options

* Amazon S3
* [Minio](https://min.io/), an open source alternative to S3

In the default installation, we use an ephemeral instance of [minio](https://min.io/),
but you'll want something more resilient when running in production to ensure you don't lose
any data.

### Amazon S3
To use Amazon S3, set your bucket name and region in _values.yaml_:

```yaml
reportStorage:
  strategy: s3
  bucket: your-bucket-name
  awsRegion: us-east-1
```

You'll also need to specify your AWS access key and secret in _secrets.yaml_:
```yaml
apiVersion: v1
data:
    aws_access_key_id: aGVsbG93b3JsZA==
    aws_secret_access_key: aGVsbG93b3JsZA==
kind: Secret
metadata:
    name: fwinsights-secrets
type: Opaque
```

Note that if you're using other AWS integrations (like SES below) they will use the same AWS credentials.

### Minio
You can use your own instance of Minio, or install a copy of Minio alongside Insights.

To have the Insights chart install Minio, you can configure it with the `minio` option:

```yaml
reportStorage:
  strategy: minio
minio:
  install: true
  accessKey: fwinsights
  secretKey: fwinsights
  persistence:
    enabled: true
```

In particular, you should set `minio.persistence.enabled=true` to use a PersistentVolume for your
data. You can see the [full chart configuration here](https://github.com/helm/charts/tree/master/stable/minio)

To use an existing installation of Minio, just set `reportStorage.minioHost`
```yaml
reportStorage:
  strategy: minio
  minioHost: minio.example.com
```

## Email
Email can be sent via AWS SES, or by specifying your own SMTP options.

### SES

First, specify the `ses` strategy in _values.yaml_
```yaml
email:
  strategy: ses
```

Then you'll need to specify your base64-encoded AWS credentials, and add them to your
_secrets.yaml_:
```yaml
apiVersion: v1
data:
    aws_access_key_id: aGVsbG93b3JsZA==
    aws_secret_access_key: aGVsbG93b3JsZA==
kind: Secret
metadata:
    name: fwinsights-secrets
type: Opaque
```

Note that if you're using other AWS integrations (like S3 above) they will use the same AWS credentials.

### SMTP
You can follow
[these instructions](https://kinsta.com/knowledgebase/free-smtp-server/#step-2-send-mail-as-google-smtp)
for using a Gmail account with SMTP.

_values.yaml_
```yaml
email:
  strategy: smtp
  smtpHost: smtp.gmail.com
  smtpUsername: you@gmail.com
  smtpPort: "465"
```

You'll need to put the password in your _secrets.yaml_
```yaml
apiVersion: v1
data:
    smtp_password: aGVsbG93b3JsZA==
kind: Secret
metadata:
    name: fwinsights-secrets
type: Opaque
```

## Ingress
The Helm chart comes with an Ingress object in order to expose a URL for connecting to Insights.
Here's an example configuration, using cert-manager and nginx-ingress.

Note that we allow up to `24m` of data in request bodies - this is important, as some of the
reports sent back from clusters can be fairly large.

_values.yaml_
```yaml
ingress:
  enabled: true
  hostedZones:
    - insights.example.com
  annotations:
    kubernetes.io/ingress.class: "nginx-ingress"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    kubernetes.io/ingress.ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-connections: "250"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-rpm: "5000"
    nginx.ingress.kubernetes.io/proxy-body-size: 24m
```

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| image.tag | string | `"3.2.4"` | Docker image tag |
| dashboardImage.repository | string | `"quay.io/fairwinds/insights-dashboard"` | Docker image repository for the front end |
| apiImage.repository | string | `"quay.io/fairwinds/insights-api"` | Docker image repository for the API server |
| migrationImage.repository | string | `"quay.io/fairwinds/insights-db-migration"` | Docker image repository for the database migration job |
| cronjobImage.repository | string | `"quay.io/fairwinds/insights-cronjob"` | Docker image repository for maintenance CronJobs. |
| options.agentChartTargetVersion | string | `"1.9.2"` | Which version of the Insights Agent is supported by this version of Fairwinds Insights |
| options.insightsSAASHost | string | `"https://insights.fairwinds.com"` | Do not change, this is the hostname that Fairwinds Insights will reach out to for license verification. |
| options.allowHTTPCookies | bool | `false` | Allow cookies to work over HTTP instead of requiring HTTPS. This generally should not be changed. |
| options.dashboardConfig | string | `"config.self.js"` | Configuration file to use for the front-end. This generally should not be changed. |
| options.autogenerateKeys | bool | `false` | Autogenerate keys for session tracking. For testing/demo purposes only |
| additionalEnvironmentVariables | string | `nil` | Additional Environment Variables to set on the Fairwinds Insights pods. |
| dashboard.pdb.enabled | bool | `false` | Create a pod disruption budget for the front end pods. |
| dashboard.pdb.minReplicas | int | `1` | How many replicas should always exist for the front end pods. |
| dashboard.hpa.enabled | bool | `false` | Create a horizontal pod autoscaler for the front end pods. |
| dashboard.hpa.min | int | `2` | Minimum number of replicas for the front end pods. |
| dashboard.hpa.max | int | `4` | Maximum number of replicas for the front end pods. |
| dashboard.hpa.cpuTarget | int | `50` | Target CPU utilization for the front end pods. |
| dashboard.resources | object | `{"limits":{"cpu":"1000m","memory":"1024Mi"},"requests":{"cpu":"250m","memory":"256Mi"}}` | Resources for the front end pods. |
| dashboard.nodeSelector | object | `{}` | Node Selector for the front end pods. |
| dashboard.tolerations | list | `[]` | Tolerations for the front end pods. |
| dashboard.securityContext.runAsUser | int | `101` | The user ID to run the Dashboard under. comes from https://github.com/nginxinc/docker-nginx-unprivileged/blob/main/stable/alpine/Dockerfile |
| api.port | int | `8080` | Port for the API server to listen on. |
| api.pdb.enabled | bool | `false` | Create a pod disruption budget for the API server. |
| api.hpa.enabled | bool | `false` | Create a horizontal pod autoscaler for the API server. |
| api.hpa.min | int | `2` | Minimum number of replicas for the API server. |
| api.hpa.max | int | `4` | Maximum number of replicas for the API server. |
| api.hpa.cpuTarget | int | `50` | Target CPU utilization for the API server. |
| api.resources | object | `{"limits":{"cpu":"1000m","memory":"1024Mi"},"requests":{"cpu":"250m","memory":"256Mi"}}` | Resources for the API server. |
| api.nodeSelector | object | `{}` | Node Selector for the API server. |
| api.tolerations | list | `[]` | Tolerations for the API server. |
| api.securityContext.runAsUser | int | `10324` | The user ID to run the API server under. |
| dbMigration.resources | object | `{"limits":{"cpu":1,"memory":"1024Mi"},"requests":{"cpu":"80m","memory":"128Mi"}}` | Resources for the database migration job. |
| dbMigration.securityContext.runAsUser | int | `10324` | The user ID to run the database migration job under. |
| alertsCronjob.resources | object | `{"limits":{"cpu":"500m","memory":"1024Mi"},"requests":{"cpu":"80m","memory":"128Mi"}}` | Resources for the Slack/Datadog integrations |
| alertsCronjob.schedules | list | `[{"cron":"5/10 * * * *","interval":"10m","name":"realtime"},{"cron":"0 16 * * *","interval":"24h","name":"digest"}]` | CRON schedules for the Slack/Datadog integrations |
| alertsCronjob.securityContext.runAsUser | int | `10324` | The user ID to run the alerts job under. |
| aggregateCronjob.resources | object | `{"limits":{"cpu":"250m","memory":"512Mi"},"requests":{"cpu":"40m","memory":"32Mi"}}` | Resources for the Workload Metrics aggregation job. |
| aggregateCronjob.schedules | list | `[{"cron":"5 0/2 * * *","interval":"120m","name":"bi-hourly"}]` | CRON schedules for the Workload Metrics aggregation job. |
| aggregateCronjob.securityContext.runAsUser | int | `10324` | The user ID to run the Workload Metrics aggregation job under. |
| emailCronjob.resources | object | `{"limits":{"cpu":"500m","memory":"1024Mi"},"requests":{"cpu":"80m","memory":"128Mi"}}` | Resources for the Action Items email job. |
| emailCronjob.schedules | list | `[{"cron":"0 16 * * 1","interval":"168h","name":"weekly-email"}]` | CRON schedules for the Action Items email job. |
| emailCronjob.securityContext.runAsUser | int | `10324` | The user ID to run the email job under. |
| deleteOldActionItemsCronjob.resources | object | `{"limits":{"cpu":"500m","memory":"1024Mi"},"requests":{"cpu":"80m","memory":"128Mi"}}` | Resources for the delete old Action Items job. |
| deleteOldActionItemsCronjob.schedules | list | `[{"cron":"0 0 * * *","interval":"24h","name":"ai-cleanup"}]` | CRON schedules for the delete old Action Items job. |
| deleteOldActionItemsCronjob.securityContext.runAsUser | int | `10324` | The user ID to run the delete Action Items job under. |
| service.port | int | `80` | Port to be used for the API and Dashboard services. |
| sanitizedBranch | string | `nil` | Prefix to use on hostname. Generally not needed. |
| ingress.tls | bool | `true` | Enable TLS |
| ingress.hostedZones | list | `[]` | Hostnames to use for Ingress |
| ingress.annotations | string | `nil` | Annotations to add to the API and Dashboard ingresses. |
| ingressApi.enabled | bool | `false` | Install API Ingress object. |
| ingressApi.annotations | string | `nil` | Annotations to add to the API ingress. |
| ingressDashboard.enabled | bool | `false` | Install Dashboard Ingress object. |
| ingressDashboard.annotations | string | `nil` | Annotations to add to the Dashboard ingress. |
| postgresql.ephemeral | bool | `true` | Use the ephemeral postgresql chart by default |
| postgresql.sslMode | string | `"require"` | SSL mode for connecting to the database |
| postgresql.existingSecret | string | `"fwinsights-postgresql"` | Secret name to use for Postgres Password |
| postgresql.postgresqlUsername | string | `"postgres"` | Username to connect to Postgres with |
| postgresql.postgresqlDatabase | string | `"fairwinds_insights"` | Name of the Postgres Database |
| postgresql.randomReadOnlyPassword | bool | `true` | Create a read only user with a random password. |
| postgresql.service.port | int | `5432` | Port of the Postgres Database |
| postgresql.persistence.enabled | bool | `true` | Create Persistent Volume with Postgres |
| postgresql.replication.enabled | bool | `false` | Replicate Postgres data |
| postgresql.resources | object | `{"limits":{"cpu":1,"memory":"1Gi"},"requests":{"cpu":"75m","memory":"256Mi"}}` | Resources section for Postgres |
| email.strategy | string | `"memory"` | How to send emails, valid values include memory, ses, and smtp |
| email.sender | string | `nil` | Email address that emails will come from |
| email.recipient | string | `nil` | Email address to send notifications of new user signups. |
| email.smtpHost | string | `nil` | Host for SMTP strategy |
| email.smtpUsername | string | `nil` | Username for SMTP strategy |
| email.smtpPort | string | `nil` | Port for SMTP strategy |
| email.awsRegion | string | `nil` | Region for SES strategy, AWS_ACCESS_KEY_ID, and AWS_SECRET_ACCESS_KEY will need to be provided in the fwinsights-secrets secret. |
| reportStorage.strategy | string | `"minio"` | How to store report files, valid values include minio, s3, and local |
| reportStorage.bucket | string | `"reports"` | Name of the bucket to use for minio or s3 |
| reportStorage.region | string | `"us-east-1"` | AWS region to use for S3 |
| reportStorage.minioHost | string | `nil` | Hostname to use for Minio |
| reportStorage.fixturesDir | string | `nil` | Directory to store files in for local. |
| minio.install | bool | `true` | Install Minio |
| minio.buckets | list | `[{"name":"reports","policy":"none"}]` | Create the following buckets for the newly installed Minio |
| minio.resources | object | `{"requests":{"cpu":"50m","memory":"256Mi"}}` | Resources for Minio |
| minio.nameOverride | string | `"fw-minio"` | nameOverride to shorten names of Minio resources |
| minio.persistence.enabled | bool | `true` | Create a persistent volume for Minio |