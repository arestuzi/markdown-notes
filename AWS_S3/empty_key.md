# How to create a folder(key) without any contents

## Issue

- s3fs failed to mount a folder with error

```bash
Nov 30 10:44:24 ip-172-31-7-64 s3fs[5578]:      check a bucket.
Nov 30 10:44:24 ip-172-31-7-64 s3fs[5578]:      URL is https://s3.cn-north-1.amazonaws.com.cn/terry-elb/1/
Nov 30 10:44:24 ip-172-31-7-64 s3fs[5578]:      URL changed is https://terry-elb.s3.cn-north-1.amazonaws.com.cn/1/
Nov 30 10:44:24 ip-172-31-7-64 s3fs[5578]:      computing signature [GET] [/1/] [] []
Nov 30 10:44:24 ip-172-31-7-64 s3fs[5578]:      url is https://s3.cn-north-1.amazonaws.com.cn
Nov 30 10:44:24 ip-172-31-7-64 s3fs[5578]:      HTTP response code 404 was returned, returning ENOENT
Nov 30 10:44:24 ip-172-31-7-64 s3fs[5578]: curl.cpp:CheckBucket(3553): Check bucket failed, S3 response: <?xml version="1.0" encoding="UTF-8"?>#012<Error><Code>NoSuchKey</Code><Message>The specified key does not exist.</Message><Key>1/</Key><RequestId>26959DBD4450AC0E</RequestId><HostId>0Iqvxjsi/A05Z4WD3rLqn4ulPMj4I5dlCAUMTkMRRQBEYsEzYPavPV1WglkMnybOKsBZqFyinUo=</HostId></Error>
```

### Solution

- create a key without any contents

    ```bash
    aws s3api put-object --body test.txt --key 1/ --bucket terry-elb
    ```
