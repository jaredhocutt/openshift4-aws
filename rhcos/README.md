# RHCOS AWS Image Import

The following are my notes on attempting to get an RHCOS image imported into
AWS manually. The reason for doing an image import manually is to support
running OpenShift 4 in alternative/private AWS regions, such as GovCloud, C2S,
and others.

While part of this document will show you how to ultimately get an image
imported, it is by no means the expected way to do this. The purpose of these
notes are to document what was tried and what the outcomes were.

## Initial Setup

In order to use the AWS image import functionality (e.g. `aws ec2
import-image`), you must create a service role to use. The full details can be
found at the link below, but here is the quick guide.

This is a one-time setup, so if you've imported images before and already have
a `vmimport` role, you can likely skip this setup.

https://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html#import-vm-image

Create an S3 bucket with your name of choice.

For example purposes, it is assumed the bucket is named
`io-rdht-govcloud-vmimport`.

Create a file named `trust-policy.json`:

```json
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": {
           "Service": "vmie.amazonaws.com"
         },
         "Action": "sts:AssumeRole",
         "Condition": {
            "StringEquals":{
               "sts:Externalid": "vmimport"
            }
         }
      }
   ]
}
```

Create the `vmimport` role:

```bash
aws iam create-role \
   --role-name vmimport \
   --assume-role-policy-document file://trust-policy.json
```

Create a file named `role-policy.json`:

```json
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetBucketLocation",
            "s3:GetObject",
            "s3:ListBucket"
         ],
         "Resource":"*"
      },
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetBucketLocation",
            "s3:GetObject",
            "s3:ListBucket",
            "s3:PutObject",
            "s3:GetBucketAcl"
         ],
         "Resource":"*"
      },
      {
         "Effect":"Allow",
         "Action":[
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*"
         ],
         "Resource":"*"
      }
   ]
}
```

Attach the policy to the `vmimport` role:

```bash
aws iam put-role-policy \
   --role-name vmimport \
   --policy-name vmimport \
   --policy-document file://role-policy.json
```

## Import RHCOS Image

This section describes the work that was done to attempt importing a RHCOS
image into AWS. It is broken down by, what we need to have, what we were able
to make work, and the thing we tried when attempting to get it to work.

### What We Need

**TL;DR** We need a RHCOS image that can be uploaded to an S3 bucket and be
usable by the `aws ec2 import-image` command.

In OpenShift 4.2, there is no specific RHCOS image for AWS, which makes it
difficult to install OpenShift 4.2 in alternative AWS regions where a RHCOS
image isn't in the marketplace.

In OpenShift 4.3 pre-release content, there is now a RHCOS image labeled for
use in AWS. However, this image does not work as is because it contains an EFI
partition and when you attempt to import it using `aws ec2 import-image`, it
fails referencing an error that UEFI booting is not supported in AWS.

I'm not sure if the fix to this problem is simply removing the EFI partition on
the RHCOS AWS VMDK image or if it requires more work than that. In any case, we
need a RHCOS image for AWS that can be uploaded to an S3 bucket as-is and be
imported using the `aws ec2 import-image` command.

### How We Got It To Work

#### OpenShift 4.2

Download the Bare Metal BIOS raw disk image, unarchive it, and upload it to S3:

```bash
wget http://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.2/latest/rhcos-4.2.0-x86_64-metal-bios.raw.gz

gunzip rhcos-4.2.0-x86_64-metal-bios.raw.gz

aws s3 cp rhcos-4.2.0-x86_64-metal-bios.raw s3://io-rdht-govcloud-vmimport
```

Create the disk containers file `containers-42-metal-bios-raw.json`:

```json
{
   "Description": "RHCOS 4.2 Metal BIOS RAW",
   "Format": "raw",
   "UserBucket": {
      "S3Bucket": "io-rdht-govcloud-vmimport",
      "S3Key": "rhcos-4.2.0-x86_64-metal-bios.raw"
   }
}
```

Import the disk as a snapshot into AWS:

```bash
aws ec2 import-snapshot \
   --region us-gov-west-1 \
   --description "RHCOS 4.2 Metal BIOS RAW" \
   --disk-container file://containers-42-metal-bios-raw.json
```

Check the status of the image import:

```bash
aws ec2 describe-import-snapshot-tasks \
   --region us-gov-west-1
```

After the import is complete, you should see similar output:

```json
{
    "ImportSnapshotTasks": [
        {
            "Description": "RHCOS 4.2 Metal BIOS RAW",
            "ImportTaskId": "import-snap-fgtxpsup",
            "SnapshotTaskDetail": {
                "Description": "RHCOS 4.2 Metal BIOS RAW",
                "DiskImageSize": 3182428160.0,
                "Format": "RAW",
                "SnapshotId": "snap-0f901db0c624c671f",
                "Status": "completed",
                "UserBucket": {
                    "S3Bucket": "io-rdht-govcloud-vmimport",
                    "S3Key": "rhcos-4.2.0-x86_64-metal-bios.raw"
                }
            }
        }
    ]
}
```

Create a volume from the snapshot:

```bash
aws ec2 create-volume \
   --region us-gov-west-1 \
   --availability-zone us-gov-west-1a \
   --volume-type gp2 \
   --snapshot-id snap-0f901db0c624c671f
```

```json
{
    "AvailabilityZone": "us-gov-west-1a",
    "CreateTime": "2020-01-22T16:05:39.000Z",
    "Encrypted": false,
    "Size": 3,
    "SnapshotId": "snap-0f901db0c624c671f",
    "State": "creating",
    "VolumeId": "vol-052d72515b93e3948",
    "Iops": 100,
    "Tags": [],
    "VolumeType": "gp2"
}
```

Check the status of the volume creation:

```bash
aws ec2 describe-volume-status \
   --region us-gov-west-1 \
   --volume-ids vol-052d72515b93e3948
```

After the volume creation is complete, you should see similar output:

```json
{
    "VolumeStatuses": [
        {
            "Actions": [],
            "AvailabilityZone": "us-gov-west-1a",
            "Events": [],
            "VolumeId": "vol-052d72515b93e3948",
            "VolumeStatus": {
                "Details": [
                    {
                        "Name": "io-enabled",
                        "Status": "passed"
                    },
                    {
                        "Name": "io-performance",
                        "Status": "not-applicable"
                    }
                ],
                "Status": "ok"
            }
        }
    ]
}
```

Create an EC2 instance you can use to modify the volume you created:

```bash
aws ec2 run-instances \
   --region us-gov-west-1 \
   --instance-type t3.medium \
   --image-id ami-5a740e3b \
   --key-name default \
   --security-group-ids sg-da4b7abc \
   --subnet-id subnet-79952d1d \
   --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=rhcos-modifications}]' \
   --associate-public-ip-address
```

```json
{
    "Groups": [],
    "Instances": [
        {
            "AmiLaunchIndex": 0,
            "ImageId": "ami-5a740e3b",
            "InstanceId": "i-0ab65eb0272069e1d",
            "InstanceType": "t3.medium",
            "KeyName": "default",
            "LaunchTime": "2020-01-22T16:50:14.000Z",
            "Monitoring": {
                "State": "disabled"
            },
            "Placement": {
                "AvailabilityZone": "us-gov-west-1a",
                "GroupName": "",
                "Tenancy": "default"
            },
            "PrivateDnsName": "ip-172-31-20-233.us-gov-west-1.compute.internal",
            "PrivateIpAddress": "172.31.20.233",
            "ProductCodes": [],
            "PublicDnsName": "",
            "State": {
                "Code": 0,
                "Name": "pending"
            },
            "StateTransitionReason": "",
            "SubnetId": "subnet-79952d1d",
            "VpcId": "vpc-d33cd0b7",
            "Architecture": "x86_64",
            "BlockDeviceMappings": [],
            "ClientToken": "",
            "EbsOptimized": false,
            "Hypervisor": "xen",
            "NetworkInterfaces": [
                {
                    "Attachment": {
                        "AttachTime": "2020-01-22T16:50:14.000Z",
                        "AttachmentId": "eni-attach-c4d96226",
                        "DeleteOnTermination": true,
                        "DeviceIndex": 0,
                        "Status": "attaching"
                    },
                    "Description": "",
                    "Groups": [
                        {
                            "GroupName": "ssh",
                            "GroupId": "sg-da4b7abc"
                        }
                    ],
                    "Ipv6Addresses": [],
                    "MacAddress": "02:a7:07:0b:eb:2e",
                    "NetworkInterfaceId": "eni-34514968",
                    "OwnerId": "676111148133",
                    "PrivateDnsName": "ip-172-31-20-233.us-gov-west-1.compute.internal",
                    "PrivateIpAddress": "172.31.20.233",
                    "PrivateIpAddresses": [
                        {
                            "Primary": true,
                            "PrivateDnsName": "ip-172-31-20-233.us-gov-west-1.compute.internal",
                            "PrivateIpAddress": "172.31.20.233"
                        }
                    ],
                    "SourceDestCheck": true,
                    "Status": "in-use",
                    "SubnetId": "subnet-79952d1d",
                    "VpcId": "vpc-d33cd0b7"
                }
            ],
            "RootDeviceName": "/dev/sda1",
            "RootDeviceType": "ebs",
            "SecurityGroups": [
                {
                    "GroupName": "ssh",
                    "GroupId": "sg-da4b7abc"
                }
            ],
            "SourceDestCheck": true,
            "StateReason": {
                "Code": "pending",
                "Message": "pending"
            },
            "Tags": [
                {
                    "Key": "Name",
                    "Value": "rhcos-modifications"
                }
            ],
            "VirtualizationType": "hvm",
            "CpuOptions": {
                "CoreCount": 1,
                "ThreadsPerCore": 2
            },
            "CapacityReservationSpecification": {
                "CapacityReservationPreference": "open"
            }
        }
    ],
    "OwnerId": "676111148133",
    "ReservationId": "r-05578e08da0263e81"
}
```

Attach the volume to the EC2 instance:

```bash
aws ec2 attach-volume \
   --region us-gov-west-1 \
   --device xvdf \
   --instance-id i-0ab65eb0272069e1d \
   --volume-id vol-052d72515b93e3948
```

When the instance is done booting, SSH into the instance for the next steps.

Find the device for your volume:

```bash
lsblk
```

```text
NAME        MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
nvme0n1     259:0    0  10G  0 disk
├─nvme0n1p1 259:1    0   1M  0 part
└─nvme0n1p2 259:2    0  10G  0 part /
nvme1n1     259:3    0   3G  0 disk
├─nvme1n1p1 259:4    0   1M  0 part
├─nvme1n1p2 259:5    0   1G  0 part
└─nvme1n1p3 259:6    0   2G  0 part
```

Mount the 2nd partition so we can make modifications:

```bash
sudo mount /dev/nvme1n1p2 /mnt
```

Modify `/mnt/grub2/grub.cfg` to change the kernel parameters
`coreos.oem.id=metal ignition.platform.id=metal` to be `coreos.oem.id=qemu
coreos.oem.id=ec2 ignition.platform.id=ec2` instead:

```bash
sudo sed -i 's/coreos.oem.id=metal ignition.platform.id=metal/coreos.oem.id=qemu coreos.oem.id=ec2 ignition.platform.id=ec2/' /mnt/grub2/grub.cfg
```

Unmount the partition:

```bash
sudo umount /mnt
```

You can now disconnect from the SSH session.

Detach the volume from your EC2 instance and then terminate the instance:

```bash
aws ec2 detach-volume \
   --region us-gov-west-1 \
   --volume-id vol-052d72515b93e3948

aws ec2 terminate-instances \
   --region us-gov-west-1 \
   --instance-ids i-0ab65eb0272069e1d
```

Create a new snapshot from the modified volume:

```bash
aws ec2 create-snapshot \
   --region us-gov-west-1 \
   --description "RHCOS 4.2 Metal BIOS RAW Modified" \
   --volume-id vol-052d72515b93e3948
```

```json
{
    "Description": "RHCOS 4.2 Metal BIOS RAW Modified",
    "Encrypted": false,
    "OwnerId": "676111148133",
    "Progress": "",
    "SnapshotId": "snap-0bc37f2db049f4864",
    "StartTime": "2020-01-22T17:59:48.000Z",
    "State": "pending",
    "VolumeId": "vol-052d72515b93e3948",
    "VolumeSize": 3,
    "Tags": []
}
```

Check the status of the snapshot creation:

```bash
aws ec2 describe-snapshots \
   --region us-gov-west-1 \
   --snapshot-ids snap-0bc37f2db049f4864
```

After the snapshot creation is complete, you should see similar output:

```json
{
    "Snapshots": [
        {
            "Description": "RHCOS 4.2 Metal BIOS RAW Modified",
            "Encrypted": false,
            "OwnerId": "676111148133",
            "Progress": "100%",
            "SnapshotId": "snap-0bc37f2db049f4864",
            "StartTime": "2020-01-22T17:59:48.000Z",
            "State": "completed",
            "VolumeId": "vol-052d72515b93e3948",
            "VolumeSize": 3
        }
    ]
}
```

Register a new image using the new snapshot:

```bash
aws ec2 register-image \
   --region us-gov-west-1 \
   --architecture x86_64 \
   --description "RHCOS 4.2 Metal BIOS RAW Modified" \
   --ena-support \
   --name "RHCOS 4.2 Metal BIOS RAW Modified" \
   --virtualization-type hvm \
   --root-device-name '/dev/sda1' \
   --block-device-mappings 'DeviceName=/dev/sda1,Ebs={DeleteOnTermination=true,SnapshotId=snap-0bc37f2db049f4864}'
```

You can now use this AMI for your RHCOS nodes when you deploy OpenShift 4.

#### OpenShift 4.3

Download the AWS VMDK disk image, unarchive it, and upload it to S3:

```bash
wget http://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.3/latest/rhcos-43.81.201912030353.0-aws.x86_64.vmdk.gz

gunzip rhcos-43.81.201912030353.0-aws.x86_64.vmdk.gz

aws s3 cp rhcos-43.81.201912030353.0-aws.x86_64.vmdk s3://io-rdht-govcloud-vmimport
```

Create the disk containers file `containers-43-aws-vmdk.json`:

```json
{
   "Description": "RHCOS 4.3 AWS VMDK",
   "Format": "vmdk",
   "UserBucket": {
      "S3Bucket": "io-rdht-govcloud-vmimport",
      "S3Key": "rhcos-43.81.201912030353.0-aws.x86_64.vmdk"
   }
}
```

Import the disk as a snapshot into AWS:

```bash
aws ec2 import-snapshot \
   --region us-gov-west-1 \
   --description "RHCOS 4.3 AWS VMDK" \
   --disk-container file://containers-43-aws-vmdk.json
```

Check the status of the image import:

```bash
aws ec2 describe-import-snapshot-tasks \
   --region us-gov-west-1
```

After the import is complete, you should see similar output:

```json
{
    "ImportSnapshotTasks": [
        {
            "Description": "RHCOS 4.3 AWS VMDK",
            "ImportTaskId": "import-snap-fh6i8uil",
            "SnapshotTaskDetail": {
                "Description": "RHCOS 4.3 AWS VMDK",
                "DiskImageSize": 819056640.0,
                "Format": "VMDK",
                "SnapshotId": "snap-06331325870076318",
                "Status": "completed",
                "UserBucket": {
                    "S3Bucket": "io-rdht-govcloud-vmimport",
                    "S3Key": "rhcos-43.81.201912030353.0-aws.x86_64.vmdk"
                }
            }
        }
    ]
}
```

Register an image using the snapshot:

```bash
aws ec2 register-image \
   --region us-gov-west-1 \
   --architecture x86_64 \
   --description "RHCOS 4.3 AWS VMDK" \
   --ena-support \
   --name "RHCOS 4.3 AWS VMDK" \
   --virtualization-type hvm \
   --root-device-name '/dev/sda1' \
   --block-device-mappings 'DeviceName=/dev/sda1,Ebs={DeleteOnTermination=true,SnapshotId=snap-06331325870076318}'
```

You can now use this AMI for your RHCOS nodes when you deploy OpenShift 4.

### Failed Attempts

#### OpenShift 4.3 AWS VMDK

Download the AWS VMDK image, unarchive it, and upload it to S3:

```bash
wget http://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.3/latest/rhcos-43.81.201912030353.0-aws.x86_64.vmdk.gz

gunzip rhcos-43.81.201912030353.0-aws.x86_64.vmdk.gz

aws s3 cp rhcos-43.81.201912030353.0-aws.x86_64.vmdk s3://io-rdht-govcloud-vmimport
```

Create the disk containers file `containers-43-aws-vmdk.json`:

```json
[
  {
    "Description": "RHCOS 4.3 AWS VMDK",
    "Format": "vmdk",
    "UserBucket": {
        "S3Bucket": "io-rdht-govcloud-vmimport",
        "S3Key": "rhcos-43.81.201912030353.0-aws.x86_64.vmdk"
    }
  }
]
```

Import the image into AWS:

```bash
aws ec2 import-image \
   --region us-gov-west-1 \
   --architecture x86_64 \
   --description "RHCOS 4.3 AWS VMDK" \
   --license-type BYOL \
   --platform Linux \
   --disk-containers file://containers-43-aws-vmdk.json
```

Check the status of the image import:

```bash
aws ec2 describe-import-image-tasks --region us-gov-west-1
```

After the import is complete, here's the error that we get:

```json
{
    "ImportImageTasks": [
        {
            "Architecture": "x86_64",
            "Description": "RHCOS 4.3 AWS VMDK",
            "ImportTaskId": "import-ami-ffo2siec",
            "LicenseType": "BYOL",
            "Platform": "Linux",
            "SnapshotDetails": [
                {
                    "Description": "RHCOS 4.3 AWS VMDK",
                    "DiskImageSize": 819056640.0,
                    "Format": "VMDK",
                    "Status": "completed",
                    "UserBucket": {
                        "S3Bucket": "io-rdht-govcloud-vmimport",
                        "S3Key": "rhcos-43.81.201912030353.0-aws.x86_64.vmdk"
                    }
                }
            ],
            "Status": "deleted",
            "StatusMessage": "ClientError: EFI partition detected. UEFI booting is not supported in EC2."
        }
    ]
}
```

### OpenShift 4.3 Metal RAW

### OpenShift 4.2 VMware OVA

### OpenShift 4.2 Metal BIOS RAW
