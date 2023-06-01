---
 categories:
   - k8s
---
# 在k8s中上报数据给Promethues

## 如何将自定义的指标，对接入Promethues中呢？

1. 在Pod上，起一个服务，暴露一个获取metrics的地址，比如说 /metrics，提供指标数据给Promethues抓取。（对于Promethues来说，叫做exporter），具体生成的数据格式，可以参考Promethues的exporter写法。
2. 创建Promethues识别的CRD抓取对象。（有很多种，这边以ServiceMonitor为例）Promethues会监控对应的CRD，从而根据配置去抓取数据。 
  * 创建一个独立的service，指向步骤1里的Pod
  * 创建一个ServiceMonitor，指向此service。

配置完ServiceMonitor，稍微等几分钟，就可以在Promethues的dashboard上查看对应的数据了。
菜单栏-Status-Targets里面可以看到新添加的ServiceMonitor

## 部署Promethues
一般直接使用：[https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

## 配置Promethues的抓取对象(Target)
Promethues的抓取对象，叫做`Target`，包含Rule，ServiceMonitor，PodMonitor等多种方式。是Promethues的CRD。

设置的方式，以ServiceMonitor方式来说：
* 可以创建独立的CRD
* 直接在values.yaml里面的属性：`defaultRules.prometheus.prometheusSpec.additionalServiceMonitors` 设置.
* 直接修改Promethues的配置文件

## Tips
### 需关注的属性设置
  由于ServiceMonitor是与namespace相关的，因此需要注意几个属性的设置：

  * `defaultRules.prometheus.prometheusSpec.serviceMonitorNamespaceSelector` Promethues只抓取有此labels的**namespace**底下的ServiceMonitor
  * `defaultRules.prometheusOperator.namespaces` 设置Promethues operator关心的namespaces. 换言之，在这些namespaces之外的`Target`，Promethues不管.

### Https相关
  如果提供的metrics服务是https，可以在ServiceMonitor.spec.enpoints.tlsConfig这边设置对应的证书信息. 
但是这边证书的路径是针对于Promethues Operator这个Pod来说。默认只包含k8s cluster的ca证书. 
如果是自签名证书，则可以考虑通过某种方式挂载到Promethues Operator这个Pod里面。比如Secrets或者ConfigMap之类的。

## Links
[https://kubevirt.io/2020/Monitoring-KubeVirt-VMs-from-the-inside.html](https://kubevirt.io/2020/Monitoring-KubeVirt-VMs-from-the-inside.html)
