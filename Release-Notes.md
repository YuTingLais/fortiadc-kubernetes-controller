# FortiADC Ingress Controller Release Notes

## 3.1.0
### What's New
1. Support Kubernetes version to 1.34
2. New RemoteServer and Host Custom Resource: FortiADC Kubernetes Controller 3.1.0 introduces a new
Fortinet-defined Kubernetes RemoteServer and Host Custom Resource that support FortiADC GLB configuration
from Kubernetes manifests. This enables advanced traffic management across geographically distributed
servers to be applied without manual GUI or CLI configuration.
3. The Custom Resource VirtualServer has been upgraded to version v1alpha2, with added support for Layer 4 TCP/UDP VirtualServers.
The v1alpha2 version extends FortiADC capabilities by supporting not only HTTP/HTTPS load balancing, but also TCP/UDP proxying. With this enhancement, a Layer 4 VirtualServer resource can now be defined to act as a proxy or load balancer for Layer 4 (TCP/UDP) services within a Kubernetes cluster.
4. Support for the Calico VXLAN CNI plugin has been added.
FortiADC can now connect to Kubernetes clusters via an overlay network tunnel using Calico VXLAN, enabling seamless integration with environments using Calico as the CNI.
5. Fix [issue](https://github.com/fortinet/fortiadc-ingress/issues/8) to resolve race conditions that occur when updating the Ingress status.

## 3.0.0
### What's New
1. Support Kubernetes version to 1.33
2. Support OpenShift version to 4.19
3. New VirtualServer Custom Resource: FortiADC Kubernetes Controller 3.0.0 introduces a new Fortinet-defined
Kubernetes VirtualServer Custom Resource that extends the Kubernetes Ingress model to support full FortiADC
virtual server configuration from Kubernetes manifests. This enables advanced traffic management, security
policies, and TLS options to be applied without manual GUI or CLI configuration.


## 2.0.3
### What's New
Support Kubernetes version to 1.32

## 2.0.2
### What's New

 1. FortiADC Ingress Controller now supports OpenShift Container Platform version 4.15.
 2. Two operational modes are supported for OpenShift Routes: VxLAN and Static Routes. VxLAN mode is compatible with OpenShift Container Platform versions 4.7-4.12, utilizing Virtual Extensible LAN (VXLAN) to encapsulate traffic for overlay networking. Static Routes mode, supported in versions 4.13-4.15, relies on predefined network routes for traffic management.
 3. FortiADC Ingress Controller now supports Kubernetes version 1.30.

## 2.0.1
### What's New
Support Kubernetes version to 1.28
### Resolved Issues
 1. Fix [issue](https://github.com/fortinet/fortiadc-ingress/issues/10) to allow valid hostname and url.

## 2.0.0
### What's New

 1. Support Openshift Container Platform 4 with Kubernetes Ingress and OpenShift Route.
 2. Support ingress to expose service with ClusterIP type. This feature requires FortiADC version >= 7.4.0 to support. FortiADC 7.4.0 and FortiADC Ingress Controller 2.0.0 support Flannel with VXLAN backend as the Kubernetes network model CNI plugin. Via the VXLAN tunnel, FortiADC can forward HTTP/HTTPS requests to the Kubernetes service with ClusterIP type.
 3. Support Virtual server FortiGSLB configuration defined in annotation of Kubernetes Ingress.
 4. Add toleration in FortiADC Ingress Controller Helm deployment template. User can customize the toleration time to wait for FortiADC ingress controller spinning up if the Kubernetes node it locates becomes unreachable or goes into NotReady state. The default toleration time for in FortiADC Ingress Controller is 30 seconds. You can check Kubernetes [taints and tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) for more details.
 5. Support declarative api for full synchronization. Be aware of using FortiADC version < 7.4, only 
 global administrator has the persmssion to call declarative api.
### Resolved Issues
 1. Fix [issue](https://github.com/fortinet/fortiadc-ingress/issues/7) to allow non-global administrator to operate the Ingress

## 1.0.2
### What's New
Support Kubernetes version to 1.27

## 1.0.1
### What's New
Support Kubernetes version to 1.24

## 1.0.0
### What's New

 FortiADC Ingress Controller combines the capabilities of an Kubernetes Ingress resource with the Ingress-managed loadbalancer, FortiADC. FortiADC, as the Ingress-managed load-balancer, not only provides flexibility in load-balancing, but also guarantees more security with features such as the Web Application Firewall (WAF), Antivirus Scanning, and Denial of Service (DoS) prevention to protect the web server resources in the Kubernetes cluster. 
 Other features such as health check, traffic log management, and FortiView on FortiADC facilitates the management of the Kubernetes ingress resources.

