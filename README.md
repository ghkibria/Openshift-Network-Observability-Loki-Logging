# Openshift-Network-Observability-Loki-Logging
Verify Network-Observability operator with Loki-Logging

Purpose
The purpose of this activity is to verify the networkobserv operator fuctionality with Loki or without Loki setup.

Prerequisites
SNO or compact cluster running with
- Loki/ODF
- no Loki setup
  
Ensure you have access to the cluster as a user with the cluster-admin role.

Helpful RH docs links

https://docs.openshift.com/container-platform/4.14/network_observability/installing-operators.html#network-observability-without-loki_network_observability
https://docs.openshift.com/container-platform/4.14/network_observability/configuring-operator.html#network-observability-flowcollector-view_network_observability
https://docs.openshift.com/container-platform/4.14/network_observability/understanding-network-observability-operator.html

Node details to be used for this activity

Single node SNO without Loki

oc get nodes
NAME                                                   STATUS   ROLES                         AGE   VERSION
master.sno-demo.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   22d   v1.27.6+f67aeb3

Compact cluster with Loki seup

NAME                                                STATUS   ROLES                         AGE    VERSION
master-0.mno.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   7d1h   v1.27.6+f67aeb3
master-1.mno.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   7d     v1.27.6+f67aeb3
master-2.mno.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   7d     v1.27.6+f67aeb3

Both cluster is running on OCP 4-14
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.14.1    True        False         22d     Cluster version is 4.14.1

