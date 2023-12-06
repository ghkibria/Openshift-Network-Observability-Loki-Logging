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

[network observability](https://docs.openshift.com/container-platform/4.14/network_observability/installing-operators.html#network-observability-without-loki_network_observability)
[flow collector](https://docs.openshift.com/container-platform/4.14/network_observability/configuring-operator.html#network-observability-flowcollector-view_network_observability)

[network operator](https://docs.openshift.com/container-platform/4.14/network_observability/understanding-network-observability-operator.html)

Node details to be used for this activity

Single node SNO without Loki
```bash
oc get nodes

NAME                                                   STATUS   ROLES                         AGE   VERSION

master.sno-demo.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   22d   v1.27.6+f67aeb3

```

Compact cluster with Loki seup
```bash
NAME                                                STATUS   ROLES                         AGE    VERSION

master-0.mno.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   7d1h   v1.27.6+f67aeb3
master-1.mno.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   7d     v1.27.6+f67aeb3
master-2.mno.hubcluster-1.lab.eng.cert.redhat.com   Ready    control-plane,master,worker   7d     v1.27.6+f67aeb3
```
Both cluster is running on OCP 4-14

NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.14.1    True        False         22d     Cluster version is 4.14.1

In my setup I installed Loki on a SNO where the node has ODF, connected to ceph storage. 
Steps to build Loki, we can do it from both OCP console or CLI
1. Install ODF operator
2. Connect to CEPH storage
3. Install Loki operator
4. Create an ObjectBucketClaim in netobserv namespace
   
loki-bucket-odf.yaml

apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: loki-bucket-odf
  namespace: netobserv
spec:
  generateBucketName: loki-bucket-odf
  storageClassName: openshift-storage.noobaa.io 

oc -n netobserv get obc
NAME              STORAGE-CLASS                 PHASE   AGE
loki-bucket-odf   openshift-storage.noobaa.io   Bound   8d


5. Create an Object Storage secret with keys as follows
   
Get bucket properties from the associated ConfigMap

BUCKET_HOST=$(oc get -n netobserv configmap loki-bucket-odf -o jsonpath='{.data.BUCKET_HOST}')
BUCKET_NAME=$(oc get -n netobserv configmap loki-bucket-odf -o jsonpath='{.data.BUCKET_NAME}')
BUCKET_PORT=$(oc get -n netobserv configmap loki-bucket-odf -o jsonpath='{.data.BUCKET_PORT}')

Get bucket access key from the associated Secret

ACCESS_KEY_ID=$(oc get -n netobserv secret loki-bucket-odf -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
SECRET_ACCESS_KEY=$(oc get -n netobserv secret loki-bucket-odf -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)

Start create lokstack secret
oc create -n netobserv secret generic lokistack-dev-odf --from-literal=access_key_id="${ACCESS_KEY_ID}" --from-literal=access_key_secret="${SECRET_ACCESS_KEY}" --from-literal=bucketnames="${BUCKET_NAME}"  --from-literal=endpoint="https://${BUCKET_HOST}:${BUCKET_PORT}"

secret.sh

BUCKET_HOST=$(oc get -n netobserv configmap loki-bucket-odf -o jsonpath='{.data.BUCKET_HOST}')
BUCKET_NAME=$(oc get -n netobserv configmap loki-bucket-odf -o jsonpath='{.data.BUCKET_NAME}')
BUCKET_PORT=$(oc get -n netobserv configmap loki-bucket-odf -o jsonpath='{.data.BUCKET_PORT}')

ACCESS_KEY_ID=$(oc get -n netobserv secret loki-bucket-odf -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
SECRET_ACCESS_KEY=$(oc get -n netobserv secret loki-bucket-odf -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)

oc create -n netobserv secret generic lokistack-dev-odf --from-literal=access_key_id="${ACCESS_KEY_ID}" --from-literal=access_key_secret="${SECRET_ACCESS_KEY}" --from-literal=bucketnames="${BUCKET_NAME}"  --from-literal=endpoint="https://${BUCKET_HOST}:${BUCKET_PORT}"

Run the file
bash secret.sh

oc -n netobserv get secret
NAME                                TYPE                                  DATA   AGE
loki-bucket-odf                     Opaque                                2      8d

6. Create an instance of LokiStack by referencing the secret name and type as s3

loki-stack-cr.yaml

apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: loki
  namespace: netobserv
spec:
  size: 1x.small
  storage:
    secret:
      name: lokistack-dev-odf
      type: s3
    tls:
      caName: openshift-service-ca.crt
  storageClassName: ocs-external-storagecluster-cephfs
  tenants:
    mode: openshift-network
    
oc -n netobserv get po

NAME                                   READY   STATUS    RESTARTS   AGE
flowlogs-pipeline-qphfs                1/1     Running   0          8d

loki-compactor-0                       1/1     Running   0          8d
loki-distributor-98b4f77d5-qhb45       1/1     Running   0          8d
loki-distributor-98b4f77d5-t48vq       1/1     Running   0          8d
loki-gateway-77f956978-6n9q7           2/2     Running   0          8d
loki-gateway-77f956978-lzzj4           2/2     Running   0          8d
loki-index-gateway-0                   1/1     Running   0          8d
loki-index-gateway-1                   1/1     Running   0          8d
loki-ingester-0                        1/1     Running   0          8d
loki-ingester-1                        1/1     Running   0          8d
loki-querier-578cfd6787-gx2rc          1/1     Running   0          8d
loki-querier-578cfd6787-k92nq          1/1     Running   0          8d
loki-query-frontend-84bd49b6db-762hj   1/1     Running   0          8d
loki-query-frontend-84bd49b6db-f2cdj   1/1     Running   0          8d
netobserv-plugin-5cdb5649bd-kxq62      1/1     Running   0          8d

7. Install the role file in netobserv namespace
   
ClusterRole reader yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: netobserv-reader    
rules:
- apiGroups:
  - 'loki.grafana.com'
  resources:
  - network
  resourceNames:
  - logs
  verbs:
  - 'get'

ClusterRole writer yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: netobserv-writer
rules:
- apiGroups:
  - 'loki.grafana.com'
  resources:
  - network
  resourceNames:
  - logs
  verbs:
  - 'create'

ClusterRoleBinding yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: netobserv-writer-flp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: netobserv-writer
subjects:
- kind: ServiceAccount
  name: flowlogs-pipeline    
  namespace: netobserv
- kind: ServiceAccount
  name: flowlogs-pipeline-transformer
  namespace: netobserv

8. Install the the Network Observability Operator from OCP console or CLI

oc get pods -n netobserv
NAME                                   READY   STATUS    RESTARTS   AGE
flowlogs-pipeline-qphfs                1/1     Running   0          8d
loki-compactor-0                       1/1     Running   0          8d
loki-distributor-98b4f77d5-qhb45       1/1     Running   0          8d
loki-distributor-98b4f77d5-t48vq       1/1     Running   0          8d
loki-gateway-77f956978-6n9q7           2/2     Running   0          8d
loki-gateway-77f956978-lzzj4           2/2     Running   0          8d
loki-index-gateway-0                   1/1     Running   0          8d
loki-index-gateway-1                   1/1     Running   0          8d
loki-ingester-0                        1/1     Running   0          8d
loki-ingester-1                        1/1     Running   0          8d
loki-querier-578cfd6787-gx2rc          1/1     Running   0          8d
loki-querier-578cfd6787-k92nq          1/1     Running   0          8d
loki-query-frontend-84bd49b6db-762hj   1/1     Running   0          8d
loki-query-frontend-84bd49b6db-f2cdj   1/1     Running   0          8d
netobserv-plugin-5cdb5649bd-kxq62      1/1     Running   0          8d


10. Create the flow collecter from GUI or CLI, example from cli
    
flow_collector.yaml

apiVersion: flows.netobserv.io/v1beta1
kind: FlowCollector
metadata:
  name: cluster
spec:
  namespace: netobserv
  deploymentModel: DIRECT
  agent:
    type: EBPF                                
    ebpf:
      sampling: 50                            
      logLevel: info
      privileged: true  # set it to true to monitor sriov interface , default is false
      resources:
        requests:
          memory: 50Mi
          cpu: 100m
        limits:
          memory: 800Mi
  processor:
    logLevel: info
    resources:
      requests:
        memory: 100Mi
        cpu: 100m
      limits:
        memory: 800Mi
    conversationEndTimeout: 10s
    logTypes: FLOWS                            
    conversationHeartbeatInterval: 30s
  loki:                                       
    url: 'https://loki-gateway-http.netobserv.svc:8080/api/logs/v1/network'
    statusUrl: 'https://loki-query-frontend-http.netobserv.svc:3100/'
    authToken: FORWARD
    tls:
      enable: true
      caCert:
        type: configmap
        name: loki-gateway-ca-bundle
        certFile: service-ca.crt
        namespace: netobserv  
  consolePlugin:
    register: true
    logLevel: info
    portNaming:
      enable: true
      portNames:
        "3100": loki
    quickFilters:                             
    - name: Applications
      filter:
        src_namespace!: 'openshift-,netobserv'
        dst_namespace!: 'openshift-,netobserv'
      default: true
    - name: Infrastructure
      filter:
        src_namespace: 'openshift-,netobserv'
        dst_namespace: 'openshift-,netobserv'
    - name: Pods network
      filter:
        src_kind: 'Pod'
        dst_kind: 'Pod'
      default: true
    - name: Services network
      filter:
        dst_kind: 'Service'

oc get flowcollector/cluster

NAME      AGENT   SAMPLING (EBPF)   DEPLOYMENT MODEL   STATUS
cluster   EBPF    50                DIRECT             Ready


oc get pods -n netobserv-privileged

NAME                         READY   STATUS    RESTARTS      AGE
netobserv-ebpf-agent-rxmqt   1/1     Running   2 (94m ago)   8d

Since we are using a Loki Operator, check the status of pods running in the openshift-operators-redhat namespace by entering the following command:

NAME                                                READY   STATUS    RESTARTS   AGE
loki-operator-controller-manager-5b8f8fc585-vlzzr   2/2     Running   0          8d

All pods status

oc get pods -n netobserv
NAME                                   READY   STATUS    RESTARTS   AGE
flowlogs-pipeline-qphfs                1/1     Running   0          8d
loki-compactor-0                       1/1     Running   0          8d
loki-distributor-98b4f77d5-qhb45       1/1     Running   0          8d
loki-distributor-98b4f77d5-t48vq       1/1     Running   0          8d
loki-gateway-77f956978-6n9q7           2/2     Running   0          8d
loki-gateway-77f956978-lzzj4           2/2     Running   0          8d
loki-index-gateway-0                   1/1     Running   0          8d
loki-index-gateway-1                   1/1     Running   0          8d
loki-ingester-0                        1/1     Running   0          8d
loki-ingester-1                        1/1     Running   0          8d
loki-querier-578cfd6787-gx2rc          1/1     Running   0          8d
loki-querier-578cfd6787-k92nq          1/1     Running   0          8d
loki-query-frontend-84bd49b6db-762hj   1/1     Running   0          8d
loki-query-frontend-84bd49b6db-f2cdj   1/1     Running   0          8d
netobserv-plugin-5cdb5649bd-kxq62      1/1     Running   0          8d




