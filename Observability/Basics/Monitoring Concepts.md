---

# Monitoring in System Design

This document provides an overview of system monitoring, its core principles, and common methodologies for collecting metrics.

---

## What is Monitoring?

Monitoring is the process of regularly collecting and visualizing data about a system's runtime health. It aims to answer three fundamental questions:

1.  **Is the service on?** (Availability)
2.  **Is the service functioning as expected?** (Correctness)
3.  **Is the service performing well?** (Performance)

For example, when monitoring a website like `savethekoala.com`:
* **Is it on?**: A successful `HTTP GET` request that returns a `200 OK` status indicates the service is available.
* **Is it functioning?**: Keeping the number of Python errors below a certain threshold (e.g., 5 per minute) can indicate the service is functioning correctly.
* **Is it performing well?**: Ensuring the response time for an `HTTP GET` request is under a specific limit (e.g., 20ms) can confirm good performance.

The data collected for this purpose is called **telemetry data**. Its primary goal is to help pinpoint *where* a problem might be in a complex system, not necessarily to diagnose the exact cause.

---

## Key DevOps Metrics

Two critical metrics are used to measure DevOps success in incident management:

* **Mean Time To Detection (MTTD):** The average time it takes from the moment an issue begins until the team detects it.
* **Mean Time To Resolve (MTTR):** The average time it takes from when an issue is detected until it is fixed and the system is operating normally again.

---

## Monitoring Layers and Methodologies

A typical microservices-based application can be broken down into three layers, each with its own monitoring methodologies:

1.  **UI Layer:** The website and mobile applications.
2.  **Service Layer:** The microservices themselves (e.g., payment service, promotion service).
3.  **Infrastructure Layer:** The underlying resources like CPU, memory, network, and disk.

``

Four common methodologies are used to collect metrics across these layers:

| Method                | Primary Layer             | Description                            |
| :-------------------- | :------------------------ | :------------------------------------- |
| **RED Method** | Service Layer             | A request-driven approach.             |
| **USE Method** | Infrastructure Layer      | A resource-oriented approach.          |
| **Four Golden Signals** | Service & Infrastructure  | An extension of the RED method.        |
| **Core Web Vitals** | UI Layer                  | Exclusively for website user experience. |

### The RED Method

The RED method focuses on the service layer and is request-driven.

* **R - Rate:** The number of requests per second the service is handling (throughput).
* **E - Errors:** The number of failed requests (e.g., HTTP 500 errors).
* **D - Duration:** The time it takes to process a request (latency).

### The USE Method

The USE method is resource-oriented and focuses on the infrastructure layer.

* **U - Utilization:** The percentage of a resource that is busy (e.g., CPU utilization).
* **S - Saturation:** The degree to which a resource has extra work it can't service, often measured by queue length. An ideal saturation is zero.
* **E - Errors:** The count of error events for a resource (e.g., disk write errors).

### The Four Golden Signals

Introduced in Google's Site Reliability Engineering (SRE) handbook, this method is essentially the **RED method plus Saturation**. It suggests that if you can only measure four metrics for a user-facing system, you should focus on these:

1.  **Latency** (Duration)
2.  **Traffic** (Rate)
3.  **Errors**
4.  **Saturation**

### Core Web Vitals

<img width="1293" height="692" alt="image" src="https://github.com/user-attachments/assets/4ed001f1-ddc9-4a1d-a9ee-e1695fcb84b6" />

This Google-introduced methodology is exclusively for the UI layer (websites) and is crucial for Search Engine Optimization (SEO).

* **Largest Contentful Paint (LCP):** Measures *perceived page load speed*. It marks the point when the page's main content has likely loaded.
* **First Input Delay (FID):** Measures *perceived responsiveness*. It quantifies the experience users feel when trying to interact with an unresponsive page.
* **Cumulative Layout Shift (CLS):** Measures *perceived visual stability*. It helps quantify how often users experience unexpected layout shifts.
