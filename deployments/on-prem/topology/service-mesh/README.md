# Learn how to integrate Citrix ADC with Istio

Citrix ADC integrations with Istio allow you to secure and optimize traffic for applications in the service mesh using Citrix ADC features. Citrix ADC integrates with Istio in two ways:

* Citrix ADC CPX, MPX, or VPX as an Istio Ingress Gateway to the service mesh.
* Citrix ADC CPX as a sidecar proxy with application containers in the service mesh. Both modes can be combined to have a unified data plane solution.

This guide covers the below use cases

| Section | Description |
| ------- | ----------- |
| [Section A] | Citrix product overview for on-prem Istio architecture and components |
| [Section B] | Kubernetes on-prem Istio setup |
| [Section C] | Deploying Citrix ADC as Ingress Gateway |
| [Section D] | Deploying Citrix ADC CPX as Sidecar in microservices |
| [Section E] | Deploying Bookinfo application in servicemesh topology  |
| [Section F] | Integration with ADM observability Service Graph solution |

## Section A (Citrix product overview for on-prem Istio architecture and components)
The Istio service mesh can be logically divided into control plane and data plane components. The data plane is composed of a set of proxies which manage the network traffic between instances of the service mesh. The control plane generates and deploys the configuration that controls the data plane's behavior.
An Istio Ingress Gateway acts as an entry point for the incoming traffic to the service mesh. You can deploy a Citrix ADC CPX, MPX, or VPX as an ingress Gateway to the Istio service mesh.
Citrix ADC CPX can be deployed as a sidecar proxy in application pods. We will demo VPX as ingress Gateway and CPX as sidecar here as shown in below architecture.

## Section B (Kubernetes on-prem Istio setup)

Refer to Istio documentation - https://istio.io/docs/setup/install/kubernetes/ for deploying the Istio cluster on top of K8s cluster.

## Section C (Deploying Citrix ADC as Ingress Gateway)

We need helm package to deploy Citrix ADC VPX as ingress gateway. Please make sure that below pre-requisites are configured in your k8s cluster.

**Prerequisites**

1. Install the helm
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

Initialize Helm on your k8s cluster
```
kubectl delete deployment tiller-deploy -n kube-system
kubectl create -f https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/charts/tiller.yaml
kubectl --namespace kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller --upgrade
```

Verify the Helm package if installed correctly
```
helm version
```
If helm is initialized properly you will get output for helm version something like:
```
Client: &version.Version{SemVer:"v2.14.3", GitCommit:"0e7f3b6637f7af8fcfddb3d2941fcc7cbebb0085", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.3", GitCommit:"0e7f3b6637f7af8fcfddb3d2941fcc7cbebb0085", GitTreeState:"clean"}
```

2. Verify the Istio version 1.1.2 is installed
```
istio version
```

3. Verify the K8s version 1.14.0+ is installed
```
kubeadm version
kubectl version
```

**Deploy Citrix ADC VPX as an Ingress Gateway**

In this example, by default ``citrix-system`` namespace is used. You can change the namespace as per your requirement. 

Please provide NSIP and port details in ``istioAdaptor.netscalerUrl=https://<nsip>[:port]`` e.g. https://10.105.158.180:443

Please provide VIP details in ``istioAdaptor.vserverIP=<IPv4 Address>`` e.g. vserverIP=10.105.158.11

Please provide VPX admin credentials in ``nslogin.username=<username> & nslogin.password=<password>``

```
helm repo add citrix https://citrix.github.io/citrix-istio-adaptor/
helm install citrix/citrix-adc-istio-ingress-gateway --namespace citrix-system --name citrix-adc-istio-ingress-gateway --set ingressGateway.EULA=YES --set istioAdaptor.netscalerUrl=https://<nsip>[:port] --set istioAdaptor.vserverIP=<IPv4 Address> --set nslogin.username=<username> --set nslogin.password=<password>
```

## Section D (Deploying Citrix ADC CPX as Sidecar in microservices)

Citrix ADC CPX can be deployed as a sidecar proxy in an application pod in the Istio service mesh.
```
curl -L https://raw.githubusercontent.com/citrix/citrix-istio-adaptor/master/charts/stable/citrix-cpx-istio-sidecar-injector/create-certs-for-cpx-istio-chart.sh > create-certs-for-cpx-istio-chart.sh 

chmod +x create-certs-for-cpx-istio-chart.sh

./create-certs-for-cpx-istio-chart.sh --namespace citrix-system

helm repo add citrix https://citrix.github.io/citrix-istio-adaptor/

helm install citrix/citrix-cpx-istio-sidecar-injector --namespace citrix-system --name cpx-sidecar-injector --set cpxProxy.EULA=YES
```

## Section E (Deploying Bookinfo application in servicemesh topology)

1. Generate the certificate and key for the application

```
openssl genrsa -out bookinfo_key.pem 2048
openssl req -new -key bookinfo_key.pem -out bookinfo_csr.pem
```
Make sure to provide Common Name(CN/Server FQDN) as **"www.bookinfo.com"** on CSR information prompt.
```
openssl x509 -req -in bookinfo_csr.pem -sha256 -days 365 -extensions v3_ca -signkey bookinfo_key.pem -CAcreateserial -out bookinfo_cert.pem

```
2. Create k8s secret using above certificate in the same namespace where ingress gateway in deployed.

```
kubectl create -n citrix-system secret tls citrix-ingressgateway-certs --key bookinfo_key.pem --cert bookinfo_cert.pem
```
3. Create a namespace for deploying bookinfo application and enable ``cpx-injector`` to deploy CPX as sidecar proxy.

```
kubectl create namespace bookinfo
kubectl label namespace bookinfo cpx-injection=enabled
```
4. Deploy the bookinfo application mutual TLS enabled

```
git clone https://github.com/citrix/citrix-istio-adaptor.git
cd citrix-istio-adaptor/examples/citrix-adc-in-istio/bookinfo/charts/stable
helm install bookinfo-citrix-ingress --name bookinfo-citrix-ingress --namespace bookinfo --set citrixIngressGateway.namespace=citrix-system --set mtlsEnabled=true
```

5. You are ready to test your application from browser

Determine the virtual server IP
```
export INGRESS_IP=$(kubectl get pods -l app=citrix-ingressgateway -n citrix-system -o 'jsonpath={.items[0].spec.containers[?(@.name=="istio-adaptor")].args}' | awk '{ for(i=1;i<=NF;++i) { if ($i=="-vserver-ip") print $(i+1) } }')
```
Access the bookinfo frontend application using Ingress IP
```
https://$INGRESS_IP/productpage
```

Please refer to Citrix ADC Istio integration documentation for more details, https://github.com/citrix/citrix-istio-adaptor