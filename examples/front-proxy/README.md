To learn about this sandbox and for instructions on how to run it please head over
to the [envoy docs](https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/front_proxy.html)

# Demo

$ docker-compose up

$ curl localhost:8000/service/1
Hello from behind Envoy (service 1)! hostname: c7fff8692c77 resolvedhostname: 172.20.0.2

$ curl localhost:8000/service/2
Hello from behind Envoy (service 2)! hostname: 2901ab8db191 resolvedhostname: 172.20.0.3

# Config

$ yamllint -d "{extends: default, rules: {line-length: {max: 140}, key-ordering: {}}}" front-envoy.yaml

# Benchmark

Without circuit breaking:

```
$ echo "GET http://127.0.0.1:8000/service/2" | vegeta attack -duration=10s | vegeta report
Requests      [total, rate]            500, 50.10
Duration      [total, attack, wait]    9.984906183s, 9.979626631s, 5.279552ms
Latencies     [mean, 50, 95, 99, max]  10.296221ms, 9.581441ms, 18.830659ms, 34.100688ms, 67.192473ms
Bytes In      [total, mean]            44500, 89.00
Bytes Out     [total, mean]            0, 0.00
Success       [ratio]                  100.00%
Status Codes  [code:count]             200:500  
Error Set:

$ curl -s localhost:8001/stats | grep service2 | grep -E "cx|rq" | grep -v ": 0"
cluster.service2.external.upstream_rq_200: 500
cluster.service2.external.upstream_rq_2xx: 500
cluster.service2.upstream_cx_active: 2
cluster.service2.upstream_cx_http2_total: 2
cluster.service2.upstream_cx_rx_bytes_buffered: 233
cluster.service2.upstream_cx_rx_bytes_total: 63691
cluster.service2.upstream_cx_total: 2
cluster.service2.upstream_cx_tx_bytes_total: 27569
cluster.service2.upstream_rq_200: 500
cluster.service2.upstream_rq_2xx: 500
cluster.service2.upstream_rq_total: 500
cluster.service2.external.upstream_rq_time: P0(nan,1) P25(nan,3.09219) P50(nan,6.00462) P75(nan,8.03571) P90(nan,11) P95(nan,13.3333) P99(nan,23) P99.9(nan,35.5) P100(nan,36)
cluster.service2.upstream_cx_connect_ms: P0(nan,3) P25(nan,3.025) P50(nan,3.05) P75(nan,3.075) P90(nan,3.09) P95(nan,3.095) P99(nan,3.099) P99.9(nan,3.0999) P100(nan,3.1)
cluster.service2.upstream_cx_length_ms: No recorded values
cluster.service2.upstream_rq_time: P0(nan,1) P25(nan,3.09219) P50(nan,6.00462) P75(nan,8.03571) P90(nan,11) P95(nan,13.3333) P99(nan,23) P99.9(nan,35.5) P100(nan,36)
```

With circuit breaking

```yaml
static_resources:
  clusters:
    - circuit_breakers:
        thresholds:
          - max_connections: 1
            max_pending_requests: 1
            max_requests: 1
            max_retries: 1
      connect_timeout: 0.25s
      hosts:
        - socket_address:
            address: service2
            port_value: 80
      http2_protocol_options: {}
      lb_policy: round_robin
      name: service2
      type: strict_dns
```

```shell
$ echo "GET http://127.0.0.1:8000/service/2" | vegeta attack -duration=10s | vegeta report
Requests      [total, rate]            500, 50.11
Duration      [total, attack, wait]    9.990570338s, 9.978277768s, 12.29257ms
Latencies     [mean, 50, 95, 99, max]  67.226317ms, 10.208619ms, 556.062277ms, 959.614973ms, 1.089870731s
Bytes In      [total, mean]            42644, 85.29
Bytes Out     [total, mean]            0, 0.00
Success       [ratio]                  88.40%
Status Codes  [code:count]             503:58  200:442  
Error Set:
503 Service Unavailable

$ cluster.service2.external.upstream_rq_200: 442
cluster.service2.external.upstream_rq_2xx: 442
cluster.service2.external.upstream_rq_503: 58
cluster.service2.external.upstream_rq_5xx: 58
cluster.service2.upstream_cx_active: 2
cluster.service2.upstream_cx_http2_total: 2
cluster.service2.upstream_cx_rx_bytes_buffered: 214
cluster.service2.upstream_cx_rx_bytes_total: 56309
cluster.service2.upstream_cx_total: 2
cluster.service2.upstream_cx_tx_bytes_total: 24474
cluster.service2.upstream_rq_200: 442
cluster.service2.upstream_rq_2xx: 442
cluster.service2.upstream_rq_503: 58
cluster.service2.upstream_rq_5xx: 58
cluster.service2.upstream_rq_pending_overflow: 57
cluster.service2.upstream_rq_total: 443
cluster.service2.external.upstream_rq_time: P0(nan,1) P25(nan,3.07971) P50(nan,5.05071) P75(nan,7.09276) P90(nan,10.4529) P95(nan,12.8722) P99(nan,18.785) P99.9(nan,29.7785) P100(nan,30)
cluster.service2.upstream_cx_connect_ms: P0(nan,1) P25(nan,1.05) P50(nan,1.1) P75(nan,4.05) P90(nan,4.08) P95(nan,4.09) P99(nan,4.098) P99.9(nan,4.0998) P100(nan,4.1)
cluster.service2.upstream_cx_length_ms: No recorded values
cluster.service2.upstream_rq_time: P0(nan,1) P25(nan,3.07971) P50(nan,5.05071) P75(nan,7.09276) P90(nan,10.4529) P95(nan,12.8722) P99(nan,18.785) P99.9(nan,29.7785) P100(nan,30)
```