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

### 📊 Sample Rate Explanation

The `$_sample_rate` parameter controls what percentage of metrics are sent:

| Sample Rate | Meaning | Result |
|-------------|---------|--------|
| `1` (default) | No sampling | **100% of metrics sent** |
| `0.5` | 50% sampling | 50% of metrics sent |
| `0.1` | 90% sampling | 10% of metrics sent |
| `0` | Everything sampled | **0% of metrics sent** |

> **Note:** Sample rate `0` means everything is sampled (dropped), so nothing is sent.
> Sample rate `1` means no sampling, so everything is sent.

The customer's default `$_sample_rate = 1` is correct - all metrics should be sent.

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

# Create values.yaml
cat <<'EOF' > /tmp/values.yaml
# Datadog Agent Helm values.yaml for DogStatsD testing
datadog:
  apiKeyExistingSecret: "datadog-secret"
  site: "datadoghq.com"
  
  # DogStatsD Configuration (CRITICAL)
  dogstatsd:
    useHostPort: true
    hostPort: 8125
    nonLocalTraffic: true      # CRITICAL: Allow traffic from other pods
    originDetection: true
    tagCardinality: "orchestrator"

  # Logs collection
  logs:
    enabled: true
    containerCollectAll: true

  # APM (optional)
  apm:
    portEnabled: true
    port: 8126

  # Tags
  tags:
    - "env:sandbox"
    - "sandbox:missing-custom-metrics"

agents:
  enabled: true
  tolerations:
    - operator: Exists

clusterAgent:
  enabled: true
  replicas: 1
EOF

# Install Agent
helm repo add datadog https://helm.datadoghq.com
helm install datadog-agent datadog/datadog \
  -f /tmp/values.yaml \
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

---

## 🐘 PHP DogStatsD Test (Kubernetes)

### Step 1: Create PHP Code ConfigMap

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: php-dogstatsd-code
  namespace: default
data:
  test.php: |
    <?php
    /**
     * DogStatsD PHP UDP Test - Sends metrics every 10 seconds (runs forever)
     */
    
    class DogStatsd {
        private $host;
        private $port;
        
        public function __construct(string $host, int $port) {
            $this->host = $host;
            $this->port = $port;
            $this->log("INFO", "Creating DogStatsD client", ['host' => $host, 'port' => $port]);
        }
        
        public function increment(string $metric, int $value = 1, array $tags = []): void {
            $tagStr = $this->formatTags($tags);
            $message = "{$metric}:{$value}|c{$tagStr}";
            
            $this->log("INFO", "Sending metric", [
                'metric' => $metric,
                'value' => $value,
                'udp_message' => $message,
            ]);
            
            $this->sendUdp($message);
        }
        
        private function formatTags(array $tags): string {
            if (empty($tags)) return '';
            $tagPairs = [];
            foreach ($tags as $key => $value) {
                $tagPairs[] = "{$key}:{$value}";
            }
            return '|#' . implode(',', $tagPairs);
        }
        
        private function sendUdp(string $message): void {
            $fp = @fsockopen("udp://{$this->host}", $this->port, $errno, $errstr, 1);
            if (!$fp) {
                $this->log("ERROR", "UDP socket failed", ['target' => "{$this->host}:{$this->port}"]);
                return;
            }
            $bytes = fwrite($fp, $message);
            fclose($fp);
            $this->log("INFO", "UDP sent", ['bytes' => $bytes, 'to' => "{$this->host}:{$this->port}"]);
        }
        
        private function log(string $level, string $message, array $context = []): void {
            $log = ['timestamp' => date('c'), 'level' => $level, 'message' => $message, 'service' => 'php-app'];
            if (!empty($context)) $log = array_merge($log, $context);
            echo json_encode($log) . "\n";
        }
    }
    
    // Configuration
    $host = getenv('DD_AGENT_HOST') ?: 'localhost';
    $port = (int)(getenv('DD_DOGSTATSD_PORT') ?: 8125);
    $interval = 10; // seconds between metrics
    
    echo "==========================================\n";
    echo "  DogStatsD PHP - Continuous Metrics\n";
    echo "==========================================\n";
    echo "Target: {$host}:{$port}\n";
    echo "Interval: {$interval}s\n";
    echo "==========================================\n\n";
    
    $statsd = new DogStatsd($host, $port);
    $counter = 0;
    
    // Send metrics forever (every 10 seconds)
    while (true) {
        $counter++;
        $statsd->increment('php.sandbox.requests', 1, ['env' => 'sandbox']);
        $statsd->increment('php.sandbox.items', rand(1, 10), ['env' => 'sandbox']);
        echo "--- Iteration {$counter} complete, sleeping {$interval}s ---\n\n";
        sleep($interval);
    }
EOF
```

### Step 2: Create PHP Pod with DD_AGENT_HOST

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: php-dogstatsd-test
  namespace: default
  labels:
    app: php-dogstatsd-test
    # Labels for Datadog tagging
    tags.datadoghq.com/service: "php-app"
    tags.datadoghq.com/env: "sandbox"
  annotations:
    # Autodiscovery annotations for log collection
    ad.datadoghq.com/php.logs: '[{"source": "php", "service": "php-app"}]'
spec:
  containers:
  - name: php
    image: php:8.2-cli
    # Runs continuously - sends metrics every 10 seconds
    command: ["php", "/app/test.php"]
    env:
    - name: DD_AGENT_HOST
      valueFrom:
        fieldRef:
          fieldPath: status.hostIP
    - name: DD_DOGSTATSD_PORT
      value: "8125"
    volumeMounts:
    - name: php-code
      mountPath: /app
  volumes:
  - name: php-code
    configMap:
      name: php-dogstatsd-code
EOF

kubectl wait --for=condition=Ready pod/php-dogstatsd-test --timeout=60s
```

### Step 3: View Metrics Being Sent (Logs)

```bash
# Watch the PHP app sending metrics (runs continuously every 10s)
kubectl logs -f php-dogstatsd-test
```

### Expected Output (Continuous)

```
==========================================
  DogStatsD PHP - Continuous Metrics
==========================================
Target: 192.168.49.2:8125
Interval: 10s
==========================================

{"timestamp":"2025-12-31T13:03:16+00:00","level":"INFO","message":"Creating DogStatsD client","service":"php-app","host":"192.168.49.2","port":8125}
{"timestamp":"2025-12-31T13:03:16+00:00","level":"INFO","message":"Sending metric","service":"php-app","metric":"php.sandbox.requests","value":1,"udp_message":"php.sandbox.requests:1|c|#env:sandbox"}
{"timestamp":"2025-12-31T13:03:16+00:00","level":"INFO","message":"UDP sent","service":"php-app","bytes":42,"to":"192.168.49.2:8125"}
--- Iteration 1 complete, sleeping 10s ---

{"timestamp":"2025-12-31T13:03:26+00:00","level":"INFO","message":"Sending metric","service":"php-app","metric":"php.sandbox.requests","value":1,"udp_message":"php.sandbox.requests:1|c|#env:sandbox"}
...
```

### Step 4: Verify Agent Received Metrics

```bash
# Check UDP packets count (should increase every 10s)
kubectl exec <agent-pod> -n datadog -c agent -- agent status | grep "Udp Packets"
```

### Key Points Demonstrated

| Aspect | Value |
|--------|-------|
| `DD_AGENT_HOST` | Set via `status.hostIP` → Gets node IP |
| `DD_DOGSTATSD_PORT` | `8125` |
| UDP Target | `192.168.49.2:8125` (node IP + hostPort) |
| Logs | JSON to stdout (collected by Agent) |

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
