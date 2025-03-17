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


    4. Wait for the pods to be running

        ```$bash
        oc -n openshift-cluster-observability-operator get pods

        NAME                                                    	READY   STATUS	RESTARTS   AGE
        obo-prometheus-operator-6b75c99f74-xz4nf                	1/1 	Running   0      	47s
        obo-prometheus-operator-admission-webhook-c55d86748-ldsvs   1/1 	Running   0      	47s
        obo-prometheus-operator-admission-webhook-c55d86748-v999m   1/1 	Running   0      	47s
        observability-operator-6b8cb6987-g2249                  	1/1 	Running   0      	47s
        ```
        
2. Create and configure MonitoringStack and ServiceMonitor

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

        1. Prometheus

            ```$bash
            oc create route edge prometheus-coo --service=federate-cmo-ms-prometheus --port=9090 -n federate-cmo
            ```
   
        2. AlertManager

            ```$bash
            oc create route edge alertmanager-coo --service=federate-cmo-ms-alertmanager --port=9093 -n federate-cmo
            ```

    6. Create ServiceMonitor
   
        The ServiceMonitor contains the configuration to enable scraping the federation endpoint of the platform Prometheus instances.

        ```$bash
        cat <<EOF | oc apply -f -
        apiVersion: monitoring.rhobs/v1
        kind: ServiceMonitor
        metadata:
          name: federate-cmo-smon
          namespace: federate-cmo
          labels:
            kubernetes.io/part-of: federate-cmo-ms
            monitoring.rhobs/stack: federate-cmo-ms
        spec:
          selector: #use the prometheus service to create a "dummy" target.
            matchLabels:
              app.kubernetes.io/managed-by: observability-operator
              app.kubernetes.io/name: federate-cmo-ms-prometheus
          endpoints:
          - params:
              'match[]': #scrape only required metrics from in-cluster prometheus
                - '{__name__=~".+"}'
                - '{__name__=~"^job:.*"}'
                - '{job="prometheus"}'
                - '{job="node"}'
                - '{__name__="server_labels"}'
                #- '{__name__=~"container_cpu_.*", namespace="federate-cmo"}'
                #- '{__name__="container_memory_working_set_bytes", namespace="federate-cmo"}'
    
            relabelings:
            # relabel example
            - targetLabel: source
              replacement: my-openshift-cluster
    
            # override the target's address by the prometheus-k8s service name.
            - action: replace
              targetLabel: __address__
              replacement: prometheus-k8s.openshift-monitoring.svc:9091
    
            #remove the default target labels as they aren't relevant in case of federation.
            - action: labeldrop
              regex: pod|namespace|service|endpoint|container
        
            # 30s interval creates 4 scrapes per minute
            #    prometheus-k8s.svc x 2 ms-prometheus x (60s/ 30s) = 4
            interval: 30s
        
            #ensure that the scraped labels are preferred over target's labels.
            honorLabels: true
        
            port: web
            scheme: https
            path: "/federate"
        
            bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
        
            tlsConfig:
              serverName: prometheus-k8s.openshift-monitoring.svc
              ca:
                configMap: #automatically created by serving-ca operator
                  key: service-ca.crt
                  name: openshift-service-ca.crt
        EOF
        ```
        Setting a very low scrape interval can lead to performance and resource-related challenges, including increased CPU/memory usage, network overhead, load on monitored targets, and management complexity. In most environments, maintaining a balance between data freshness and resource consumption is essential, and setting these values to reasonable intervals (such as 30 to 60 seconds) is a good practice to achieve efficient and scalable monitoring.

    8. Validation

        Access the COO Prometheus UI and check that the endpoint https://prometheus-k8s.openshift-monitoring.svc:9091/federate state is UP.
       
        ![Alt text](./pics/prometheus_target.jpg?raw=true "Prometheus ") 

        - Samples of promql queries:
            - cluster:node_cpu:ratio_rate5m{cluster=""}
       
                ![Alt text](./pics/prometheus_sample1.jpg?raw=true "Prometheus ") 

            - sum(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{cluster="", namespace="openshift-multus"}) by (pod)
       
                ![Alt text](./pics/prometheus_sample2.jpg?raw=true "Prometheus ") 

3. Create a PrometheusRule defining the required rules

    1. Create a PrometheusRule on the federate-cmo namespace

        The PrometheusRules constants the rules to trigger the alerts.
        I used as a reference the PrometheusRules from the platform.
        Make sure to configure the yaml manifest with the API of the COO (i.e. prometheusrules.monitoring.rhobs).

        ```$bash
        git clone https://github.com/CSA-RH/rosahcp_platform-alerts_visibility.git
        oc create -f rosahcp_platform-alerts_visibility/files/all_prometheusrules.yaml
        ```

    2. Validate

        Check in Prometheus UI that the alerts rules have been created
        Alerts configured in PrometheusRules are loaded in Prometheus UI
       
        ![Alt text](./pics/prometheus_rules.jpg?raw=true "Prometheus ") 

4. Configure AlertManager

    In order to create the alerts and send them to an external Alarms server, the alert manager has to be configured.

    1. Configure AlertManager requests and limits under spec.

        ```$bash
        oc -n federate-cmo edit alertmanagers.monitoring.rhobs federate-cmo-ms
    
        ...
        ...
        spec:
          …
          …
          resources:
            limits:
              cpu: 500m 
              memory: 1Gi
            requests:
              cpu: 200m
              memory: 500Mi
        ```

    2. Create a AlertManager receiver to send the Alerts to a Alert Server:
       
        To simulate an Alert Server we will use the service provided in https://webhook.site.

        1. Setup the AlertServer
           
            To simulate the alert server were the alerts will be sent, we used the site https://webhook.site and sent HTTP Posts to the Webhook configured using the secret  alertmanager-<namespace_monitoring-stack>-ms”.
            My Webhook link was, (note: yours webhook URL must be different!: https://webhook.site/767c84e8-4707-4a85-bd2a-07174d2ad948

        3. Create namespace
           
            NOTE: The customized secret name should be: alertmanager-<namespace_monitoring-stack>-ms
           
            For my test the secret name is “alertmanager-federate-cmo-ms”

            ```$bash
            cat <<EOF | oc apply -f -
            apiVersion: v1
            kind: Secret
            metadata:
              name: alertmanager-federate-cmo-ms
              namespace: federate-cmo
            stringData:
              alertmanager.yaml: |
                receivers:
                  - name: Default
                    webhook_configs:
                      - url: 'https://webhook.site/767c84e8-4707-4a85-bd2a-07174d2ad948'
                  - name: Watchdog
                  - name: Critical
                route:
                  group_by:
                    - namespace
                  group_interval: 5m
                  group_wait: 30s
                  receiver: Default
                  repeat_interval: 12h
                  routes:
                    - matchers:
                      receiver: Default
            EOF
            ```

        5. (Optional): One can generate alarms against the alertmanager
        
            In order to check if the AlertManager is forwarding the alerts towards the alert server, one can create alerts on the AlertManager using the following commands. Anyhow the ROSA cluster must always have a Watchdog alert firing.

            - Connect to one of the alertmanager pods

                ```$bash
                oc -n federate-cmo get pods | grep alert
                alertmanager-federate-cmo-ms-0   2/2 	Running   0      	29m
                alertmanager-federate-cmo-ms-1   2/2 	Running   0      	29m
                ```

            - Get to the shell of one of the alertmanager pods

                ```$bash
                oc -n federate-cmo rsh alertmanager-federate-cmo-ms-0
                sh-4.4$            
                ```

            - Generate a alarm against the alertmanager
            
                ```$bash
                curl -vvv -XPOST http://localhost:9093/api/v2/alerts -H "Content-Type: application/json" -d '[
                  {
                    "status": "firing",
                    "labels": {
                      "alertname": "TestManualAlert",
                      "severity": "critical"
                    },
                    "annotations": {
                      "summary": "Test Manual Alert",
                      "description": "Esta es una alerta enviada manualmente a Alertmanager"
                    }
                  }
                ]'
                ```

                ```$bash
                ---output
                sh-4.4$ curl -vvv -XPOST http://localhost:9093/api/v2/alerts -H "Content-Type: application/json" -d '[
                >   {
                >     "status": "firing",
                >     "labels": {
                >       "alertname": "TestManualAlert",
                >       "severity": "critical"
                >     },
                >     "annotations": {
                >       "summary": "Test Manual Alert",
                >       "description": "Esta es una alerta enviada manualmente a Alertmanager"
                >     }
                >   }
                > ]'
                Note: Unnecessary use of -X or --request, POST is already inferred.
                *   Trying ::1...
                * TCP_NODELAY set
                * Connected to localhost (::1) port 9093 (#0)
                > POST /api/v2/alerts HTTP/1.1
                > Host: localhost:9093
                > User-Agent: curl/7.61.1
                > Accept: */*
                > Content-Type: application/json
                > Content-Length: 267
                    > 
                * upload completely sent off: 267 out of 267 bytes
                < HTTP/1.1 200 OK
                            < Cache-Control: no-store
                < Vary: Origin
                < Date: Thu, 13 Mar 2025 14:20:02 GMT
                < Content-Length: 0
                < 
                * Connection #0 to host localhost left intact
                sh-4.4$
                ```

        6. Validate that the Alert was sent to the alert server
           
            https://webhook.site/#!/view/767c84e8-4707-4a85-bd2a-07174d2ad948/b9876c8d-6ec0-41f5-9ac3-e4583c3a61a1/1

            In my test I see two alerts forwarded by the alertmanager to the alert server. The first two alerts are firing in ROSA, the first is the Watchdog and the second is a memory warning. The third alert is the one created manually, above, in a previous step of this procedure:

            - Alert1:
              
                ![Alt text](./pics/alert1.jpg?raw=true "Alert ") 

            - Alert2:
              
                ![Alt text](./pics/alert2.jpg?raw=true "Alert ") 

            - Alert3:
              
                ![Alt text](./pics/alert3.jpg?raw=true "Alert ") 

## Also recommend checking the following blog:
Step-by-step guide to configuring alerts in Cluster Observability Operator
https://developers.redhat.com/articles/2024/12/16/step-step-guide-configuring-alerts-cluster-observability-operator?source=sso
