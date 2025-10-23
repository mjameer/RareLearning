The process of collecting data and metrics for Prometheus involves two primary methods: direct instrumentation and using **Exporters** or the **Push Gateway**.

***

## 1. Collecting Metrics from Instrumented Applications

When you have the **source code** of an application (e.g., in Ruby, Python, Java, etc.), you can use a Prometheus **client library** directly within the application code. This allows the application to expose metrics that Prometheus can then **scrape** (pull) directly.

***

## 2. Collecting Metrics from Systems Without Source Code

For systems where you cannot modify the source code (e.g., databases like **MySQL**, third-party services like **HAProxy**, **CloudWatch**, or vast numbers of **IoT devices**), the approach is different:

### Exporters

* **Problem:** You can't directly add the Prometheus client library to these systems.
* **Solution:** Use a piece of software called an **Exporter**.
* **Function:** An Exporter is installed **next to** or **on** the system (e.g., on a Linux server, next to an IoT cluster) and is responsible for collecting metrics from that system and translating them into a format Prometheus can understand.
* **Scraping:** **Prometheus** then connects to the Exporter and **pulls** the metrics. This pulling action is called **scraping**.
* **Configuration:** Scraping is configured in the Prometheus config file, with a default interval of **15 seconds**.
* **Benefit:** This is the standard, scalable solution for monitoring complex infrastructure.

***

## 3. Handling Short-Lived Jobs and Asynchronous Pushes

Prometheus is fundamentally a **pull** time-series database; it does not accept data pushed directly to it. However, for applications or jobs that are **short-lived** (making them unavailable to be scraped) or that need to **asynchronously send data**, the **Push Gateway** is used:

### Push Gateway

* **Role:** The Push Gateway is a component that acts as a **temporary storage** buffer.
* **Process:**
    1.  The application **pushes** its metrics to the Push Gateway.
    2.  The Push Gateway has a **built-in Exporter**.
    3.  **Prometheus** then **scrapes** (pulls) the metrics from the Push Gateway's Exporter.
* **Key Point:** The Push Gateway *does not* change Prometheus into a push database; it merely acts as a go-between, maintaining Prometheus's core **pull-based** data collection architecture.
