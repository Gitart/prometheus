**Prometheus**

Для сбора метрик с использованием Golang используются библиотеки из репозитория [prometheus](https://github.com/prometheus/client_golang)

```go
// библиотека со всеми наборами метрик
import "github.com/prometheus/client_golang/prometheus"
```

* `Counter` (счетчик) - хранит значения, которые увеличиваются с течением времени
* `Gauge` (шкала) - хранит значения, которые с течением времени могут как увеличиваться, так и уменьшаться.
* `Histogram` (гистограмма) - хранит информацию об изменении некоторого параметра в течение определённого промежутка.
* `Summary` (сводка результатов) - как и гистограмма, хранит информацию об изменении значения некоторого параметра за временной интервал, но также позволяет рассчитывать квантили для скользящих временных интервалов.

Для каждого типа есть свой набор опций все они встраивают в себя структуру Opts.
```go
type Opts struct {
    // Namespace, Subsystem, и Name являются частью полного 
    // имени метрики (имя создается путем разделения полей через "_"). 
    // Только поле Name является обязательным, остальные поля просто 
    // помогают структурировать имя. Следует обратить внимание, 
    // что полное имя должно быть корректным именем метрики для Prometheus.
    Namespace string
    Subsystem string
    Name      string

    // Help представляет информацию о метрике. Рекомендуется указывать,
    // для информативности
    // Метрики с похожими полными именами ДОЛЖНЫ иметь такой же Help
    Help string
    ...
}
```
`Opts` объединяет опции для инициализации большинства типов метрик. Каждая реализация метрики XXX имеет свой собственный тип XXXOpts, но в большинстве случаев это просто псевдоним этого типа. 

**Рекомендация**
Для лучшего понимания, какие имеются поля структуры `Opts` для каждого типа, рекомендуется ознакомиться с каждой структурой в отдельности. Например у метрики типа `Histogram` есть поле позволяющее указывать размеры бакетов.

**Создание и изменение Counter**
```go
var promRegisteredUsersCount = prometheus.NewCounter(
    prometheus.CounterOpts{
        Name: "registered_users_count",
        Help: "Количество зарегистрированных пользователей",
    },
)
/*
# HELP registered_users_count Количество зарегистрированных пользователей
# TYPE registered_users_count counter
registered_users_count 5.7
*/

// инициализируем
prometheus.MustRegister(promRegisteredUsersCount)

...
promRegisteredUsersCount.Inc()     // увеличение на единицу
promRegisteredUsersCount.Add(4.7)  // увеличение на заданную величину
```

**Создание и изменение Gauge**

```go
var UsersSessionsGauge = prometheus.NewGauge(
    prometheus.GaugeOpts{
        Name: "users_sessions_gauge",
        Help: "Количество пользовательских сессий",
    },
)
/*
# HELP users_sessions_gauge Количество пользовательских сессий
# TYPE users_sessions_gauge gauge
users_sessions_gauge 1.58442934998837e+09
*/

// инициализируем
prometheus.MustRegister(UsersSessionsGauge)

...
UsersSessionsGauge.Set(5.5) // установка значения
UsersSessionsGauge.Inc()    // увеличение на единицу
UsersSessionsGauge.Dec()    // уменьшение на единицу
UsersSessionsGauge.Add(1.5) // увеличение на заданную величину
UsersSessionsGauge.Sub(0.5) // уменьшение на заданную величину

// Дополнительные методы
UsersSessionsGauge.SetToCurrentTime() // установить текущее время (Unix time) в секундах

```

**Создание и изменение Histogram**

```go
var ResponseLatencySecGauge = prometheus.NewHistogram(
    prometheus.HistogramOpts{
        Name: "request_duration_seconds",
        Help: "Продолжительность выполнения запроса",
        // Можно указать свой тип бакета или оставить значение по умолчанию
        // Buckets: []float64{0.5, 10, 20},
        // Buckets: ExponentialBuckets(100, 1.2, 3),
        // Buckets: LinearBuckets(-15, 5, 6),
    },
)
/*
# HELP request_duration_seconds Продолжительность выполнения запроса
# TYPE request_duration_seconds histogram
request_duration_seconds_bucket{le="0.005"} 0
request_duration_seconds_bucket{le="0.01"} 0
request_duration_seconds_bucket{le="0.025"} 0
request_duration_seconds_bucket{le="0.05"} 0
request_duration_seconds_bucket{le="0.1"} 0
request_duration_seconds_bucket{le="0.25"} 0
request_duration_seconds_bucket{le="0.5"} 0
request_duration_seconds_bucket{le="1"} 0
request_duration_seconds_bucket{le="2.5"} 2
request_duration_seconds_bucket{le="5"} 2
request_duration_seconds_bucket{le="10"} 2
request_duration_seconds_bucket{le="+Inf"} 2
request_duration_seconds_sum 3.5001457279999997
request_duration_seconds_count 2
*/

// инициализируем
prometheus.MustRegister(ResponseLatencySecGauge)

// Задержка ответа
startedAt := time.Now()
...
elapsed := time.Since(startedAt)

ResponseLatencySecGauge.Observe(float64(elapsed) / float64(time.Second))

// указание значения вручную
ResponseLatencySecGauge.Observe(2.5) // секунды
```

**Создание и изменение Summary**

```go
var ResponseLatencySecSummary = prometheus.NewSummary(
    prometheus.SummaryOpts{
        Name: "request_latency_sec",
	Help: "Время задержки обработки запроса в секундах",
    },
)
/*
# HELP request_latency_sec Время задержки обработки запроса в секундах
# TYPE request_latency_sec summary
request_latency_sec{quantile="0.5"} 1.000199219
request_latency_sec{quantile="0.9"} 2.5
request_latency_sec{quantile="0.99"} 2.5
request_latency_sec_sum 3.5001992189999998
request_latency_sec_count 2
*/

// инициализируем
prometheus.MustRegister(ResponseLatencySecSummary).

// Задержка ответа
startedAt := time.Now()
...
elapsed := time.Since(startedAt)

ResponseLatencySecSummary.Observe(float64(elapsed) / float64(time.Second))

// указание значения вручную
ResponseLatencySecSummary.Observe(2.5) // секунды
```


**Создание и изменение метрики с метками**

Рассмотрим как работать с метриками у которых имеются метки. Для этого потребуется из обычной метрики сделать вектор. Для этого у каждого типа есть `NewXXXVec` (где `XXX` тип метрики Prometheus).

```go
var RequestsTotalCounter = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "requests_total",
	Help: "HTTP Failures",
    },
    []string{"method", "endpoint"},
)
/*
# HELP requests_total HTTP Failures
# TYPE requests_total counter
requests_total{endpoint="/",method="GET"} 1
requests_total{endpoint="/submit",method="POST"} 1
*/

// инициализируем
prometheus.MustRegister(RequestsTotalCounter)

// устанавливаем значение
RequestsTotalCounter.With(
    prometheus.Labels{
        "method": "GET",
        "endpoint": "/",
    },
).Inc()

// устанавливаем значение (потоконебезопасно)
RequestsTotalCounter.WithLabelValues("POST", "/submit").Inc()
```

Похожим образом метки работают и с другими метриками
