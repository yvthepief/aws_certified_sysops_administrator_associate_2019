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

# Storage and Data Management

## S3 Encryption
	- Encryption In-Transit
		○ SSL/TLS (HTTPS)
	- Encryption At Rest
		○ Server Side Encryption
			§ SSE-S3
			§ SSE-KMS
			§ SSE-C
		○ Client Side Encryption

	- If you want to enforce the use of encryption for your files stored in S3, use an S3 bucket policy to deny all PUT requests that don't include the x-amz-server-side-encryption parameter in the request header

## EC2 Types - EBS vs Instance Store
	- Delete on Termination is default for all EBS root device volumes. You can set it to false, however buy only at instance creation time
	- Additional volumes will persist automatically
	- Instance store is known as ephemeral storage, meaning it will always be lost when instance is deleted

## Volumes & Snapshots
	- Volumes exist on EBS:
		○ Virtual hard disk
	- Snapshots exist on S3
	- Snapshots are point in time copies of volumes
	- Snapshots are incremental -  this means that only the blocks that have changed since your last snapshot are moved to S3
	- To create a snapshot for Amazon EBS volumes that serve as root devices, you should stop the instance before taking a snapshot
	- However you can take a snapshot while the instance is running
	- You can create AMI's from both images and snapshots
	- You can change EBS volume sizes on the fly, including changing the size and storage type
	- Volumes will ALWAYS be in the same AZ as the EC2 instance
	- To move an EC2 volume from one AZ/Region to another, take a snapshot or an image of it, then copy it to a new AZ/Region

## Volumes vs Snapshots - Security
	- Snapshots of encrypted volumes are encrypted automatically
	- Volumes restored from encrypted snapshots are encrypted automatically
	- You can share snapshots, but only if they are unencrypted
		○ These snapshots can be shared with other AWS accounts or made public

## Encryption and Downtime
	- For most services, encryption needs to be activated on creation (EFS, RDS, EBS volumes)
	- S3 is more flexible, can be put on anytime

## KMS vs CloudHSM
	- Both KMS and CloudHSM enable you to generate, store and manage your own encryption keys to encrypt data stored in AWS
	- KMS is multi-tenancy and good for use cases which do not require dedicated hardware
	- If your application has a requirement for dedicated hardware for managing keys, use CloudHSM
	- KMS uses symmetric encryption (use same key for encrypting and decrypting), CloudHSM could use Asymmetric encryption (different keys for encrypting and decrypting)

# Security and Compliance
## AWS Shield
## Aws Shield Advance provides:
	- Always on, flow based monitoring of network traffic and active application monitoring to provide near real-time notifications of DDOS attacks.
	- DDOS response team (DRT) 24x7 to manage and mitigate application layer DDOS attacks
	- Protects you AWS bill against higher fees due to elastic loadbalancing (ELB), Amazon Cloud Front and Route53 usage spikes during DDOSS attack
	- $3000.- a month

## AWS Marketplace Security Products
	- Can purchase security products from third party vendors on the AWS market place
	- Firewalls, Hardened OS's, WAF's Antivirtus, Security Monitoring etc
	- Free, Hourly, Monthly, Annual, BYOL etc
	- CIS OS Hardening

## Roles & Custom Policies
	- You can create custom policies using the visual editor or using json
	- You can now attach roles to ec2 instances at any time using the command line or aws console
	- Once attached the role takes effect immediately
	- Any policy change also takes effect immediately

## MFA Reporting and IAM
	- You can enable MFA using the command line and by using the console
	- MFA can be enabled on both the root account and user accounts
	- You can enforce the use of MFA with the CLI by using the STS token service
	- You can report on who’s using MFA on a per user basis using credential reports

## AWS Hypervisors
	- Choose HVM over PV where possible
	- PV is isolated by layers, Guest OS sits on Layer 1, Applications Layer 3
	- Only AWS Administrators have access to hypervisors
	- AWS staff do not have access to EC2, that is the responsibility as a customer
	- All storage memory and RAM memory is scrubbed before it's delivered to you

## Dedicated instances vs dedicated hosts
	- Both dedicated instances and dedicated host have dedicated hardware
	- Dedicated instances are charged by the instance, dedicated hosts are charge by the host
	- If you have specific regulatory requirements or licensing conditions, choose dedicated hosts
	- Dedicated instances may share the same hardware with other AWS instances from the same account that are not dedicated
	- Dedicated hosts give you much better visibility in to things like sockets, cores and host id

## AWS System Manager Run Command
	- Commands can be applied to a group of systems based on AWS instance tags or by selecting manually
	- SSM agent needs to be installed om al your managed instances
	- The commands and parameters are defined in a system manager document
	- Commands can be issued using AWS console, AWS CLI, AWS Tools for Windows PowerShell, Systems Manager Api or Amazon SDKs
	- You can use this service with you on-premise systems as well as ec3 instance

## AWS System Manager Parameter store
	- Confidential information such as passwords, database connection strings, and license codes can be stored in SSM parameter store
	- You can store values as plain text or you can encrypt the data
	- You can reference these values by using their names
	- You can use this services with ec2, CloudFormation , lambda, ec2 run command etc

## AWS Config Rules with S3
	- Be aware of the following config rules
		○ No public read access
		○ No public write access

## AWS Inspector vs AWS trusted advisor
	- AWS Inspector: 
		○ Create "Assessment target"
		○ Install agents on EC2 instances
		○ Create "Assessment template"
		○ Perform "Assessment run"
		○ Review "Findings" against "Rules"
		○ Rules Packages:
			§ Common Vulnerabilities and Exposures
			§ CIS Operating System security Configuration Benchmarks
			§ Security Best Practices
			§ Runtime behavior Analysis
		○ Severity Levels for Rules in Amazon Inspector:
			§ High
			§ Medium
			§ Low
			§ Informational
		○ Monitor the network, files system and process activity within the specified target
		○ Compare wat it 'sees' to security rules
		○ Report on security issues observed within target during run
		○ Report findings and advise remediation
	- Trusted Advisor
		○ Cost optimization
		○ Availability
		○ Performance
		○ Security (F.E. Security groups) 

## Shared Responsibility Model
	- AWS responsible of the cloud
	- You responsible in the cloud

## AWS Artifact
Here you can download all AWS security and compliance documents

## CloudHSM vs KMS
	- Cloud HSM:
		○ Tendency: Single
		○ Scale & HA: HA service from AWS
		○ Key Control: you
		○ Integration: Broad AWS Support
		○ Symmetry: Symmetric and asymmetric keys
		○ Compliance: FIPS 140-2 & EAL-4
		○ Price: $$
	- KMS:
		○ Tendency: Multi
		○ Scale & HA: HA service from AWS
		○ Key Control: you + AWS
		○ Integration: Broad AWS Support
		○ Symmetry: Symmetric keys
		○ Compliance: good
		○ Price: $

## Encryption:
	- Instant Encryption: S3
	- Encryption with migration: DynamoDB, RDS, EFS and EBS
	
IAM:Passrole:
Temporary allow passing role to a service or other account

# Automation
## Cloudformation
Cloudformation allows you to manage, configure, and provision AWS infrastructure as code (YAML/JSON)
	- Template exist of:
	- Parameters: input custom values
	- Conditions: e.g. provision resources based on environment
	- Resources: Mandatory - the AWS resources to create
	- Mappings: create custom mappings like Region: AMI
	- Transforms: reference code located in S3 e.g. lambda code or reusable snippets of Cloudfromation code
