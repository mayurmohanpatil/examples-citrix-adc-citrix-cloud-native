# Learn how to expose Tier 2 ADC CPX to Tier 1 ADC VPX using NodePort type K8s service)

This guide covers the below use cases

| Section | Description |
| ------- | ----------- |
| [Section A] | Citrix product overview for on-prem K8s architecture and components |
| [Section B] | Kubernetes on-prem cluster setup |
| [Section C] | Deploy and Expose the microservices using NodePort type services |
| [Section D] | Integration with CNCF tools for Monitoring (Prometheus/Grafana) |
| [Section E] | Integration with CRDS |
| [Section F] | Integration with ADM observability Service Graph solution |
| [Section G] | Packer flow Visualizer - How does North-South traffic flows in microservices |



## Section A (Citrix product overview for on-prem K8s architecture and components)

Citrix ADC works in the two-tier architecture deployment solution to load balance the enterprise grade applications deployed in microservices and access those through internet. Tier 1 can have traditional load balancers such as VPX/SDX/MPX, or CPX (containerized Citrix ADC) to manage high scale north-south traffic. Tier 2 has CPX deployment for managing microservices and load balances the north-south & east-west traffic.

![2tierarchitecture](https://user-images.githubusercontent.com/5059506/52114542-518e2080-2632-11e9-8d17-eb0b5623b74f.png)


In the Kubernetes cluster, pod gets deployed across worker nodes. Below screenshot demonstrates the microservice deployment which contains 3 services marked in blue, red and green colour and 12 pods running across two worker nodes. These deployments are logically categorized by Kubernetes namespace (e.g. team-hotdrink namespace)

![hotdrinknamespacek8s](https://user-images.githubusercontent.com/42699135/50677395-99179180-101f-11e9-93f0-566cf179ce25.png)

## Section B (Kubernetes on-prem cluster setup)

Here are the detailed demo steps in cloud native infrastructure which offers the tier 1 and tier 2 seamless integration along with automation of proxy configuration using yaml files. 

1.	Bring your own nodes (BYON)
Kubernetes is an open-source system for automating deployment, scaling, and management of containerized applications. Please install and configure Kubernetes cluster with one master node and at least two worker node deployment.
Recommended OS: Ubuntu 16.04 desktop/server OS. 
Visit: https://kubernetes.io/docs/setup/scratch/ for Kubernetes cluster deployment guide.
Once Kubernetes cluster is up and running, execute the below command on master node to get the node status.
``` 
kubectl get nodes
```
 ![getnodes](https://user-images.githubusercontent.com/42699135/50677393-987efb00-101f-11e9-8580-4d27746bb96a.png)
 
 (Screenshot above has Kubernetes cluster with one master and two worker node).

2.	Set up a Kubernetes dashboard for deploying containerized applications.
Please visit https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/ and follow the steps mentioned to bring the Kubernetes dashboard up as shown below.

![k8sdashboard](https://user-images.githubusercontent.com/42699135/50677396-99179180-101f-11e9-95a4-1d9aa1b9051b.png)

3. Make sure that route configuration is present in Tier 1 ADC so that Ingress Citrix ADC should be able to reach Kubernetes pod network for seamless connectivity. Please refer to https://github.com/citrix/citrix-k8s-ingress-controller/blob/master/docs/network/staticrouting.md#manually-configure-route-on-the-citrix-adc-instance for Network configuration.


## Section C (Deploy and Expose the microservices using NodePort type services)

Citrix ADC CPX deployed as Tier 2 proxy exposed as Ingress type service to Tier 1 ADC VPX. We will deploy 3 CPX's to load balance microservices such as hotdrink, colddrink and guestbook beverages applications.
Hotdrink, colddrink and Guestbook microservices are also exposed as Ingress to Tier 2 CPX.

1.	Clone the GitHub repository to your Master node using following command

``git clone https://github.com/citrix/example-cpx-vpx-for-kubernetes-2-tier-microservices.git ``

Now change the directory to access the yaml files 

``
cd example-cpx-vpx-for-kubernetes-2-tier-microservices/on-prem/NodePort-config/ ``

2. Create a namespaces using Kubernetes master CLI console.
```
kubectl create -f namespace.yaml
```
Once you execute above commands, you should see the output given in below screenshot using command: 
```
kubectl get namespaces
```
![getnamespace](https://user-images.githubusercontent.com/42699135/50677390-97e66480-101f-11e9-9a69-cc132407bd1e.png)

3.	Go to Kubernetes dashboard and deploy the ``rbac.yaml`` in the default namespace
```
kubectl create -f rbac.yaml 
```

4.	Deploy the CPX for hotdrink, colddrink and guestbook microservices using following commands,

```
kubectl create -f cpx.yaml -n tier-2-adc
kubectl create -f hotdrink-secret.yaml -n tier-2-adc
```

5.	Deploy the three types of hotdrink beverage microservices using following commands
```
kubectl create -f team-hotdrink.yaml -n team-hotdrink
kubectl create -f hotdrink-secret.yaml -n team-hotdrink
```

6.	Deploy the colddrink beverage microservice using following commands
```
kubectl create -f team-colddrink.yaml -n team-colddrink
kubectl create -f colddrink-secret.yaml -n team-colddrink
```

7.	Deploy the guestbook no SQL type microservice using following commands
```
kubectl create -f team-guestbook.yaml -n team-guestbook
```
8.	Login to Tier 1 ADC (VPX/SDX/MPX appliance) to verify no configuration is pushed from Citrix Ingress Controller before automating the Tier 1 ADC.

9.	Deploy the VPX ingress and ingress controller to push the CPX configuration into the tier 1 ADC automatically.
**Note:-** 
Go to ``ingress-vpx.yaml`` and change the IP address of ``ingress.citrix.com/frontend-ip: "x.x.x.x"`` annotation to one of the free IP which will act as content switching vserver for accessing microservices.
e.g. ``ingress.citrix.com/frontend-ip: "10.105.158.160"``
Go to ``cic-vpx.yaml`` and change the NS_IP value to your VPX NS_IP.         
``- name: "NS_IP"
  value: "x.x.x.x"``
Now execute the following commands after the above change.
```
kubectl create -f ingress-vpx.yaml -n tier-2-adc
kubectl create -f cic-vpx.yaml -n tier-2-adc
```

  
10.	Add the DNS entries in your local machine host files for accessing microservices though internet.
Path for host file: ``C:\Windows\System32\drivers\etc\hosts``
Add below entries in hosts file and save the file

```
<frontend-ip from ingress-vpx.yaml> hotdrink.beverages.com
<frontend-ip from ingress-vpx.yaml> colddrink.beverages.com
<frontend-ip from ingress-vpx.yaml> guestbook.beverages.com
```
  
11.	Now you are ready to access your application from browser.
e.g. ``https://hotdrink.beverages.com``

![hotbeverage_webpage](https://user-images.githubusercontent.com/42699135/50677394-987efb00-101f-11e9-87d1-6523b7fbe95a.png)

## Section D (Integration with Kubernetes Monitoring tools like Prometheus, Grafana)
We will integrate with Prometheus and Grafana for visualizing time series data based on specific metrics such as LB hits count, CPU and memory utilization

12.	Deploy the CNCF monitoring tools such as Prometheus and Grafana to collect ADC proxies’ stats. Monitoring ingress yaml will push the configuration automatically to Tier 1 ADC.
**Note:-**
Go to ``ingress-vpx-monitoring.yaml`` and change the frontend-ip address from ``ingress.citrix.com/frontend-ip: "x.x.x.x"`` annotation to one of the free IP which will act as content switching vserver Prometheus and Grafana portal or you can use the same frontend-IP used in Step 11. 
e.g. ``ingress.citrix.com/frontend-ip: "10.105.158.161"``
```
kubectl create -f monitoring.yaml -n monitoring
kubectl create -f ingress-vpx-monitoring.yaml -n monitoring
```

13.	Add the DNS entries in your local machine host files for accessing monitoring portals though internet.
Path for host file: ``C:\Windows\System32\drivers\etc\hosts``
Add below entries in hosts file and save the file
```
<frontend-ip from ingress-vpx-monitoring.yaml> grafana.beverages.com
<frontend-ip from ingress-vpx-monitoring.yaml> prometheus.beverages.com
```
14.	Login to ``http://grafana.beverages.com:8080`` and do the following one-time setup
Login to portal using admin/admin credentials.
Click on Add data source and select the Prometheus data source. Do the settings as shown below and click on save & test button.
 
 ![grafana_webpage](https://user-images.githubusercontent.com/42699135/50677392-987efb00-101f-11e9-993a-cb1b65dd96cf.png)
 
From the left panel, select import option and upload the json file provided in folder yamlFiles ``/example-cpx-vpx-for-kubernetes-2-tier-microservices/config/grafana-config.json``
Now you can see the Grafana dashboard with basic ADC stats listed.
 
 ![grafana_stats](https://user-images.githubusercontent.com/42699135/50677391-97e66480-101f-11e9-8d42-87c4a2504a96.png)
 

## Section E (Integration with CRDs)

**Configure Rewrite and Responder policies in Tier 1 ADC using Kubernetes CRD deployment**

Now it's time to push the Rewrite and Responder policies on Tier1 ADC (VPX) using the custom resource definition (CRD).

1. Deploy the CRD to push the Rewrite and Responder policies in to tier-1-adc in default namespace.

   ```
   kubectl create -f crd-rewrite-responder.yaml
   ```

1. **Blacklist URLs** Configure the Responder policy on `hotdrink.beverages.com` to block access to the coffee beverage microservice.

   ```
   kubectl create -f responderpolicy-hotdrink.yaml -n tier-2-adc
   ```

   After you deploy the Responder policy, access the coffee page on `https://hotdrink.beverages.com/coffee.php`. Then you receive the following message.
   
   ![cpx-ingress-image16a](https://user-images.githubusercontent.com/48945413/55129538-7f2cad00-513d-11e9-9191-72a385fad377.png)

1. **Header insertion** Configure the Rewrite policy on `https://colddrink.beverages.com` to insert the session ID in the header.

   ```
   kubectl create -f rewritepolicy-colddrink.yaml -n tier-2-adc
   ```

   After you deploy the Rewrite policy, access `colddrink.beverages.com` with developer mode enabled on the browser. In Chrome, press F12 and preserve the log in network category to see the session ID, which is inserted by the Rewrite policy on tier-1-adc (VPX).

   ![cpx-ingress-image16b](https://user-images.githubusercontent.com/48945413/55129567-9075b980-513d-11e9-9926-d1207d7d1e16.png)

## Section G (Packer flow Visualizer - How does North-South traffic flows in microservices)

**Packet Flow Diagrams**


Citrix ADC solution supports the load balancing of various protocol layer traffic such as SSL,  SSL_TCP, HTTP, TCP. Below screenshot has listed different flavours of traffic supported by this demo.
![traffic_flow](https://user-images.githubusercontent.com/42699135/50677397-99179180-101f-11e9-8a40-26ba7d0d54e0.png)


**How user traffic reaches hotdrink-beverage microservices?**

Client sends the traffic to Tier 1 ADC through Content Switching virtual server and reaches to pods where hotdrink beverage microservices are running. Detailed traffic flow is allocated in following gif picture (please wait for a moment on gif picture to see the packet flow).
![hotdrink-packetflow-gif](https://user-images.githubusercontent.com/42699135/53723239-4a566e80-3e8d-11e9-99d1-dd9bd53dea53.gif)
 
**How user traffic reaches guestbook-beverage microservices?**
Client sends the traffic to Tier 1 ADC through Content Switching virtual server and reaches to pods where guestbook beverage microservices are running. Detailed traffic flow is allocated in following gif picture (please wait for a moment on gif picture to see the packet flow).

![guestbook-app](https://user-images.githubusercontent.com/42699135/53723248-50e4e600-3e8d-11e9-8036-c27c9af22bf7.gif)

Please refer to Citrix ingress controller documentation for more details, https://github.com/citrix/citrix-k8s-ingress-controller
