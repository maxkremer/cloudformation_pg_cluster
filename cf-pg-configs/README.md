
# EC2 Instance type postgresql.conf

This project focuses on 3 instance types, the r5a, t3a and i3en this does not preclude you from using other instance types and sizes. The r5a is a general purpose memory optimized instance good for ingestion and batch processing workloads. The i3en which will serve as our read replica (via PostGreSQL streaming replication) is good at fast IO and performs OLAP workloads very quickly on large datasets. The Cloudformation template references a **cf-pg-configs S3 bucket** that contains an inventory of [instance_type.size].conf files representing the Postgresql.conf parameter file for the associated instance type prefixed by its use in the cluster (rw primary & ro replica) like this:


```
cf-pg-configs.s3.us-west-2.amazonaws.com/rw/r5a.2xlarge.conf
cf-pg-configs.s3.us-west-2.amazonaws.com/rw/r5a.xlarge.conf
cf-pg-configs.s3.us-west-2.amazonaws.com/rw/t3a.medium.conf
cf-pg-configs.s3.us-west-2.amazonaws.com/ro/i3en.2xlarge.conf
cf-pg-configs.s3.us-west-2.amazonaws.com/ro/i3en.xlarge.conf
```

The above naming convention is relied upon in the Cloudformation template.  