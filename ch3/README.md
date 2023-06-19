# CH3

```
docker run -d --name httpbin citizenstig/httpbin
```

```
docker run -it --rm --link httpbin curlimages/curl curl -X GET http://httpbin:8000/headers 
```
> Return
```
{
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin:8000", 
    "User-Agent": "curl/7.83.0-DEV"
  }
}
```


```
docker run -it --rm envoyproxy/envoy:v1.19.0 envoy  
```
> Return
```
...
[2022-05-07 03:09:35.316][1][info][main] [source/server/server.cc:342]   envoy.http.stateful_header_formatters: preserve_case
[2022-05-07 03:09:35.316][1][info][main] [source/server/server.cc:342]   envoy.quic.server.crypto_stream: envoy.quic.crypto_stream.server.quiche
[2022-05-07 03:09:35.316][1][critical][main] [source/server/server.cc:112] error initializing configuration '': At least one of --config-path or --config-yaml or Options::configProto() should be non-empty
[2022-05-07 03:09:35.316][1][info][main] [source/server/server.cc:855] exiting
At least one of --config-path or --config-yaml or Options::configProto() should be non-empty
```

```
docker run -it --rm envoyproxy/envoy:v1.19.0 envoy --config-yaml "$(cat ch3/simple.yaml)"
```


 - X-Envoy-Expected-Rq-Timeout-Ms
    
 - X-Request-Id

```
docker run -it --rm --link proxy curlimages/curl curl  -X GET http://proxy:15001/headers
```
> Return
```
{
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin", 
    "User-Agent": "curl/7.83.0-DEV", 
    "X-Envoy-Expected-Rq-Timeout-Ms": "15000", 
    "X-Request-Id": "f986a3c9-8ee4-4a63-8ae5-043814da0f22"
  }
}
```

```
curl -X GET https://httpbin.org/headers
```
> Return
```
{
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/7.79.1", 
    "X-Amzn-Trace-Id": "Root=1-6275e8e4-45623edf1a515ccf22877177"
  }
}
```

```
docker rm -f proxy
```

```
docker run --name proxy --link httpbin envoyproxy/envoy:v1.19.0  --config-yaml "$(cat ch3/simple_change_timout.yaml)"
```

```
docker run -it --rm --link proxy curlimages/curl curl  -X GET http://proxy:15001/headers
```
> Return
```
{
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin", 
    "User-Agent": "curl/7.83.0-DEV", 
    "X-Envoy-Expected-Rq-Timeout-Ms": "1000", 
    "X-Request-Id": "b368c66f-9371-4305-9f8b-854d71795d47"
  }
}
```

## Admin

- [ ] [Override the default configuration](https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/run-envoy#override-the-default-configuration)

```
admin:
  address:
    socket_address: { address: 0.0.0.0, port_value: 15000 }
```

```
- match: { prefix: "/" }
  route:
    auto_host_rewrite: true
    cluster: httpbin_service
    retry_policy: 
        retry_on: 5xx    ❶
        num_retries: 3   ❷

```

```
docker run --name proxy --link httpbin envoyproxy/envoy:v1.19.0  --config-yaml "$(cat ch3/simple_retry.yaml)"
```


```
docker run -it --rm --link proxy curlimages/curl curl  -X GET http://proxy:15000/stats
```
> Return
```
cluster.httpbin_service.assignment_stale: 0
cluster.httpbin_service.assignment_timeout_received: 0
cluster.httpbin_service.bind_errors: 0
cluster.httpbin_service.circuit_breakers.default.cx_open: 0
cluster.httpbin_service.circuit_breakers.default.cx_pool_open: 0
cluster.httpbin_service.circuit_breakers.default.rq_open: 0
...
```

```
docker run -it --rm --link proxy curlimages/curl curl  -X GET http://proxy:15000 > ch3/stats.html
```

```
docker run -it --rm --link proxy curlimages/curl curl  -X GET http://proxy:15000/server_info | jq '.version'
```
> Return
```
"68fe53a889416fd8570506232052b06f5a531541/1.19.0/Clean/RELEASE/BoringSSL"
```

```
docker run -it --rm --link proxy curlimages/curl curl  -X GET http://proxy:15001/status/500                
```

```
docker run -it --rm --link proxy curlimages/curl curl  -X GET http://proxy:15000/stats | grep retry
```
> Return
```
cluster.httpbin_service.circuit_breakers.default.rq_retry_open: 0
cluster.httpbin_service.circuit_breakers.high.rq_retry_open: 0
cluster.httpbin_service.retry.upstream_rq_500: 3
cluster.httpbin_service.retry.upstream_rq_5xx: 3
cluster.httpbin_service.retry.upstream_rq_completed: 3
cluster.httpbin_service.retry_or_shadow_abandoned: 0
cluster.httpbin_service.upstream_rq_retry: 3
cluster.httpbin_service.upstream_rq_retry_backoff_exponential: 3
cluster.httpbin_service.upstream_rq_retry_backoff_ratelimited: 0
cluster.httpbin_service.upstream_rq_retry_limit_exceeded: 1
cluster.httpbin_service.upstream_rq_retry_overflow: 0
cluster.httpbin_service.upstream_rq_retry_success: 0
vhost.httpbin_local_service.vcluster.other.upstream_rq_retry: 0
vhost.httpbin_local_service.vcluster.other.upstream_rq_retry_limit_exceeded: 0
vhost.httpbin_local_service.vcluster.other.upstream_rq_retry_overflow: 0
vhost.httpbin_local_service.vcluster.other.upstream_rq_retry_success: 0
```


