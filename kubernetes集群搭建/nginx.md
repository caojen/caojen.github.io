# 通过创建Nginx服务来打通全过程

通过创建一个Nginx服务来理清楚K8s的大部分概念。

## 概念

首先我们应该知道了容器（container）的概念。

本文中，还会遇到以下几个概念。

1. `pod`. pod = 一个或多个contianers的集合。例如，pod里面可以有一个nginx。当然，你也可以写一个定时`GET /`的容器，和nginx放在一起。此时，这一个pod就是nginx+新容器的组合。
2. `deployment`. 一般来说我们不直接创建pod，而是使用deployment来管理pod（比如做replica之类的，升级回滚也用这个）
3. `service`. service用于将服务**转发流量**，譬如将deployment的某个端口映射到集群外面。service也可以做一个简单的负载均衡。

## 创建nginx deployment
