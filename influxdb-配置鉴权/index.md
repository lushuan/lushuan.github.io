# Influxdb 添加鉴权

## InfluxDB 基础了解
### 介绍
一个开源的时间序列数据库
### 特性
1、内置HTTP-API,无需编写服务端代码来启动和运行。  
2、数据可以被标记，允许非常灵活的查询  
3、类似SQL一样的查询语言  
4、简单的安装和管理， 快速的获取和输出数据  
5、它的目标是符合实时查询， 这意味着每一个数据点都会被索引，并且可以立即在小于100ms的查询中获得

docker 创建容器实例
```shell
docker run --name=cm2-influxdb -d -p 8530:8083 -p 8586:8086 \
-v /opt/influxdata:/var/lib/influxdb \
-m 10g \
--restart=always \
influxdb:1.8
```

## 配置鉴权
### 添加鉴权配置
```
[http]  
  enabled = true  
  bind-address = ":8086"  
  auth-enabled = true  
  log-enabled = true  
  ping-auth-enabled = true  
```
重启influxdb服务

未添加鉴权  
```shell
$ curl -G http://172.16.24.220:8586/query --data-urlencode "q=SHOW DATABASES"  
{"results":[{"statement_id":0,"series":[{"name":"databases","columns":["name"],"values":[["host_metrics"],["vnf_metrics"],["events"],["alarm"],["custom_metrics"],["_internal"],["k8s_metrics"],["mw_metrics"],["bss_metrics"],["application_metrics"],["oss_metrics"],["itracing_metrics"],["prometheus_metrics"],["dcm_metrics"]]}]}]}
```

添加鉴权未创建用户,提示需要创建鉴权用户或者关闭鉴权  
```shell
curl -G http://172.16.17.82:8586/query --data-urlencode "q=SHOW DATABASES"
[lushuan@220 nms-influxdb]$ curl -G http://172.16.24.220:8586/query --data-urlencode "q=SHOW DATABASES"
{"error":"error authorizing query: create admin user first or disable authentication"}
```

登录influxdb 容器创建用户  
```
docker exec -it cm2-influxdb sh
> influx
> CREATE USER admin WITH PASSWORD 'password' WITH ALL PRIVILEGES 
```
再次测试  
```
$ curl -G http://172.16.24.220:8586/query -u admin:abc@123A --data-urlencode "q=SHOW DATABASES" 

{"results":[{"statement_id":0,"series":[{"name":"databases","columns":["name"],"values":[["host_metrics"],["vnf_metrics"],["events"],["alarm"],["custom_metrics"],["_internal"],["k8s_metrics"],["mw_metrics"],["bss_metrics"],["application_metrics"],["oss_metrics"],["itracing_metrics"],["prometheus_metrics"],["dcm_metrics"]]}]}]}
```

## 参考
- [influxdb 官网](https://docs.influxdata.com/influxdb/v1/introduction/download/)
