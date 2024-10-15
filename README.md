# Pingmesh System Based on Blackbox

## Background

Data centers are inherently complex, and the network aspect becomes even more complicated due to the large number of devices involved. A large data center typically has hundreds or thousands of nodes, network cards, switches, routers, and numerous cables and optical fibers. On top of this hardware, many software systems such as search engines, distributed file systems, and distributed storage are built. While running these systems, we face certain challenges: How can we determine if a fault is a network issue? How do we define and track network SLAs? How do we troubleshoot when something goes wrong?

![IDC](https://kubeservice.cn/img/devops/IDC_hu8ec2fdff58b0ea09e7358f84cbaf1df1_175984_filter_3454788233369042773.png)

`Monitoring network performance data` is particularly difficult to achieve. If we directly use the `ping` command to collect results, each server would need to ping all other `(N-1)` servers, which results in an `N^2` complexity. This presents challenges in both stability and performance.

For example: 
If an IDC contains 10,000 servers, the total number of ping tasks is `10000*9999`. If each machine has multiple IP requests, the result would double.

Data storage also becomes an issue. If we perform a ping every 30 seconds, and each ping requires a payload size of 64 bytes, the data storage requirement would be:
`10000*9999*2*64*24*3600/30` = `3.6860314e+13 bytes` = `33.52TB`

Only recording `fail` and `timeout` events could save `99.99%` of the storage space.

## Industry Implementation

This system is an `enhanced` implementation based on the `Microsoft Pingmesh paper`.

Original Microsoft Pingmesh paper:
[《Pingmesh: A Large-Scale System for Data Center Network Latency Measurement and Analysis》](https://conferences.sigcomm.org/sigcomm/2015/pdf/papers/p139.pdf)

`Microsoft Pingmesh` represents a significant breakthrough in network monitoring. (Please refer to the original paper for details.)

However, there are several limitations in actual use:

1. Agent data flow: After each ping, the `Agent` records the result in a log, and the infrastructure collects this `log` data. Using a `log analysis` system increases system complexity.

2. Ping mode support: It only supports `UDP` mode. Support for `DNS TCP`, `ICMP ping`, etc., is lacking.

3. Ping dimension: It only supports `IPv4` ping. However, in many scenarios, it is necessary to support whether public networks are connected, such as `domain/DNS` pings.

4. No support for manual real-time ping attempts: This can be achieved through network probing based on the `blackbox-exporter`.

5. No support for IPv6.

## Architecture After the Pingmesh Upgrade

![Pingmesh+](https://kubeservice.cn/img/devops/pingmesh_hu8c196f2563a4108ff3fa8682517063fd_177531_filter_4759638724306006349.png)
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fkubeservice-stack%2Fpingmesh-agent.svg?type=shield)](https://app.fossa.com/projects/git%2Bgithub.com%2Fkubeservice-stack%2Fpingmesh-agent?ref=badge_shield)

### Controller

The `Controller` is mainly responsible for generating the `pinglist.yaml` file. There are three sources for the `pinglist` generation:

> Automatically obtaining the podIP and nodeIP list of the entire cluster through the `IP Controller`.

> Obtaining the `Agent Setting` configuration through the `Pinglist Controller`.

> Adding external addresses through `Custom Define Pinglist` in the `pinglist.yaml` file. Supports `DNS addresses`, `external HTTP addresses`, `domain addresses`, `NTP addresses`, `Kubernetes apiserver addresses`, etc.

After generating the `pinglist` file, the `Controller` makes it available through `HTTP/HTTPS`. The `Agent` periodically fetches the `pinglist` to update its configuration, which is referred to as a `pull` model. To ensure high availability, multiple instances of the `Controller` should be configured behind a `Service`, and each instance should have the same algorithm and `pinglist` file to ensure consistency and availability.

### Agent
Each ping action opens a new connection. To reduce `TCP` concurrency caused by `Pingmesh`, the minimum ping interval between two servers is 10 seconds, and the packet size is limited to 64KB.

```yaml
setting:
  # the maximum amount of concurrent to ping, uint
  concurrent_limit: 20
  # interval to exec ping in seconds, float
  interval: 60.0
  # The maximum delay time to ping in milliseconds, float
  delay: 200
  # ping timeout in seconds, float
  timeout: 2.0
  # send ip addr
  source_ip_addr: 0.0.0.0
  # send ip protocol
  ip_protocol: ip6

mesh:
  add-ping-public: 
    name: ping-public-demo
    type: OtherIP
    ips :
      - 127.0.0.1
      - 8.8.8.8
      - www.baidu.com
      - kubernetes.default.svc.cluster.local
```

Additionally, `overload protection` has been implemented:
1. If there are too many entries in the `pinglist`, and they cannot all be processed in one cycle (e.g., 10s), the process will complete the current cycle before starting the next one, ensuring that a full round is completed.
2. The configuration allows setting the number of concurrent threads for the `agent`, ensuring that the `pingmesh agent` has a minimal impact (less than 0.1%) on the entire cluster.
3. Metrics are calculated separately in each cycle using `Prometheus Gauge`.

```
# HELP ping_fail ping fail
# TYPE ping_fail gauge
ping_fail{target="8.8.8.8",tor="ping-public-demo"} 1
```

4. To ensure that ping requests are evenly distributed within a `time window interval`, memory-based calculations are performed for the request jobs, and a `ratelimit` is applied to the concurrent coroutines.

## Network Condition Design

In the `interval` time window set in the `pinglist.yaml`:
- If the request exceeds the `timeout` time, the request is marked as `ping_fail`.
- If the request exceeds the `delay` but does not exceed the `timeout`, the request is marked as `ping_duration_milliseconds`.
- If the request does not exceed the `delay`, it is not recorded in the metrics.

## Integration with Prometheus

Add the following to the scrape_configs section of prometheus.yaml, where `pingmeship` is the IP of the server.

```yaml
scrape_configs:
  - job_name: net_monitor
    honor_labels: true
    honor_timestamps: true
    scrape_interval: 60s
    scrape_timeout: 5s
    metrics_path: /metrics
    scheme: http
    static_configs:
    - targets:
      - $pingmeship:9115
```

## License
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fkubeservice-stack%2Fpingmesh-agent.svg?type=large)](https://app.fossa.com/projects/git%2Bgithub.com%2Fkubeservice-stack%2Fpingmesh-agent?ref=badge_large)
