# prometheus-stack

 In Values.yaml file these things must be edited 

 See Design must be like below
1) if you have separate Node group for the EKS Cluster in the specific availability zone , the nodes are launched in specific az as the requirement is that we have pv for all the three prometheus, grafana and alertmanager.
2) I have added Below config in values.yaml file for the Prometheus
    i) First have the storage class created with waitforfirstconsumer and with the policy Retain and CHANGE THE STORAGE CLASS NAME
   ii) Decide the Storage with the Client How much GB you want and Change it
  iii) If you want even you can decide the Resource Limits 


⚙️ General Configurations for All Pods

⚠️ These values must be added in the correct places in values.yaml (do NOT copy-paste blindly)


🔧 Tolerations

```yaml
tolerations:
  # Example:
  #- key: "key"
  #  operator: "Equal"
  #  value: "value"
  #  effect: "NoSchedule"
```
💻 Resources
## Resources Configuration

Used to define CPU and memory limits for pods.

```yaml
resources:
  #requests:
  #  memory: 400Mi
  #  cpu: 200m
```

###ALERTMANAGER
alertmanagerSpec:
   storage: 
      volumeClaimTemplate:
       spec:
         storageClassName: gp3
         accessModes: ["ReadWriteOnce"]
         resources:
           requests:
             storage: 20Gi



 ###GRAFANA
i) Decide the size
ii) let the admin user be admin and let the grafana create the passowrd which is stored as secret
grafana:
   persistence:
      enabled: true
      storageClassName: "gp3"
      accessModes:
        - ReadWriteOnce
      size: 20Gi


 ###PROMETHEUS
 i) Decide the size
 storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi 


THIS IS SLACK BASED ALERTMANAGER.YAML

1. Create a Slack Channel
   
   i)Open Slack (web or app)
   ii)Click “+” next to Channels
   iii)Choose Create a channel
   iv)Name it like:
      alerts-monitoring
   v)Click Create

2. Create Slack App (for Webhook)
   i)Go to: https://api.slack.com/apps
   ii)Click Create New App
   iii)Choose From scratch
   iv) App Name: Alertmanager
   v) Pick your workspace → Create App

3. Enable Incoming Webhooks
   i)Inside your app, go to Incoming Webhooks
   ii)Toggle Activate Incoming Webhooks = ON
   iii)Click Add New Webhook to Workspace
   iv)Select your channel (alerts-monitoring)
   v)Click Allow

4. Get Webhook URL (IMPORTANT)
      You will get a URL like:
            https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
      This is what you use in alertmanager.yaml


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


