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
aws iam create-role --role-name vmimport --assume-role-policy-document file://trust-policy.json
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
aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document file://role-policy.json
```

## Import RHCOS Image

The sections below describe the steps that were taken to attempt importing a
RHCOS image into AWS. Each section will describe the steps that were taken, any
modifications of the provided RHCOS images that took place, and what the
outcomes were.

### OpenShift 4.3 AWS VMDK

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
aws ec2 import-image --architecture x86_64 --description "RHCOS 4.3 AWS VMDK" --license-type BYOL --platform Linux --region us-gov-west-1 --disk-containers file://containers-43-aws-vmdk.json
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

### OpenShift 4.3 VMware OVA

### OpenShift 4.3 Metal BIOS RAW
