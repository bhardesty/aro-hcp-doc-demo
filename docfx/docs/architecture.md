
# Azure Red Hat OpenShift with hosted control planes architecture

With Microsoft Azure Red Hat OpenShift with hosted control planes, the control plane is hosted in a Red Hat account and the worker nodes are deployed in the customer’s Azure account. The Azure Red Hat OpenShift with hosted control planes service hosts a highly-available, single-tenant OpenShift control plane. 

The worker nodes are deployed in your Azure account and run on your virtual network. You can add additional private subnets from one or more availability zones to ensure high availability. Worker nodes are shared by OpenShift components and applications. OpenShift components such as the ingress controller, image registry, and monitoring are deployed on the worker nodes hosted on your virtual network.
