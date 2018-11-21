<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Monitoring and Reporting](#monitoring-and-reporting)
	- [CloudWatch](#cloudwatch)
	- [Elastic Load balancers (ALB/NetworkLB/Classic)](#elastic-load-balancers-albnetworklbclassic)
	- [CloudTrail](#cloudtrail)
	- [Elasticache](#elasticache)
		- [CPU Utilization](#cpu-utilization)
		- [Swap Usage](#swap-usage)
		- [Evictions](#evictions)
		- [Concurrent Connections](#concurrent-connections)
	- [AWS Organizations](#aws-organizations)
	- [AWS Config Rules](#aws-config-rules)
- [Deployment and Provisioning](#deployment-and-provisioning)
	- [EC2 Launch issues](#ec2-launch-issues)
	- [Elastic Block Storage](#elastic-block-storage)
	- [Elastic Loadbalancer](#elastic-loadbalancer)
	- [Systems Manager](#systems-manager)
- [High Availability](#high-availability)
	- [Elasticity VS Scalability](#elasticity-vs-scalability)
	- [RDS Multi AZ](#rds-multi-az)
	- [RDS Read Replica](#rds-read-replica)

<!-- /TOC -->

# Monitoring and Reporting
## CloudWatch
Monitoring service to monitor your AWS resources, as well as the applications that you run on AWS. By default CloudWatch monitors in a 5 minute interval.

_**Standard Metrics**_ - Host level metrics of EC2 instances consist of:

	- CPU
	- Network
	- Disk
	- Status Checks (EC2 host/EC2 instance)

_**Custom Metrics**_ - minimum granularity is 1 minute. (Ram for example is a custom metric)

_**Terminated Instances**_ - You can retrieve data from any terminated EC2 or ELB instance after its termination. CloudWatch Logs are stored indefinitely.

_**Metric Granularity:**_

	- 1 minute for detailed Monitoring
	- 5 minutes for standard Monitoring

_**CloudWatch can be used on premise**_ - Not restricted to just AWS resources. Can be on premise with the SSM agent and CloudWatch Agent

_**Dashboards**_ - CloudWatch Dashboards are multi-region and can display any widget to any region. To add the widget, change to the region that you need and then add the widget to your dashboard.

## Elastic Load balancers (ALB/NetworkLB/Classic)
ELB monitoring types: 4 different ways to monitor your load balancers: CloudWatch Metrics, Access Logs, Request tracing and CloudTrail logs.

_**Access Logs:**_

	- Access logs can store data where the EC2 instance has been deleted. F.E. when running an ASG for a fleet of EC2 instances, and EC2 instances are deleted, you still have the access logs available if access log is enabled on the ELB

_**Request Tracing:**_

	- Available for ALB only

## CloudTrail
	- You can use CloudTrail to capture detailed information about the calls made to the ELB API and store them as logs in S3.

_**CloudWatch VS CloudTrail**_

	- CloudWatch monitors performance
	- CloudTrail monitors API calls in the AWS Platform

## Elasticache
Two types of engines: Memcached or Redis

### CPU Utilization

_**Memcached:**_

	- Multi-threaded
	- Can handle loads up to 90%, when >90%, add more nodes

_**Redis**_

	- Not multi-threaded, scalingpoint = 90 / number of cores. F.E. 4 cores, CPU Utilization = 90/4=22,5%

### Swap Usage
_**Memcached:**_

	- Should be 0, or not exceed 50Mb
	- If >50Mb increase memcached_connections_overhead parameter (defines the amount of mem to be reserved for memcached connections and other miscellaneous overhead)

_**Redis:**_

	- No SwapUsage metric, instead use reserved-memory

### Evictions
An Eviction occurs when a new item is added and an old item must be removed due to lack of free space in the system

_**Memcached:**_

	- No recommended setting, choose threshold based off application
	- Scale up (increase mem) or Scale out (more nodes)

_**Redis:**_

	- No recommended setting, choose threshold based off application
	- Only Scale out

### Concurrent Connections
_**Memcached & Redis:**_

	- No recommended setting, choose threshold based off application
	- If there is a large and sustained spike in the number of concurrent connections this can either mean a large traffic spike or your application is not releasing connections.
	- Remember to set an alarm on number of concurrent connections for Elasticache

## AWS Organizations
	- Centrally manage policies across multiple AWS accounts (Service Control Policies)
	- Control access to AWS Services
	- Automate AWS account creation and management
	- Consolidate billing across multiple AWS accounts

## AWS Config Rules
_**Compliance checks:**_

	- Trigger
		○ Periodic
		○ Configuration changes
	- Managed Rules
		○ About 40
		○ Basic, but fundamental

_**Permissions needed for Config:**_

	- AWS Config requires an IAM Role with:
		○ Read Only Permissions to the record resources
		○ Write access to a S3 logging bucket
		○ Publish access to SNS

_**Restrict Access:**_

	- Users need to be authenticated with AWS and have appropriate permissions set with IAM policies to gain access
	- Only Admins needing to set up and manage config require full access
	- Provide read-only permissions for Config day-to-day use

_**Monitoring Config:**_

	- Use CloudTrail with Config to provide deeper insight into resources
	- Use CloudTrail to monitor access to Config, such as someone stopping the Config recorder


# Deployment and Provisioning

## EC2 Launch issues
Remember the two common reasons for an instance failing to launch:

	- InstanceLimitExceeded error (You have exceeded the default limit of number of ec2 instances you can launch in the region)
	- InsufficientInstanceCapacity error (AWS does not currently have enough available On-Demand capacity to service your request)

## Elastic Block Storage

2 different variants of SSD:

	- GP2 - General Purpose 2 - boot volumes
	- IO1 - Provisioned IOPS - I/O intensive, NoSQL/Relational Databases, latency sensitive workloads.

IOPS (Input/Output Operations per Second) used to benchmark performance for SSD volumes.
IOPS Capability is dependent on the size of your volume.

	- GP2 volumes: (minimum 100 IOPS) 3 IOPS/GB up to maximum of 10.000 IOPS
	- IO1 volumes: 50 IOPS/GB to a maximum of 32.000 IOPS

If your workload is hitting the IOPS limit for your volume:

	- Increase the volume size - (Only works if your GP2 volume is < 3333GB
	- Change to Provisioned IOPS (IO1)

## Elastic Loadbalancer
3 types of loadbalancers:

	- Application loadbalancer (OSI lvl 7), checking http and https headers
	- Network loadbalancer (OSI lvl 4), for high performance, static ip
	- Classic loadbalancer, legacy

Pre-warm your ELB when expecting a sudden and significant increase in traffic to your application, therefor AWS needs to know start and end date, traffic expected (expected request rate per second) and packet size of typical request.

_**Loadbalancer Errors**_

	- 200 - successful response from the loadbalancer
	- 4XX - Message indicates client side error
		○ 400 - Bad or malformed request
		○ 401 - Unauthorized - user access denied
		○ 403 - Forbidden - request is blocked by WAF or ACL
		○ 404 - File not found
		○ 406 - Client closed connection before LB could respond, client timeout to short
		○ 463 - LB received an X-Forwarded-For (source ip address) request header with >30 ip addresses - similar to malformed request
	- 5XX - Message indicates server side error
		○ 500 - Internal server error - Error on the loadbalancer
		○ 502 - Bad gateway - Server closed connection or send back a malformed response
		○ 503 - Servers unavailable - no registered targets
		○ 504 - Gateway timeout - application is not responding
		○ 561 - Unauthorized - received error code from the ID Provider when trying to authenticate user

_**Loadbalancer Metrics => CloudWatch**_

	- Metrics for general health:
		○ HealthyHostCount
		○ UnHealthyHostCount
		○ HTTPCode_Backend_2XX
	- Metrics for performance:
		○ Latency
		○ RequestCount
		○ SurgeQueuelength (Classic LB only)
		○ SpillOverCount (Classic LB Only)

## Systems Manager
	- Systems Manager is used to give visibility and control over your AWS infrastructure
	- Integrates with CloudWatch dashboards
	- Allows you to organize your inventory and logically group resources
	
Run Command enables you to perform common operational tasks on groups of instances simultaneously without needing log in to each one.


# High Availability
## Elasticity VS Scalability
	- Elasticity - Scale with Demand (Short term)
		○ EC2 - Increase the number of instances, based on autoscaling
		○ DynamoDB - Increase additional IOPS for additional spikes in traffic, decrease after spike
		○ RDS - not very elastic, can't scale RDS on demand
		○ AURORA - Aurora serverless
	- Scalability - Scale out infrastructure (Long term)
		○ EC2 - Increase instance sizes as required, using reserved instances
		○ DynamoDB - Unlimited amount of storage
		○ RDS - Increase instance size eg from small to medium
		○ AURORA - Modify the instance type

## RDS Multi AZ
Multi AZ is for disaster recovery only, not for performance win.

	- High Availability
	- Backups are taken from secondary which avoids I/O suspension to the primary
	- Restore's are taken from secondary which avoids I/O suspension to the primary
	- Force failover by rebooting instance

## RDS Read Replica
Read only instance of your database. Supported versions are:

	- MySQL (5 read replica's)
	- PostgreSQL (5 read replica's)
	- MariaDB (5 read replica's)
	- Aurora

Read replica's can be stored in different regions.
Replication is asynchronous.
Read replica's can be built off Multi-AZ's databases.
Read replica's themselves can be Multi-AZ.
You can have read replica's of read replica's, but beware of latency.
DB Snapshots or Automated backups can't be taken from read replica's.
Key metric for read replica's is REPLICA LAG
