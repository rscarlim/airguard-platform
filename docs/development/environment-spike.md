# AG-T02 — pré-requisitos e versões do laboratório

Spike da [issue #7](https://github.com/rscarlim/airguard-platform/issues/7), revisado em 06/10/2026 (`America/Sao_Paulo`). A dependência AG-T01 foi integrada ao `main` pelo PR #25. Escopo: verificar SDK, Docker e compatibilidade básica das imagens; registrar escolhas e impedimentos antes da implementação. Laboratório de uma instalação e um compressor, exclusivamente simulados.

## Decisões de versão

| Componente | Versão escolhida | Fonte oficial e justificativa |
| --- | --- | --- |
| SDK .NET | `10.0.401` | [Metadados de releases Microsoft](https://raw.githubusercontent.com/dotnet/core/main/release-notes/10.0/releases.json) e [download .NET 10](https://dotnet.microsoft.com/en-us/download/dotnet/10.0); SDK estável instalado e verificado |
| Runtime ASP.NET Core | `10.0.12` | Mesmos metadados Microsoft; runtime da linha .NET 10 compatível com o SDK escolhido |
| MQTT | `eclipse-mosquitto:2.0.22` | [Catálogo oficial da imagem](https://github.com/docker-library/official-images/blob/master/library/eclipse-mosquitto); linha 2.0 com suporte a MQTT 3.1.1/5, suficiente para o laboratório |
| Kafka | `apache/kafka:4.1.2` | [Downloads Apache](https://kafka.apache.org/community/downloads/) e [imagem oficial](https://kafka.apache.org/41/getting-started/docker/); release estável listado como suportado na consulta. Usar imagem JVM e KRaft; não adicionar ZooKeeper |
| Redis | `redis:7.4.11-bookworm` | [Catálogo oficial](https://github.com/docker-library/official-images/blob/master/library/redis); patch explícito da linha 7.4, evitando introduzir a linha 8 sem necessidade do laboratório |
| PostgreSQL com TimescaleDB | `timescale/timescaledb:2.30.2-pg17` (PostgreSQL `17.11`) | [Release TimescaleDB](https://github.com/timescale/timescaledb/releases/tag/2.30.2) e [repositório oficial da imagem](https://github.com/timescale/timescaledb-docker); imagem com extensão e PostgreSQL 17 empacotados juntos; patch 17.11 confirmado pelo binário |
| SDK em container (referência futura) | `mcr.microsoft.com/dotnet/sdk:10.0.401-noble` | [Imagem SDK Microsoft](https://github.com/dotnet/dotnet-docker/blob/main/README.sdk.md); Ubuntu 24.04, evitando depender de Alpine/musl nos futuros serviços |
| ASP.NET em container (referência futura) | `mcr.microsoft.com/dotnet/aspnet:10.0.12-noble` | [Imagem ASP.NET Microsoft](https://github.com/dotnet/dotnet-docker/blob/main/README.aspnet.md); mesma família de distribuição da imagem SDK |

`global.json` exige SDK `10.0.401`, com `rollForward: disable` e pré-releases desabilitados. Isso fixa a ferramenta deste repositório; não define TargetFramework nem aplica automaticamente aos demais repositórios. Atualizações de patch serão explícitas, com nova verificação e revisão. [Semântica oficial do global.json](https://learn.microsoft.com/en-us/dotnet/core/tools/global-json).

As imagens devem ser consumidas pelas próximas tarefas com tag explícita **e digest**, usando as referências abaixo. Nenhuma escolha usa a tag flutuante `latest`, tag somente de major ou de minor. O PostgreSQL empacotado em TimescaleDB não deve ser trocado por outra imagem nem atualizado isoladamente. A definição dos bancos e usuários por serviço é da AG-T04; este spike não cria bancos, usuários ou migrations. A necessidade de instâncias distintas será decidida nessa tarefa.

### Referências imutáveis verificadas para Linux amd64

Os digests abaixo são dos manifestos **da plataforma `linux/amd64`**, consultados com `docker manifest inspect`; não são os digests do índice multiarch. As tags podem ser republicadas; o digest mantém a seleção reproduzível. Outra arquitetura exige nova verificação e registro, sem emulação implícita.

```text
eclipse-mosquitto:2.0.22@sha256:cba7cd79d4e48b2e668b599c100bc5bb612a2c85aada76ab5a27987ecc81e3d7
apache/kafka:4.1.2@sha256:d50ab7b5df612b3c303f9d8afe8fee59626a5de798addfd626fe1924e3205965
redis:7.4.11-bookworm@sha256:cd745595f143052dd6a743bc5651d3ce4b03979fe5c99c7fcfab73461f6f217b
timescale/timescaledb:2.30.2-pg17@sha256:836e3eda797a144f6327b9a280c7c30368e4158ecb5f4bc0012aa3247c2b4018
mcr.microsoft.com/dotnet/sdk:10.0.401-noble@sha256:0eeb52c76e35a5431ca707ad2bc75e38006a05393045d8532ae44c15d9474523
mcr.microsoft.com/dotnet/aspnet:10.0.12-noble@sha256:0fa044f682cb7d93a5a90401a00c626c66f7b00b86922be9441f869eae039f80
```

## Pré-requisitos mínimos e orçamento inicial

- Sistema x64 com suporte a containers Linux, virtualização habilitada e rede para Docker Hub, Microsoft Artifact Registry e NuGet. Para Windows, usar Docker Desktop com backend WSL 2 e engine Linux ativo. No Linux nativo, usar Docker Engine com Compose plugin v2; esse caminho não foi executado neste spike.
- Para instalação/atualização no Windows, seguir os [requisitos atuais do Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/): WSL 2.1.5 ou superior, 8 GB de RAM e virtualização/SLAT. A edição Home permite containers Linux; não planejar containers Windows. Confirmar também a versão/edição do Windows suportada na documentação durante a instalação.
- SDK exato de `global.json`; o runtime sozinho não substitui o SDK. Editor é opcional, pois as verificações usam CLI. Git e Docker Compose v2 devem estar disponíveis.
- Baseline **observada**, não recomendação de versão antiga para novas instalações: Docker Desktop `4.26.1`, Engine/CLI `24.0.7`, Compose `2.23.3-desktop.2`. Essas versões responderam ao spike. Novas instalações devem usar uma versão estável suportada do Docker Desktop, registrando suas versões exatas e repetindo as verificações. Não há Compose de aplicação nesta issue; sua compatibilidade será verificada na AG-T03.
- Orçamento mínimo inicial **proposto para a futura stack**, ainda sem medição de carga: host com 4 processadores lógicos, 16 GiB de RAM e 20 GiB livres; disponibilizar ao engine ao menos 4 CPUs e 6 GiB. Os 20 GiB cobrem inicialmente imagens, cache e dados pequenos de laboratório, não retenção ilimitada. São premissas de planejamento, não mínimos oficiais nem garantia de desempenho. Medir e ajustar na AG-T03 antes de declarar ambiente saudável.

## Ambiente observado e decisões sobre limitações

| Item | Observado | Decisão |
| --- | --- | --- |
| Host | Windows 11 Home Single Language, x64, build `26200`; 8 processadores lógicos; aproximadamente 15,8 GiB de RAM; aproximadamente 199 GiB livres em C: | Usar containers Linux; memória atende aproximadamente ao orçamento de 16 GiB comercial |
| SDK/runtime | SDK `10.0.401`, ASP.NET Core e .NET Runtime `10.0.12` | Compatível com a escolha; não instalar outro SDK |
| Docker inicial | CLI presente; pipe do daemon ausente no contexto `default` | Não interpretar CLI instalada como engine disponível; reconsultar o contexto Linux após iniciar o Desktop |
| Docker após retomada | Desktop `4.26.1`; Engine `24.0.7`; servidor `linux/amd64`; contexto `desktop-linux` | Smoke test viável; comandos usam contexto explícito, sem trocar o padrão do usuário |
| WSL | `2.0.14.0`, kernel `5.15.133.1-1` | Abaixo do mínimo atual de 2.1.5; registrar divergência. Planejar atualização de WSL/Desktop antes da implementação do ambiente completo, em sessão própria; não atualizar sistema neste ticket |
| Recursos do engine | 2 CPUs; `4111699968` bytes de RAM (aproximadamente 3,8 GiB) | Abaixo do orçamento da futura stack; revisar alocação antes da AG-T03. Sucesso de um processo de versão não comprova capacidade para a stack |

Não foi feita alteração de WSL, BIOS, configuração de recursos, Docker Desktop ou contexto padrão. O retorno inicial de daemon indisponível foi superado; a divergência do WSL com requisitos atuais permanece documentada. Decisão antes da implementação: continuar apenas com verificações descartáveis neste spike e usar a AG-T03 para comprovar prontidão após tratar essas limitações. Não afirmar prontidão completa agora.

## Verificação reproduzível

Executar na raiz do repositório, com Docker Desktop/engine iniciado. Comandos de inspeção não iniciam serviços. Não é preciso fazer build de aplicação: ainda não há código nesta entrega.

```powershell
dotnet --info
dotnet --version
docker context ls
docker --context desktop-linux version
docker compose version
docker --context desktop-linux info --format '{{.OSType}}/{{.Architecture}} CPUs={{.NCPU}} MemoryBytes={{.MemTotal}}'
wsl --version
```

Esperado: SDK `10.0.401`, servidor Docker Linux amd64 acessível, Compose v2. Para reprovar um pré-requisito, registrar saída e decidir a correção antes de iniciar a stack. Em Linux nativo, omitir `wsl` e selecionar explicitamente o contexto apropriado.

Para conferir metadados, repetir `docker manifest inspect` para cada tag da tabela. Localizar `platform.os=linux`, `platform.architecture=amd64` e comparar o digest ao registro. Isso verifica publicação/arquitetura, não funcionamento da aplicação nem integração entre componentes.

Imagem descartável mínima, sem rede no processo, portas publicadas ou mounts do host:

```powershell
docker --context desktop-linux run --rm --network none --entrypoint /usr/sbin/mosquitto eclipse-mosquitto:2.0.22@sha256:cba7cd79d4e48b2e668b599c100bc5bb612a2c85aada76ab5a27987ecc81e3d7 -h
```

Esperado: saída `mosquitto version 2.0.22`, código 0 e remoção do container ao terminar. O download da imagem requer rede, apesar de o processo no container usar `--network none`. O cache da imagem permanece; não executar limpeza global de imagens, containers ou volumes do usuário.

Verificações adicionais de versão, com as referências imutáveis acima no lugar das tags:

```powershell
docker --context desktop-linux run --rm --network none --entrypoint redis-server redis:7.4.11-bookworm --version
docker --context desktop-linux run --rm --network none --hostname ag-t02-kafka --add-host ag-t02-kafka:127.0.0.1 --entrypoint /opt/kafka/bin/kafka-topics.sh apache/kafka:4.1.2 --version
docker --context desktop-linux run --rm --network none --entrypoint sh timescale/timescaledb:2.30.2-pg17 -c 'postgres --version; grep default_version /usr/local/share/postgresql/extension/timescaledb.control'
```

Esses comandos não iniciam brokers nem servidor de banco; não verificam publicação MQTT/Kafka, persistência, extensões carregadas em um banco ou conectividade de clientes .NET. `--rm` remove os containers e eventuais volumes anônimos da execução; não são criados volumes nomeados.

O hostname explícito do smoke test Kafka resolve localmente, sem rede externa. A primeira execução sem essa entrada retornou versão 4.1.2 e código 0, mas também erro Log4j `UnknownHostException`; a decisão é tornar o hostname resolvível no teste descartável e validar nomes/listeners da rede real na AG-T03. Não tratar código 0 sozinho como ausência de erros.

## Evidência de execução

| Entrada | Resultado esperado | Resultado obtido em 06/10/2026 |
| --- | --- | --- |
| `dotnet --info` e `dotnet --version` após global.json | SDK fixado e runtime .NET 10 | SDK `10.0.401`; runtimes `10.0.12`; seleção pelo global.json confirmada |
| `docker version` inicial | Cliente e servidor acessíveis | Cliente `24.0.7`, servidor inacessível por pipe ausente; limitação registrada |
| `docker --context desktop-linux version` após retomada | Engine Linux acessível | Engine `24.0.7`, Desktop `4.26.1`, Linux amd64 |
| `docker compose version`, `wsl --version`, `docker info` | Versões e recursos identificados | Compose `2.23.3-desktop.2`, WSL `2.0.14.0`, 2 CPUs e aproximadamente 3,8 GiB; divergências registradas acima |
| `docker manifest inspect` das seis imagens escolhidas | Publicação e plataforma Linux amd64 | Todos retornaram manifesto correspondente; digests registrados |
| Mosquitto descartável por tag `2.0.22`, `-h` | Versão 2.0.22, saída 0 | `mosquitto version 2.0.22`, saída 0; container removido. Índice baixado: `sha256:199ea8ef2e35ec2b1b37e59cfd1dbae538ed4dfa4a2251a121a52215a6248a21` |
| Redis descartável `--version` | Versão 7.4.11, saída 0 | `Redis server v=7.4.11`, 64 bits, saída 0 |
| Mosquitto descartável pela referência com digest | Mesma versão e saída 0 | `mosquitto version 2.0.22`, saída 0; digest de plataforma baixado igual ao registro |
| TimescaleDB descartável: `postgres --version` e arquivo de controle | PostgreSQL 17 e extensão 2.30.2 empacotados | `PostgreSQL 17.11`, `default_version = '2.30.2'`, saída 0; não foi carregada extensão em banco |
| Kafka descartável inicial sem hostname local | Versão 4.1.2 e saída 0 sem erros | Versão 4.1.2 e saída 0, mas erro de resolução do hostname; decisão registrada acima |
| Kafka com hostname local e digest fixado | Versão 4.1.2, saída 0 sem erro de hostname | `4.1.2`, saída 0, sem erro; digest de plataforma baixado igual ao registro |

Os quatro containers de verificação terminaram e foram removidos por `--rm`; não ficaram processos de serviço do spike. A limpeza de imagens realizada pelo usuário durante a sessão removeu tags do cache; isso não altera as saídas já observadas, e a repetição por digest de Mosquitto e Kafka voltou a baixar as referências fixadas. Não foi feita limpeza global pelo spike.

A evidência separa a versão declarada pela imagem, a arquitetura publicada e a execução de binários. Não constitui teste de integração nem validação de capacidade, resiliência ou segurança da futura stack.

## Critérios de aceitação e próximos passos

| Critério da issue | Evidência |
| --- | --- |
| Versões escolhidas com fontes oficiais | Tabela de decisões e links Microsoft, Apache, catálogos oficiais e Timescale |
| Imagens sem latest | Tags exatas e referências com digest para seis imagens |
| Incompatibilidade gera decisão antes da implementação | WSL antigo, recursos do engine e daemon inicial documentados com decisão explícita |
| Verificações de versão e imagem descartável | Registro acima, incluindo saída 0 de Mosquitto e Redis |

Uma pessoa com duas horas por dia: reservar uma sessão para inspeção, consulta das fontes, smoke test e registro. Downloads ou problemas do host podem exceder a sessão; não ampliar este ticket para resolver toda a máquina. As próximas tarefas implementam Compose (AG-T03), bancos/usuários (AG-T04) e contratos/clientes, com suas próprias validações.

Não há serviços .NET, Dockerfiles, Compose, rede de aplicação ou infraestrutura de produção neste PR. Antes de integrações, repetir verificações no host atualizado, medir recursos, validar inicialização saudável de cada componente e fixar versões dos clientes .NET e sua compatibilidade de protocolo nas tarefas responsáveis.
