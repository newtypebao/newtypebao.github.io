---
layout: post
title: 基于tomcat服务器的CPU使用率，重启tomcat
date: 2024-09-23
tag: tomcat,linux,cpu,script
---

基于tomcat部署的javaweb项目中可能会遇到下面这个场景：由于应用功能里有个大报表、大查询又或者别的复杂的逻辑，把某一个tomcat所在服务器的cpu给搞满了。此时这个tomcat上的用户都巨卡。
原则上遇到上述情况应当排查具体的程序或者接口是不是有逻辑上的优化方法。以下是退一万步而言的终极方案，一个基于tomcat服务器的CPU使用率，重启tomcat。可以结合操作系统的定时任务执行，比如每5分钟扫描一次。如果命中规则(cpu>90%)就重启这个tomcat.

```
	#!/bin/bash
	#需要在服务器内安装一下两个组件1.sysstat   2.bc

	# 定义toncat目录
	tomcatPath=/usr/local/tomcat
	# 定义日志文件
	log_file="/tmp/tomcat_thread_monitor.log"


	# 获取Tomcat进程的主进程ID
	tomcat_pid=$(pgrep -f catalina)

	# 检查是否找到了Tomcat进程
	if [ -z "$tomcat_pid" ]; then
		echo "Tomcat process not found."
		exit 1
	fi

	# 获取当前cpu使用率
	cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2 + $4}')

	# 获取当前日期和时间
	current_date=$(date)

	# 记录开始检查的时间
	echo "Tomcat usage at $current_date is $cpu_usage"

	# 检查CPU使用率是否超过90%
	if (( $(echo "$cpu_usage > 90" | bc -l) )); then
		echo "CPU usage is above 90% ($cpu_usage%). Restarting Tomcat..."
		echo "$current_date: CPU usage is above 90% ($cpu_usage%). Restarting Tomcat">> $log_file
		# 重启Tomcat服务
		sh "$tomcatPath/bin/shutdown.sh"
		sleep 10
		kill -9 $tomcat_pid
		sleep 5
		#启动tomcat
		sh "$tomcatPath/bin/startup.sh"
	else
		echo "CPU usage is below 90% ($cpu_usage%). No action taken."
	fi
```
