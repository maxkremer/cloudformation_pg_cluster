
## Recipe for creating a PostgreSQL Base AMI using PostgreSQL 14 and ZFS

Start with the Amazon Linux 2 AMI (HVM) - Kernel 5.10:

**ami-0895022f3dac85884 (64-bit (x86))** 


### Install zfs

ZFS is huge performance booster for improving I/O for Postgres. By trading a bit of CPU time for compression we are able to read/write data off disk faster.

https://openzfs.github.io/openzfs-docs/Getting%20Started/RHEL-based%20distro/index.html



#### Install ZFS and it's dependencies
```
sudo amazon-linux-extras install epel
sudo yum install -y kernel-devel
sudo yum install -y epel-release
sudo yum install https://zfsonlinux.org/epel/zfs-release-2-2.el7.noarch.rpm
sudo sed -i 's/\$releasever/7/g' /etc/yum.repos.d/zfs.repo**
reboot
sudo yum install -y zfs
sudo /sbin/modprobe zfs
```
Note the funny business above with sed replacing $releasever with 7 to is get unix compatible version for AWS Linux2


### Install PostgreSQL using amazon-linux-extras

```
sudo amazon-linux-extras install postgresql14  
sudo yum install -y postgresql-server postgresql-contrib postgresql-server-devel.x86_64  
sudo yum update
```

### Install a few handy Perl utils we'll need

```
sudo yum install -y perl-Switch \
  perl-DateTime \
  perl-Sys-Syslog \
  perl-LWP-Protocol-https \
  perl-Digest-SHA.x86_64
```

### Get and unzip CloudWatch Monitoring scripts
```
curl https://aws-cloudwatch.s3.amazonaws.com/downloads/CloudWatchMonitoringScripts-1.2.2.zip -O
unzip CloudWatchMonitoringScripts-1.2.2.zip && \
  rm CloudWatchMonitoringScripts-1.2.2.zip && \
  cd aws-scripts-mon
```


### Install WAL-G for (AWS Linux 2 compatible version)
```
wget https://github.com/wal-g/wal-g/releases/download/v2.0.1/wal-g-pg-ubuntu-18.04-amd64.tar.gz
tar -xvf wal-g-pg-ubuntu-18.04-amd64.tar.gz 
mv wal-g-pg-ubuntu-18.04-amd64 wal-g
sudo chown root:root wal-g
sudo mv wal-g /usr/local/bin/
```

### Install dependencies for WAL-G.
```
sudo yum install -y lzo
```
### Install nvme-cli for querying nvme drives
```
sudo yum install nvme-cli
```

Once you've done the above create your own AMI and use the ami-#### ID in your Cloudformation template