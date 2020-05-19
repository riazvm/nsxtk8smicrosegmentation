VMware Advanced Customer Engagements (ACE) Team

*** Authored by Riaz Mohamed and Raghu Pemmaraju ***
May 2020

Table of Contents {#table-of-contents .TOC-Heading}
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

Containers provide a great deal of benefits to your development pipeline
and leverage resource isolation but they're not designed with strong
security boundaries or workload isolation. Tanzu Kubernetes Grid
Integrated (TKGI) or Enterprise PKS (PKS) integrates with VMware NSX-T
Data Center (NSX-T) and built on top of VMware vSphere to provide an
agile software defined infrastructure to build cloud-native
applications. NSX-T primarily focuses on networking and security for
these applications and used to create stronger workload isolation.

In TKGI, workload isolation can be performed by physically isolating
workload at the vSphere level or by logically isolating workloads within
the TKGI deployed clusters (at cluster or namespace isolation). The
following design patterns can be considered for workload isolation.

In this document, we focus on how you can leverage the
micro-segmentation feature within NSX-T to provide workload isolation.
NSX-T comes with a distributed firewall that can provide complete
control of both North-South Traffic but also East-West Traffic and can
isolate workloads, even if they are next to each other. For example,
traditional firewalls only isolate network traffic between network VLANs
or segments but not within a network segment. But with NSX-T distributed
firewall, you can create rules to isolate workload on the same segment
and with Kubernetes tags, you can isolate even POD to POD communication.

In this document, we take a simple application that has several
components or services. These services are required to communicate with
each other in a very defined manner. For example, service-a need to
communicate with service-c and service-b but not with any other service.
Similarly, service-c needs to communicate with service-d but not with
service-a or service-b.

In such a scenario, we look at how you can isolate the workload using
NSX-T and this is all done dynamically as pods are created and
destroyed.

Now, let us assume that all ingress traffic from outside the cluster
comes into service-a. service-a can is running in namespace x1 and is a
typically ingress controller like nginx or contour. service-a routes to
service-b or service-c and service-c might need to talk to database
service like that defined in service-d. This diagram shows which
workload should is allowed and what is not between the services.

![A screenshot of a video game Description automatically generated](./media/image1.png)

Use-case 
========

The k8s cluster has three namespaces (namespace x1, y1 and z1). All
ingress into the cluster is restricted to namespace x1 and pod
service-a.

Namespace y1 runs a pod service-b and namespace z1 runs pods service-c
and service-d.

Traffic flows
-------------

All ingress traffic into the cluster can only be serviced by service-a
on namespace x1

Service-b allows traffic from service-a

Service-b denies traffic from service-c

Service-c allows traffic from service-a and service-b

Service-d allows traffic from only service-c

Service-c denies traffic from service-d

Assumptions
-----------

K8s cluster exists and there is integration with NSX-T

K8s Resources
=============

Pod Definition
--------------

We will be using a pod definition with 2 containers

-   nginx (to service web requests)

-   busybox with curl (container to test service to service traffic)

Deployments And Services
------------------------

service-a, service-b, service-c and service-d are identical and run the
above pod definition on their respective namespaces, they run two
replicas and are exposed as services svc-service-a, svc-service-b,
svc-service-c, svc-service-d

K8s Resource Deployments
========================

This section goes through the steps to set up the K8 environment with
the deployments in their respective namespaces. At the end of the setup
a test will be performed to establish that the traffic flows between the
different services deployed

K8 Deployments
--------------

1.  **Login to k8s cluster**

> pks login -a <pks-api\> -u <pksuser\> -p <pks-password\> -k
>
> Eg.
>
> pks login -a pks.corp.local -u pksadmin -p VMware1! -k

2.  **Get Kubeconfig**

> pks get-kubeconfig \<cluster-name\> -a \<pks-api\> -u \<pksuser\> -p
> \<pks-password\> -k
>
> Eg.
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
> ![A screenshot of a cell phone Description automatically
> generated](./media/image2.png){width="3.577991032370954in"
> height="1.1805555555555556in"}

5.  **Check if namespace is created**

> kubectl ge ns
>
> ![A screen shot of a social media post Description automatically
> generated](./media/image3.png){width="4.055555555555555in"
> height="2.4662160979877514in"}

6.  **Create Deployments & Services**

> Download the yaml file or copy contents to a local file Eg. micro.yaml
> . This file declares the K8 resources required . Create the necessary
> resources in their respective namespaces
>
> [[https://github.com/riazvm/nsxtk8smicrosegmentation/blob/master/yaml/micro.yaml]{.underline}](https://github.com/riazvm/nsxtk8smicrosegmentation/blob/master/yaml/micro.yaml)
>
> kubectl apply -f micro.yaml
>
> ![A picture containing bird Description automatically
> generated](./media/image4.png){width="5.369564741907261in"
> height="2.0in"}

7.  **Check services and pods created in each namespace**

> kubectl get all -n x1
>
> ![A screenshot of a cell phone Description automatically
> generated](./media/image5.png){width="5.965991907261592in"
> height="1.8611111111111112in"}
>
> kubectl get all -n y1
>
> ![A screenshot of a cell phone Description automatically
> generated](./media/image6.png){width="6.1440791776028in"
> height="1.9166666666666667in"}
>
> kubectl get all -n z1
>
> ![A screenshot of a cell phone Description automatically
> generated](./media/image7.png){width="6.625761154855643in"
> height="2.763888888888889in"}

8.  **Expose service-a as a load-balancer service**

> kubectl expose deployment service-a \\
>
> \--name=service-a-lb \--port=80 \--target-port=8080
> \--type=LoadBalancer \--namespace=x1

![](./media/image8.png){width="7.499926727909012in"
height="0.7083333333333334in"}

9.  **Check the external URL/IP address assigned to the service (make
    note of the first IP addres under External-IP).**

> kubectl get svc -n x1
>
> ![](./media/image9.png){width="6.8856211723534555in"
> height="0.6805555555555556in"}

Current Traffic Flow
--------------------

**Traffic flow from external network to k8 cluster**.

Open a browser and browse to the external ip from the previous step

![A screenshot of a cell phone Description automatically
generated](./media/image10.png){width="6.680250437445319in"
height="2.4027777777777777in"}

service-a can be reached from the external network through the load
balancer which is created in NSX-T

service-b, service-c and service-d are not exposed and hence are not
reachable from the external network.

**Traffic flow between pods**

We will be using the busy-box container within service-a pod and use
curl to check for responses between service.

Source Service -- service-a (namespace x1)

Target Service -- service-b (namespace y1)

1.  Get pods running on namespace x1

kubectl get po -n x1

> ![A screenshot of a cell phone Description automatically
> generated](./media/image11.png){width="6.206597769028871in"
> height="0.9027777777777778in"}

2.  Get service name for service-b running on namespace y1

> kubectl get svc -n y1\
> \
> ![](./media/image12.png){width="5.742222222222222in"
> height="0.4722222222222222in"}

3.  Exec into the busybox container running on a service-a pod

> kubectl exec -n \<namespace\> \<podname\> -it -c busybox \-- /bin/sh
>
> Eg
>
> kubectl exec -n x1 service-a-84965f57cc-nhbrx -it -c busybox \--
> /bin/sh
>
> ![](./media/image13.png){width="6.7692311898512685in"
> height="0.4583333333333333in"}

4.  Use curl to reach service-b in namespace y1

> curl <http://svc-service-b.y1>
>
> NOTE: the service name is prepended by svc in this case, this is
> defined in the yaml we used to create the K8 resources. The service
> name is also appended by the namespace.

5.  This results in a successful response

![A screenshot of a cell phone Description automatically
generated](./media/image14.png){width="7.433908573928259in"
height="1.0833333333333333in"}

6.  Use the curl command to check the service response of service-d in
    namespace z1

curl <http://svc-service-d.z1>

This should be successful as well

![A screenshot of a cell phone Description automatically
generated](./media/image15.png){width="7.70576990376203in"
height="2.6944444444444446in"}

At this point all services should be able to communicate to each other.
The only traffic allowed from outside the cluster is to service-a via
the loadbalancer.

Configure Micro Segmentation with NSX-T
=======================================

This section goes through the configuration of NSX-T DFW rules that are
required to allow, restrict or drop pod to pod traffic.

1.  **Check labels on the pods**

> kubectl get pod \--show-labels -n x1

Check the labels on the pods in namespace y1 and z1 as well. All tags
are defined as app={service name}. e.g.: app=service-a

> ![A screenshot of a cell phone Description automatically
> generated](./media/image16.png){width="7.097222222222222in"
> height="2.75in"}

2.  **Check tags for pods in NSXT**

Login to NSXT and Navigate to Advanced Networking & Security Switching
Ports

> The pods service-a , service-b, service-c and service-d would have a
> logical port assigned to them
>
> ![A screenshot of a cell phone Description automatically
> generated](./media/image17.png){width="7.0in"
> height="4.105555555555555in"}

![A screenshot of a cell phone Description automatically
generated](./media/image18.png){width="7.5in"
height="8.284722222222221in"}

> Click on one of the Logical ports associated with the services we are
> working with and notice that the tags on NSXT is the labels defined
> for the pods in K8s.

![A screenshot of a social media post Description automatically
generated](./media/image19.png){width="7.5in"
height="5.759722222222222in"}

3.  **Create NSX-T NSGroup (NSX-T Security Group)**

> Login to NSXT and Navigate to Advanced Networking & Security Inventory
> Groups
>
> ![A screenshot of a social media post Description automatically
> generated](./media/image20.png){width="7.180555555555555in"
> height="4.194444444444445in"}
>
> Create a new NSGroup by clicking on ADD
>
> Create an NSGroup for service-a
>
> NSGroup name -- nsg-service-a
>
> ![A screenshot of a cell phone Description automatically
> generated](./media/image21.png){width="7.5in"
> height="2.5395833333333333in"}
>
> Click on [Membership Critera]{.underline}
>
> And add the following criteria
>
> Logical Port \> Tag \> Equals \> service-a \> Scope \> equals \> app
>
> Note these are the values of the tag retrieved in step 2.
>
> For example, if you defined a Kubernetes tag of "app=service-a" then
> "service-a" should be used for "Tag" value for the Logical Port and
> "app" should be used for the "scope".
>
> If you want to further refine the membership criteria to include only
> pods in a namespace, you can use the namespace name as the tag and
> scope should be defined as "ncp/project". So for this service, the tag
> will be "x1" and scope "ncp/project".
>
> You can use multiple tags to define your membership criteria.
>
> ![A screenshot of a cell phone Description automatically
> generated](./media/image22.png){width="6.666666666666667in"
> height="3.102777777777778in"}

Repeat the same to create NSGroups for service-b, service-c and
service-d

![A screenshot of a social media post Description automatically
generated](./media/image23.png){width="7.5in"
height="2.7743055555555554in"}

4.  **Check for pool members**

> Click on the newly created groups and select Members, check IP
> addresses
>
> ![A screenshot of a social media post Description automatically
> generated](./media/image24.png){width="7.5in"
> height="2.4229166666666666in"}
>
> Run the following kubectl command from the terminal to verify that the
> ip-address match the pods ip addresses
>
> kubectl get po -n x1 -o wide

![](./media/image25.png){width="7.833333333333333in"
height="0.5694444444444444in"}

> Repeat the same for the other NSGroups created

5.  **Create DFW rules**

> Login to NSXT and navigate to Advanced Networking & Security Security
> Distributed Firewall
>
> ![A screenshot of a computer Description automatically
> generated](./media/image26.png){width="7.5in"
> height="3.9604166666666667in"}
>
> Select the first section listed and navigate to ADD SECTION Add
> Section Above
>
> ![A screenshot of a cell phone Description automatically
> generated](./media/image27.png){width="7.5in" height="2.9in"}
>
> Section Name -- PtoPDFW-MicroSegmentation
>
> Object Type -- NSGroup
>
> Select -- nsg-service-b, nsg-service-c and nsg-service-d
>
> ![A screenshot of a cell phone Description automatically
> generated](./media/image28.png){width="7.5in"
> height="7.129861111111111in"}
>
> Select the newly created PtoPDFW-MicroSegmentation segment and click
> on Add rule

Rule 5:  Denies all Traffic to nsg-service-a, nsg-service-b ,
nsg-service-c and nsg-service-d

> ID : DenyAll
>
> Source : Any
>
> Destination: Any
>
> Service: Any
>
> ContextProfile: Any
>
> Action : Reject

Rule 4: Allow service-a to service-b traffic

> ID : service-aToservice-b
>
> Source : nsg-service-a
>
> Destination: nsg-service-b
>
> Service: Any
>
> ContextProfile: Any
>
> Action : Allow
>
> To select the Source click on the icon next to the source filed and
> click on Edit Rule Source
>
> ![A screenshot of a social media post Description automatically
> generated](./media/image29.png){width="7.5in"
> height="1.9458333333333333in"}
>
> Select Object Type as NSGroup
>
> ![A screenshot of a cell phone Description automatically
> generated](./media/image30.png){width="7.5in" height="5.60625in"}
>
> Similarly select nsg-service-b for destination.
>
> ![A screenshot of a cell phone screen with text Description
> automatically generated](./media/image31.png){width="7.5in"
> height="1.6423611111111112in"}

Rule 3: Allow service-a to service-c traffic

> ID : service-aToservice-c
>
> Source : nsg-service-a
>
> Destination: nsg-service-c
>
> Service: Any
>
> ContextProfile: Any
>
> Action : Allow

Rule 2: Allow service-b to service-c traffic

> ID : service-bToservice-c
>
> Source : nsg-service-b
>
> Destination: nsg-service-c
>
> Service: Any
>
> ContextProfile: Any
>
> Action : Allow

Rule 1: Allow service-c to service-d traffic

> ID : service-cToservice-d
>
> Source : nsg-service-c
>
> Destination: nsg-service-d
>
> Service: Any
>
> ContextProfile: Any
>
> Action : Allow

![A screenshot of a computer Description automatically
generated](./media/image32.png){width="7.5in"
height="2.696527777777778in"}

> Select PtoPDFW-Microsegmentation and click on Publish

Test Traffic Flow
=================

**Traffic flow from external network to k8 cluster**.

Open a browser and browse to the external ip from the previous step

![A screenshot of a cell phone Description automatically
generated](./media/image10.png){width="6.680250437445319in"
height="2.4027777777777777in"}

service-a can be reached from the external network through the load
balancer which is created in NSX-T

service-b, service-c and service-d are not exposed and hence are not
reachable from the external network.

**Traffic flow between pods**

We will be using the busy-box container within service-a pod and use
curl to check for responses between service.

  **Source Pod**   **Destination Pod**   **Result**
  ---------------- --------------------- ------------
  service-a        service-b             Success
  service-a        service-c             Success
  service-a        service-d             Fail
  service-b        service-c             Success
  service-b        service-d             Fail
  service-c        service-d             Success
  service-c        service-b             Fail
  service-d        service-c             Fail
  service-d        service-b             Fail
  service-c        service-a             Fail
  service-b        service-a             Fail

1.  Get pods running on namespace x1

kubectl get po -n x1

> ![A screenshot of a cell phone Description automatically
> generated](./media/image11.png){width="6.206597769028871in"
> height="0.9027777777777778in"}

2.  Get service name for service-b running on namespace y1

> kubectl get svc -n y1\
> \
> ![](./media/image12.png){width="5.742222222222222in"
> height="0.4722222222222222in"}

3.  Exec into the busybox container running on a service-a pod

> kubectl exec -n \<namespace\> \<podname\> -it -c busybox \-- /bin/sh
>
> Eg
>
> kubectl exec -n x1 service-a-84965f57cc-nhbrx -it -c busybox \--
> /bin/sh
>
> ![](./media/image13.png){width="6.7692311898512685in"
> height="0.4583333333333333in"}
>
> Use curl to reach service-b in namespace y1
>
> curl <http://svc-service-b.y1>
>
> NOTE: the service name is prepended by svc in this case, this is
> defined in the yaml we used to create the K8 resources. The service
> name is also appended by the namespace.

4.  This results in a successful response

![A screenshot of a cell phone Description automatically
generated](./media/image14.png){width="7.433908573928259in"
height="1.0833333333333333in"}

5.  Use the curl command to check the service response of service-c in
    namespace z1

curl <http://svc-service-c.z1>

This should be successful as well

6.  Use the curl command to check the service response of service-d in
    namespace z1

curl <http://svc-service-d.z1>

> This would fail

![](./media/image33.png){width="7.5in" height="0.4354166666666667in"}

Exec into each pod and check connectivity between pods.
