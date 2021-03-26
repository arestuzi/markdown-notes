# The status of worknode is NotReady with error "Initialization failure: ipamd"

## Issue

- EKS worknode is not ready with following error

    ```bash
    {"level":"error","ts":"2021-01-27T00:17:16.845Z","caller":"aws-k8s-agent/main.go:28","msg":"Initialization failure: ipamd: can not initialize with AWS SDK interface: refreshSGIDs: unable to update the ENI's SG: WebIdentityErr: failed to retrieve credentials\ncaused by: InvalidIdentityToken: Incorrect token audience\n\tstatus code: 400, request id: xxxxxx"}
    ```

## Root Cause

- Trust relationship for EC2 instance profile is wrong.

## Solution

- the service `ec2.amazonaws.com` should be allowed to assume the role.

    ```bash
    # Add following policy to trust relationship of the role
    {
    "Version": "2012-10-17",
    "Statement": [
        {
        "Effect": "Allow",
        "Principal": {
            "Service": "ec2.amazonaws.com.cn"
        },
        "Action": "sts:AssumeRole"
        }
    ]
    }
    ```
