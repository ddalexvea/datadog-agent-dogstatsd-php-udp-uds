# Missing Custom Metrics - DogStatsD PHP Client Sandbox

## 📋 Overview

This sandbox documents troubleshooting steps for missing custom metric datapoints when using the **DogStatsD PHP client** (`datadog/php-datadogstatsd` package version ^1.5).

---

## 🔍 Customer Code Analysis

### Customer Implementation

```php
protected function createStatsdService($host, $port)
{
    return new DogStatsd([
        'host' => $host,
        'port' => $port,
    ]);
}

protected function logCounter($_name, $_value = 1, $_tags = [], $_sample_rate = 1)
{
    $this->init();
    foreach ($this->statsd_services as $service) {
        if ($_value > 0) {
            $service->increment($_name, $_sample_rate, $_tags, $_value);
        } else {
            $service->decrement($_name, $_sample_rate, $_tags, $_value);
        }
    }
}
```

### ✅ Code Analysis Result: No Bugs Detected

| Aspect | Status |
|--------|--------|
| DogStatsd instantiation | ✅ Correct |
| `increment()` parameters order | ✅ Correct |
| Loop logic | ✅ Correct |
| Value handling | ✅ Correct (positive → increment, else → decrement) |

**The PHP code is correct.** The issue is likely infrastructure or configuration related.

---

## 📋 What We Know vs Don't Know

| We Have | We Don't Have |
|---------|---------------|
| ✅ Datadog Agent `values.yaml` | ❌ PHP pod deployment manifest |
| ✅ PHP code snippet (Logger class) | ❌ How `$host` and `$port` are set |
| ✅ DogStatsD Agent config looks correct | ❌ What `init()` method does |
| | ❌ How `statsd_services` is populated |
| | ❌ Whether ALL or SOME metrics are missing |

---

## 🔍 Customer's Agent Configuration Analysis

Their `values.yaml` has correct DogStatsD settings:

```yaml
dogstatsd:
  nonLocalTraffic: true   # ✅ Allows traffic from other pods
  useHostPort: true       # ✅ Exposes port 8125 on host
  originDetection: true   # ✅ Container tagging
```

**Note:** `logLevel: CRITICAL` makes debugging difficult - no Agent-side visibility.

---

## 🎯 Recommendations for Customer

### Step 1: Set Agent Log Level to DEBUG (Quick Win)

```yaml
datadog:
  logLevel: DEBUG  # Was: CRITICAL
```

Then check agent logs:
```bash
kubectl logs <agent-pod> -c agent | grep -i dogstatsd
```

### Step 2: Add Minimal PHP Logging

Since we don't know if `init()` works or if `statsd_services` is populated:

```php
protected function logCounter($_name, $_value = 1, $_tags = [], $_sample_rate = 1)
{
    $this->init();
    
    // Add this one line to verify services are configured
    error_log(sprintf("[DogStatsD] services_count=%d", count($this->statsd_services)));
    
    foreach ($this->statsd_services as $service) {
        if ($_value > 0) {
            $service->increment($_name, $_sample_rate, $_tags, $_value);
        } else {
            $service->decrement($_name, $_sample_rate, $_tags, $_value);
        }
    }
}
```

**What to look for:**
- `services_count=0` → `init()` failed or misconfigured
- `services_count>0` → Services configured, issue is elsewhere

### Step 3: Verify UDP Connectivity

From a pod on the same node as the PHP app:

```bash
# Test UDP connectivity to Agent
echo "test.metric:1|c" | nc -u -w1 $DD_AGENT_HOST 8125
```

### Step 4: Check Agent DogStatsD Stats

```bash
kubectl exec <agent-pod> -c agent -- agent dogstatsd-stats
```

Look for:
- `Udp Packets` - Is Agent receiving UDP traffic?
- `Metric Packets` - Are metrics being processed?

### Step 5: Try UDS (Unix Domain Socket) - Eliminates UDP Issues

UDS is more reliable than UDP (no packet loss).

**Agent values.yaml:**
```yaml
datadog:
  dogstatsd:
    useSocketVolume: true
    socketPath: /var/run/datadog/dsd.socket
```

**PHP App Pod:**
```yaml
volumes:
  - name: dsdsocket
    hostPath:
      path: /var/run/datadog/
containers:
  - name: php-app
    volumeMounts:
      - name: dsdsocket
        mountPath: /var/run/datadog
        readOnly: true
```

**PHP Code:**
```php
$statsd = new DogStatsd([
    'socket_path' => '/var/run/datadog/dsd.socket',
]);
```

---

## ❓ Questions to Ask Customer

1. **Are ALL metrics missing or just SOME?**
   - ALL → Configuration/init issue
   - SOME → Possible UDP packet loss

2. **How does your PHP app get the DogStatsD host/port?**
   - Environment variables?
   - Config file?
   - Hardcoded?

3. **Can you share your PHP pod's deployment manifest?**
   - Need to verify `DD_AGENT_HOST` is set

4. **What does your `init()` method do?**
   - How is `statsd_services` populated?

5. **How many metrics/second are you sending?**
   - High volume → UDP buffer overflow risk

---

## 🔧 Sandbox Test Environment

### Start Minikube

```bash
minikube delete --all
minikube start --driver=docker --memory=4096 --cpus=2
```

### Deploy Datadog Agent

```bash
# Create namespace and secret
kubectl create namespace datadog
kubectl create secret generic datadog-secret \
  --from-literal=api-key="<YOUR_API_KEY>" \
  -n datadog

# Install Agent
helm repo add datadog https://helm.datadoghq.com
helm install datadog-agent datadog/datadog \
  -f values.yaml \
  -n datadog
```

### Verify DogStatsD is Working

```bash
# Check Agent status
kubectl exec -it <agent-pod> -n datadog -c agent -- agent status | grep -A 15 "DogStatsD"

# Verify hostPort and nonLocalTraffic
kubectl get pod <agent-pod> -n datadog -o jsonpath='{.spec.containers[0].ports}'
kubectl get pod <agent-pod> -n datadog -o jsonpath='{.spec.containers[0].env}' | jq '.[] | select(.name | contains("STATSD"))'
```

### Test UDP from a Pod

```bash
# Create test pod
kubectl run test-pod --image=alpine --restart=Never -- sleep 3600

# Send test metric
kubectl exec test-pod -- sh -c '
  apk add --no-cache netcat-openbsd
  DD_HOST=$(ip route | grep default | awk "{print \$3}")
  echo "test.metric:1|c|#env:sandbox" | nc -u -w1 $DD_HOST 8125
'

# Verify Agent received it
kubectl exec <agent-pod> -n datadog -c agent -- agent status | grep "Udp Packets"
```

---

## 📊 How `useHostPort: true` Works

```
┌─────────────────────────────────────────────────────────────────────┐
│                         KUBERNETES NODE                             │
│                      (IP: 192.168.49.2)                             │
│                                                                     │
│    useHostPort: true                                                │
│    Opens port 8125 on the NODE itself                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                             │   │
│  │   NODE:8125  ◀──────────────────────────────────────────┐   │   │
│  │      │                                                  │   │   │
│  │      ▼                                                  │   │   │
│  │  ┌────────────────────┐      ┌────────────────────┐    │   │   │
│  │  │  Datadog Agent Pod │      │    PHP App Pod     │    │   │   │
│  │  │                    │      │                    │    │   │   │
│  │  │  Container:8125 ◀──┘      │  sends to:         │    │   │   │
│  │  │  (DogStatsD)       │      │  NODE_IP:8125  ────┘    │   │   │
│  │  │                    │      │                    │    │   │   │
│  │  └────────────────────┘      └────────────────────┘    │   │   │
│  │                                                         │   │   │
│  └─────────────────────────────────────────────────────────┘   │   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**PHP app needs `DD_AGENT_HOST` to know the NODE's IP:**

```yaml
# In PHP app deployment
env:
  - name: DD_AGENT_HOST
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
  - name: DD_DOGSTATSD_PORT
    value: "8125"
```

---

## 📚 Reference Documentation

- [DogStatsD PHP Client GitHub](https://github.com/DataDog/php-datadogstatsd)
- [DogStatsD Documentation](https://docs.datadoghq.com/developers/dogstatsd/)
- [Troubleshooting DogStatsD](https://docs.datadoghq.com/developers/dogstatsd/troubleshooting/)
- [Unix Domain Socket Setup](https://docs.datadoghq.com/developers/dogstatsd/unix_socket/)

---

## 🎯 Summary

| Component | Status |
|-----------|--------|
| PHP Code | ✅ Correct - no bugs |
| Agent DogStatsD Config | ✅ Looks correct |
| Missing Info | ❌ PHP pod manifest, init() method |
| Root Cause | ❓ Unknown - need more info from customer |

**Next Steps:**
1. Ask customer for PHP pod deployment manifest
2. Ask customer to add one line of logging to verify `services_count`
3. Set Agent log level to DEBUG temporarily
4. Consider testing with UDS instead of UDP

---

*Last updated: December 2024*
