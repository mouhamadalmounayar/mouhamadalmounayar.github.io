---
layout: "post"
author: "Al Mounayar Mouhamad"
date: 2025-07-30
title: "Building a custom prometheus exporter"
tags: "prometheus"
---

The need to track business-specific metrics often arises in real-world scenarios. For instance, I was recently tasked with monitoring CO2 emissions for Kubernetes pods using an internal time-based API. Developing a custom Prometheus exporter was the natural solution that came to mind. In this article, I’ll guide you through building your own custom exporter to handle similar challenges.

In a multi-cluster environment, a best practice is to have a Prometheus instance per cluster which collects metrics exposed by pods. Some organizations take this a step further and have an even more centralized Prometheus instance that collects metrics across all the clusters.

Developing an exporter simply involves exposing metrics in a format that Prometheus can understand and collect. In my specific case, the values for the metrics I wanted to expose were only accessible on demand via an internal api which provides a value per timepoint. Thus, I needed to schedule calls to this api at a certain frequency.

The Prometheus exporter I implemented queries the API every 30 minutes and exposes metrics on the /metrics endpoint in a format that a Prometheus instance can scrape.

The python Prometheus client library provides a very easy and quick way t o create and expose metrics. Here is an example of a metric I exposed representing the energy consumed by a pod at a certain timepoint.

```python
energy_consumed_total = Gauge(
    "energy_consumed_total", "Total energy consumed", ["pod_name", "namespace"]
)
```

The first argument passed to the `Gauge` constructor is the metric name. When naming metrics, it's best to follow Prometheus naming conventions. The second argument is a brief description of the metric, while the third is an array of label names, which allows the metric to be queried with more granularity. The following code block demonstrates how this Prometheus metric is exposed. It follows the standard format: `<metric_name>{<labels>} <value>`.

```
energy_consumed_total { pod_name = pod_example, namespace = namespace_example } 10
```

Next, let’s take a look at the fetch_and_update_metrics() function, which task is to query the api and update the Gauge metric.

```python
def fetch_and_update_metrics() -> None:
    try:
        api_url, query_params, headers = get_config()
        logging.info(f"Fetching metrics from API: {api_url}")
        response = session.get(api_url, params=query_params, headers=headers)
        data = response.json()
        for _, apps in data.items():
            for _, ns_pods in apps.items():
                for namespace, pods in ns_pods.items():
                    for pod in pods:
                        name = pod["name"]
                        total_energy_consumed = pod.get("energy_consumed", 0)
                        energy_consumed_total.labels(
                            pod_name=name, namespace=namespace
                        ).set(total_energy_consumed[-1])

    except Exception as e:
        logging.error(f"Error fetching or updating metrics: {e}")
```

In this function, we query the API URL to retrieve a data object, parse the response, and update the metric using the set() method.

Finally, in the main part of my script, I start an HTTP server using the `start_http_server()` function provided by the library and schedule the above function to run every 30 minutes. As mentioned earlier, the metrics will be available at the /metrics endpoint.

```python
if __name__ == "__main__":
    logging.info("Starting prometheus server on port 8080.")
    start_http_server(8080)

    while True:
        logging.info("Fetching and updating metrics...")
        fetch_and_update_metrics()
        time.sleep(1800)
```

The exporter can be deployed virtually anywhere and added as a target in your Prometheus configuration. Once set up, Prometheus will scrape the metrics, which can then be queried and visualized in Grafana.
