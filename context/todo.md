# Mission: Exposed web services & access info audit

## M1: Enumerate listeners, classify HTTP vs non-HTTP, capture machine IPs | status: completed
### T1.1: ss -tlnp enumeration + curl probes + docker ps mapping | agent:Commander(direct read-only)
- [x] S1.1.1: Enumerate all TCP listeners via ss -tlnp | size:S
- [x] S1.1.2: Probe each port with curl to classify HTTP services (200/301/302/404=HTTP alive) | size:S
- [x] S1.1.3: Map docker-proxy ports to container names via docker ps | size:S
- [x] S1.1.4: Capture host IPs via ip -brief addr | size:S
> Note: prior DeepSeek-V4 mission (9/29) archived to archive/ — PAUSED pending user decision to resume.
