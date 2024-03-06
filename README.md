# Cloudformation Script for creating PostgreSQL Clusters

This project is aimed at turnkey deployment of PostgreSQL clusters on the AWS cloud. It uses [AWS Cloudformation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacks.html) to create "Stack" by defining the Postgres Primary, Replica and related resources. Things like instance type, AV zone, DB password and keypair are parameterized so they can be set at run time. This implies you can parameterize much more than I have depending your needs. This is intended to be a starting point.

## Motivation

The motivation behind this project is to create containerless Postgres cluster with the click of a button on varrying instance types. This implies that tuning needs to take place for each instance type to get the right values for the various params that affect performance in **postgresql.conf**. You can use tuning guides like this https://postgresqlco.nf/ or simpler one like https://pgtune.leopard.in.ua

#### More on motivation:

[HERE](https://medium.com/@mkremer_75412/why-postgres-rds-didnt-work-for-us-and-why-it-won-t-work-for-you-if-you-re-implementing-a-big-6c4fff5a8644) and HERE

## EC2 Instances and Tuning postgresql.conf

This project focuses on two instance types, the r5a and i3en this does not preclude you from using other instance types and sizes. The r5a is a general purpose memory optimized instance good for ingestion and batch processing workloads. The i3en which will serve as our read replica (via PostGreSQL streaming replication) is good at fast IO and performs OLAP workloads very quickly on large datasets. [Read more about how setting instance specific postgresql.conf parameters are accomplished in Cloudformation](cf-pg-configs/README.md).

> **Note:** When launching instances make sure to pair the instance sizes accordingly. `large` primary means `large` replica, `xlarge` with `xlarge`, `2xl` with `2xl`, etc.

#### The Memory Optimized r5a instance types (Read/Write Primary)
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-store-volumes.html#instance-store-vol-mo

#### The Storage Optimized i3en instance types (Read Only Replica)
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-store-volumes.html#instance-store-vol-gp


## Instrumental debug data for stack development

Use this command to check (grep) for errors during stack creation by sshing into the EC2 instance.

https://repost.aws/knowledge-center/cloudformation-failed-signal

```sh
grep -ni 'error\|failure' $(sudo find /var/log -name cfn\* -or -name cloud-init\*)
```

### Huge Pages and Mappings

Both primary and replica Postgres servers rely on Huge Pages performance optimization. The config files in S3 for the contain `huge_pages = on`.
There may be a way to compute nPages dynamically by querying the instance type's total available memory - for now its statically coded in the Mappings section of the Cloudformation template like this:

```json
"Mappings": {
      "HugePages": {
        "r5a.2xlarge": {
          "nPages": "16000"
        },
        "r5a.xlarge": {
          "nPages": "8000"
        },
        "t3a.medium": {
          "nPages": "1500"
        },
        "i3en.2xlarge": {
          "nPages": "16000"
        },
        "i3en.xlarge": {
          "nPages": "8000"
        }
      }
    }
```  

### ZFS

The instances rely on ZFS which provides tremendous I/O performance improvements over conventional file systems by trading computation for compression and hence faster fetches. The [base_image.md](base_image.md) contains installation steps for ZFS  - this should be part of your base AMI.

https://openzfs.github.io/openzfs-docs/Getting%20Started/RHEL-based%20distro/index.html

### VPC securtiy groups

The template demonstrates referencing existing vpc groupId IDs and also creating them on the fly as resources as using them when associating security on the EC2 resources. Replace my security group ID like `sg-0055ac66` with yours.

>Beware: There are weird limitation around creating ingress and egress rules as part of the security group in one JSON model. This is why SecurityGroupIngress and SecurityGroup resources are created separately in the template


### WAL-G and replication

WAL archiving is performed by WAL-G via PostgreSQL’s `archive_command`, and streaming replication is supported by PostgreSQL’s protocol. These functions are distinct, but closely related. WAL-G handles backups and point-in-time-restore (PITR via WAL log shipping), while streaming replication supports the hot standby server. However, if the standby falls behind the primary and misses a WAL segment it can also retrieve it from S3 using WAL-G. WAl-G should be installed on the base AMI

### Cloudwatch

Cloudwatch alarms are handy for monitoring ZFS pool usage AKA disk space consumption. The script sets alarms for 80% thresholds but you may change as you see fit for parameterize.


### DNS

I create route53 DNS records for an existing local hosted zone. This is handy for setting server names for the cluster for addressing the servers upstream. You end up with DNS names like this `[stack-name]-rw.myapp.local`. Replace `myapp` with your namespace.  