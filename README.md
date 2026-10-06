# airguard-platform
AirGuard MVP — infraestrutura local, contratos de eventos, arquitetura e documentação da plataforma IoT de monitoramento de ar comprimido.

## Especificação do teste de produto

[Cenários e resultados esperados do teste do MVP](docs/product/mvp-test-scenarios.md) — laboratório com uma instalação, um compressor e dados exclusivamente simulados (AG-T01).

## Pré-requisitos e versões

[Spike do ambiente de laboratório](docs/development/environment-spike.md) — SDK .NET 10 fixado em `global.json`, imagens com versões e digests, verificações executadas e limitações do host (AG-T02).

## Dependências locais

Na raiz do repositório, com Docker Desktop/engine Linux iniciado:

```powershell
docker --context desktop-linux compose up -d --wait --wait-timeout 240
```

Sobe MQTT, Kafka e Redis com portas publicadas somente em `127.0.0.1`, volumes e health checks. Usa credenciais públicas fictícias de laboratório; `.env.example` documenta os defaults e pode ser copiado opcionalmente para `.env`.

[Instruções, endpoints, credenciais, reinício, persistência e evidências](docs/development/local-dependencies.md) (AG-T03). Não usar essa configuração em produção.
