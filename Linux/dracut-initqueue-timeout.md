# 实例无法启动并包含报错`dracut-initqueue timeout`

## 环境

- Nitro 实例
- CentOS 7

## 事件

- "dracut-initqueue[282]: Warning: dracut-initqueue timeout - starting timeout scripts"

## 解决方案

1. 出现此报错可能是由于initramfs文件中没有包含Nitro实例需要的NVMe驱动模块导致，CentOS 7系统上较常见。
2. 停止当前实例，并使类型相同的上一代实例启动，比如m5.xlarge，修改为m4.xlarge
3. 启动实例后，执行命令校验当前实例是否可以在Nitro实例上运行。

    ```bash
    ```