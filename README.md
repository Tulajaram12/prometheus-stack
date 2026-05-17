# prometheus-stack

 In Values.yaml file these things must be edited 

 See Design must be like below
1) if you have separate Node group for the EKS Cluster in the specific availability zone , the nodes are launched in specific az as the requirement is that we have pv for all the three prometheus, grafana and alertmanager.
2) I have added Below config in values.yaml file for the Prometheus    
    i) First have the storage class created with waitforfirstconsumer and with the policy Retain and change the storageClass Name  
   ii) Decide the Storage with the Client How much GB you want and Change it  
  iii) If you want even you can decide the Resource Limits  


**⚙️ General Configurations for All Pods**  
          
          1) Tolerations  
          2) Resources    
**⚠️ These values must be added in the correct places in values.yaml (do NOT copy-paste blindly)**


🔧 Tolerations

```yaml
tolerations: []
  # Example:
  #- key: "key"
  #  operator: "Equal"
  #  value: "value"
  #  effect: "NoSchedule"
```
💻 Resources

Used to define CPU and memory limits for pods.

```yaml
resources: {}
  #requests:
  #  memory: 400Mi
  #  cpu: 200m
```

Create a Storage Class as Below  

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  fsType: ext4
```



Add the Alert Manager Specification in values.yaml and change the size you want and storageclassname     
```yaml
alertmanagerSpec:
   storage: 
      volumeClaimTemplate:
       spec:
         storageClassName: gp3
         accessModes: ["ReadWriteOnce"]
         resources:
           requests:
             storage: 20Gi

```


 Add the Grafana spec as below  
i) Decide the size  
ii) let the admin user be admin and let the grafana create the password which is stored as secret    

``` yaml
grafana:
   persistence:
      enabled: true
      storageClassName: "gp3"
      accessModes:
        - ReadWriteOnce
      size: 20Gi
```


 Add the Prometheus specification in values.yaml  
 i) Decide the size  

```yaml
 storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi 

```


This is the Slack Based Alert Manager.yaml

1. Create a Slack Channel  
   
   i) Open Slack (web or app)  
   ii) Click “+” next to Channels  
   iii) Choose Create a channel  
   iv) Name it like:  
          alerts-monitoring  
   v) Click Create  

2. Create Slack App (for Webhook)  
   i) Go to: https://api.slack.com/apps  
   ii) Click Create New App  
   iii) Choose From scratch  
   iv)  App Name: Alertmanager  
   v) Pick your workspace → Create App  

3. Enable Incoming Webhooks  
   i) Inside your app, go to Incoming Webhooks  
   ii) Toggle Activate Incoming Webhooks = ON  
   iii) Click Add New Webhook to Workspace  
   iv) Select your channel (alerts-monitoring)  
   v) Click Allow  

4. Get Webhook URL (IMPORTANT)  
      You will get a URL like:  
            https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX  
      This is what you use in alertmanager.yaml  


```yaml
ii) Change the Slack URL here and repeat interval i.e no of hours you want in values.yaml 

  config:
    global:
      resolve_timeout: 5m
    inhibit_rules:
      - source_matchers:
          - 'severity = critical'
        target_matchers:
          - 'severity =~ warning|info'
        equal:
          - 'namespace'
          - 'alertname'
      - source_matchers:
          - 'severity = warning'
        target_matchers:
          - 'severity = info'
        equal:
          - 'namespace'
          - 'alertname'
      - source_matchers:
          - 'alertname = InfoInhibitor'
        target_matchers:
          - 'severity = info'
        equal:
          - 'namespace'
      - target_matchers:
          - 'alertname = InfoInhibitor'
    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'slack-notifications'

      routes:
      - receiver: 'null'
        matchers:
          - alertname = "Watchdog"
    receivers:
    - name: "null"

    - name: "slack-notifications"
      slack_configs:
      - api_url: "https://hooks.slack.com/services/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"
        username: "Alertmanager"
        icon_emoji: ":rotating_light:"
        send_resolved: true
        title: '{{ .CommonLabels.alertname }}' 
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
    templates:
    - '/etc/alertmanager/config/*.tmpl'
```

AlertMangerIngress Creation and use below commmand

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alertmanager-ingress
  namespace: monitoring

  annotations:
    kubernetes.io/ingress.class: alb
    # Shared ALB Group
    alb.ingress.kubernetes.io/group.name: monitoring
    # Internet Facing ALB
    alb.ingress.kubernetes.io/scheme: internet-facing
    # Use Pod IPs as Targets
    alb.ingress.kubernetes.io/target-type: ip
    # HTTPS Listener
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    # Redirect HTTP -> HTTPS
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    # ACM Certificate
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-north-1:188776114860:certificate/d99003fb-1027-48bf-8af6-5004baf57a95
    # Health Check
    alb.ingress.kubernetes.io/healthcheck-path: /-/healthy
    alb.ingress.kubernetes.io/healthcheck-port: "9093"
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/success-codes: "200"

spec:
  ingressClassName: alb
  rules:
    - host: alertmanager.tulaja.shop
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: monitoring-kube-prometheus-alertmanager
                port:
                  number: 9093
```
Command  
```yaml
kubectl create -f IngressName.yaml  
```
Create GrafanaIngress.yaml  
```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring

  annotations:
    kubernetes.io/ingress.class: alb

    # Shared ALB Group
    alb.ingress.kubernetes.io/group.name: monitoring

    # Internet Facing ALB
    alb.ingress.kubernetes.io/scheme: internet-facing

    # Use Pod IPs as Targets
    alb.ingress.kubernetes.io/target-type: ip

    # HTTPS Listener
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'

    # Redirect HTTP -> HTTPS
    alb.ingress.kubernetes.io/ssl-redirect: "443"

    # ACM Certificate
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-north-1:188776114860:certificate/d99003fb-1027-48bf-8af6-5004baf57a95

    # Health Check Configuration
    alb.ingress.kubernetes.io/healthcheck-path: /api/health

    alb.ingress.kubernetes.io/healthcheck-port: "3000"

    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP

    alb.ingress.kubernetes.io/success-codes: "200"

spec:
  ingressClassName: alb

  rules:
    - host: grafana.tulaja.shop
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: monitoring-grafana
                port:
                  number: 80
```

Create the PrometheusIngress.yaml  
```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/group.name: monitoring
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-north-1:188776114860:certificate/d99003fb-1027-48bf-8af6-5004baf57a95
    alb.ingress.kubernetes.io/healthcheck-path: /-/healthy
    alb.ingress.kubernetes.io/healthcheck-port: "9090"
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/success-codes: "200"
spec:
  ingressClassName: alb
  rules:
    - host: prometheus.tulaja.shop
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: monitoring-kube-prometheus-prometheus
                port:
                  number: 9090
```

Command to Create the Ingress.yaml  
```yaml
kubectl create -f PrometehusIngress.yaml
```

```yaml  
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cluster-and-app-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:        
  - name: cluster-health
    rules:
    - alert: KubeAPIDown
      expr: up{job="kubernetes-apiservers"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: Kubernetes API server is down

    - alert: EtcdInstanceDown
      expr: up{job="etcd"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: etcd instance is down
  
    - alert: KubeSchedulerDown
      expr: up{job="kube-scheduler"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Kubernetes Scheduler is down"
        description: "kube-scheduler is not responding in Prometheus scrape"

    - alert: KubeControllerManagerDown
      expr: up{job="kube-controller-manager"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Kubernetes Controller Manager is down"
        description: "kube-controller-manager is not responding in Prometheus scrape"  

  - name: pod-status
    rules:
    - alert: PodCrashLooping
      expr: kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: Pod is in CrashLoopBackOff

    - alert: PodNotReady
      expr: kube_pod_status_ready{condition="true"} == 0
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: Pod is not ready

    - alert: PodNotReady
      expr: kube_pod_status_ready{condition="true"} == 0
      for: 5m
      labels:
        severity: Critical
      annotations:
        summary: Pod is not ready

    - alert: PodRestartingFrequently
      expr: increase(kube_pod_container_status_restarts_total[10m]) > 3
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod restarting frequently"
        description: "Pod restarted more than 3 times in last 10 minutes"

    - alert: PodErrImagePull
      expr: kube_pod_container_status_waiting_reason{reason="ErrImagePull"} > 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Pod image pull failed (ErrImagePull)"
        description: "Kubernetes is unable to pull container image (check image name or registry auth)"

    - alert: PodImagePullBackOff
      expr: kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"} > 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Pod in ImagePullBackOff"
        description: "Kubernetes is retrying image pull and backing off"

    - alert: PodOOMKilled
      expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Pod killed due to OOM"
        description: "Container exceeded memory limit and was killed"

    - alert: PVCAlmostFull
      expr: (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) * 100 > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "PVC disk usage is high"
        description: "PVC usage is above 80% for more than 5 minutes"

    - alert: PVCCriticalUsage
      expr: (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) * 100 > 90
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "PVC critically full"
        description: "PVC usage is above 90%"

  - name: node-status
    rules:
    - alert: NodeNotReady
      expr: kube_node_status_condition{condition="Ready",status="true"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: Node is not ready

    - alert: NodeMemoryPressure
      expr: kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: Node memory pressure

    - alert: NodeDiskPressure
      expr: kube_node_status_condition{condition="DiskPressure",status="true"} == 1
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: Node disk pressure

    - alert: HighNodeCPUUsage
      expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100) > 80
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage on node"

    - alert: HighNodeCPUUsage
      expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100) > 90
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "High CPU usage on node"

    - alert: HighNodeMemoryUsage
      expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 80
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage on node"

    - alert: HighNodeMemoryUsage
      expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage on node"

    - alert: HighNodeDiskUsage
      expr: (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} 
        / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"})) * 100 > 80
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High disk usage on node"

    - alert: HighNodeDiskUsage
      expr: (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} 
        / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"})) * 100 > 90
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "High disk usage on node"
```


                  


