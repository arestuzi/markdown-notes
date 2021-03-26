# EC2 Linux实例健康检查失败一般排错步骤 - 实例无法启动篇

## 事件

- 运行在EC2实例上Linux操作系统实例健康检查失败
- 操作系统无法通过SSH正常登录
- 系统健康检查正常，实例健康检查失败

## 诊断

1. 查看控制台状态检查信息，正常应显示为系统状态检查和实例状态检查均通过。如下图所示，即说明系统健康检查失败。关于状态检查，请参考[文档](https://docs.amazonaws.cn/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html)
![EC2 Health Check](https://images.cnblogs.com/cnblogs_com/terryares/1932718/o_210218060520ec2_health_check.PNG "EC2 Health Check")

2. 查看系统日志和实例屏幕截图， 选中实例 -> 操作 -> 监控和故障排除 -> 获取系统日志/获取实例屏幕截图

## 解决方案

- 系统日志显示 [`"dracut-initqueue[282]: Warning: dracut-initqueue timeout - starting timeout scripts"`](https://www.cnblogs.com/terryares/p/14411551.html)

- 系统日志显示为空，屏幕截图显示Error 15, 请[查看](https://www.cnblogs.com/terryares/p/14411573.html)。  
![MBR Issue](https://images.cnblogs.com/cnblogs_com/terryares/1932716/o_210218052633mbr.png "MBR Issue")
