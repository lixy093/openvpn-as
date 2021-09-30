
# 安装配置OpenVPN Access Server
### 安装openvpn-as
```
[root@openvpn ~]# yum -y install https://as-repository.openvpn.net/as-repo-centos7.rpm
[root@openvpn ~]# yum -y install openvpn-as
```

如果yum网络原因无法安装，可以手动下载安装包进行安装：
```
wget https://swupdate.openvpn.net/as/openvpn-as-2.9.5_82d54e5b-CentOS7.x86_64.rpm
wget https://swupdate.openvpn.net/as/clients/openvpn-as-bundled-clients-19.rpm
yum install -y openvpn-as-bundled-clients-19.rpm
yum install -y openvpn-as-2.9.5_82d54e5b-CentOS7.x86_64.rpm
```
可在官网查看最新的版本号https://openvpn.net/sha256sum-data/   ，openvpn-as和penvpn-as-bundled-clients需要对应。

### 安装完成后会自动配置openvpn-as的WebUI，需要设置openvpn用户密码才能访问
```
[root@openvpn ~]# passwd openvpn
```

### 网页打开https://IP:943/admin，   用openvpn用户登录WebUI
![](https://devopsvn.xyz/wp-content/uploads/2021/07/Untitled-1.png)
![](https://devopsvn.xyz/wp-content/uploads/2021/07/Untitled-2.png)

### 缺省只有2个免费的connection
![](https://devopsvn.xyz/wp-content/uploads/2021/07/Untitled-4.png)

# 解除openvpn-as连接数限制
### 停止openvpn服务
```
[root@openvpn ~]# systemctl stop openvpnas
```

### 进入openvpn目录，查找pvovpn-2.0-py**.egg文件
```
[root@openvpn ~]# # cd /usr/local/openvpn_as/lib/python
[root@openvpn python]# ls -l pyovpn-2.0-py*.egg
```
本例文件为pyovpn-2.0-py3.6.egg，不同的python版本py**的名称会有所不同。

### 备份pyovpn-2.0-py*.egg，并解压
```
[root@openvpn python]# mv pyovpn-2.0-py3.6.egg pyovpn-2.0-py3.6.egg.bak
[root@openvpn python]# mkdir unlock_lic
[root@openvpn python]# cp -rp pyovpn-2.0-py3.6.egg.bak unlocl_lic/pyovpn-2.0-py3.6.zip
[root@openvpn python]# cd unlock_lic
[root@openvpn unlock_lic]# unzip pyovpn-2.0-py3.6.egg
```

```
[root@openvpn unlock_lic]# ll -h
total 5.7M
drwxr-xr-x  2 root root   79 Jul 29 11:37 common
drwxr-xr-x  2 root root  106 Jul 29 11:37 EGG-INFO
drwxr-xr-x 37 root root 4.0K Jul 29 11:37 pyovpn
-rw-r--r--  1 root root 5.7M Jul  7 20:24 pyovpn-2.0-py3.6.zip
```

### 备份OpenVPN Access Server连接数配置文件
```
[root@openvpn lic]# cd pyovpn/lic
[root@openvpn lic]# ls -lh uprop.pyc
[root@openvpn lic]# mv uprop.pyc uprop2.pyc
```
openvpn-as的2.9.0以上版本连接数配置文件是uprop.pyc，2.9.0以下版本uprop.pyo文件

## 新建一个名为uprop.py的python文件
修改ret['concurrent_connections'] = 1024的数值即可指定最大连接数

openvpn-as 2.9.0 及以上版本uprop.py内容:
```
from pyovpn.lic import uprop2
old_figure = None

def new_figure(self, licdict):
    ret = old_figure(self, licdict)
    ret['concurrent_connections'] = 2048
    return ret


for x in dir(uprop2):
    if x[:2] == '__':
        continue
    if x == 'UsageProperties':
        exec('old_figure = uprop2.UsageProperties.figure')
        exec('uprop2.UsageProperties.figure = new_figure')
    exec('%s = uprop2.%s' % (x, x))
```


## 将上面的uprop.py编译为库文件uprop.pyc
```
[root@openvpn lic]# python3 -O -m compileall uprop.py
[root@openvpn lic]# mv __pycache__/uprop.cpython-36.opt-1.pyc uprop.pyc
[root@openvpn lic]# ls -lh uprop.pyc
[root@openvpn lic]# rm -rf __pycache__ uprop.py
```

## 重新打包pyovpn-2.0-py3.6文件
```
[root@openvpn ]# cd /usr/local/openvpn_as/lib/python/unlock_lic
[root@openvpn unlock_lic]# ll -h
total 5.7M
drwxr-xr-x  2 root root   79 Jul 29 11:37 common
drwxr-xr-x  2 root root  106 Jul 29 11:37 EGG-INFO
drwxr-xr-x 37 root root 4.0K Jul 29 11:37 pyovpn
-rw-r--r--  1 root root 5.7M Jul  7 20:24 pyovpn-2.0-py3.6.zip

[root@openvpn unlock_lic]# zip -r pyovpn-2.0-py3.6_unlock.zip common EGG-INFO pyovpn

[root@openvpn unlock_lic]# ll -h
total 12M
drwxr-xr-x  2 root root   79 Jul 29 11:37 common
drwxr-xr-x  2 root root  106 Jul 29 11:37 EGG-INFO
drwxr-xr-x 37 root root 4.0K Jul 29 11:37 pyovpn
-rw-r--r--  1 root root 5.7M Jul 29 12:14 pyovpn-2.0-py3.6_unlock.zip
-rw-r--r--  1 root root 5.7M Jul  7 20:24 pyovpn-2.0-py3.6.zip
[root@openvpn unlock_lic]# mv pyovpn-2.0-py3.6_unlock.zip pyovpn-2.0-py3.6.egg
[root@openvpn unlock_lic]# mv pyovpn-2.0-py3.6.egg ../ 
[root@openvpn unlock_lic]cd ../
[root@openvpn python]# rm -rf __pycache__/*
```

## 重新初始化配置openvpn
```
[root@openvpn ~]# cd /usr/local/openvpn_as/bin
[root@openvpn bin]# ./ovpn-init
Detected an existing OpenVPN-AS configuration.
Continuing will delete this configuration and restart from scratch.
Please enter 'DELETE' to delete existing configuration: $\color{FF0000}{DELETE}$
 
          OpenVPN Access Server
          Initial Configuration Tool
------------------------------------------------------
OpenVPN Access Server End User License Agreement (OpenVPN-AS EULA)
...
Please enter 'yes' to indicate your agreement [no]: yes

……
……
------------------------------------------------------
```

## 重新启动VPN后，发现连接数限制已解除
![](https://devopsvn.xyz/wp-content/uploads/2021/07/Untitled-5.png)
