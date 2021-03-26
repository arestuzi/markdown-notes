# Running (e.g.) CentOS 7 on unapproved hardware

The official CentOS 7 marketplace AMI incurs no extra fees and usually works just fine, but due to its product code and the associated marketplace metadata, AWS prohibits launching instance types not listed on that page.
At the time of posting, it appears that all AWS instance types are approved for use with the current CentOS 7 AMI, but when I originally developed these instructions, the CentOS 7 AMI could not be run on the p2.\* or x1.\* (etc.) instance types.

Start up a new instance that will host all the dd'ing work.

```bash
# the image id comes from https://aws.amazon.com/marketplace/pp/B00O7WM7QW ->
# "Continue" -> "Manual Launch" -> US East (N. Virginia)
aws ec2 run-instances \
  --image-id ami-46c1b650 \
  --key-name aws-ec2-20151025 \
  --instance-type m4.large \
  --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=8,DeleteOnTermination=true}' \
  --subnet-id subnet-6df23d24 \
  --associate-public-ip-address
```

Store the Instance ID as **HOST_INSTANCE**, and Volume ID as **HOST_VOLUME**:
Then stop it:

```bash
aws ec2 stop-instances --instance-id <host-instance-id>
```

Start up a new instance:

```bash
aws ec2 run-instances \
  --image-id ami-46c1b650 \
  --instance-type m4.large \
  --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=8,DeleteOnTermination=false}' \
  --subnet-id subnet-6df23d24
```

Store the Volume ID as **SOURCE_VOL**, and then terminate the instance (all we want is the volume):

```bash
aws ec2 terminate-instances --instance-id <its-instance-id>
```

Create a spare volume:

```bash
# the original CentOS image has VolumeType=standard
aws ec2 create-volume \
  --size 8 \
  --availability-zone us-east-1c \
  --volume-type standard
```

Store the Volume ID as **TARGET_VOL**.
Attach the volumes to the stopped instance:

```bash
aws ec2 attach-volume \
  --volume-id $SOURCE_VOL \
  --instance-id $HOST_INSTANCE \
  --device /dev/sdc
aws ec2 attach-volume \
  --volume-id $TARGET_VOL \
  --instance-id $HOST_INSTANCE \
  --device /dev/sdf
```

Start the stopped instance:

```bash
aws ec2 start-instances --instance-id $HOST_INSTANCE
```

SSH in and clone the source volume to the target volume:

```bash
aws ec2 describe-instances --instance-id $HOST_INSTANCE
#
ssh <host-instance-ip>
#
sudo mkfs -t xfs /dev/xvdf
sudo dd bs=65536 if=/dev/xvdc of=/dev/xvdf
```

Output will look something like:

```bash
131072+0 records in
131072+0 records out
8589934592 bytes (8.6 GB) copied, 334.42 s, 25.7 MB/s
```

Back on the local machine, stop the instance:
aws ec2 stop-instances --instance-id $HOST_INSTANCE
Detach all the volumes:

```bash
echo $HOST_VOLUME $SOURCE_VOLUME $TARGET_VOLUME |\
  xargs -n1 aws ec2 detach-volume --instance-id $HOST_INSTANCE --volume-id
```

Attach the cloned volume at the standard location:

```bash
aws ec2 attach-volume --instance-id $HOST_INSTANCE --volume-id $TARGET_VOLUME --device /dev/sda1
```

Trigger an AMI snapshot:

```bash
aws ec2 create-image \
  --instance-id $HOST_INSTANCE \
  --name 'centos7/base' \
  --description "CentOS 7 (x86_64) build 1602 (aw0evgkw8e5c1q413zgy5pjce) clone without product codes"
```

Wait a while for the image to be ready:

```bash
aws ec2 wait image-available --image-ids ami-491c095e
```

All done! Terminate the instance:

```bash
aws ec2 terminate-instances --instance-id $HOST_INSTANCE
```
