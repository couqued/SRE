Prometheus Query(이하 PromQL)

1. CPU 리소스 조회
- CPU Average

```PromQL
1000 * avg_over_time(
    sum by (pod, namespace)(rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]))[$__range:]
)
```

```PromQL
1000 * avg_over_time(
    sum by (pod_base, namespace)(
        label_replace(
            rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]), "pod_base", "$1", "pod", "(.+)-[a-z0-9]{5,}$"))[$__range:]
)
```

- CPU Max

``` PromQL
1000 * max_over_time(
  sum by(pod, namespace)(
    rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]))[$__range:]
)
```
``` PromQL
1000 * max_over_time(
  sum by(pod_base, namespace)(
    label_replace(
      rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]), "pod_base", "$1", "pod", "(.+)-[a-z0-9]{5,}$"))[$__range:]
)
```

- CPU Min
``` PromQL
1000 * min_over_time(
  sum by(pod, namespace)(
    rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]))[$__range:]
)
```

``` PromQL
1000 * min_over_time(
  sum by(pod_base, namespace)(
      label_replace(
        rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]), "pod_base", "$1", "pod", "(.+)-[a-z0-9]{5,}$"))[$__range:]
)
```

- CPU Limit (현재 할당된 스펙)
``` PromQL
1000 * sum by (pod, namespace) (
    kube_pod_container_resource_limits{resource="cpu", unit="core", namespace=~"$namespace", pod=~"$pod"}
)
```

- CPU Max/Limit (최대 사용량 대비 현재 할당된 스펙)
``` PromQL
100 * 
    max_over_time(
        sum by(pod, namespace)(
            rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container",   container!=""}[1m]))[$__range:]
    )
    /
    sum by (pod, namespace) (
        kube_pod_container_resource_limits{resource="cpu", unit="core", namespace=~"$namespace", pod=~"$pod"}
    )
```

- cpu 몇% 사용했는지
``` PromQL
100 * (
    sum by (pod, namespace) (
        rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", image!=""}[$__range])
    )
    /
    sum by (pod, namespace) (
        kube_pod_container_resource_limits{resource="cpu", unit="core", namespace=~"$namespace", pod=~"$pod"}
    )
)
```

- ReplicaSet pod명 suffix 제거 변환 정규식
``` PromQL
1000 * avg by(pod_base, namespace)(
    avg_over_time(
        label_replace(
            rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]), "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$")[$__range:]
    )
)
```

- ReplicaSet, statefulset 변환 정규식
``` PromQL
1000 * avg by(pod_base, namespace)(
    avg_over_time(
        label_replace(
            label_replace(
                rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]
                ), "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
            ), "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
        )[$__range:]
    )
)
```

- pod suffix 변환 정규식 적용
``` PromQL
A
1000 * avg by(pod_base, namespace)(
    avg_over_time(
        label_replace(
            label_replace(
                rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]
                ), "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
            ), "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
        )[$__range:]
    )
)

B
1000 * max by(pod_base, namespace)(
    max_over_time(
        label_replace(
            label_replace(
                rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]
                ), "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
            ), "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
        )[$__range:]
    )
)

C
1000 * min by(pod_base, namespace)(
    min_over_time(
        label_replace(
            label_replace(
                rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]
                ), "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
            ), "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
        )[$__range:]
    )
)

D
1000 * sum by(pod_base, namespace)(
    label_replace(
        label_replace(
            kube_pod_container_resource_limits{resource="cpu", unit="core", namespace=~"$namespace", pod=~"$pod"},
            "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
        ), "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
    )
)
```

- 기간별 cpu 사용량 그래프
``` PromQL
1000 * 
sum by(pod_base, namespace)(
    label_replace(
        label_replace(
            rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]),"pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
            ), "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
        
    )
)
```

- 종료된 pod 정보 조회
``` PromQL
count by (reason) (
    kube_pod_container_status_last_terminated_reason{namespace=~"$namespace",pod=~"$pod"} == 1
)
```

``` PromQL
1000 * max by(pod_base, namespace)(
    max_over_time(
        label_replace(
            label_replace(
                rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]
                ), "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
            ), "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
        )[$__range:]
    )
)

1000 * max without(instance, job, pod)(
    max_over_time(
        label_replace(
            label_replace(
                rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]
                ), "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
            ), "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
        )[$__range:]
    )
)

1000 * 
        label_replace(
            label_replace(
                rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]),
                 "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
            ), 
            "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
    )

1000 * avg by(pod_base, namespace)(
    avg_over_time(
        label_replace(
            label_replace(
                rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]
                ), "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
            ), "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
        )[$__range:]
    )
)

1000 * max by(pod_base, namespace) (
  max_over_time(
    label_replace(
      label_replace(
        rate(container_cpu_usage_seconds_total{
          namespace=~"$namespace", 
          pod=~"$pod", 
          container=~"$container", 
          container!=""
        }[1m]),
        "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
      ),
      "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
    )[$__range:]
  )
)

1000 * max by (pod_base, namespace) (
  max_over_time(
    label_replace(
      label_replace(
        rate(container_cpu_usage_seconds_total{
          container!="", image!="", namespace=~"$namespace", pod=~"$pod"
        }[1m]),
        "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9]+$"
      ),
      "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
    )[$__range:]
  )
)

그래프
1000 * max by(pod_base, namespace) (
  max_over_time(
    label_replace(
      label_replace(
        rate(container_cpu_usage_seconds_total{
          namespace=~"$namespace", 
          pod=~"$pod", 
          container=~"$container", 
          container!=""
        }[1m]),
        "pod_base", "$1", "pod", "^([a-z0-9-]+)-[0-9-]+$"
      ),
      "pod_base", "$1", "pod", "^([a-z0-9-]+?)(-[a-z0-9]{4,}){1,2}$"
    )[$__range:]
  )
)

1000 * max by (pod) (
  max_over_time(
        rate(container_cpu_usage_seconds_total{
          container!="", image!="", namespace=~"$namespace", pod=~"$pod"
        }[1m])[$__range:]
  )
)
```

퍼센트 비율 색상 적용
Overrides
 - Frelds with name
 - Standard options > Unit : Percent (0-100)
 - Thresholds
 - Cell options > Cell type : Colored text
 - standard options > Color scheme : From thresholds (by value)

===============================================

- 컨테이너 종료되었을때 종료 사유 나타냄
``` PromQL
kube_pod_container_status_terminated_reason{reason="OOMKilled"} > 0
```

- 기동중인 컨테이너 정보
``` PromQL
kube_pod_status_phase{phase="Running"} == 1
```

``` PromQL
sum by (namespace, app) (
   increase(kube_pod_container_status_restarts_total{namespace="app-sample", app="app-name"}[1y]) > 0
)

increase(kube_pod_container_status_restarts_total{namespace="app-sample"}[7d]) > 0

sum by (namespace, label_app) (
  increase(kube_pod_container_status_restarts_total[7d])
  *
  on(pod, namespace)
  group_left(label_app)
  kube_pod_labels
)

increase(kube_pod_container_status_restarts_total{namespace="app-sample"}[7d])
  *
on(pod, namespace)
group_left(app)
kube_pod_labels

max by(pod, namespace, label_app) (
  kube_pod_container_status_restarts_total{namespace="app-sample"}
  *
  on(pod, namespace)
  group_left(label_app)
  kube_pod_labels{namespace="app-sample"}
)

max by(pod, namespace) (
  kube_pod_container_status_restarts_total{namespace="app-sample"}
)
```


- pod 재기동 현황
``` PromQL
max by(pod) (
  kube_pod_container_status_restarts_total{namespace="app-sample"} != 0
)

count by (phase) (
  kube_pod_status_phase{namespace="app-sample"}
)

kube_pod_status_phase{namespace="app-sample", phase="Running"}
```

- pod 운영 현황
``` PromQL
count by (phase) (
  kube_pod_status_phase{namespace="app-sample"} == 1
)

count(kube_pod_status_phase{namespace="app-sample", phase="Running"} == 1)

label_replace(vector(0), "phase", "Pending", "", "")
or
label_replace(vector(0), "phase", "Failed", "", "")
or
label_replace(vector(0), "phase", "Unknown", "", "")
or
label_replace(vector(0), "phase", "Running", "", "")
or
count by (phase) (kube_pod_status_phase{namespace="app-sample"} == 1)

label_replace(count_over_time(kube_pod_status_phase{namespace="app-sample", phase="Running"}[1d]), "phase", "Pending", "", "")

avg_over_time(probe_success{job="$http_job",instance="$target"}[$__interval]) * 100
```

---

1. Cpu Resource Usage
``` PromQL
100 * (
    sum by (pod, namespace) (
        rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", image!=""}[$__range])
    )
    /
    sum by (pod, namespace) (
        kube_pod_container_resource_limits{resource="cpu", unit="core", namespace=~"$namespace", pod=~"$pod"}
    )
)
```

2. Cpu Resource Usage
``` PromQL
1000 * sum by (pod, namespace) (
    rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", image!=""}[$__range])
)

1000 * sum by (pod, namespace) (
    kube_pod_container_resource_limits{resource="cpu", unit="core", namespace=~"$namespace", pod=~"$pod"}
)

100 * (
    sum by (pod, namespace) (
        rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", image!=""}[$__range])
    )
    /
    sum by (pod, namespace) (
        kube_pod_container_resource_limits{resource="cpu", unit="core", namespace=~"$namespace", pod=~"$pod"}
    )
)
```

3. 평균
``` PromQL
1000 * sum by (container)(
    avg_over_time(rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", image!=""}[1m])[10d:])
)

1000 * 
    avg_over_time(rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", image!=""}[1m])[10d:])

1000 * avg_over_time(
    sum by (container)(rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", container!=""}[1m]))[$__range:]
)
```

4. 최대
``` PromQL
1000 * max_over_time(
    sum by(container)(
    rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", image!=""}[1m]))[30m:]
)
```

5. 최소
``` PromQL
1000 * min_over_time(
  sum by(container)(
    rate(container_cpu_usage_seconds_total{namespace=~"$namespace", pod=~"$pod", container=~"$container", image!=""}[1m]))[5d:]
)
```
