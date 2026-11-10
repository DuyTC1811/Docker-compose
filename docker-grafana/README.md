# docker-grafana: Observability stack cho Spring Boot

Grafana LGTM (**L**oki, **G**rafana, **T**empo, **M**imir) kết hợp Alloy và OBI (eBPF), dùng lại được cho nhiều dự án.

```
                      ┌────────────── Alloy :12345 ──────────────┐
App :8081/actuator ──►│ scrape (targets/*.json) ─► remote_write  │──► Mimir :9009 ──┐
App/OBI ──OTLP──────► │ :4317 / :4318 ───────────► otlp          │──► Tempo :3200 ──┤──► Grafana :3000
App log file ───────► │ tail ─► parse JSON ─► mask PII           │──► Loki  :3100 ──┘
                      └──────────────────────────────────────────┘
               Tempo ──span metrics / service graph──► Mimir
```

## Cấu trúc thư mục

```
docker-grafana/
├── docker-compose.yml          # stack dev
├── .env.example                # cấu hình theo dự án (log dir, version, port)
├── Makefile                    # make up / up-ebpf / down / reset / check
├── mimir/mimir.yaml
├── loki/loki.yaml
├── tempo/tempo.yaml
├── alloy/
│   ├── config.alloy            # pipeline metrics + traces + logs
│   └── targets/spring-apps.json  ← SỬA FILE NÀY cho từng dự án
├── obi/obi-config.yml          # eBPF auto-instrumentation
├── grafana/
│   ├── provisioning/           # datasource (đã liên kết metric↔trace↔log) + dashboard provider
│   └── dashboards/spring-boot-overview.json
├── spring-boot/                # copy vào app
│   ├── application-observability.yml
│   ├── build.gradle.snippet
│   └── ObservabilityConfig.java
└── k8s/                        # bản production (Helm values)
```

## Dùng cho một dự án mới (5 bước)

**1. Chạy stack**
```bash
cd docker-grafana
cp .env.example .env            # sửa APP_LOG_DIR thành đường dẫn tuyệt đối, ví dụ /home/duy/logs
make up                         # hoặc: make up-ebpf (Linux, có OBI)
make check                      # mọi thành phần phải trả 200
```

**2. Khai báo app cần scrape** trong `alloy/targets/spring-apps.json` (Alloy tự reload sau khoảng 30 giây):
```json
[
  { "targets": ["host.docker.internal:8081"], "labels": { "job": "spring-boot", "application": "fx-disbursement" } },
  { "targets": ["host.docker.internal:9081"], "labels": { "job": "spring-boot", "application": "e-invoice" } }
]
```

**3. Thêm dependency vào app**: xem `spring-boot/build.gradle.snippet`.

**4. Cấu hình app**: copy `spring-boot/application-observability.yml` vào `src/main/resources/`, rồi trong `application.yml`:
```yaml
spring:
  application:
    name: fx-disbursement        # = label application = thư mục log
  profiles:
    include: observability
```
Đặt biến môi trường `APP_LOG_DIR` cho app (Run Configuration trong IntelliJ) **trùng** giá trị trong `.env`.

**5. Mở Grafana** tại http://localhost:3000, vào thư mục *Spring Boot* và mở dashboard *Spring Boot – Overview*.

## Đi từ metric → trace → log

| Từ | Thao tác | Đến |
|---|---|---|
| Panel *Latency p95/p99* | Bấm vào chấm exemplar | Trace trong Tempo |
| Trace (Tempo) | Bấm *Logs for this span* | Log có cùng `traceId` trong Loki |
| Dòng log (Loki) | Mở log details, bấm *Xem trace* | Trace trong Tempo |
| Tempo → *Service Graph* | | Sơ đồ service gọi nhau, dựng từ trace |

Truy vấn mẫu trong Explore:
```promql
# Mimir: p95 theo endpoint
histogram_quantile(0.95, sum by (le, uri) (rate(http_server_requests_seconds_bucket{application="fx-disbursement"}[5m])))
```
```logql
# Loki: lỗi của một service, kèm trace_id
{service_name="fx-disbursement", level="ERROR"} | json
```
```traceql
# Tempo: request chậm hơn 1s
{ resource.service.name = "fx-disbursement" && duration > 1s }
```

## Trace: chọn MỘT trong hai cách (tránh trùng span)

| | A. Micrometer Tracing (khuyến nghị) | B. OBI eBPF |
|---|---|---|
| Sửa code | Thêm dependency | Không |
| `traceId` trong log | Có | Không |
| Exemplar trên metric | Có | Không |
| Span nghiệp vụ (`@Observed`) | Có | Không, chỉ HTTP/gRPC/SQL |
| Chạy | `make up` | `make up-ebpf` và **bỏ** `management.otlp.tracing.endpoint` trong app |

Có thể chạy OBI chỉ để lấy metric: xóa `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` trong service `obi`.

## Quy ước để không làm hỏng Mimir/Loki

- **Không** gắn tag/label có giá trị không giới hạn (`userId`, `txnId`, `accountNo`). `ObservabilityConfig` đã chặn sẵn tag `uri` ở 100 giá trị.
- Log: `service_name` và `level` là label; `trace_id` là structured metadata.
- Alloy che sẵn dãy 12–19 chữ số (số tài khoản/thẻ) và email. Chỉnh regex trong `stage.replace` cho phù hợp nghiệp vụ.

## Xử lý sự cố

| Triệu chứng | Kiểm tra |
|---|---|
| Dashboard không có dữ liệu | http://localhost:12345, component `prometheus.scrape.spring_apps` phải health; `curl localhost:8081/actuator/prometheus` |
| `up = 0` | App chưa mở port 8081, hoặc firewall chặn `host.docker.internal` (Linux: `sudo ufw allow from 172.16.0.0/12`) |
| Không có log | Đường dẫn log phải là `${APP_LOG_DIR}/<service>/*.log`; `docker compose logs alloy` |
| Không có exemplar | App phải có `micrometer-tracing-bridge-otel` và `percentiles-histogram` = true |
| Tempo/Loki báo permission denied | `make reset` rồi `make up` (service `init` sẽ chown lại volume) |
| OBI không chạy | Cần Linux kernel ≥ 5.8 có BTF (`ls /sys/kernel/btf/vmlinux`) |

## Spring Boot 4

- `spring-boot-starter-aop` đổi thành `spring-boot-starter-aspectj`.
- Một số key `management.otlp.*` / `management.tracing.*` đã đổi tên. Đối chiếu với Spring Boot 4 migration guide, hoặc chạy app với `spring-boot-properties-migrator` để được cảnh báo key cũ.

## Lên production

Xem thư mục `k8s/`. Thứ tự triển khai: **S3/MinIO → Mimir/Loki/Tempo → Grafana (SSO Keycloak) → Alloy DaemonSet → app**.

```bash
kubectl create ns observability
kubectl -n observability create secret generic observability-s3 \
  --from-literal=S3_ACCESS_KEY=... --from-literal=S3_SECRET_KEY=...
helm repo add grafana https://grafana.github.io/helm-charts && helm repo update
helm upgrade --install mimir   grafana/mimir-distributed -n observability -f k8s/mimir-values.yaml
helm upgrade --install loki    grafana/loki              -n observability -f k8s/loki-values.yaml
helm upgrade --install tempo   grafana/tempo-distributed -n observability -f k8s/tempo-values.yaml
helm upgrade --install grafana grafana/grafana           -n observability -f k8s/grafana-values.yaml
helm upgrade --install alloy   grafana/alloy             -n observability -f k8s/alloy-values.yaml
```

Khác biệt chính so với bản dev: lưu trữ trên object storage, replication 3, multi-tenant (`X-Scope-OrgID`), Grafana đăng nhập qua SSO và lưu DB trên Postgres, log đọc từ stdout của pod. App chỉ cần annotation `prometheus.io/*` (xem `k8s/app-deployment-example.yaml`).

> Key trong Helm values có thể thay đổi giữa các version chart. Luôn đối chiếu với `helm show values <chart>` trước khi cài.
