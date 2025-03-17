# ROSA HCP Federating Platform Metrics to Custom Monitoring Stack

# Disclaimer
?????This repo contains an opinionated demo and is NOT an official Red Hat documentation.

# Introduction
The objective of this document is to define a procedure to workaround the gap in ROSA HCP, which doesn't allow configuring an alert manager receiver to send platform alerts from the platform monitoring stack, towards a alerts server. 
The RFE (https://issues.redhat.com/browse/XCMSTRAT-976) is opened to allow configuring the monitoring stack alert manager with a receiver.

# Architecture
The objective is to use the MonitoringStack CRD from the Cluster Observability Operator, this CRD will create an additional Monitoring stack (prometheus + alertmanager). The additional stack will scrap the federation endpoint of the ROSA platform Monitoring stack (running in openshift-monitoring namespace). The PrometheusRules object contains the rules that trigger alerts. The COO prometheus evaluates the PrometheusRules and sends the alerts to the COO alertmanager, which forwards the alert to a webhook.

![Alt text](./pics/architecture.jpg?raw=true "Architecture ") 

<code style="color : Gold">Yellow: Cluster Monitoring Operator resources (platform monitoring stack)</code>
<code style="color : Blue">Blue: Cluster Observability Operator resources</code>

# Procedure
The Cluster Observability Operator (COO) is an optional component of the OpenShift Container Platform designed for creating and managing highly customizable monitoring stacks. It enables cluster administrators to automate configuration and management of monitoring needs extensively, offering a more tailored and detailed view of each namespace compared to the default OpenShift Container Platform monitoring system.

1. Install Cluster Observability Operator

    1. Create namespace

    ```$bash
    cat <<EOF | oc apply -f -
    apiVersion: v1
    kind: Namespace
    metadata:
      name: openshift-cluster-observability-operator
    EOF
    ```

    2. Create Operator Group

    ```$bash
    cat <<EOF | oc apply -f -
    apiVersion: operators.coreos.com/v1
    kind: OperatorGroup
    metadata:
      name: openshift-cluster-observability-operator
      namespace: openshift-cluster-observability-operator
    EOF
    ```

    3. Create Subscription

    ```$bash
    cat <<EOF | oc apply -f -
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      labels:
        operators.coreos.com/cluster-observability-operator.openshift-cluster-observability: ""
      name: cluster-observability-operator
      namespace: openshift-cluster-observability-operator
    spec:
      channel: stable
      installPlanApproval: Automatic
      name: cluster-observability-operator
      source: redhat-operators
      sourceNamespace: openshift-marketplace
      startingCSV: cluster-observability-operator.v1.0.0
    EOF
    ```

2. Create and configure COO

    1. Create namespace

    ```$bash
    cat <<EOF | oc apply -f -
    apiVersion: v1
    kind: Namespace
    metadata:
      name: federate-cmo
    EOF
    ```

    2. Grant Permission to Federate In-Cluster Prometheus

    ```$bash
    cat <<EOF | oc apply -f -
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: federate-cmo-ms-view
      labels:
        kubernetes.io/part-of: federate-cmo-ms
        monitoring.rhobs/stack: federate-cmo-ms
    
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-monitoring-view
    subjects:
    - kind: ServiceAccount
      # ServiceAccount used in the prometheus deployed by ObO.
      # SA name follows <monitoring stack name>-prometheus nomenclature
      name: federate-cmo-ms-prometheus
      namespace: federate-cmo
    EOF
    ```

    3. Install COO Monitoring Stack

    ```$bash
    cat <<EOF | oc apply -f -
    apiVersion: monitoring.rhobs/v1alpha1
    kind: MonitoringStack
    metadata:
      name: federate-cmo-ms
      namespace: federate-cmo
    spec:
      #Used to select the ServiceMonitor in the federate-cmo namespace
      #NOTE: there isn't a need for namespaceSelector
      resourceSelector:
        matchLabels:
          monitoring.rhobs/stack: federate-cmo-ms

      logLevel: info #use debug for verbose logs
      retention: 8d #retention of the metrics
    
      alertmanagerConfig:
        disabled: false

      prometheusConfig:

        replicas: 2  #ensures that at least one prometheus is running during upgrade
        #prometheus storage
        persistentVolumeClaim:
          storageClassName: gp3-csi
          resources:
            requests:
              storage: 10Gi

      # prometheus resources: #ensure that you provide sufficient amount of resources
      resources:
         limits:
           cpu: 1
           memory: 2Gi
         requests:
           cpu: 500m
           memory: 1Gi
    EOF
    ```

    4. Wait until all pods are running

    ```$bash
    oc -n federate-cmo get pods

    NAME                         	READY   STATUS	RESTARTS   AGE
    alertmanager-federate-cmo-ms-0   2/2 	Running   0      	91s
    alertmanager-federate-cmo-ms-1   2/2 	Running   0      	91s
    prometheus-federate-cmo-ms-0 	3/3 	Running   0      	92s
    prometheus-federate-cmo-ms-1 	3/3 	Running   0      	92s
    ```

    5. Configure Routes to expose the Prometheus and Alertmanager UI deployed by COO
