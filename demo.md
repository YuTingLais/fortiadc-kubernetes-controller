
# Demo: FortiADC Kubernetes Controller Integration
In this demo, we’ll showcase how to deploy a **TCP Virtual Server** on **FortiADC** that proxies traffic to a **PostgreSQL database service** running inside a **Kubernetes cluster**.

```mermaid
flowchart LR

  subgraph "Kubernetes Cluster(Microk8s with Calico CNI)"
    pgsvc["PostgreSQL Service"]
    sslpg["🔒 PostgreSQL Pod<br>SSL enabled + Cert Verification"]
    pgsvc --> sslpg
  end

  client["psql client"] --> l4vs["FortiADC Layer4 TCP VirtualServer<br>(172.23.133.109:5432)"]
  l4vs --> route["Content Routing<br>sourceAddress 172.23.133.0/24"]
  route --> pgsvc
```
## Outline

1.  Scenario Overview:  
    Deployment of a TCP Virtual Server to expose a PostgreSQL server with SSL enabled on Kubernetes, using Calico VXLAN as the CNI plugin.
2.  Integration Steps:  
    How to integrate FortiADC with Kubernetes in a Calico VXLAN environment.
3.  Deployment Steps:  
    How to deploy the TCP Virtual Server to expose the PostgreSQL server with SSL enabled.

## Scenario Overview

```mermaid
flowchart LR

  %% ---------------------
  %% Control Plane
  %% ---------------------
  subgraph controlplane["Control Plane"]
    direction LR

    %% K8s Cluster steps
    subgraph cp_k8s["Kubernetes Cluster"]
      direction TB
      step12["Deploy PostgreSQL Pod & Service<br>+ Deploy VirtualServer Resource"]:::fixed
      step3["FortiADC Kubernetes Controller monitors resources"]:::fixed
      step12 --> step3
    end

    %% FortiADC Appliance also inside Control Plane
    FortiADC["FortiADC Appliance"]:::forti
    step3 --> |"Controller syncs configuration to FortiADC"| FortiADC

    %% FortiADC Config Objects inside a subgraph
    subgraph configs["FortiADC Config Objects"]
      direction LR
      vs["(VirtualServer)"]:::oval
      cr["(Content Routings)"]:::oval
      pool["(Real Server Pool & Real Server)"]:::oval
      vxlan["(VXLAN)"]:::oval
    end

    FortiADC -.-> vs
    FortiADC -.-> cr
    FortiADC -.-> pool
    FortiADC -.-> vxlan
  end


  %% ---------------------
  %% Styles
  %% ---------------------
  classDef fixed fill:#ede7f6,stroke:#673ab7,stroke-width:1px;
  classDef forti fill:#fce4ec,stroke:#c2185b,stroke-width:2px;
  classDef oval fill:#fff,stroke:#333,stroke-width:1px,rx:40px,ry:40px;
  classDef controlplane fill:#fffde7,stroke:#333,stroke-width:2px;
  class controlplane controlplane
```
```mermaid
flowchart LR
  %% ---------------------
  %% Data Plane
  %% ---------------------
  subgraph dataplane["Data Plane"]
    direction LR

    %% Kubernetes Cluster
    subgraph subGraph0["Kubernetes Cluster<br>(Microk8s with Calico CNI)"]
      direction TB
      subgraph subGraph0a["K8s Node 172.23.133.48"]
        direction TB
        pgsvc["PostgreSQL Service"]
        sslpg["🔒 PostgreSQL Pod<br>SSL Enabled + Cert Verification<br>Pod IP: 10.1.183.193"]
        pgsvc --> sslpg
      end
    end

    %% Calico VXLAN Overlay
    subgraph subGraph1["FortiADC 172.23.133.171"]
      direction TB
      l4vs["FortiADC Layer4 TCP VirtualServer<br>172.23.133.109:5432"]
      vxroute["Calico VXLAN Routing"]
    end

   
    client["psql client"] --> l4vs
    l4vs == "VXLAN Tunnel<br>Encapsulated TCP Payload<br>VTEP: 172.23.133.171" ==> vxroute
    vxroute == "Decapsulate to Pod IP<br>VTEP: 172.23.133.48" ==> pgsvc

    
    linkStyle 2 stroke:#ff0000,stroke-width:3px,stroke-dasharray: 5 5,fill:none
    linkStyle 3 stroke:#ff0000,stroke-width:3px,stroke-dasharray: 5 5,fill:none
  end

  %% ---------------------
  %% Styles
  %% ---------------------
  classDef dataplane fill:#e8f5e9,stroke:#333,stroke-width:2px;
  class dataplane dataplane

```
## integrate FortiADC with Kubernetes in a Calico VXLAN environment.


## Deploy the TCP Virtual Server to expose the PostgreSQL server with SSL enabled.


### Installation
Install the FortiADC Kubernetes Controller using Helm Charts.

:bulb: Currently, only Helm 3 (version 3.6.3 or later) is supported.

Helm Charts ease the installation of the FortiADC Kubernetes Controller in the Kubernetes cluster. By using the Helm 3 installation tool, most of the Kubernetes objects required for the FortiADC Ingress Controller can be deployed in one simple command.

To get the verbose output, add --debug option for all the Helm commands.

#### Install cert-manager.io

The version of cert manager we had verified is v1.18.2

https://cert-manager.io/docs/installation/


#### Create credential to access FortiADC Kubernetes Controller image

Please refer to the private_install.pdf

#### Get Repo Information


    helm repo add fortiadc-kubernetes-controller https://YuTingLais.github.io/fortiadc-kubernetes-controller/

    helm repo update

#### Install FortiADC Ingress Controller

    helm install --devel --debug first-release --namespace fortiadc-ingress --create-namespace --wait fortiadc-kubernetes-controller/fadc-k8s-ctrl

#### Check the installation

    helm history -n fortiadc-ingress first-release

    kubectl get -n fortiadc-ingress deployments

    kubectl get -n fortiadc-ingress pods

Check the log of the FortiADC Kubernetes Controller.

    kubectl logs -n fortiadc-ingress -f [pod name]

#### Upgrading chart

    helm repo update
    helm upgrade --devel --debug --reset-values -n fortiadc-ingress first-release fortiadc-kubernetes-controller/fadc-k8s-ctrl


## Configuration parameters
### FortiADC Authentication Secret

As shown in above figure, the FortiADC Kubernetes Controller satisfies an Ingress by FortiADC REST API call, so the authentication parameters of the FortiADC must be known to the FortiADC Kubernetes Controller.

To preserve the authentication securely on the Kubernetes cluster, you can save it with the Kubernetes secret. For example

    kubectl create secret generic fad-login -n [namespace] --from-literal=username=admin --from-literal=password=[admin password]

The secret is named fad-login. This value will be specified in the Ingress annotation "fortiadc-login" for the FortiADC Kubernetes Controller to get permission access on the FortiADC.

:warning:  The namespace of the authentication secret must be the same as the Ingress/virtualserver which references this authentication secret.
### Virtual Server Configuration Fields
#### Annotation in VirtualServer
Configuration parameters are required to be specified in the VirtualServer annotation to enable the FortiADC Kubernetes Controller to determine how to deploy the VirtualServer resource.

|Parameter  | Description | Default |
|--|--|--|
| fortiadc-ip | The VirtualServer will be deployed on the FortiADC with the given IP address or domain name. <br> **Note**: This parameter is **required**. | |
| fortiadc-admin-port | FortiADC https service port. | 443|
| fortiadc-login | The Kubernetes secret name preserves the FortiADC authentication information. <br> **Note**: This parameter is **required**. | |
| fortiadc-ctrl-log | Enable/disable the FortiADC Kubernetes Controller log. Once enabled, the FortiADC Kubernetes Controller will print the verbose log the next time the VirtualServer is updated. |enable |

#### VirtualServer Resource Specification


| Field                     | Type     | Description                                                                                                                                                                     | Default              |
|--------------------------|----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------|
| addressType              | string   | Specifies the IP version for the virtual server (ipv4 or ipv6).                                                                                                                 | ipv4                 |
| address                  | string   | The VirtualServer IP on FortiADC where client traffic is received. This parameter is required.                                                                                  |                      |
| port                     | int      | The listening port of the virtual server, typically 443 for HTTPS.                                                                                                              | 443                  |
| interface                | string   | The FortiADC network interface for the client to access the virtual server (e.g., port1).                                                                                       | port1                |
| loadBalanceProfile       | string   | The load balancing profile to be used. Allowed values: `LB_PROF_HTTPS`, `LB_PROF_HTTP`, `LB_PROF_TCP`, `LB_PROF_UDP`                                                           | LB_PROF_HTTPS        |
| loadBalanceMethod        | string   | Load balancing method, such as `LB_METHOD_ROUND_ROBIN`.                                                                                                                         | LB_METHOD_ROUND_ROBIN |
| wafProfile               | string   | The name of the Web Application Firewall (WAF) profile to apply.                                                                                                                |                      |
| captchaProfile           | string   | CAPTCHA profile to enable human verification security features.                                                                                                                 |                      |
| avProfile                | string   | Antivirus profile to scan traffic for threats.                                                                                                                                  |                      |
| trafficGroup             | string   | The traffic group that this virtual server belongs to for traffic forwarding.                                                                                                   | default              |
| fortiview                | string   | Enables/disables FortiView, FortiADC's traffic visualization and analytics (`enable` or `disable`).                                                                            | disable              |
| trafficLog               | string   | Enables/disables logging of traffic handled by this virtual server (`enable` or `disable`).                                                                                     | disable              |
| wccp                     | string   | Enables/disables WCCP (Web Cache Communication Protocol) (`enable` or `disable`).                                                                                               | disable              |
| fortigslbPublicIpType    | string   | Type of public IP used for FortiGSLB (usually ipv4).                                                                                                                            | ipv4                 |
| fortigslbPublicAddress   | string   | The public IP address used for global server load balancing (GSLB).                                                                                                             |                      |
| fortigslbOneClick        | string   | Enables/disables FortiADC’s one-click GSLB configuration feature (`enable` or `disable`).                                                                                       | disable              |
| fortigslbHostName        | string   | Hostname registered for GSLB.                                                                                                                                                   |                      |
| fortigslbDomainName      | string   | Domain name associated with the GSLB hostname.                                                                                                                                  |                      |
| realServerPool           | object   | Direct backend service definition. Use instead of `contentRoutings`.                                                                                                            |                      |
| contentRoutings          | list     | Defines Layer 4 and Layer 7 routing rules. See table below for fields.                                                                                                          |                      |
| natSourcePoolList        | list     | List of SNAT (source NAT) pools. Ensure each SNAT pool already exists on FortiADC.                                                                                              |                      |
| tls                      | list     | TLS certificate configurations. Each includes: `hosts` (list of hostnames), `secretName` (Kubernetes TLS secret).                                                              |                      |
| vdom                     | string   | Specifies the VDOM (Virtual Domain) on FortiADC where the virtual server is deployed.                                                                                           |                      |

---

### `realServerPool` Field Details

| Field            | Type                 | Description                                                                                                                                       |
|------------------|----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| service          | string               | Name of the Kubernetes service that this virtual server routes traffic to. Must match valid DNS label format (e.g., my-service).                  |
| servicePort      | integer or string    | Port of the target service. Can be numeric (e.g., 80) or named (e.g., "http").                                                                   |
| serviceNamespace | string (optional)    | Namespace of the target service. Defaults to the namespace of the VirtualServer resource if not specified. Useful for cross-namespace routing.   |

---

### `contentRoutings` Field Details

| Field                      | Type     | Description                                                                                                                                                   | Default        |
|---------------------------|----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------|
| name                      | string   | A unique name for the routing rule.                                                                                                                           |                |
| type                      | string   | Type of content routing. Valid values: `l4-content-routing` or `l7-content-routing`.                                                                          | l7-content-routing |
| host                      | string   | The hostname (FQDN) to match for this rule (e.g., example.com, *.foo.com). Only for `l7-content-routing`.                                                    |                |
| path                      | string   | The URL path to match (e.g., /, /info, /hello). Only for `l7-content-routing`.                                                                                |                |
| pathType                  | string   | Path matching type: `Prefix` or `Exact`. Only for `l7-content-routing`.                                                                                       |                |
| sourceAddressType         | string   | IP version for source address matching (ipv4 or ipv6). Only for `l4-content-routing`.                                                                         | ipv4           |
| sourceAddress             | string   | Source address in CIDR form (e.g., 0.0.0.0/0, ::/0). Only for `l4-content-routing`.                                                                           | 0.0.0.0/0      |
| realServerPool.service    | string   | Name of the Kubernetes Service to route traffic to.                                                                                                           |                |
| realServerPool.servicePort| int      | Target port of the Kubernetes Service.                                                                                                                        |                |
| realServerPool.serviceNamespace | string | Kubernetes namespace where the service resides.                                                                                                              |                |
### Deployment steps

#### Deploy the Pods and expose the Services

Download the PostgresSQL with SSL enabled service yaml file:

     curl -k https://raw.githubusercontent.com/fortinet/fortiadc-kubernetes-controller/main/service_examples/postgresql_ssl_service.yaml -o postgresql_ssl_service.yaml

:warning:  Please make sure you had installed the cert-manager. The certificates used in this example are all auto-signed and issued by cert-manager.

Modify the Service Annotation in postgresql_ssl_service.yaml to accommodate to your environment, ex: change the name of overlay_tunnel if your want to expose your service with ClusterIP type. Then deploy the PostgresSQL with SSL enabled service with kubectl command

    kubectl apply -f postgresql_ssl_service.yaml

#### Deploy the VirtualServer

Download the virtualserver_postgres_ssl.yaml


    curl -k https://raw.githubusercontent.com/fortinet/fortiadc-kubernetes-controller/main/customResource/virtualserver_postgres_ssl.yaml -o virtualserver_postgres_ssl.yaml

Modify the VirtualServer Annotation in virtualserver_postgres_ssl.yaml to accommodate to your environment, ex: fortiadc-ip, fortiadc-admin-port, etc.. Also, modify the VirtualServer Spec, ex: address, contentRoutings.SourceAddress.  Then deploy the virtualserver with kubectl command

    kubectl apply -f virtualserver_postgres_ssl.yaml

Try to access PostgreSQL use psql client.
```bash
~$ psql "host=192.168.1.108 port=5432 dbname=mydb user=admin password=StrongP@ssw0rd sslmode=require"
psql (10.23 (Ubuntu 10.23-0ubuntu0.18.04.2), server 16.10)
WARNING: psql major version 10, server major version 16.
         Some psql features might not work.
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

mydb=#

```
