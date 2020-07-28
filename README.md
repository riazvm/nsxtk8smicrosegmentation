VMware Advanced Customer Engagements (ACE) Team

Authored by Riaz Mohamed and Raghu Pemmaraju <br>

May 2020

Table of Contents
=================

[Introduction ](#introduction)

[Use-case ](#use-case)

[Traffic flows ](#traffic-flows)

[Assumptions ](#assumptions)

[K8s Resources ](#k8s-resources)

[Pod Definition ](#pod-definition)

[Deployments And Services ](#deployments-and-services)

[K8s Resource Deployments ](#k8s-resource-deployments)

[K8 Deployments ](#k8-deployments)

[Current Traffic Flow ](#current-traffic-flow)

[Configure Micro Segmentation with NSX-T](#configure-micro-segmentation-with-nsx-t)

[Test Traffic Flow ](#test-traffic-flow)

Introduction
============

Containers provide a great deal of benefits to your development pipeline and support resource isolation but they’re not designed with strong network security boundaries or workload isolation. By default, pods are non-isolated; they accept traffic from any source.

Tanzu Kubernetes Grid Integrated (TKGI previously knowns as Enterprise PKS) coupled with VMware NSX-T Data Center (NSX-T), and VMware vSphere provides an agile, software defined platform to deliver cloud-native applications. NSX-T primarily focuses on networking and security for these applications and the features support implementing stronger workload isolation. 

In TKGI, workload isolation can be performed by physically isolating workload at the vSphere cluster (group of physical hosts) level or by logically isolating workloads within the TKGI deployed clusters (at cluster or namespace isolation). The following design patterns can be considered for workload isolation.
![A screenshot of a video game Description automatically generated](./media/image34.png)

In this document, we focus on how to leverage the micro-segmentation feature within NSX-T to provide workload isolation. NSX-T comes with a distributed firewall that can provide complete control of both North-South Traffic but also East-West Traffic and can isolate workloads, even if they are next to each other. For example, traditional firewalls only isolate network traffic between network VLANs or segments but not within a network segment. But with NSX-T distributed firewall, you can create rules to isolate workload on the same segment and with Kubernetes tags, you can isolate even Kubernetes pod-to-pod communication. 


In this document, we take a simple application that has several components or services. These services are required to communicate with each other in a very defined manner. For example, service-a needs to communicate with service-c and service-b but not with any other service. Similarly, service-c needs to communicate with service-d but not with service-a or service-b. 

In such a scenario, we look at how to isolate the workload using NSX-T, and also show how this is done dynamically as pods are created and destroyed. 




For this guide, consider the following application with few loosely coupled services. Assume that all ingress traffic from outside the cluster comes into service-a. service-a is running in namespace x1 and typically will use ingress controller like Nginx or Contour. service-a routes to service-b or service-c, and service-c might need to talk to service-d. This diagram shows which workload is allowed and not allowed between the services.


![A screenshot of a video game Description automatically generated](./media/image1.png)

Use-case 
========

The k8s cluster has three namespaces (namespace x1, y1 and z1). All ingress into the cluster must be restricted to namespace x1 and pod service-a. 
Namespace y1 runs a pod service-b and namespace z1 runs pods service-c and service-d.


Namespace y1 runs a pod service-b and namespace z1 runs pods service-c
and service-d.

Traffic flows
-------------

All ingress traffic into the cluster can only be serviced by service-a on namespace x1 which means the following traffic flow rules must be enforced:

Service-b allows traffic from service-a

Service-b denies traffic from service-c

Service-c allows traffic from service-a and service-b

Service-d allows traffic from only service-c

Service-c denies traffic from service-d

Assumptions
-----------

K8s cluster managed by TKGI (PKS) exists with NSX-T as the networking layer

K8s Resources
=============
In this section, the K8S resources associated with the sample application and services is described.

Pod Definition
--------------

The sample application will have a pod definition with 2 containers:
•	nginx (to service web requests)
•	busybox with curl (container to test service-to-service traffic)

Deployments and Services
------------------------
service-a, service-b, service-c and service-d are identical and run the above pod definition on their respective namespaces. 
They run two replicas and are exposed as services svc-service-a, svc-service-b, svc-service-c, and svc-service-d.

K8s Resource Deployments
========================

This section goes through the steps to set up the K8 environment with the deployments in their respective namespaces. 
At the end of the setup, a test will be performed to verify that the traffic flows between the services.

K8 Deployments
--------------

1.  **Login to k8s cluster**

> pks login -a <pks-api\> -u <pksuser\> -p <pks-password\> -k
>
> e.g.
>
> pks login -a pks.corp.local -u pksadmin -p VMware1! -k

2.  **Get Kubeconfig**

> pks get-kubeconfig \<cluster-name\> -a \<pks-api\> -u \<pksuser\> -p
> \<pks-password\> -k
>
> e.g.
>
> pks get-kubeconfig ci-cluster -a pks.corp.local -u pksadmin -p
> VMware1! -k

3.  **Change context**

> kubectl config use-context ci-cluster

4.  **Create Namespace x1, y1, z1**

> kubectl create ns x1
>
> kubectl create ns y1
>
> kubectl create ns z1
>
![A screenshot of a cell phone Description automatically generated](./media/image2.png)

5.  **Check if namespace is created**

> kubectl ge ns
>
![A screen shot of a social media post Description automatically generated](./media/image3.png)

6.  **Create Deployments & Services**

> Download the yaml file or copy contents to a local file Eg. micro.yaml. This file declares the K8 resources required . Create the necessary
> resources in their respective namespaces
>
[https://github.com/riazvm/nsxtk8smicrosegmentation/blob/master/yaml/micro.yaml](https://github.com/riazvm/nsxtk8smicrosegmentation/blob/master/yaml/micro.yaml)
>
> kubectl apply -f micro.yaml
>
> ![A picture containing bird Description automatically generated](./media/image4.png)

7.  **Check services and pods created in each namespace**

> kubectl get all -n x1
>
![A screenshot of a cell phone Description automatically generated](./media/image5.png)
>
> kubectl get all -n y1
>
> ![A screenshot of a cell phone Description automatically generated](./media/image6.png)
>
> kubectl get all -n z1
>
![A screenshot of a cell phone Description automatically generated](./media/image7.png)

8.  **Expose service-a as a load-balancer service**

> kubectl expose deployment service-a --name=service-a-lb --port=80 --target-port=8080 --type=LoadBalancer --namespace=x1

![](./media/image8.png)

9.  **Check the external URL/IP address assigned to the service (make note of the first IP addres under External-IP).**

> kubectl get svc -n x1
>
![](./media/image9.png)

Current Traffic Flow
--------------------

**Traffic flow from external network to k8 cluster**.

Open a browser and browse to the external ip from the previous step

![A screenshot of a cell phone Description automatically generated](./media/image10.png)

service-a can be reached from the external network through the loadbalancer which is created in NSX-T

service-b, service-c and service-d are not exposed and hence are not reachable from the external network.

**Traffic flow between pods**

We will be using the busy-box container within service-a pod and use
curl to check for responses between service.

Source Service -- service-a (namespace x1)

Target Service -- service-b (namespace y1)

1.  Get pods running on namespace x1

> kubectl get po -n x1

> ![A screenshot of a cell phone Description automatically generated](./media/image11.png)

2.  Get service name for service-b running on namespace y1

> kubectl get svc -n y1\
> \
![](./media/image12.png)

3.  Exec into the busybox container running on a service-a pod

> kubectl exec -n \<namespace\> \<podname\> -it -c busybox \-- /bin/sh
>
> Eg
>
> kubectl exec -n x1 service-a-84965f57cc-nhbrx -it -c busybox \--
> /bin/sh
>
![](./media/image13.png)

4.  Use curl to reach service-b in namespace y1

> curl <http://svc-service-b.y1>
>
NOTE: the service name is prepended by svc in this case, this is defined in the yaml we used to create the K8 resources. The service
name is also appended by the namespace.

5.  This results in a successful response

![A screenshot of a cell phone Description automatically generated](./media/image14.png)

6.  Use the curl command to check the service response of service-d in
    namespace z1

> curl <http://svc-service-d.z1>
>
This should be successful as well

![A screenshot of a cell phone Description automatically generated](./media/image15.png)

At this point all services should be able to communicate to each other. The only traffic allowed from outside the cluster is to service-a via
the loadbalancer.

Configure Micro Segmentation with NSX-T
=======================================

This section goes through the configuration of NSX-T DFW rules that are required to allow, restrict or drop pod to pod traffic.

1.  **Check labels on the pods**

> kubectl get pod \--show-labels -n x1

Check the labels on the pods in namespace y1 and z1 as well. All tags
are defined as app={service name}. e.g.: app=service-a

![A screenshot of a cell phone Description automatically generated](./media/image16.png)

2.  **Check tags for pods in NSXT**

Login to NSX-T Manager and navigate to Advanced Networking & Security > Switching > Ports
The pods service-a, service-b, service-c and service-d would have a logical port assigned to them.


The pods service-a , service-b, service-c and service-d would have a logical port assigned to them

![A screenshot of a cell phone Description automatically generated](./media/image17.png)

![A screenshot of a cell phone Description automatically generated]

Click on one of the Logical ports associated with the services we are working with and notice that the tags on NSXT is the labels defined
for the pods in K8s.

![A screenshot of a social media post Description automatically generated](./media/image19.png)

3.  **Create NSX-T NSGroup (NSX-T Security Group)**

Login to NSXT and Navigate to Advanced Networking & Security Inventory Groups

![A screenshot of a social media post Description automatically generated](./media/image20.png)

***Create a new NSGroup by clicking on ADD***

Create an NSGroup for service-a

NSGroup name -- nsg-service-a

![A screenshot of a cell phone Description automatically generated](./media/image21.png)

Click on [Membership Critera]{.underline}

And add the following criteria

> Logical Port > Tag > Equals > service-a > Scope > equals > app

Note these are the values of the tag retrieved in step 2.

For example, if you defined a Kubernetes tag of "app=service-a" then "service-a" should be used for "Tag" value for the Logical Port and
"app" should be used for the "scope".

If you want to further refine the membership criteria to include only pods in a namespace, you can use the namespace name as the tag and
scope should be defined as "ncp/project". So for this service, the tag will be "x1" and scope "ncp/project".

You can use multiple tags to define your membership criteria.

![A screenshot of a cell phone Description automatically generated](./media/image22.png)

Repeat the same to create NSGroups for service-b, service-c and service-d

![A screenshot of a social media post Description automaticall generated](./media/image23.png)

4.  **Check for pool members**

Click on the newly created groups and select Members, check IP addresses

![A screenshot of a social media post Description automatically generated](./media/image24.png)

Run the following kubectl command from the terminal to verify that the ip-address match the pods ip addresses

> kubectl get po -n x1 -o wide

![](./media/image25.png)

Repeat the same for the other NSGroups created

5.  **Create DFW rules**

Login to NSXT and navigate to Advanced Networking & Security Security > Distributed Firewall

![A screenshot of a computer Description automatically generated](./media/image26.png)

Select the first section listed and navigate to ADD SECTION Add Section Above

![A screenshot of a cell phone Description automatically generated](./media/image27.png)

Section Name -- PtoPDFW-MicroSegmentation<br>
Object Type -- NSGroup <br>
Select -- nsg-service-b, nsg-service-c and nsg-service-d

![A screenshot of a cell phone Description automatically generated](./media/image28.png)

Select the newly created PtoPDFW-MicroSegmentation segment and click on Add rule

***Rule 5:  Denies all Traffic to nsg-service-a, nsg-service-b , nsg-service-c and nsg-service-d***

> ID : DenyAll <br>
> Source : Any<br>
> Destination: Any<br>
> Service: Any<br>
> ContextProfile: Any<br>
> Action : Reject<br>

***Rule 4: Allow service-a to service-b traffic***

> ID : service-aToservice-b<br>
> Source : nsg-service-a<br>
> Destination: nsg-service-b<br>
> Service: Any<br>
> ContextProfile: Any<br>
> Action : Allow<br>
>
To select the Source click on the icon next to the source filed and click on Edit Rule Source

![A screenshot of a social media post Description automatically generated](./media/image29.png)

Select Object Type as NSGroup

![A screenshot of a cell phone Description automatically generated](./media/image30.png)

Similarly select nsg-service-b for destination.

![A screenshot of a cell phone screen with text Description automatically generated](./media/image31.png)

***Rule 3: Allow service-a to service-c traffic***

>ID : service-aToservice-c<br>
> Source : nsg-service-a<br>
> Destination: nsg-service-c<br>
> Service: Any<br>
> ContextProfile: Any<br>
> Action : Allow<br>

***Rule 2: Allow service-b to service-c traffic***

> ID : service-bToservice-c<br>
> Source : nsg-service-b<br>
> Destination: nsg-service-c<br>
> Service: Any<br>
> ContextProfile: Any<br>
> Action : Allow<br>

***Rule 1: Allow service-c to service-d traffic***

> ID : service-cToservice-d<br>
> Source : nsg-service-c<br>
> Destination: nsg-service-d<br>
> Service: Any<br>
> ContextProfile: Any<br>
> Action : Allow<br>

![A screenshot of a computer Description automatically generated](./media/image32.png)
 
 Select PtoPDFW-Microsegmentation and click on Publish

Test Traffic Flow
=================

**Traffic flow from external network to k8 cluster**.

Open a browser and browse to the external ip of the load balancer for service-a (a previous step describes  this)

![A screenshot of a cell phone Description automatically generated](./media/image10.png)

service-a can be reached from the external network through the load
balancer which is created in NSX-T

service-b, service-c and service-d are not exposed and hence are not
reachable from the external network.

**Traffic flow between pods**

We will be using the busy-box container within service-a pod and use
curl to check for responses between service.

 ----------------------------------------------------------
|  **Source Pod**    | **Destination Pod**   | **Result**  | 
| ------------------ | --------------------- | ----------- | 
| service-a          | service-b             | Success     | 
| service-a          | service-c             | Success     | 
| service-a          | service-d             | Fail        | 
| service-b          | service-c            | Success      | 
| service-b          | service-d             | Fail        | 
| service-d          | service-d             | Success     | 
| service-d          | service-b             | Fail        | 
| service-d          | service-c             | Fail         | 
| service-d          | service-b             | Fail        | 
| service-c          | service-a             | Fail        | 
| service-b          | service-a             | Fail        | 


1.  Get pods running on namespace x1

> kubectl get po -n x1

![A screenshot of a cell phone Description automatically generated](./media/image11.png)

2.  Get service name for service-b running on namespace y1

> kubectl get svc -n y1

![](./media/image12.png)

3.  Exec into the busybox container running on a service-a pod

> kubectl exec -n \<namespace\> \<podname\> -it -c busybox \-- /bin/sh
>
> Eg
>
> kubectl exec -n x1 service-a-84965f57cc-nhbrx -it -c busybox \--
> /bin/sh
>
![](./media/image13.png)

Use curl to reach service-b in namespace y1

> curl <http://svc-service-b.y1>

NOTE: the service name is prepended by svc in this case, this is defined in the yaml we used to create the K8 resources. The service
name is also appended by the namespace.

4.  This results in a successful response

![A screenshot of a cell phone Description automatically generated](./media/image14.png)

5.  Use the curl command to check the service response of service-c in namespace z1

> curl <http://svc-service-c.z1>

This should be successful as well

6.  Use the curl command to check the service response of service-d in namespace z1

> curl <http://svc-service-d.z1>

This would fail

![](./media/image33.png)

Exec into each pod and check connectivity between pods.
