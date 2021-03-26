# EC2 instances fail to launch

## Issue

- the status of instance changes to shutting-down after launch

## Environment

- All instances types
- gp3 EBS types

## Solution

### Situation One

1. Whether the AMI are encrypted.

    ```bash
    aws ec2 describe-images --image-ids ami-08982b5e42307a601 --query "Images[].BlockDeviceMappings"
    [
        [
            {
                "DeviceName": "/dev/sda1",
                "Ebs": {
                    "DeleteOnTermination": true,
                    "SnapshotId": "snap-06f4f392a5db8bd09",
                    "VolumeSize": 30,
                    "VolumeType": "gp2",
                    "Encrypted": true
                }
            }
        ]
    ]
    ```

2. You have to grant access to KMS if the AMI is encrypted by custom key, For example,

    ```json
    {
        "Version": "2012-10-17",
        "Id": "key-consolepolicy-3",
        "Statement": [
            {
                "Sid": "Enable IAM User Permissions",
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws-cn:iam::xxx:root"
                },
                "Action": "kms:*",
                "Resource": "*"
            },
            {
                "Sid": "Allow use of the key",
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws-cn:iam::xxxx:user/xx"
                },
                "Action": [
                    "kms:Encrypt",
                    "kms:Decrypt",
                    "kms:ReEncrypt*",
                    "kms:GenerateDataKey*",
                    "kms:DescribeKey"
                ],
                "Resource": "*"
            },
            {
                "Sid": "Allow attachment of persistent resources",
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws-cn:iam::xxx:user/xx"
                },
                "Action": [
                    "kms:CreateGrant",
                    "kms:ListGrants",
                    "kms:RevokeGrant"
                ],
                "Resource": "*",
                "Condition": {
                    "Bool": {
                        "kms:GrantIsForAWSResource": "true"
                    }
                }
            },
            {
                "Sid": "Allow GuardDuty to use the key",
                "Effect": "Allow",
                "Principal": {
                    "Service": "guardduty.amazonaws.com"
                },
                "Action": "kms:GenerateDataKey",
                "Resource": "*"
            }
        ]
    }
    ```

### Situation Two

1. EBS type is gp3, but you are trying to use gp2 as the EBS volume type during the EC2 instance launching, which is not allowed. You can use following command to check the EBS volume type of the AMI.

    ```bash
    aws ec2 describe-images --image-ids ami-0834505eb6210f5aa --query "Images[].BlockDeviceMappings"
    [
        [
            {
                "DeviceName": "/dev/sda1",
                "Ebs": {
                    "DeleteOnTermination": true,
                    "Iops": 3000,
                    "SnapshotId": "snap-06f4f392a5db8bd09",
                    "VolumeSize": 30,
                    "VolumeType": "gp3",
                    "Throughput": 125,
                    "Encrypted": true
                }
            }
        ]
    ]
    ```
