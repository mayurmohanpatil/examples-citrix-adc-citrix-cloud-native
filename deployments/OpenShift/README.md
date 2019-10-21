# Deployment architecture

The common deployment architectures emerging in OpenShift platform are of single-tier, dual-tier and service mesh load balancing.

Citrix ADC with Ingress Controller provides solution for unified ingress, service mesh lite and service mesh deployments on OpenShift distribution. The Citrix ingress controller automates the configuration of Citrix ADC load balancing microservices in Kubernetes environment.

**North-South traffic Load balancing**: North-South traffic is the traffic heading in and out of your Kubernetes Cluster. It is the traffic that comes from the client and hits the frontend microservices.

**East-West traffic Load balancing**: East-West traffic is the traffic from one microservice to another inside the Kubernetes Cluster. East-west traffic can be routes through either ingress Citrix ADC (VPX/MPX/CPX) or Citrix ADC CPX sidecar.

In usual k8s environment the E-W traffic is load balanced by kube-proxy and N-S traffic is load balanced by Ingress load balancer like Citrix ADC.

Citrix provides the solution for below topologies,
* Unified Ingress (Tier 1 ADC + CIC)
* 2-Tier service mesh lite (Tier 1+2 ADC + CIC)
* service mesh (Tier 1 ADC + Sidecar CPX + CIC)