Here’s a clean blog draft based on your full Q&A thread.

---

# **Monitoring Legacy Java Applications with Prometheus — A Complete Guide**

Modern Spring Boot applications expose Prometheus metrics easily using **Micrometer**, but legacy Java applications—built with **Struts2**, **servlets**, or plain Java—need extra steps.
This post explains **how to monitor such legacy systems** without rewriting code, using Prometheus and the JMX exporter.

---

## **1. Why Legacy Apps Need a Different Approach**

Spring Boot integrates Micrometer, exposing a `/actuator/prometheus` endpoint automatically.
Legacy applications lack this, so we must surface metrics externally.

Prometheus collects metrics by scraping endpoints that expose them in text-based format. For legacy apps, the challenge is **creating such an endpoint**.

---

## **2. The Simplest Approach — JMX Exporter (Java Agent Mode)**

### **What It Is**

[JMX Exporter](https://github.com/prometheus/jmx_exporter) is a lightweight Prometheus agent that attaches to any Java process.
It reads metrics through the Java Management Extensions (JMX) interface and serves them at an HTTP endpoint (e.g., `http://host:9404/metrics`).

### **How to Set It Up**

1. **Download the jar**

   ```
   https://github.com/prometheus/jmx_exporter/releases
   ```

   Example:
   `jmx_prometheus_javaagent-0.20.0.jar`

2. **Create `config.yaml`**

   ```yaml
   startDelaySeconds: 0
   rules:
     - pattern: "java.lang<type=Memory><>(.*)"
     - pattern: "java.lang<type=GarbageCollector,name=(.*)><>CollectionCount"
     - pattern: "java.lang<type=GarbageCollector,name=(.*)><>CollectionTime"
   ```

3. **Attach the agent when starting Tomcat or your Java app**

   ```bash
   java -javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent-0.20.0.jar=9404:/opt/jmx_exporter/config.yaml \
        -jar your-app.jar
   ```

For **Tomcat**, edit `TOMCAT_HOME/bin/setenv.sh`:

```bash
export CATALINA_OPTS="$CATALINA_OPTS -javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent-0.20.0.jar=9404:/opt/jmx_exporter/config.yaml"
```

Now visit:

```
http://<server-ip>:9404/metrics
```

You’ll see Prometheus-formatted JVM data like heap usage, GC counts, and thread stats.

---

## **3. Handling Multiple Servers**

Each Tomcat instance or Java server runs its own agent.
Prometheus scrapes them individually.

Example Prometheus configuration:

```yaml
scrape_configs:
  - job_name: legacy-app
    static_configs:
      - targets:
        - app01:9404
        - app02:9404
        - app03:9404
        - app04:9404
        - app05:9404
        labels:
          app: legacy-app
          env: prod
```

Each target becomes a unique `instance`. No merging happens at ingestion; Prometheus aggregates metrics during query time.

---

## **4. Aggregating in Grafana**

Prometheus labels let you visualize combined metrics easily.

* **Heap usage across all servers**

  ```promql
  sum by(app) (jvm_memory_bytes_used{area="heap", job="legacy-app"})
  ```

* **Average CPU time**

  ```promql
  avg by(app) (rate(process_cpu_seconds_total{job="legacy-app"}[5m]))
  ```

* **Per-server breakdown**

  ```promql
  rate(process_cpu_seconds_total{job="legacy-app"}[5m]) by (instance)
  ```

Grafana variables like `$instance` or `$app` can switch between per-host and global views.

---

## **5. When You Need More Than JVM Metrics**

JMX Exporter surfaces JVM and Tomcat internals, but not application-level KPIs (like API latency or business counters).
To go deeper:

### **Option A — Embed Micrometer**

Add Micrometer dependencies to your servlet or Struts2 app and expose `/metrics`.
Best if you can modify the app slightly.

### **Option B — Prometheus Java Client**

Use `simpleclient` to create counters and histograms manually.
Example:

```java
static final Counter requests = Counter.build()
  .name("web_requests_total").help("Total web requests").register();
```

### **Option C — OpenTelemetry Agent**

Attach the OpenTelemetry Java agent to collect metrics and traces together.
Useful if you plan to modernize later.

---

## **6. Security**

* Restrict access to the exporter port using firewalls or by binding to localhost:

  ```
  -javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent-0.20.0.jar=127.0.0.1:9404:/opt/jmx_exporter/config.yaml
  ```
* Let Prometheus be the only scraper.

---

## **7. Summary**

| Need                      | Best Option                     | Notes               |
| ------------------------- | ------------------------------- | ------------------- |
| JVM & Tomcat metrics only | JMX Exporter (javaagent)        | No code change      |
| Add custom app metrics    | Micrometer or Prometheus client | Small code change   |
| Unified metrics + traces  | OpenTelemetry agent             | Modernization-ready |

---

**In short:**
Even legacy Java apps can be observed like modern ones.
Attach the **JMX exporter**, configure **Prometheus** to scrape it, and visualize with **Grafana**—no rewrite required.

---

