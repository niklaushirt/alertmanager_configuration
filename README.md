# IBM Cloud Private Alerting with Prometheus
![](http://nkls.info/wp-content/uploads/2018/06/Screenshot-2018-06-07-16.30.00.png)


With the new versions of IBM Cloud Private (ICP) a new version of Prometheus has also been implemented.
For ICP 2.1.0.3 this would be Prometheus version 2.0.0 to be precise. 

Amongst other changes and improvements, this brings with it a new format for configuring the Alertmanager rules.
After some digging around and a lot of trial and error I got a working configuration up and running that I thought I'd share with you.

Remark: the configuration files can be downloaded from GitHub [here](https://github.com/niklaushirt/alertmanager_configuration/tree/master)
</BR>

## Receivers
First I configured two receivers:

- Slack
- IBM Cloud Event Management - available on [IBM Public Cloud](https://console.bluemix.net/docs/services/EventManagement/index.html#event_gettingstarted)
</BR>
</BR>
</BR>
![](http://nkls.info/wp-content/uploads/2018/06/Screenshot-2018-06-07-16.38.32.png)

### Slack

This is handled by the slack receiver.

```yaml
    global:
      slack_api_url: 'https://hooks.slack.com/services/xxxx/yyyy/zzzz'

....

        slack_configs:
        - channel: '#ibmcloudprivate'
          text: 'Nodes: {{ range .Alerts }}{{ .Labels.instance }} {{ end }}      ---- Summary: {{ .CommonAnnotations.summary }}      ---- Description: {{ .CommonAnnotations.description }}       ---- https://*yourICPClusterAddress*:8443/prometheus/alerts '
```

Replace the *yourICPClusterAddress* address with the real ingress IP of your ICP Cluster.

</BR>
![](http://nkls.info/wp-content/uploads/2018/06/Screenshot-2018-06-07-16.37.39.png)

### IBM Cloud Event Management

This is handled by a webhook receiver.
The URL can be obtained directly in the web interface under:
>*Administration* / *Event sources* / *Configure an event source* / *Prometheus*


```yaml
        webhook_configs:
          - send_resolved: true
            url: 'https://cem-normalizer-us-south.opsmgmt.bluemix.net/webhook/prometheus/xxxxx/yyyyy/zzzzz'
```


## Routes
Then we define the two routes pulling in both receivers:

```yaml
   route:
      receiver: webhook
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 1h
    route:
      receiver: 'slack-notifications'
      group_by: [alertname, datacenter, app]
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 1h
```

You can configure the timing parts to your likings.
Typically for testing purposes you would want to chose much shorter group- and repeat intervals.
 
</BR>
<HR>
# RULES
Now on to the more challenging part - the new rule format.
You can read up on this [here](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/).
And there is a [nice crisp article](https://www.robustperception.io/converting-rules-to-the-prometheus-2-0-format/) on how to convert old rules into the new format.

So basically we go from this:

```yaml
ALERT HighErrorRate
  IF job:request_latency_seconds:mean5m{job="myjob"} > 0.5
  FOR 10m
  ANNOTATIONS {
    summary = "High request latency",
  }
```

to this:

```yaml
groups:
- name: alert.rules
  rules:
  - alert: HighErrorRate
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
    for: 10m
    annotations:
      summary: High request latency
```

Not complete rocket science but takes some time to convert and test it out.

</BR>
## Example rules

So to give you a starting point and get you of the ground, you might want to start with this set of rules:

```yaml
    groups:
    - name: alert.rules
      rules:
      - alert: high_cpu_load
        expr: node_load1 > 5
        for: 10s
        labels:
          severity: critical
        annotations:
          description: Docker host is under high load, the avg load 1m is at {{ $value}}.
            Reported by instance {{ $labels.instance }} of job {{ $labels.job }}.
          summary: Server under high load
      - alert: high_memory_load
        expr: (sum(node_memory_MemTotal) - sum(node_memory_MemFree + node_memory_Buffers
          + node_memory_Cached)) / sum(node_memory_MemTotal) * 100 > 85
        for: 30s
        labels:
          severity: warning
        annotations:
          description: Docker host memory usage is {{ humanize $value}}%. Reported by
            instance {{ $labels.instance }} of job {{ $labels.job }}.
          summary: Server memory is almost full
      - alert: high_storage_load
        expr: (node_filesystem_size{fstype="aufs"} - node_filesystem_free{fstype="aufs"})
          / node_filesystem_size{fstype="aufs"} * 100 > 15
        for: 30s
        labels:
          severity: warning
        annotations:
          description: Docker host storage usage is {{ humanize $value}}%. Reported by
            instance {{ $labels.instance }} of job {{ $labels.job }}.
          summary: Server storage is almost full
```

Again, adapt the thresholds to your likings/needs.

</BR>
</BR>
# Putting it all together
So to implement those alerts, basically you'll have to update two ConfigMaps in ICP.

To do this:

* Go to Configuration / ConfigMaps
* Filter for "alert"
* And you should find three ConfigMaps

Now you can either modify them by hand by clicking on the action handle to the very right (three small vertical points)


Or you can just select "Create Resource" in the top menu and paste the two followind scripts one after another.

</BR>
## Update Alert Rules
To modify the monitoring-prometheus-alertrules ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: monitoring-prometheus
    component: prometheus
  name: monitoring-prometheus-alertrules
  namespace: kube-system
data:
  sample.rules: |-
    groups:
    - name: alert.rules
      rules:
      - alert: high_cpu_load
        expr: node_load1 > 5
        for: 10s
        labels:
          severity: critical
        annotations:
          description: Docker host is under high load, the avg load 1m is at {{ $value}}.
            Reported by instance {{ $labels.instance }} of job {{ $labels.job }}.
          summary: Server under high load
      - alert: high_memory_load
        expr: (sum(node_memory_MemTotal) - sum(node_memory_MemFree + node_memory_Buffers
          + node_memory_Cached)) / sum(node_memory_MemTotal) * 100 > 85
        for: 30s
        labels:
          severity: warning
        annotations:
          description: Docker host memory usage is {{ humanize $value}}%. Reported by
            instance {{ $labels.instance }} of job {{ $labels.job }}.
          summary: Server memory is almost full
      - alert: high_storage_load
        expr: (node_filesystem_size{fstype="aufs"} - node_filesystem_free{fstype="aufs"})
          / node_filesystem_size{fstype="aufs"} * 100 > 15
        for: 30s
        labels:
          severity: warning
        annotations:
          description: Docker host storage usage is {{ humanize $value}}%. Reported by
            instance {{ $labels.instance }} of job {{ $labels.job }}.
          summary: Server storage is almost full
```

</BR>
## Update Alert Routes
And to modify the monitoring-prometheus-alertmanager ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: monitoring-prometheus
    component: alertmanager
  name: monitoring-prometheus-alertmanager
  namespace: kube-system
data:
  alertmanager.yml: |-
    global:
      resolve_timeout: 20s
      slack_api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'

    receivers:
      - name: 'webhook'
        webhook_configs:
          - send_resolved: true
            url: 'https://cem-normalizer-us-south.opsmgmt.bluemix.net/webhook/prometheus/xxx/yyy/zzz'
      - name: 'slack-notifications'
        slack_configs:
        - channel: '#ibmcloudprivate'
          text: 'Nodes: {{ range .Alerts }}{{ .Labels.instance }} {{ end }}      ---- Summary: {{ .CommonAnnotations.summary }}      ---- Description: {{ .CommonAnnotations.description }}       ---- https://*yourICPClusterAddress*:8443/prometheus/alerts '
    route:
      receiver: webhook
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 1h
    route:
      receiver: 'slack-notifications'
      group_by: [alertname, datacenter, app]
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 1h
```


# Test the whole thing

## Check Rules

![](http://nkls.info/wp-content/uploads/2018/06/Screenshot-2018-06-07-15.58.31.png)

Check if it works by going to 
>https://*yourICPClusterAddress*:8443/prometheus/rules 

and verify that the rules have been loaded.
**This might take a while to update!**


</BR>
## Create Load

If you like to test the CPU rule firing you can do the following:

Start a simple BusyBox shell

```yaml
docker run -it --rm busybox
```

and then generate some simple load by pasting several times the following (depends on the number of cores you're running)

```yaml
yes > /dev/null &
```

</BR>

## Check Load

![](http://nkls.info/wp-content/uploads/2018/06/Screenshot-2018-06-07-16.27.09.png)

To check out the generated load open the following URL:
>https://*yourICPClusterAddress*:8443/prometheus/graphg0.range_input=1h&g0.expr=node_load1%20%3E%201.5&g0.tab=0

</BR>

## Check Alerts
![](http://nkls.info/wp-content/uploads/2018/06/Screenshot-2018-06-07-16.31.45.png)

And to see if the alerts are firing either go to 
>https://*yourICPClusterAddress*:8443/alertmanager/#/alerts

or to 

>https://*yourICPClusterAddress*:8443/prometheus/alerts
for more details



</BR></BR></BR>

**I hope that this short writeup might help some poor soul out there, trying to get the Prometheus Alerting system working in the recent versions of ICP.**

