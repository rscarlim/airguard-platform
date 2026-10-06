# AirGuard — cenários do teste de produto do MVP

Especificação da [AG-T01, issue #6](https://github.com/rscarlim/airguard-platform/issues/6). Entrega documental; os comportamentos abaixo são expectativas para tarefas futuras, ainda não executadas em um sistema.

## Objetivo e resultado da demonstração

Demonstrar, exclusivamente com dados simulados de uma instalação e um compressor, que o AirGuard pode apresentar indícios de operação fora da agenda com contexto suficiente para investigação. A demonstração será bem-sucedida quando cada entrada deste documento produzir o resultado esperado: operação normal e exceção autorizada sem incidente de operação fora do expediente; operação persistente sem autorização como suspeita; ausência de leituras como perda de comunicação. O investigador deverá conseguir distinguir evidência, hipótese e decisão humana, inclusive quando faltarem dados.

Nesta tarefa, sucesso significa fixar esses resultados de modo verificável por revisão da especificação. Não significa que a demonstração funcional já existe.

## Persona e decisão apoiada

Marina Costa, responsável fictícia pela manutenção, consulta agenda, potência, pressão e disponibilidade das leituras. Precisa decidir se deve investigar a operação observada, verificar uma autorização ou primeiro restabelecer a comunicação. Pode registrar uma conclusão como “operação necessária”, “agenda incorreta” ou “inspeção presencial necessária”, acompanhada de justificativa. Os sinais isolados não permitem diagnosticar vazamento, sua origem nem desperdício confirmado.

## Laboratório e premissas

| Elemento | Valor fictício |
| --- | --- |
| Instalação | Oficina Aurora; `site-lab-001` |
| Compressor | Compressor Aurora C01; `compressor-lab-001` |
| Potência nominal de referência | 7,5 kW; informação fictícia, sem uso para inferir potência real |
| Fuso da agenda e dos exemplos | `America/Sao_Paulo` |
| Expediente | Segunda a sexta, intervalo [08:00, 18:00); sem feriados nesta fixture |
| Sinais | Potência ativa em kW; pressão manométrica em bar; instante de observação com deslocamento UTC |
| Cadência | Uma leitura simulada por minuto, no início do minuto |

Todos os exemplos ocorrem na terça-feira 06/10/2026. Horários abreviados nas tabelas têm a data `2026-10-06` e o deslocamento `-03:00` (por exemplo, `2026-10-06T18:10:00-03:00`). Às 08:00 o expediente começa; às 18:00 já terminou. Não haverá dados de sensores reais nem ligação com equipamento físico.

Cada leitura contém instalação, compressor, instante observado, potência, pressão, origem `simulada` e natureza da potência. A fixture principal tem natureza `medida_simulada`: representa uma medição fictícia, não uma medição física. Potência estimada, se usada futuramente, deve ser identificada como `estimada_simulada`, junto de método e premissas, sem ser apresentada como medida. A potência nominal não substitui uma leitura e não permite preencher lacunas. Não se exige implementar estimadores nesta tarefa.

Parâmetros ilustrativos, sem validação industrial:

- Suspeita: potência **maior ou igual a 2,0 kW**, fora da agenda e sem exceção válida, em seis leituras consecutivas separadas por um minuto, abrangendo cinco minutos. A primeira leitura é o início da evidência; a sexta permite justificar a suspeita. Pressão contextualiza a investigação e não é condição nem prova de vazamento.
- Ausência: ao completar três minutos sem nova leitura, indicar perda de comunicação. Uma última leitura às 20:00 leva ao estado offline às 20:03 se nenhuma nova leitura chegar. Com leitura no instante limite, considerá-la antes de avaliar ausência.
- Continuidade: qualquer minuto faltante interrompe a sequência de suspeita. Não somar períodos separados para completar os cinco minutos. Leituras abaixo de 2,0 kW, início de período autorizado ou de expediente encerram o sinal suspeito em curso.
- Identidade: dentro de cada execução independente, uma sequência contínua produz um incidente de operação fora do expediente, sem duplicação por leitura. Essas expectativas orientam mecanismos futuros sem prescrever serviços, contratos ou infraestrutura.

Exceções são específicas à instalação e ao compressor, possuem intervalo [início, fim), motivo e responsável fictícios, e estão válidas antes da sequência ser avaliada. Ausência de exceção nos exemplos significa agenda completa conhecida, e não falha de consulta. Agenda indisponível ou desconhecida deve indicar classificação inconclusiva, sem afirmar operação não autorizada; sua política detalhada fica para validação futura.

## Cenários verificáveis

Cada cenário começa com histórico próprio e sem incidente aberto, salvo continuação explicitamente descrita. Intervalos de leituras incluem ambos os extremos; cada minuto indicado representa uma leitura efetiva, sem interpolação.

### C01 — Operação normal

**Contexto:** expediente regular, nenhuma exceção necessária.

| Instantes | Potência | Pressão | Entrada adicional |
| --- | --- | --- | --- |
| 08:00 a 08:05, a cada minuto | 6,0 kW | 7,0 bar | Agenda regular conhecida |
| 17:54 a 17:59, a cada minuto | 5,5 kW | 6,8 bar | Mesmo compressor |
| 18:00 | 0,4 kW | 6,6 bar | Primeiro minuto fora do expediente |

**Sequência e resultado esperado:** às 08:00 e até 17:59 a operação está autorizada pelo expediente. Nenhuma dessas leituras abre incidente de operação fora do expediente, independentemente de exceder 2,0 kW. Às 18:00 a leitura está fora da agenda, mas abaixo do limiar; também não abre incidente. Os dois blocos são recortes independentes para revisão; o intervalo não exibido não comprova comunicação contínua nem permite calcular consumo diário.

**Justificativa:** funcionamento no horário previsto não é evidência de operação irregular. O caso verifica também os limites inclusivo e exclusivo do expediente.

### C02 — Operação fora do expediente sem autorização

**Contexto:** agenda conhecida, sem exceção para o compressor.

| Instantes | Potência | Pressão | Entrada adicional |
| --- | --- | --- | --- |
| 18:10 a 18:15, a cada minuto | 3,0 kW | 6,2 bar | Origem e natureza conforme fixture principal |
| 18:16 | 0,3 kW | 6,1 bar | Sinal de potência abaixo do limiar |
| 18:20 | Sem nova evidência de operação | — | Marina registra conclusão e justificativa |

**Sequência e resultado esperado:** às 18:10 inicia a sequência; até 18:14 ainda não há duração suficiente. Às 18:15, seis leituras abrangem cinco minutos e justificam um único incidente de **suspeita de operação fora do expediente**, com evidência de 18:10 a 18:15, agenda e potência. O texto apresentado não afirma vazamento nem economia potencial confirmada. Pressão de 6,2 bar é contexto; não altera essa conclusão.

Às 18:16 o sinal suspeito termina, mas o incidente continua pendente de investigação. Às 18:20 Marina pode resolvê-lo com “inspeção presencial necessária; operação observada sem autorização registrada”, preservando evidência, autoria, instante e justificativa. Essa resolução encerra a análise humana deste incidente; não prova reparo, vazamento ou economia. A ação humana de 18:20 não é uma leitura e nada informa sobre comunicação desde 18:16.

**Justificativa:** persistência fora da agenda merece investigação. Potência e pressão também podem corresponder a outros usos ou problemas de agenda; diagnóstico requer outras evidências.

### C03 — Operação excepcional autorizada

**Contexto:** teste de manutenção registrado previamente por Marina. Exceção fictícia `exception-lab-001`, para `site-lab-001` e `compressor-lab-001`, válida das 18:00 às 19:00; motivo “teste de manutenção”.

| Instantes | Potência | Pressão | Entrada adicional |
| --- | --- | --- | --- |
| 18:10 a 18:15, a cada minuto | 3,0 kW | 6,2 bar | Exceção válida e conhecida |
| 18:59 | 3,0 kW | 6,2 bar | Último minuto autorizado |
| 19:00 a 19:05, a cada minuto | 2,0 kW | 6,0 bar | Sem outra exceção |

**Sequência e resultado esperado:** o bloco de 18:10 a 18:15 reproduz C02 com autorização: não abre incidente. Às 18:59 ainda há autorização. Às 19:00 a exceção terminou; inicia nova sequência, sem contar minutos autorizados. Às 19:05 há seis leituras no limiar e cinco minutos de persistência: abre uma suspeita de operação fora do expediente. O intervalo não exibido entre os blocos não sustenta afirmação de continuidade nem consumo total.

**Justificativa:** uma autorização válida impede abertura indevida, mas não autoriza operação indefinidamente. O caso verifica fim exclusivo da exceção e igualdade ao limiar.

### C04 — Equipamento offline

**Contexto:** fora do expediente, sem exceção; nenhuma suspeita prévia neste cenário.

| Instantes | Potência | Pressão | Entrada adicional |
| --- | --- | --- | --- |
| 20:00 | 0,4 kW | 6,5 bar | Última leitura recebida |
| 20:01 a 20:09 | Ausente | Ausente | Nenhuma leitura recebida |
| 20:10 | 0,4 kW | 6,4 bar | Comunicação retomada |

**Sequência e resultado esperado:** às 20:01 e 20:02 há lacuna, ainda sem atingir o prazo de offline. Às 20:03 indicar perda de comunicação e exibir a última leitura com seu instante, sem apresentá-la como atual. Não abrir incidente de operação fora do expediente com base na ausência. Às 20:10 indicar retomada da comunicação. O intervalo sem observações permanece desconhecido; não deve ser preenchido com zero nem com 0,4 kW.

**Justificativa:** ausência de leitura informa indisponibilidade de observação, não o estado físico do compressor. Não prova consumo zero, desperdício ou vazamento. O alerta de comunicação é distinto de uma suspeita de operação.

### C05 — Lacuna durante indício de operação

**Contexto:** fora da agenda, sem exceção, com agenda conhecida.

| Instantes | Potência | Pressão | Entrada adicional |
| --- | --- | --- | --- |
| 21:00 a 21:02, a cada minuto | 3,0 kW | 6,2 bar | Três leituras |
| 21:03 | Ausente | Ausente | Minuto faltante |
| 21:04 a 21:09, a cada minuto | 3,0 kW | 6,2 bar | Seis novas leituras |
| 21:10 a 21:12 | Ausente | Ausente | Comunicação interrompida |

**Sequência e resultado esperado:** 21:03 interrompe a continuidade. Às 21:04 reinicia a sequência; não abrir suspeita às 21:05 somando blocos. Às 21:09 abrir uma suspeita com evidência de 21:04 a 21:09. Às 21:12 indicar offline (última leitura às 21:09); o incidente continua pendente de investigação. A lacuna torna a continuidade do sinal desconhecida: não registrar cessação comprovada nem resolver automaticamente o incidente.

**Justificativa:** intervalos sem dados não demonstram persistência nem término físico da operação. Perda de comunicação e investigação podem coexistir.

## Lacunas, consumo e ciclo do incidente

Não extrapolar consumo ou custo silenciosamente. As fixtures não especificam integração de energia nem tarifa: não apresentar totais em kWh, valores em R$ ou economia calculada nesta demonstração. Qualquer evolução que calcule energia deverá explicitar método, cobertura, natureza medida ou estimada, período e lacunas; cálculo de custo também exigirá tarifa e premissas. Dado ausente permanece ausente, mesmo após retomada.

Separar o estado da evidência (persistente, cessou por leitura/agenda, ou continuidade desconhecida por lacuna) do estado da investigação (pendente ou resolvida por pessoa com justificativa). Encerramento do sinal não resolve o incidente; resolução humana não altera o histórico dos sinais nem garante que a causa foi eliminada. Política de recorrência após resolução e agrupamento de episódios será definida futuramente.

## Critérios de aceitação e rastreabilidade

| Critério | Resultado verificável | Cenários/evidência |
| --- | --- | --- |
| CA01 — Normal não abre incidente | Nenhum incidente de operação fora do expediente em período regular | C01, inclusive 08:00 e 17:59 |
| CA02 — Fora do expediente é hipótese | Suspeita após persistência; nenhuma afirmação de vazamento confirmado | C02, C03 após 19:00, C05 |
| CA03 — Dados exclusivamente simulados | Identidades fictícias, origem simulada e natureza explícita em todas as leituras | Premissas e C01–C05 |
| CA04 — Exceção válida é respeitada | Nenhum incidente no período excepcional autorizado | C03, contraste com C02 |
| CA05 — Offline não significa zero | Ausência distinta de leitura e de desperdício; retomada preserva lacuna | C04 e C05 |
| CA06 — Entradas e saídas verificáveis | Instantes, valores, agenda e decisões suficientes para comparar resultados | Tabelas e sequências C01–C05 |
| CA07 — Lacunas não são extrapoladas | Sequência reiniciada, consumo/custo desconhecidos sem cálculo implícito | C05 e seção de lacunas |
| CA08 — Resolução humana é independente | Cessação e offline não resolvem investigação automaticamente | C02 e C05 |
| CA09 — Orientação sem implementação | Expectativas de negócio documentadas, sem serviços ou infraestrutura | Este documento e diff da entrega |

## Roteiro e evidência da revisão manual

1. Confirmar escopo de uma instalação, um compressor e origem simulada; localizar CA03 em todas as fixtures.
2. Conferir data, dia da semana, deslocamento UTC, unidades e limites [08:00, 18:00) e [18:00, 19:00).
3. Percorrer cada tabela minuto a minuto: contar seis leituras em cinco minutos; verificar ausência de abertura antecipada, igualdade a 2,0 kW e reinício após lacuna.
4. Comparar C02 e C03 no mesmo bloco de 18:10 a 18:15: somente a autorização muda a classificação.
5. Conferir offline às 20:03 e 21:12 a partir da última leitura. Verificar que potência/pressão ausentes não viram zero ou valores atuais.
6. Verificar linguagem de hipótese, separação entre sinal e decisão humana, ausência de totais de consumo/custo e cobertura de CA01–CA09.
7. Conferir link do README e diff: somente documentação; registrar divergências antes de aprovar.

**Registro da revisão da especificação em 06/10/2026:** as entradas das tabelas foram confrontadas com os parâmetros e resultados textuais. Resultado obtido na revisão: C01 sem abertura; C02 com limiar de persistência às 18:15 e cessação distinta de resolução; C03 autorizado até 18:59 e suspeita às 19:05; C04 offline às 20:03; C05 reinício às 21:04, suspeita às 21:09 e offline às 21:12. Horários, contagens e unidades são consistentes. A matriz cobre os critérios obrigatórios e os casos adicionais. Esta é evidência de revisão documental, não de execução ou teste de comportamentos em software. Não há necessidade de build nem testes automatizados artificiais para esta entrega.

## Plano de trabalho e limites

Para uma pessoa com duas horas por dia: uma sessão documental indicativa de 20 minutos de leitura e delimitação, 60 minutos de redação, 25 minutos de revisão e 15 minutos de registro/PR. Se acesso ou revisão exigirem mais tempo, continuar no próximo dia sem ampliar o escopo. A demonstração funcional depende de tarefas posteriores; não cabe nesta sessão.

A simulação não comprova economia real, acurácia industrial, diagnóstico de vazamentos, retorno financeiro ou validação de mercado. Não modela ciclo de carga/alívio, todos os modos de falha, calibração, consumo produtivo ou sazonalidade.

Fora do escopo: hardware, sensores reais, comercialização, controle remoto do compressor e implementação dos demais tickets. Não implementar microserviços .NET 10, MQTT, Kafka, Redis, bancos por serviço, DDD, observabilidade ou resiliência nesta entrega.

## Premissas e dúvidas para validação futura

- Validar persona, fluxo de decisão e motivos aceitos para resolução com profissionais de manutenção.
- Calibrar limiar, persistência, cadência e prazo offline com dados reais e modos de operação; os valores atuais só servem ao laboratório.
- Definir feriados, turnos, mudanças de fuso, exceções sobrepostas, autorização retroativa e governança da agenda.
- Definir política para leituras atrasadas, duplicadas, fora de ordem, relógios divergentes, qualidade inválida e agenda indisponível. Fixtures atuais pressupõem ordem, pontualidade, unicidade e valores válidos.
- Validar origem e qualidade de potência medida/estimada, métodos de energia, cobertura mínima, tarifas e apresentação de incerteza antes de mostrar consumo ou custo.
- Definir recorrência, agrupamento, responsabilização e evidências adicionais para diagnóstico. Operação fora do expediente, por si só, permanece hipótese de necessidade de investigação.
