# How to rebalance hot files to each OST?

## Environment

- AWS FSx for Lustre

## Solution

1. Initially, 10 files(1GiB each) are stored on OST:0.

    ```bash
    [root@ip-172-31-7-64 fsx]# lfs df -h
    UUID                       bytes        Used   Available Use% Mounted on
    twxsdbmv-MDT0000_UUID      137.4G        6.4M      137.4G   0% /fsx[MDT:0]
    twxsdbmv-OST0000_UUID        1.1T       10.0G        1.1T   1% /fsx[OST:0]    <-----
    twxsdbmv-OST0001_UUID        1.1T        4.6M        1.1T   0% /fsx[OST:1]
    twxsdbmv-OST0002_UUID        1.1T        4.6M        1.1T   0% /fsx[OST:2]
    twxsdbmv-OST0003_UUID        1.1T        4.6M        1.1T   0% /fsx[OST:3]

    ```

2. Save file names to a file.

    ```bash
    ll | grep fio| awk '{print $NF}' > filelist
    ```

3. Run following script to migrate files to each OST node.

    ```bash
    #!/bin/bash
    ost_index=0
    while read -r line;
    do
    lfs migrate -c 1 -i ${ost_index} $line
    let ost_index+=1
    if [ $ost_index == 4 ]; then
            ost_index=0
    fi
    done < filelist
    ```

4. Check the result and which files are stored on OST.

    ```bash
    [root@ip-172-31-7-64 task1]# lfs df -h
    UUID                       bytes        Used   Available Use% Mounted on
    twxsdbmv-MDT0000_UUID      137.4G        6.4M      137.4G   0% /fsx[MDT:0]
    twxsdbmv-OST0000_UUID        1.1T        3.0G        1.1T   0% /fsx[OST:0]
    twxsdbmv-OST0001_UUID        1.1T        3.0G        1.1T   0% /fsx[OST:1]
    twxsdbmv-OST0002_UUID        1.1T        2.0G        1.1T   0% /fsx[OST:2]
    twxsdbmv-OST0003_UUID        1.1T        2.0G        1.1T   0% /fsx[OST:3]

    [root@ip-172-31-7-64 task1]# lfs find --ost 0 /fsx/
    /fsx//task1/fio_test_file.8.0
    /fsx//task1/rebalance.sh
    /fsx//task1/fio_test_file.0.0
    /fsx//task1/fio_test_file.4.0
    /fsx//task1/filelist
    ```
