# Presto Bosh Release

## Description

This boshrelease installs and configures presto with the hive plugin for use with a backing s3.

## Usage

Minimum deployment requries the following:

1. A backing database. Examples for postgres are included.
1. S3-compatible object storage. Examples for both Minio and s3 are available.
1. At least 1 hive metastore instance. This should be deployed via the `hive` job.
1. Exactly 1 presto coordinator. You are unable to deploy multiple coordinators. This is a known possible point of failure.
1. At least 1 presto worker.

### Backing Database

Hive requires a backing database server to share metadata information between metastore instances. This can be provided via bosh-deployed postgres, or an IaaS-provided postgres. You can choose either option, and there is not really any gain going one way or another. The only information stored in this database is metadata.

*NOTE: While using other databases is possible, I only provided drivers for Postgres in this release. You would need to add any other drivers you'd want to use to the release separately.*

### S3-Compatible Object Storage

Both Hive and Presto require connection information to the S3-compatiible object storage. Deployment models as follows:

1. IaaS-Provided S3-compatible object storage.
1. Minio gateway to non-s3 compatible object storage (Azure Object storage, flat nas deployments, etc).
1. Minio deployment, either in-release or out-of-release.

Examples are available in the [manifest](manifest/) folder.

### Hive Standalone Metastore

In this release, I target the Hive Standalone Metastore. Each deployment can have multiple metastores, and should be scaled according to use case and performance requirements.
Requirements:

1. Hadoop, which is used by Hive in library form, but not daemon form. This is included in the release as part of the job.
1. openjdk-11 (included).

### Presto

Presto has 2 different deployment types, being a coordinator and worker. I will cover both, and what they do.
Hard requirements for installation are the following:

1. Hive Metastore (provided via the hive job)
1. S3-compatible object storage (multiple deployment models discussed above)
1. Java 11 (included)
1. Python 3.7 (included)

#### Coordinator

The deployment model for the coordinator instance is pretty straightforward. You can only deploy one coordinator instance. Using links, you can share information between the coordinator and worker instances. This is also the api connection location for jdbc calls.

#### Worker

This can be scaled to how ever many instances are necessary. Once the coordinator splits the work, it sends to the respective workers to run the queries.

## Config Variables

### Hive Job

| Name | Description | Default |
| - | - | - |
| `hive.db.type` | Hive metastore db type | `postgresql` |
| `hive.db.host` | Hive DB Host. Not used if postgres link available | *N/A* |
| `hive.db.port` | Hive DB Port. Not used if postgres link available | `5432` |
| `hive.db.name` | Hive DB Name. Not used if postgres link available | `metastore` |
| `hive.db.flags` | Hive Meastore DB Flags. Must add starting `?` otherwise it won't work. | `?allowPublicKeyRetrieval=true&amp;useSSL=false&amp;serverTimezone=UTC` |
| `hive.db.driver` | Hive DB Driver | `org.postgresql.Driver` |
| `hive.db.username` | Hive Username. Not used if postgres link available | *N/A* |
| `hive.db.password` | Hive DB Password. Not used if postgres link available | *N/A* |
| `hive.heapsize` | Java Heap for hive. set in megabytes. Leave 10% overhead for the OS | `1024` |
| `s3.access.key` | S3 Access key. Not used if minio link available | *N/A* |
| `s3.secret.key` | S3 Secret Access Key. Not used if minio link available | *N/A* |
| `s3.ssl_enabled` | Boolean if s3 is available. Not used if minio link available. | `false` |
| `s3.port` | Port. Not used if minio link available. | `9000` |
| `s3.endpoint` | Endpoint url. Not used if minio link available. | *N/A* |

### Presto Job

| Name | Description | Default |
| - | - | - |
| `presto.discovery.uri` | Presto Discovery uri. Not used if presto coordinator link is available. | `http://localhost:9000` |
| `presto.jvm.config` | JVM Configuration for Presto | [see jobs/presto/spec for more information](jobs/presto/spec) |
| `presto.coordinator` | Boolean. Is this instance a coordinator node? | `false` |
| `presto.node-schedule.include-coordinator` | Boolean. Does this scheduler include a coordinator? | `false` |
| `presto.http-server.http.port` | Presto http server port | `8080` |
| `presto.query.max-memory` | Max memory per query | `16GB` |
| `presto.query.max-memory-per-node` | Max memory per query per node | `1GB` |
| `presto.query.max-total-memory-per-node` | Max total memory for queries per node | `2GB` |
| `presto.discovery-server.enabled` | Boolean. Presto discovery server enabled? | `true` |
| `hive.metastore.uri` | Hive Metastore URI. Not used if hive link is available. [see jobs/presto/spec for more information](jobs/presto/spec) | *N/A* |
| `s3.access_key` | S3 Access key. Not used if minio link is available. | *N/A* |
| `s3.secret_key` | S3 Secret Access Key. Not used if minio link is available. | *N/A* |
| `s3.ssl_enabled` | Is ssl enabled on s3? Not used if minio link available | `false` |
| `s3.port` | S3 Port. Not used if minio link available. | `9000` |
| `s3.endpoint` | S3 Endpoint host and port. Overrides minio link if specified. | *N/A* |

## Deployment Models

### Test Deployment Manifest - Internal Minio, Postgres

This is an omnibus deployment with internal Minio, Postgres, Hive and Presto.
No flags are required for this.

Example manifest is available in [manifest/presto-development.yml](manifest/presto-development.yml).

```bash
bosh deploy -d presto manifest/presto-development.yml
```

### Production Deployment Manifest - External S3, Internal Postgres

This example is what is used in production on AWS. [manifest/presto-external-s3.yml](manifest/presto-external-s3.yml)

Required Values:

1. `metastore_password` - DB Password
1. `aws_access_key_id` - S3 Access Key
1. `aws_secret_access_key` - S3 Secret Key
1. `s3_endpoint` - s3 Endpoint. Example: `s3.amazonaws.com:443`

```bash
bosh deploy -d presto manifest/presto-external-s3.yml -v metastore_password=$PASSWORD -v aws_secret_access_key=$SECRET_KEY -v aws_access_key_id=$ACCESS_KEY -v s3_endpoint=s3.amazonaws.com:443 --no-redact
```

## CI/CD

TODO
