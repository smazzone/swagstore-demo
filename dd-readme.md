helm repo add datadog https://helm.datadoghq.com
helm repo update

Let us create a namespace for deploying our operator

kubectl create ns observe

<!-- 
Let us switch namespace
kubectl config set-context --current --namespace=observe 
-->

helm install my-datadog-operator datadog/datadog-operator -n observe


Let us make sure to replace <DATADOG_API_KEY> and <DATADOG_APP_KEY> with your Datadog API and application keys.

kubectl create secret generic datadog-secret --from-literal api-key=<DATADOG_API_KEY> --from-literal app-key=<DATADOG_APP_KEY> -n observe

if you ever need to visualize your keys again:

kubectl get secret datadog-secret -n observe -o jsonpath='{.data.api-key}' | base64 --decode

kubectl get secret datadog-secret -n observe -o jsonpath='{.data.app-key}' | base64 --decode


NOTE:
Single Step Instrumentation does NOT instrument applications in the namespace where you install the Datadog Agent. It's recommended to install the Agent in a separate namespace in your cluster where you don't run your applications.


Create datadog-agent.yaml with the spec of your Datadog Agent deployment configuration. The simplest configuration is as follows:

```
apiVersion: datadoghq.com/v2alpha1
kind: DatadogAgent
metadata:
  name: datadog
spec:
  global:
    clusterName: docker-desktop
    kubelet:
      tlsVerify: false
    tags:
      - env:ssi-dev
    credentials:
      apiSecret:
        secretName: datadog-secret
        keyName: api-key
      appSecret:
        secretName: datadog-secret
        keyName: app-key
  override:
    clusterAgent:
      image:
        #we use this version of the agent to make sure we could use the new SSI features
        tag: 7.64.1
    nodeAgent:
      image:
        tag: 7.64.1
      containers:
        trace-agent:
          env:
            - name: DD_APM_COMPUTE_STATS_BY_SPAN_KIND
              value: "true"
            - name: DD_APM_PEER_TAGS_AGGREGATION
              value: "true"
        agent:
          env:
            - name: "DD_IGNORE_AUTOCONF"
              value: "kube_controller_manager kube_scheduler etcd"
  features:
    #comment out these the following lines if you want to swtich back and use admission controller instead of SSI
    # admissionController:
    #   enabled: true
    #   mutateUnlabelled: false
    apm:
      enabled: true
      #comment out these lines if you want admission controller to use hostip instead of UDS
      hostPortConfig:
        enabled: true
      unixDomainSocketConfig:
        enabled: false
      #comment out these lines if you want to use SSI instead of admission controller
      instrumentation:
        enabled: true
        targets:
          # to target a namespace
          # - name: "my target namespace"
          #   namespaceSelector:
          #     matchNames:
          #       - "default"
          #   ddTraceVersions:
          #     java: "1.47.3"
          #     dotnet: "3.13.0"
          #     python: "3.3.0"
          #     js: "5.45.0"
          # to target some specific pods
          - name: "my-target"
            podSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                  - adservice
                  - cartservice
                  - checkoutservice
                  - currencyservice
                  - emailservice
                  - paymentservice
                  - recommendationservice
                  - shippingservice
            ddTraceVersions:
              java: "1.47.3"
              dotnet: "3.13.0"
              python: "3.3.0"
              js: "5.45.0"
    logCollection:
      enabled: true
      containerCollectAll: true
    remoteConfiguration:
      enabled: true
    # clusterChecks:
    #   enabled: true
    #   useClusterChecksRunners: true
    #comment out these lines if you want to configure dogstatsd to use UDP instead of UDS
    dogstatsd:
      hostPortConfig:
        enabled: true
      unixDomainSocketConfig:
        enabled: false
    eventCollection:
      collectKubernetesEvents: true
    liveContainerCollection:
      enabled: true
    liveProcessCollection:
      enabled: true
    npm:
      collectDNSStats: true
      enableConntrack: true
      enabled: true
    otlp:
      receiver:
        protocols:
          grpc:
            enabled: true
            endpoint: 0.0.0.0:4317
          http:
            enabled: true
            endpoint: 0.0.0.0:4318
```

Let us deploy the cluster agent 

kubectl apply -f datadog-agent.yaml -n observe

After waiting a few minutes for the Datadog Cluster Agent changes to apply, (restart your applications if they are already running)

You can check your pods are up & running

kubectl get pods -n observe

We can now deploy the demo app in the default namespace

skaffold run



