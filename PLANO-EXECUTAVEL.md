# TSM EX — Plano Executável Consolidado

> Consolida a tese deste conselho (TESE-TECHLOG.md) com a tese do segundo conselheiro
> ("TransMesquita 2030+", out/2026) em UM plano. Onde os conselhos divergem, a divergência
> está nomeada e decidida aqui. Data: outubro/2026.

---

## 0. Parecer sobre a tese do segundo conselheiro

### Convergências — ratificadas, não discutir mais

As duas teses chegaram, por caminhos independentes, às mesmas conclusões. Isso encerra o debate de visão:

1. **O produto é o compromisso cumprido com comprovação**, não o frete nem o software.
2. **Outcome-first (Sequoia): o sistema executa trabalho, humano trata exceção** — subida gradual, nunca salto.
3. **Lonngren fica** como motor fiscal/financeiro até existir paridade comprovada. Não se reconstrói fiscal agora.
4. **Começar na malha que já dominamos**: São Paulo/Guarulhos → Maranhão (Imperatriz, São Luís) + distribuição regional.
5. **Eventos auditáveis + máquina de estados por processo** (não um campo `status` único) + evidência para cada fato crítico.
6. **IA propõe; regras verificam; autorizações limitam; evidências confirmam.** Nunca autoridade final sobre fiscal.
7. **Exceção com dono, prioridade, prazo e desfecho** — alerta sem ação é só mais uma tarefa.
8. **Não vender software/plataforma agora.** Primeiro provar na própria operação.
9. **Métricas antes de otimização**: OTIF, divergências/1.000 volumes, minutos de coordenação por remessa, margem por remessa/rota/cliente.
10. **Stack enxuto**: PostgreSQL bem modelado, fila assíncrona, APIs versionadas, evidências armazenadas. Sem microsserviços prematuros.

### Ajustes — onde este conselho diverge ou aperta o sequenciamento

**A1. "Promessa calculada antes de aceita" (Capacidade A dele) não é ponto de partida.**
Calcular promessa com base em capacidade real e histórico é L2/L3 da régua de autonomia — exige dados que ainda não existem estruturados. No início, promessa = **tabela de prazo por cobertura de hub** (regra declarada, conservadora). A promessa calculada entra quando o grafo de eventos tiver meses de dados. Classificação: **Depois**.

**A2. Consolidação/manifesto multi-pedido é o coração econômico e precisa ser critério de aceite, não detalhe.**
Na tese dele a consolidação aparece como etapa do plano (Capacidade C), mas o ganho de margem da TransMesquita vem de ocupação de veículo — é A métrica que paga o projeto. O manifesto multi-pedido com ocupação por peso/cubagem é entregável central da Fase 2, com meta medida. Classificação: **Agora**.

**A3. Risco não nomeado por nenhuma das teses: hoje existem TRÊS sistemas em potencial.**
Lonngren (rodando), o TSM que você já começou a construir, e o "Logistics OS" proposto. **Dois sistemas de registro operacional = morte por divergência de dados.** Decisão obrigatória antes de qualquer linha nova de código: auditar o que você construiu (checklist na seção 4) e decidir — ele **é** o núcleo e evolui, ou vira protótipo de referência e se descarta. Não existe terceira opção "vai convivendo".

**A4. Cadeia de custódia por volume: começar nos 2 pontos de maior perda, não na operação inteira.**
Etiquetar e conferir todo volume em todo ponto é mudança de processo físico — cara, lenta, gera resistência. Começar onde a divergência dói: **carregamento da transferência tronco** e **descarga no hub destino**. Expandir depois com dado na mão. (A tese dele já aponta nessa direção nos dias 41–70; aqui vira regra explícita.)

**A5. Stack360 / Revenue Orchestrator: integra por identidade, não agora.**
A visão do ciclo comercial-operacional é correta e fica registrada. Mas conectar CRM/inteligência comercial ao núcleo operacional antes do núcleo existir é dispersão. Classificação: **Depois** — pré-requisito: Fase 2 concluída. Única exigência agora: **IDs de empresa/pessoa consistentes** para a integração futura não exigir migração.

**A6. O "serviço de abastecimento coordenado" (seção 11 dele): Futuro.**
Excelente tese comercial, muda a posição competitiva — e só é vendível quando o sistema provar OTIF alto na rota piloto. Vender isso antes é prometer o que a operação ainda não sustenta (erro clássico pré-Convoy).

---

## 1. Resposta à pergunta central: "parte de um TMS? o fluxo validado é um certo?"

**Sim — com uma correção de enquadramento.**

O fluxo validado é invariante e ninguém vai reinventá-lo:

> Pedido → Coleta → Consolidação no CD → Manifesto/Transferência tronco → Recebimento no hub destino → Distribuição regional → Entrega → Comprovação → Faturamento

As entidades de um TMS são o esqueleto certo: **Pedido, Volume, Hub, Cobertura, Link, Rota, Manifesto, Trecho, Evento** (modelo já definido na skill do projeto). Nisso, não se inventa a roda.

A correção: **a base é um núcleo de TMS, mas construído como sistema de execução, não como sistema de cadastro.** A diferença concreta, nos três pontos que separam um do outro:

| TMS tradicional (Lonngren, Aleff, SSW) | Núcleo TSM EX |
|---|---|
| Campo `status` que alguém atualiza | **Máquina de estados** com transições válidas; estado contraditório é impossível |
| Registro do que o humano fez | **Evento imutável com porquê**: quem decidiu, qual era a sugestão, se foi aceita/alterada, evidência |
| Tela espera alguém lembrar | **Fila de próxima ação**: o sistema sabe qual é o próximo passo válido de cada pedido e cobra o responsável |

Ou seja: ~70% do modelo de dados é TMS clássico (e deve ser — é o fluxo validado). Os 30% que diferenciam — eventos com porquê, estados, filas, custódia, exceções — custam pouco a mais **se entrarem no desenho desde o dia 1** e são impossíveis de enxertar depois. É por isso que a auditoria do que você já construiu (seção 4) decide tudo.

---

## 2. Decisões congeladas

Estas sete decisões não se rediscutem até o fim do piloto (dia 100):

- **D1.** Produto = compromisso logístico cumprido com comprovação. O sistema existe para produzir isso.
- **D2.** **Um único sistema de registro operacional.** Lonngren permanece como motor fiscal/financeiro/documental até paridade comprovada. O TSM EX nunca duplica o que o Lonngren já resolve bem — integra ou importa.
- **D3.** O fluxo validado é o esqueleto. Entidades: as nove já definidas. Não se cria entidade nova sem passar pelo critério da skill (ajuda a consolidar, transferir, distribuir, medir?).
- **D4.** Evento com porquê desde o dia 1 (decisão de arquitetura inegociável).
- **D5.** Máquina de estados por processo (coleta, transferência, remessa, distribuição, documento fiscal — ciclos independentes).
- **D6.** IA propõe → regra verifica → humano autoriza o crítico → evidência confirma. Nenhum modelo generativo tem autoridade final sobre fiscal, financeiro ou baixa de entrega.
- **D7.** Rota piloto: **Guarulhos → Imperatriz / São Luís + distribuição regional**. Nenhuma funcionalidade entra no escopo se não servir a essa rota.

---

## 3. O plano de 100 dias consolidado

Funde o 100 dias do segundo conselheiro com o roadmap de 5 fases da skill. Regra herdada dele, ratificada: **nenhuma fase avança porque o software ficou pronto — avança com evidência de uso operacional, dados consistentes e ganho medido.**

### Fase 0 — Diagnóstico e decisão de núcleo (dias 1–15)

Três frentes em paralelo:

| Frente | Trabalho | Entrega |
|---|---|---|
| **a) Auditoria do TSM que você já construiu** | Rodar o checklist da seção 4 contra o código/modelo | Decisão: evolui como núcleo OU vira referência descartada |
| **b) Diagnóstico da execução real** | ~50 remessas reais SP→MA no Lonngren, incluindo atraso, avaria, coleta frustrada, divergência, entrega parcial. Mapear onde há intervenção manual, perda de evidência, custo extra | Mapa de eventos reais + linha de base das métricas |
| **c) Capacidade de integração do Lonngren** | Verificar API, webhooks, exportações, acesso ao banco | Estratégia de integração/importação definida |

**Gate de saída (dia 15):** decisão de núcleo tomada + baseline numérica publicada (OTIF atual, minutos de coordenação/remessa, divergências, ocupação média). Sem baseline, nenhum ganho futuro é demonstrável.

### Fase 1 — Núcleo de rede + verdade operacional (dias 16–45)

- Entidades implementadas com máquina de estados e evento-com-porquê: Pedido, Volume, Hub, **Cobertura**, Link, Rota.
- **Cobertura de hub carregada de verdade** (cidades/prazos reais dos hubs MA) — sem isso não existe orquestração.
- Importação/integração dos dados do Lonngren **sem interferir nos fluxos existentes** (shadow mode: o TSM EX espelha, não comanda ainda).
- **Fila de consolidação por rota/hub destino** funcionando: pedido entra → sistema classifica → cai na fila certa.
- Painel único: onde está cada pedido, o que falta de informação, o que está contraditório.

**Critérios de aceite:** 100% dos pedidos da rota piloto visíveis no painel com hub origem/destino e fila atribuída automaticamente; operador consegue apontar dado faltante em < 1 min.

### Fase 2 — Manifesto, custódia e exceções (dias 46–75)

- **Manifesto multi-pedido**: seleção a partir da fila, cálculo de peso/cubagem/valor/ocupação, atribuição de Link, **status em lote** (manifesto saiu → todos os pedidos em trânsito).
- **Custódia nos 2 pontos críticos**: conferência por volume (etiqueta/QR) no carregamento da transferência e na descarga do hub destino. Divergência (esperado 12, conferido 11) **bloqueia o encerramento silencioso** e abre exceção.
- **Fila única de exceções**: problema, responsável, prioridade, próximo passo, prazo, desfecho.
- App/fluxo de doca funcionando **offline-first** (conectividade precária é a realidade da rota).

**Critérios de aceite:** um manifesto real montado no sistema em menos tempo que o processo atual (medir); ocupação por peso e cubagem calculada em 100% dos manifestos da rota; primeira divergência detectada pelo sistema antes de reclamação externa.

### Fase 3 — Distribuição regional + orquestrador v0 + primeiras automações (dias 76–100)

- Recebimento no hub destino → fila de distribuição → manifesto regional ou entrega direta → comprovação.
- **Orquestrador v0 por regras**: sugere hub destino, fila, agrupamento de manifesto, link elegível — **sempre com justificativa**, operador aceita ou altera (e a alteração vira dado).
- **2–3 automações de coordenação** escolhidas pelo diagnóstico da Fase 0 (ex.: confirmar coleta com fornecedor, cobrar atualização de parceiro, avisar cliente de risco de atraso confirmado). Primeiro assistidas, depois autônomas sob regra.
- Comparação planejado × realizado por manifesto (custo, prazo, ocupação) alimentando o painel.

**Critérios de aceite:** uma operação real ponta a ponta (pedido → entrega comprovada) rodando inteira no sistema; taxa de aceitação das sugestões do orquestrador medida; minutos de coordenação por remessa reduzidos contra a baseline (meta definida no dia 15, não agora).

### Fora de escopo nos 100 dias (consolidado dos dois conselhos)

Motor fiscal novo · marketplace de fretes · CRM adicional / integração Stack360 · WMS complexo · otimização de rotas por IA · promessa calculada preditiva · app completo de motorista · onboarding público de hubs · venda da plataforma a terceiros · expansão de rota antes do piloto provar.

---

## 4. Checklist de auditoria do TSM já construído (Fase 0a)

O TSM que você começou **evolui como núcleo** se responder SIM à maioria, com custo de correção baixo nas que faltam:

1. **Volume existe como entidade separada de Pedido** (ou é só um campo "qtde volumes")?
2. **Cobertura de hub existe** (cidades/prazos por hub) ou hub é só um cadastro de endereço?
3. **Manifesto é multi-pedido** com peso/cubagem/ocupação calculados, ou viagem aponta para um pedido?
4. **Status é máquina de estados** (transições válidas controladas) ou campo livre/enum que qualquer update troca?
5. **Eventos são imutáveis e com contexto** (quem, quando, por quê, evidência) ou logs sobrescrevíveis?
6. **Trecho existe** (viagem com paradas/pernas) ou origem→destino único?
7. **Status em lote**: mudar o manifesto propaga para os pedidos?
8. **O modelo separa compromisso comercial (remessa) de execução física (volumes/viagens)?**
9. **Há acoplamento com fiscal** que duplique o Lonngren (emissão própria de CT-e/MDF-e já construída)?
10. **Migração é viável**: dá para corrigir o modelo sem perder o que já foi feito?

Interpretação: 7+ SIM → evolui. 4–6 → evolui com refatoração dirigida (plano específico antes de qualquer feature nova). 0–3 → vira protótipo de referência; núcleo nasce limpo com as lições dele. **Quem responde é uma auditoria técnica no código, não a memória de quem construiu.**

---

## 5. O que o fundador precisa providenciar (insumos do plano)

1. **Acesso ao TSM construído** (código/repositório) — destrava a Fase 0a. É o insumo mais urgente.
2. **Acesso/export do Lonngren** (API? banco? relatórios?) + as ~50 remessas do diagnóstico.
3. **Números da operação**: remessas/mês, rotas ativas, nº de pessoas na coordenação, hubs/parceiros atuais no MA.
4. **Quem opera**: nomes/papéis de quem vai usar o sistema na doca e na torre — eles validam cada fase.
5. A visão do segundo conselheiro já está incorporada; **novos documentos de visão não são insumo** até o dia 100.

---

## 6. Governança do plano

- **Ritmo semanal**: revisão de 30 min contra os critérios de aceite da fase corrente. O que não tem critério de aceite não entra em desenvolvimento.
- **Protocolo com times de dev** (skill §17): sócio sênior gera prompt → time devolve plano → aprovação/corte de escopo → implementação → teste na operação real.
- **Classificação obrigatória de toda ideia nova**: Agora / Depois / Futuro / Cuidado / Não fazer. Este documento já classifica as ideias dos dois conselhos; novas entram pela mesma régua.
- **Regra de ouro (herdada do conselheiro 2, ratificada):** se o app não funciona na doca, não está pronto. Se a IA recomenda e ninguém executa, não criou valor. Se a nova camada duplica trabalho, corrige-se o desenho antes de expandir.
