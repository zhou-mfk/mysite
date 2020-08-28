---
title: "Python中使用ldap实现用户认证"
date: 2019-11-19T17:02:32+08:00
draft: false
tags: ["python", "ldap", "python-ldap"]
categories: ["Python"]
keywords: ["python", "ldap", "python-ldap"]
---

使用python实现域账号认证，并增加对ldaps的支持

版本信息如下:

* python 3
* python-ldap 3.2.0

## 安装python-ldap

参考文档 [python-ldap](https://www.python-ldap.org/en/python-ldap-3.2.0/index.html)

```python
python -m pip install python-ldap
```

## 代码实例

ldapauth 函数中增加了对ldaps的支持

```python
import ldap
from random import choice

LDAP_URL = ["ldap://xx.xx.xx.xxx", "ldaps://xxx.xxx.xxx.xxx", ]

def ldapauth(username, passwd):
    dn = '%s@corp.xx.com' % username
    ldap_url = choice(LDAP_URL)  # 从域链接中也随机读取一个访问
    if ldap_url.startswith('ldaps'):  # 增加对ldaps协议支持
        ldap.set_option(ldap.OPT_X_TLS_REQUIRE_CERT, ldap.OPT_X_TLS_NEVER)
        conn = ldap.initialize(ldap_url)  # 初始化ldap链接
        conn.set_option(ldap.OPT_X_TLS, ldap.OPT_X_TLS_DEMAND)
        conn.set_option(ldap.OPT_X_TLS_DEMAND, True)
    else:
        conn = ldap.initialize(ldap_url)
    conn.set_option(ldap.OPT_PROTOCOL_VERSION, 3)
    conn.set_option(ldap.OPT_REFERRALS, 0)

    try:
        # 开始认证
        res = conn.simple_bind_s(dn, passwd)  ## 返回的结果默认为: (97, [], 1, []) 
        if res and isinstance(res, tuple) and res[0] == 97:
            return {'status': 1, 'msg': 'Auth OK'}
        else:
            # 这段内容，有时返回值为空，但是不代表认证失败，在这里统一为认证成功
            return {'status': 1, 'msg': res}
    except ldap.INVALID_CREDENTIALS as e:  ## 这里认证失败时捕捉的异常信息
        if isinstance(e, dict) and 'desc' in e:
            return {'status': 0, 'msg': e['desc']}
        else:
            return {'status': 0, 'msg': 'auth error'}
    except Exception as e:  ## 这里捕捉其他异常信息，也统一表示为认证失败
        if isinstance(e, dict) and 'desc' in e:
            return {'status': 0, 'msg': e['desc']}
        else:
            return {'status': 0, 'msg': 'auth error'}
```

以上就是使用python-ldap实现域认证

有时间会试下ldap3这个模块