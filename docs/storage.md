# Storage Fundamentals for AWS
AWS provides many services for storing data. Data storage can be divided into categories such as:

- Block Storage
    - Data is stored in chunks known as blocks
    - Blocks are stored on a volume and attached to a single instance
    - Provide very low latency
- File Storage
    - Data is stored as seperate files within a series of directories
    - Data is stored on top of a file system
    - Shared access is provided for multiple users
- Object Storage
    - Objects are stored across a flat address space
    - Objects are referenced by a unique key
    - Each object can also have associated metadata to help categorize and identify the object

## Overview of Amazon S3
Amazon S3 (Simple Storage Service) is a fully managed Object-based storage service by Amazon, that is - 

- Highly Available
- Durable
- Cost Effective
- Widely Accessible

It is a object based storage service so each file uploaded does not belong to hierarchial file system but rather file is stored in a flat path and which is referenced by a unique URL. S3 is also a Regional service. 

To store object in S3, you first need to define and create a bucket. This bucket name must be completetly unique. By default your account can have upto a 100 buckets but this is a soft limit and a request can be made to increase this.

Any object uploaded to the bucket are given a unique object key for identification. 

## EC2 Instance Storage
This is also referred to Instance Store Volume. The volumes reside on the same host that provides the EC2 instances itself. The instance storage provides Ephemeral storage for your instance volumes. So it is recommended not to store critical data on these ephemeral volumes. 

These are the cirumstances where data stored in the EC2 Instance Storage will be lost - 

- Stop
- Termniate

However, if the instance was simply rebooted, the data would remain intact

#### Benefits of EC2 Instance Storage

- No additional cost for storage
- Very high I/O speed
- Optimized instance families
- Instance store volumes are ideal as a cache or buffer for rapidly changing data without the need for retention
- Often used within a load balancing group, where data is replicated and pooled between the fleet

## Amazon Elastic Block Store (EBS)
EBS provides storage to EC2 instances with EBS Volume

- Provides persistent and durable block level storage
- EBS volumes offer far more flexiblity with regards to managing the data

EBS volumes are attached to your EC2 instance and are primiarily used for fast changing data that requires a certain amount of Input/Output operations per second (IOPS).

EBS also provides the ability to provide point-in-time backup of the storage, called Snapshots. You can manually create a Snapshot of of your storage or automate it using Amazon CloudWatch. The Snapshots themselves are stored in S3 so that they are very durable and reliable.

Snapshots are also incremental meaning they will only copy data that has changed since the previous snapshot was taken.

Once you have the snapshot of an EBS volume, you can create a new volume from that snapshot. It is also possible to have Snaposhots in different regions. 

Two types of EBS volumes available -

- SSD Backed Storage
    - Suited for work with smaller blocks
    - As boot volumes for EC
- HDD Backed Storage
    - Suited for workloads that require a higher rate of throughput
    - E.g.: big data and logging information

#### EBS Security
If you have sensitive information in your storage, then you can just check the box for 'enable EBS security' and EBS will ensure security using Amazon's AES-256 encryption method and AWS KMS (Key Management System). Any snapshot taken from a encrypted volume would also be encrypted. But it is to be noted that this encryption is only available on selected instance types.

#### Ways to Create EBS Volumes:

- During the creation of a new instance. You can attach it at the time of launch.
- From within the EC2 dashboard of the AWS management console. Create it as a standalone volume, ready to be attached to an instance when required.

## EBS Multi-Attach
Normally one EBS can only be attached to only one EC2. But with Multi-Attach one EBS can be attached with multiple EC2. But this is heavily dependent on the type of volume and the type of instance. For the time being Multi-Attach is only available in - 

- Provisioned IOPS SSD (io1)
- Provisioned IOPS SSD (i02)

#### File Systems
It is recommended to use a **Clustered File System** such as GFS2. This will safely manag multi-instance access to the shared volume.

#### Nitro Systems
EC2 instances based on the **Nitro Systems** can run EC2 on maximum performance. Nitro instances is a requirement for using Multi-Attach. 

**Nitro:** The underlying virtualization platform of the EC2 instance. 

**Bare Metal Insances:** The nitro also introduces the concept of Bare Metal Instances. That means you can run EC2 instances without any hypervisor or the customer can use their own hypervisor.

## Amazon Elastic File System (EFS)
It is like your typical directory based file system. This also supports low latency file storage. But unlike EBS, it can provide support to multiple file systems. It also has features of traditional file systems such as - locking files, updating files and renaming files. It also has a hierarchial file structure. This type of storage allows you to store files that are accessible to network resources.

## Hybrid Storage Solution using AWS Storage Gateway
AWS Storage Gateway for hybrid storage between on-premises and cloud infrastructure. 

#### Deployment Options

- On-Premises on a Virtual Machine or AWS Storage Gateway Hardware Appliance
- Deploy it on a AWS EC2 instance using AWS Storage Gateway Management Service

When deploying the gateway, it will ask for at least 150MB which will work as the local cache. This will act as - 

- A staging area for data thatthe gateway will upload to AWS
- A true cache to save data for low-latency access

There are four types of storage gateway - 

- S3 File Gateway
- FSx File Gateway
- Tape Gateway
- Volume Gateway

## AWS Elastic Disaster Recovery System (DRS)
AWS Elastic DRS is AWS' service for recovering failed applications. It is more efficient and cost-effective than on premises data recovery systems. AWS DRS supports **Recovery Point Objectives (RPOs)** of seconds and **Recovery Time Objectives (RTOs)** of minutes by performing ongoing replication of your source servers' disks to EBS volumes in the AWS cloud.