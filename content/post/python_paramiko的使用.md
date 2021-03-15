---
title: "Python paramiko的使用"
date: 2021-03-15T10:40:32+08:00
draft: false
tags: ["python", "paramiko"]
categories: ["Python"]
keywords: ["python", "paramiko"]
---


使用paramiko在远程服务器上执行命令:


```python
import paramiko
from typing import Dict


def ssh_command(hostname: str, port: str, username: str, password: str, command: str, timeout: int = 90,
                get_pty: bool = False) -> Dict:
    """
    连接远程服务器执行shell命令
    :param hostname: 远程服务器ip地址
    :param port: 指定远程服务器的ssh端口
    :param username: 远程用户名
    :param password: 密码
    :param command: 执行的命令
    :param timeout: 执行命令的超时时间
    :param get_pty: 是否启用tty 即伪终端
    :return:
    """
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    try:
        ssh.connect(hostname=hostname, port=port, username=username, password=password, timeout=8)
        stdin, stdout, stderr = ssh.exec_command(command=command, timeout=timeout, get_pty=get_pty)
        exit_code = stdout.channel.recv_exit_status()
        ssh_result = {
            'output': stdout.readlines(),
            'IP': hostname,
            'msg': 1,
            'error': stderr.readlines(),
            'exit_code': exit_code,

        }
        ssh.close()
    except Exception as e:
        ssh_result = {
            'output': 0,
            'IP': hostname,
            'msg': 0,
            'error': [str(e)],
            'exit_code': -1,
        }
        ssh.close()
    return ssh_result
```



其中`get_pty` 需要注意

当为`True`时 此时所有命令执行的结果输出到stdout中，错误信息也会到stdout中

当为`False`时 此时错误输出会到stderr中 

当需要执行sudo命令时也需要使用到此参数

