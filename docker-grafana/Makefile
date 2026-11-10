.PHONY: up up-ebpf down reset ps logs check

up:            ## Chạy stack (không có eBPF)
	@test -f .env || cp .env.example .env
	docker compose up -d

up-ebpf:       ## Chạy stack + OBI (Linux)
	@test -f .env || cp .env.example .env
	docker compose --profile ebpf up -d

down:          ## Dừng, giữ dữ liệu
	docker compose --profile ebpf down

reset:         ## Dừng và XÓA toàn bộ dữ liệu
	docker compose --profile ebpf down -v

ps:
	docker compose --profile ebpf ps

logs:          ## make logs s=alloy
	docker compose logs -f --tail=100 $(s)

check:         ## Kiểm tra các thành phần đã sẵn sàng
	@printf "mimir   : "; curl -s -o /dev/null -w "%{http_code}\n" localhost:9009/ready
	@printf "loki    : "; curl -s -o /dev/null -w "%{http_code}\n" localhost:3100/ready
	@printf "tempo   : "; curl -s -o /dev/null -w "%{http_code}\n" localhost:3200/ready
	@printf "alloy   : "; curl -s -o /dev/null -w "%{http_code}\n" localhost:12345/-/ready
	@printf "grafana : "; curl -s -o /dev/null -w "%{http_code}\n" localhost:3000/api/health
	@printf "app     : "; curl -s -o /dev/null -w "%{http_code}\n" localhost:8081/actuator/prometheus || true
