# nmap使用

## 1、获取某个主机open的端口

~~~shell
sudo nmap IP
~~~

## 2、判断主机端口是否开放

~~~shell
sudo map -p port (domain)
~~~

## 3、扫描整个局域网的IP-MAC

```shell
# 内网主机发现
nmap -sP 192.168.0.0/24 
```

## 4、获取主机的所有信息

```shell
nmap -A IP 
```

