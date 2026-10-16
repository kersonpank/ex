# Auditoria técnica — TSM EX OMNI (kersonpank/tsmex-omni)

> Auditoria do Gate da Fase 0a do PLANO-EXECUTAVEL.md, feita sobre o código real
> (clone do repositório, outubro/2026): schema Prisma (2.592 linhas, ~90 modelos),
> serviços do servidor, testes e frontend. Não baseada nos .md — os .md foram
> conferidos contra o código.

---

## Veredito

> **O TSM EX OMNI EVOLUI COMO NÚCLEO. Não se começa um sistema novo.**
> Checklist: **7 de 10 SIM** (3 parciais/não, todos corrigíveis por evolução dirigida, nenhum exige reescrita).

Isso muda o plano: a Fase 0a está respondida, e o "terceiro sistema" (Logistics OS do zero) **está descartado**. O risco agora é outro — dispersão de escopo dentro do OMNI (CRM, chat multi-LLM, identidade visual já vivem no mesmo monólito) e a dívida declarada de UI.

O sistema é muito mais maduro do que "feito na tora": 98 arquivos de teste, transações com invariantes, idempotência em pontos críticos, fiscal abstraído por provider, decisões de arquitetura documentadas e — mais importante — **implementadas de verdade** (conferido código contra TSM_NORTE.md).

---

## Checklist — resposta por pergunta, com evidência

| # | Pergunta | Resposta | Evidência no código |
|---|---|---|---|
| 1 | Volume como entidade separada? | **NÃO** | `Demand.units Int @default(1)` — volume é só quantidade; não há `model Volume`, sem identidade/etiqueta por volume. Mitigação parcial: `DeliveryAttempt.expectedQuantity/deliveredQuantity` concilia quantidade na entrega |
| 2 | Cobertura de hub existe? | **PARCIAL** | `Hub.coverageCities String[]` + `coverageRadius` existem, mas sem prazo/cidade, faixa de CEP, tipo de atendimento ou prioridade — e só são usadas em `routes/hubs.ts` (cadastro), **nenhuma decisão de roteamento consome cobertura** |
| 3 | Manifesto multi-pedido com ocupação? | **SIM** | `Manifest.orders[]`, `legs[]`, `totalWeight/totalVolume/totalValue`; capacidade validada contra `VehicleType.maxWeight/maxVolume` com bloqueio de excesso e cálculo de utilização (`manifestLifecycle.ts:194-310`) |
| 4 | Status é máquina de estados? | **SIM (por guardas)** | Não há tabela declarativa de transições, mas cada transição é guardada em serviço testado: `startManifest` exige `ACCEPTED`, roda `getManifestReadiness` (issues bloqueantes) e checa invariante "pedido não pode estar em outro manifesto IN_PROGRESS" |
| 5 | Eventos imutáveis com contexto? | **PARCIAL** | `OrderHistory`/`DemandEvent` são append-only mas **sem ator nem porquê** (só eventType, descrição, local, foto). Em compensação, `ConsolidationFeedback` registra exatamente o "evento com porquê" da consolidação: ator, ACCEPTED/MODIFIED/REJECTED, sugerido vs final, utilização e margem antes/depois. `DeliveryAttempt` tem `recordedByUserId`, evidência e `idempotencyKey` |
| 6 | Trecho existe? | **SIM** | `OrderLeg` (sequence, COLETA/TRANSFERENCIA/ENTREGA, executedBy HUB/LINK, hubs origem/destino, custo alocado) + `ManifestStop` (N paradas/descargas por viagem, conclusão por parada) |
| 7 | Status em lote? | **SIM** | `startManifest`/`completeManifest` propagam em transação: manifesto → legs → stops → status de cada pedido + `OrderHistory` por pedido |
| 8 | Separa compromisso comercial de execução física? | **SIM** | Fronteira explícita Demanda≠Pedido = **posse** (decisão #9/#10 do TSM_NORTE, implementada: `Demand.order?` 1:1, pedido = fonte única do estado físico, jornada multi-perna N:N com manifesto) |
| 9 | Acoplamento fiscal duplica o Lonngren? | **SIM — e isso é bom** | Pipeline fiscal próprio e sério: CT-e/MDF-e assíncronos (`CteEmissionJob`), CIOT via eFrete, Distribuição NF-e via ACBr, abstração de provider (SIMULATED/ACBR/NUVEM_FISCAL/FOCUS_NFE/TECNOSPEED), encerramento fiscal best-effort que nunca bloqueia a operação (decisão #20). Em vez de "risco de duplicação", isto é a **independência do Lonngren já construída** |
| 10 | Migração viável? | **SIM** | Prisma com ~50 migrações disciplinadas, 98 arquivos de teste, comentários de compatibilidade ("espelho legado até a limpeza"), padrões de não-destruição de dados |

Também já existem, além do checklist: fila de exceções embrionária (`SystemAlert`), read model "onde está cada carga" por hub com buckets stock/incoming/awaiting/triage (`hubCargoReadModel.ts`), acerto de Hub com máquina de estados própria, Carta Frete real como fonte de custo (margem por viagem), gestão de risco/GR no fluxo.

---

## Os 5 gaps reais (em ordem de prioridade)

### G1 — Cobertura rica + fila de consolidação (o degrau L1→L2 inteiro)
A decisão #12 do TSM_NORTE ("pernas emergem na operação, sem planejador automático") foi correta para nascer — e agora é **o próximo degrau**. Hoje nenhuma lógica decide ou sugere destino/fila: o humano monta tudo e o sistema não perde o rastro (L1 quase completo). Falta: enriquecer `Cobertura` (prazo por cidade, tipo de atendimento, prioridade, ativo) e criar a **fila de consolidação por rota/hub destino** — pedido entra, sistema classifica e enfileira. É o pré-requisito do orquestrador v0, e o `ConsolidationFeedback` já existe esperando por ele.

### G2 — Identidade por volume nos 2 pontos críticos
`units: Int` não permite conciliar "esperados 12, conferidos 11" por volume na transferência. Criar `Volume` (pedido, etiqueta/código, peso/dimensões, status) e conferência por leitura **só no carregamento da transferência e na descarga do hub destino** (conforme ajuste A4 do plano). `DeliveryAttempt` já dá o padrão de evidência+idempotência a seguir.

### G3 — Ator e porquê em todos os eventos
`OrderHistory` e `DemandEvent` precisam de `actorUserId/actorRole` (e, quando decisão: sugestão do sistema, aceita/alterada, motivo) — o padrão já existe no `ConsolidationFeedback`; é generalizá-lo. Barato agora, impossível depois.

### G4 — Plano de corte do Lonngren (pergunta estratégica respondida abaixo)
O OMNI já tem o encanamento fiscal próprio. O que falta não é software, é **plano de cutover por rota**: shadow (OMNI espelha) → paridade medida → corte. E a migração de dados (cadastros, tabelas de frete, histórico) via **export**, não engenharia reversa.

### G5 — Dívida de UI contida, não resolvida agora
104 componentes, ~35.000 linhas, arquivos de 1.000–2.200 linhas (`ManifestDetailPage.tsx` com 2.204). A decisão #16 ("refatorar depois; /hub é a superfície principal") está certa. Regra prática: **nenhum arquivo novo acima de ~400 linhas; todo toque em arquivo gigante extrai pelo menos um componente**. Refatoração big-bang de frontend agora seria a distração clássica.

Risco adicional a conter: o monólito já carrega CRM (Account/Opportunity), chat multi-LLM com rotação de chaves e identidade visual. Nada disso cresce até G1–G3 estarem entregues.

---

## Resposta às duas questões estratégicas

### "Não dá para criar um negócio estruturado em cima de um SaaS contratado"
**Concordo — e o código mostra que você já agiu sobre isso.** O pipeline fiscal próprio (ACBr + eFrete + providers) é exatamente a independência que importa. O Lonngren deixa de ser "fundação" e vira **sistema legado em produção durante a transição**: continua operando o dia a dia até o OMNI provar paridade na rota piloto, e então é cortado rota a rota. Prazo de contrato, export de dados e janela de operação paralela entram no plano de corte (G4).

### "Usar o REA para recriar o que tem de importante no Lonngren?"
**Não. Posição firme de sócio, por três razões:**

1. **Jurídica/contratual**: o Lonngren é software proprietário licenciado para a TransMesquita. Engenharia reversa para replicá-lo viola os termos típicos de licença e expõe a empresa (Lei 9.609/98 — lei do software). Para uma empresa que vende **confiança B2B**, criar passivo jurídico na fundação é o oposto da tese.
2. **Desnecessária**: o que o Lonngren tem de "importante" não é segredo — são fluxos padrão de TMS (que o OMNI já modela melhor) e o motor fiscal (que o OMNI já tem, com provider abstraído). O que a TransMesquita precisa extrair do Lonngren é **o que é seu por direito: os dados** — cadastros, tabelas de frete, histórico de remessas, numeração de documentos. Isso sai por export/relatório/API, caminho limpo.
3. **Estratégica**: a tese inteira é que software é copiável e o fosso está na malha medida + grafo de eventos. Gastar energia copiando um TMS commodity é investir no ativo que não se defende.

Onde o REA **tem** uso legítimo: estudar padrões de produtos para **aprendizado de design** de mercado, e eventualmente depurar integrações próprias. Não para clonar o sistema de um fornecedor contratado.

---

## Impacto no PLANO-EXECUTAVEL.md

- **Fase 0a: RESPONDIDA.** OMNI evolui como núcleo. "Logistics OS do zero": descartado.
- **Fase 0b/0c continuam**: diagnóstico das ~50 remessas reais + baseline de métricas + inventário de export do Lonngren. São os únicos insumos que ainda faltam e não dependem de código.
- **Fase 1 vira**: G1 (cobertura rica + fila de consolidação) + G3 (ator/porquê nos eventos) **dentro do OMNI**.
- **Fase 2 vira**: G2 (volume nos 2 pontos críticos) + exceções promovidas de `SystemAlert` para fila única com dono/prazo/desfecho.
- **Fase 3 mantém**: orquestrador v0 (sugestões com justificativa alimentando `ConsolidationFeedback`) + automações de coordenação + G4 (corte do Lonngren na rota piloto).
- **Congela**: crescimento de CRM/chat/LLM-config no monólito; refatoração big-bang de UI; qualquer engenharia reversa do Lonngren.
