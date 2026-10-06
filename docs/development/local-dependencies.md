# AG-T03 — MQTT, Kafka e Redis no laboratório local

[Issue #8](https://github.com/rscarlim/airguard-platform/issues/8). Escopo: três dependências no Compose, volumes, portas de host em loopback, credenciais fictícias e readiness. Não inclui bancos, serviços .NET, simulador, contratos de eventos ou observabilidade da aplicação. Os testes abaixo usam somente mensagens sintéticas.

## Iniciar e observar readiness

Na raiz do repositório, com Docker Desktop e engine Linux disponíveis, executar uma única instrução:

```powershell
docker --context desktop-linux compose up -d --wait --wait-timeout 240
```

O Compose usa defaults explícitos de laboratório; não exige copiar `.env.example`. O primeiro download precisa de rede e pode exceder o tempo de startup, pois o timeout de espera é aplicado à prontidão dos containers. Em Linux nativo, usar o contexto Linux apropriado no lugar de `desktop-linux`. Imagens fixadas por tag e digest para `linux/amd64`, conforme [AG-T02](environment-spike.md); outra arquitetura exige revisão.

Para alterar portas ou credenciais fictícias, copiar o exemplo **somente se `.env` ainda não existir**, editar e repetir o comando de início (ele recria containers cuja configuração mudou):

```powershell
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
```

`.env` é ignorado pelo Git. Usar apenas credenciais fictícias nesta configuração local; defaults, exemplo e health checks não representam gestão de segredos de produção. As senhas devem ser valores simples não vazios; manter o arquivo privado se posteriormente adaptado para outro contexto.

```powershell
docker --context desktop-linux compose ps
docker --context desktop-linux compose logs --tail 50
docker --context desktop-linux stats --no-stream
```

Esperado: serviços `mqtt`, `kafka` e `redis` com status `healthy`. `up --wait` retorna sucesso somente após os três estarem saudáveis; em falha, retorna código diferente de zero. `running` sozinho não demonstra readiness. Para diagnóstico detalhado, consultar `.State.Health` via `docker inspect` do ID retornado por `compose ps -q <serviço>`.

| Serviço | Health check | O que comprova |
| --- | --- | --- |
| MQTT | Publicação autenticada em `lab/health`, QoS 1 | Broker aceita conexão e confirma publicação MQTT; não comprova ingestão pela aplicação |
| Kafka | Listagem de tópicos pelo listener interno | Broker responde à API; não comprova pipeline de consumidores |
| Redis | `PING` autenticado e comparação exata com `PONG` | Servidor aceita autenticação e comando; não comprova lógica de cache da aplicação |

O prazo inicial do Kafka é maior por envolver JVM e eleição KRaft. O probe usa heap de 32–128 MiB separado do heap de 256–512 MiB do broker; as duas JVMs compartilham o limite de 1 GiB do container. Os três serviços são independentes e não precisam de `depends_on`; consumidores futuros deverão respeitar readiness e ter suas próprias políticas de reconexão.

## Endereços e credenciais de laboratório

| Serviço | Cliente no host (defaults) | Cliente na rede deste Compose | Autenticação |
| --- | --- | --- | --- |
| MQTT | `127.0.0.1:1883` | `mqtt:1883` | Usuário `airguard-lab`; senha pública fictícia `airguard-mqtt-lab-only`; anônimo desabilitado |
| Kafka | `localhost:19092` | `kafka:9092` | PLAINTEXT, **sem usuário/senha e sem SASL** neste laboratório |
| Redis | `127.0.0.1:6379` | `redis:6379` | Senha pública fictícia `airguard-redis-lab-only`; usuário Redis padrão `default` |

Todas as portas publicadas têm bind `127.0.0.1`; não usar `0.0.0.0` nem endereço de LAN. O controller Kafka em `9093` e o listener interno `9092` não são publicados no host. Os listeners dentro dos containers precisam aceitar conexões da rede Compose; isso é distinto do bind das portas do host. A rede é exclusiva ao projeto `airguard-lab`, mas não impede processos locais com acesso ao Docker de conectar containers a ela.

O listener Kafka `HOST` anuncia `localhost:<KAFKA_PORT>` para clientes no host, e `INTERNAL` anuncia `kafka:9092` para futuros containers na mesma rede. Ao alterar `KAFKA_PORT` em `.env`, mudam o bind de host e o endereço anunciado, mantendo o listener do container em 19092. Cliente em outro container não deve usar `localhost:19092`, pois esse endereço aponta para ele próprio.

O ID KRaft fixo `MkU3OEVBNTcwNTJENDM2Qk` é um identificador de laboratório, não uma credencial. Há um nó combinando broker/controller e fator de replicação 1; não existe alta disponibilidade. TLS e autenticação Kafka não foram configurados para esta rede local. Essa escolha não autoriza exposição em LAN/produção.

MQTT gera seu arquivo de senha com `mosquitto_passwd` no startup, a partir das variáveis do Compose; não há hash estático a sincronizar manualmente com `.env`. O broker usa o usuário `mosquitto`. Redis usa o entrypoint oficial para iniciar como usuário `redis`; Kafka mantém `appuser` (UID 1000).

## Dados, recursos e ciclo de vida

| Volume do projeto | Conteúdo | Política |
| --- | --- | --- |
| `airguard-lab_mqtt-data` | Persistência Mosquitto (por exemplo, mensagens retidas) | Salva periodicamente a cada 30 segundos e no encerramento normal |
| `airguard-lab_kafka-data` | Logs e metadados KRaft | Montado em `/var/lib/kafka/data`, diretório gravável do usuário da imagem; retenção nominal de 24h |
| `airguard-lab_redis-data` | Dados Redis/AOF | AOF habilitado, fsync a cada segundo; até cerca de 1 segundo pode ser perdido em falha abrupta |

Persistência demonstrada abaixo é para encerramento/recriação normal; não equivale a backup nem garantia de durabilidade em pane. Kafka tem segmentos de 16 MiB e retenção de 24h; a retenção depende também do fechamento dos segmentos e não limita rigorosamente o espaço de disco. Redis limita dados a 128 MiB com `noeviction` (recusa novas escritas quando necessário). MQTT não implementa histórico de telemetria. Contratos de confirmação e persistência da aplicação serão definidos em outros tickets.

Limites de container: MQTT 128 MiB/0,25 CPU; Kafka 1 GiB/1 CPU; Redis 256 MiB/0,5 CPU. Logs Docker rotacionados em dois arquivos de 5 MiB por container. Esses valores destinam-se a testes pequenos; não são capacidade industrial nem garantia para a futura stack completa.

Para reiniciar e esperar novamente pelos health checks:

```powershell
docker --context desktop-linux compose restart
docker --context desktop-linux compose up -d --wait --wait-timeout 240
```

Para parar e remover containers/rede, preservando os volumes nomeados:

```powershell
docker --context desktop-linux compose down
```

Novo `up` reaproveita os volumes. **Reset destrutivo e voluntário:** executar o comando abaixo somente se quiser apagar todos os dados deste laboratório (inclusive mensagens, metadados e chaves). Não é etapa obrigatória de startup, validação ou revisão:

```powershell
docker --context desktop-linux compose down --volumes
```

Não executar `docker system prune` ou limpeza global como parte deste roteiro. As imagens Kafka e MQTT também declaram volumes anônimos de configuração/log; o reset com `--volumes` remove os anônimos ligados aos containers além dos nomeados declarados no Compose. Volumes anônimos de containers removidos anteriormente sem `--volumes` podem permanecer no Docker; não contêm os dados persistentes do laboratório e não são alvos de limpeza automática deste ticket.

## Roteiro de validação funcional e de persistência

Usar somente fixtures de laboratório; o tópico MQTT `lab/ag-t03-smoke`, a chave Redis `lab:ag-t03-smoke` e o tópico Kafka `lab.ag-t03-smoke` são probes de infraestrutura, não contratos da aplicação. Com defaults ou `.env`, os comandos MQTT/Redis usam as credenciais do próprio container. Executar no PowerShell na raiz, um passo por vez e conferir saídas/códigos.

1. Validar configuração com `docker --context desktop-linux compose config --quiet`; subir com o comando único e verificar os três `healthy`.
2. Publicar mensagem retida MQTT e recebê-la:

```powershell
docker --context desktop-linux compose exec -T mqtt sh -ec 'mosquitto_pub -h 127.0.0.1 -u "$MQTT_USERNAME" -P "$MQTT_PASSWORD" -q 1 -t lab/ag-t03-smoke -m ag-t03-persisted -r; mosquitto_sub -h 127.0.0.1 -u "$MQTT_USERNAME" -P "$MQTT_PASSWORD" -q 1 -t lab/ag-t03-smoke -C 1 -W 10'
```

3. Escrever e ler Redis (esperado `OK` e `ag-t03-persisted`):

```powershell
docker --context desktop-linux compose exec -T redis sh -ec 'export REDISCLI_AUTH="$REDIS_PASSWORD"; redis-cli SET lab:ag-t03-smoke ag-t03-persisted; redis-cli GET lab:ag-t03-smoke'
```

4. Criar tópico Kafka explicitamente, produzir e consumir a mensagem:

```powershell
docker --context desktop-linux compose exec -T -e 'KAFKA_HEAP_OPTS=-Xms32m -Xmx128m' kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server kafka:9092 --create --if-not-exists --topic lab.ag-t03-smoke --partitions 1 --replication-factor 1
'ag-t03-persisted' | docker --context desktop-linux compose exec -T -e 'KAFKA_HEAP_OPTS=-Xms32m -Xmx128m' kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server kafka:9092 --topic lab.ag-t03-smoke
docker --context desktop-linux compose exec -T -e 'KAFKA_HEAP_OPTS=-Xms32m -Xmx128m' kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:19092 --topic lab.ag-t03-smoke --partition 0 --offset earliest --max-messages 1 --timeout-ms 15000
```

Esperado: mensagem `ag-t03-persisted` e um registro consumido. O comando de consumo verifica também o listener `HOST` dentro do broker, distinto do teste de TCP a partir do Windows. Inspecionar metadados com `kafka-broker-api-versions.sh --bootstrap-server localhost:19092`; o broker anunciado deve ser `localhost:19092` nos defaults. Não foi executado cliente Kafka nativo no Windows ou serviço .NET.

5. Verificar rejeição de acessos sem autenticação: `compose exec -T mqtt mosquitto_pub -h 127.0.0.1 -t lab/ag-t03-unauth -m rejected` deve falhar; `compose exec -T redis redis-cli ping` deve retornar `NOAUTH`. O redis-cli pode retornar código 0 para erro do servidor: conferir a resposta, não só o código. Nos exemplos abreviados deste passo, manter o prefixo `docker --context desktop-linux`.
6. Inspecionar os bindings Docker dos containers: todos os `HostIp` publicados devem ser `127.0.0.1`; abrir conexões TCP a partir do host para as três portas. Isso confirma alcance loopback, sem afirmar que foi executado teste a partir de outra máquina na LAN.
7. Reiniciar e esperar saúde. Repetir somente as leituras de MQTT, Redis e Kafka, sem republicar/escrever: as três devem retornar `ag-t03-persisted`.
8. Executar `down` **sem `--volumes`**, depois `up --wait`; repetir somente as leituras novamente. Isso verifica persistência nos volumes após remoção dos containers, não só no filesystem de um container reiniciado.
9. Após registrar as saídas, remover somente as fixtures criadas neste roteiro: mensagem retida MQTT com publicação `-n -r`, chave Redis com `DEL lab:ag-t03-smoke` e tópico Kafka com `kafka-topics.sh --delete --topic lab.ag-t03-smoke`. Não apagar fixtures de outras pessoas. A rede e os volumes permanecem prontos para as próximas tarefas.

Se uma porta estiver ocupada, escolher outra no `.env`, sem encerrar processos existentes. Em falha de saúde, consultar logs e health check; corrigir a causa antes de declarar prontidão. Alterar credenciais exige novo `up`, pois `restart` sozinho não aplica mudanças de configuração. Não alterar o cluster ID com dados Kafka existentes.

## Evidência e limites do host

Data: 06/10/2026, `America/Sao_Paulo`. Docker Desktop 4.26.1, Engine 24.0.7, Compose 2.23.3, servidor Linux amd64, 2 CPUs e aproximadamente 3,8 GiB. WSL permanece na versão 2.0.14 documentada pela AG-T02, abaixo do requisito oficial atual. A decisão desta tarefa é validar somente as três dependências compactas no engine já operacional; atualização de WSL/Desktop e dimensionamento da stack completa continuam pendências de ambiente. Não alterar configurações do host neste PR.

As versões/digests são os escolhidos na AG-T02. Fontes para as configurações: [Mosquitto](https://mosquitto.org/man/mosquitto-conf-5.html), [exemplo Apache Kafka 4.1.2](https://github.com/apache/kafka/blob/4.1.2/docker/examples/docker-compose-files/single-node/plaintext/docker-compose.yml), [Dockerfile Kafka e usuário do diretório de dados](https://github.com/apache/kafka/blob/4.1.2/docker/jvm/Dockerfile), [persistência Redis](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/) e [readiness no Compose](https://docs.docker.com/compose/how-tos/startup-order/).

O orçamento da AG-T02 era uma hipótese para a futura stack completa, não mínimo comprovado destes três processos. Nesta tarefa, snapshot durante probes mostrou aproximadamente MQTT 1,4 MiB, Redis 8,2 MiB e Kafka 421 MiB; uso varia e inclui probes/transientes. Não é benchmark, teste de longa duração, carga, recuperação de pane ou comprovação de capacidade para todos os serviços futuros.

### Registro da execução

| Entrada | Esperado | Resultado obtido |
| --- | --- | --- |
| Configuração Compose | Configuração válida | `config --quiet`, saída 0 |
| Startup com volumes novos | Três dependências saudáveis com uma instrução | MQTT, Kafka e Redis `healthy`; `up --wait`, saída 0 |
| MQTT autenticado, publicação retida QoS 1 e assinatura | Mesmo payload | `ag-t03-persisted`, saída 0 |
| Redis autenticado, SET/GET | OK e mesmo valor | `OK`, `ag-t03-persisted`, saída 0 |
| Kafka, criação de tópico e produção/consumo | Um registro com mesmo payload | Tópico criado, `ag-t03-persisted`, um registro consumido, saída 0 |
| Listener Kafka HOST | Metadados anunciados para host | `localhost:19092`, broker id 1 não fenced; API consultada com sucesso |
| Acesso anônimo MQTT/Redis | Recusa explícita | MQTT `not authorised`, saída 5; Redis `NOAUTH Authentication required` |
| Bindings e TCP de host | Publicação somente em loopback e alcance local | Conexões TCP para 1883, 19092 e 6379 em 127.0.0.1 bem-sucedidas; bindings inspecionados |
| Reinício seguido de espera | Três healthy e dados preservados | Três healthy; leituras retornaram o payload original sem novas escritas |
| Down sem volumes, novo up e releitura | Dados persistentes fora dos containers antigos | Três healthy; mensagem MQTT, chave Redis e registro Kafka originais lidos sem republicar/escrever |
| Inspeção final de containers | Bindings locais, volumes corretos, processos sem root | HostIp 127.0.0.1 nas três portas; volumes de dados nos destinos esperados; configuração MQTT/Redis somente para leitura; UIDs 1883/999/1000; OOMKilled false e RestartCount 0 nos containers recriados |
| Configuração com `.env.example` e limpeza das fixtures | Exemplo válido; remover somente dados do roteiro | Configuração válida; publicação retida removida, DEL retornou 1 e tópico Kafka ausente na listagem posterior |

Durante a preparação, o primeiro diretório Kafka usado em `/tmp` não permitiu gravar metadados no volume. A configuração final usa o diretório gravável `/var/lib/kafka/data`, mantendo o broker sem root; a inicialização passou. O entrypoint MQTT foi ajustado para não tentar alterar o arquivo de configuração montado somente para leitura. Esses problemas foram corrigidos antes de registrar sucesso; não ficaram ignorados por health checks.

Critérios da issue: startup único demonstrado; readiness explícita demonstrada; credenciais fictícias e ausência de autenticação Kafka declaradas. Nenhum banco ou serviço .NET foi implementado. Validação é de infraestrutura executada com fixtures sintéticas; não testa os cenários de negócio da AG-T01.

Para uma pessoa com duas horas por dia, planejar uma sessão de configuração e startup (até 60 minutos), validação/reinício/persistência (até 40 minutos) e registro/PR (20 minutos). Downloads e diagnóstico podem exceder essa hipótese; tratar a causa sem ampliar para bancos, serviços ou atualização completa da máquina. Ao final desta execução, as três dependências foram deixadas em funcionamento; `compose down` permite pará-las preservando os dados.
