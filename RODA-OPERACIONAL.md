# TSM EX OMNI — A Roda Operacional (redesenho dos processos e da UI)

> Resposta ao bloqueio atual: "está complexo, só consigo fazer com clareza; temos quase tudo,
> mas precisa ser uma roda — o ciclo tem que fechar e ser constante."
> Este documento define a roda, o contrato de cada estação/tela e o método de redesenho.
> Baseado no código auditado (AUDITORIA-TSMEX-OMNI.md). Outubro/2026.

---

## 0. Atualizações de posição (correções do fundador aceitas)

**Lonngren:** com o consentimento do dono e a aposentadoria do sistema marcada para 2027, a objeção jurídica cai. Estudar o Lonngren para entender fluxos e melhorá-los é legítimo — a posição que permanece é só esta: **não replicar 1:1**; o OMNI já modela a operação melhor (multi-perna, posse, margem). O que muda de verdade:

- **A aposentadoria vira deadline imposto.** O corte do Lonngren (G4 da auditoria) deixa de ser "quando provar paridade" e passa a ter data-limite externa: **a operação precisa rodar 100% no OMNI antes do desligamento em 2027**. Isso sobe a prioridade do plano de migração de dados.
- **O scraper vira ponte oficial de dados** durante a transição (não há API). Cuidados mínimos: rodar como job agendado com log do que importou, idempotência (não duplicar remessa já importada) e alarme quando o layout da tela mudar e o scraper quebrar silenciosamente. Scraper que falha calado = dado divergente = a roda abre.

---

## 1. O princípio do redesenho (o ajuste de método)

Concordo com redesenhar os processos — com uma correção de partida: **não se parte da UI; parte-se da roda, e a UI vira as estações da roda.** "Redesenhar a partir da UI" produz telas melhores para os mesmos fluxos confusos. O caminho certo:

> **Processo (a roda) → papel de quem opera → estação de trabalho → tela.**

A causa da complexidade atual não é falta de funcionalidade — a auditoria mostrou que o backend cobre quase tudo. A causa é que a UI está organizada **por entidade** (abas de cadastro: pedidos, pessoas, manifestos, tarifas...) e não **por estação de trabalho**. O operador precisa saber onde clicar; o sistema é que deveria dizer "isto precisa de você agora".

**O contrato de estação** — toda tela operacional do OMNI passa a responder, obrigatoriamente, a 4 coisas:

1. **Pergunta única** que a estação responde (ex.: "o que preciso despachar hoje?");
2. **Fila de trabalho** ordenada ("o que precisa de mim agora"), nunca uma lista de cadastro;
3. **Ação primária** (1 botão dominante que conclui o item e o empurra para a estação seguinte);
4. **Evento emitido** ao concluir (com ator e porquê — gap G3), que **alimenta a fila da próxima estação automaticamente**.

A roda "fecha e é constante" exatamente quando o item concluído numa estação **aparece sozinho** na fila da seguinte. Hoje isso acontece em parte (manifesto → status em lote dos pedidos); onde não acontece, o operador vira o motor da roda — e é aí que cansa e trava.

---

## 2. A Roda — 8 estações

```
        ┌──────────────────────────────────────────────────────────┐
        ▼                                                          │
  1. ENTRADA → 2. POSSE → 3. CONSOLIDAÇÃO → 4. VIAGEM             │
  (demanda/NF)  (receber)   (montar carga)   (transferir)          │
                                                                   │
  8. APRENDIZADO ← 7. FECHAMENTO ← 6. ENTREGA ← 5. DISTRIBUIÇÃO   │
  (margem/feedback)  (fiscal+acerto)  (comprovar)  (última perna)  │
        └──────────────────────────────────────────────────────────┘
```

| # | Estação | Pergunta da estação | Fila | Ação primária | O que já existe no OMNI | Estado |
|---|---|---|---|---|---|---|
| 1 | **Entrada** | O que está chegando e o que falta para virar pedido? | Demandas incompletas + NF-e descobertas (triagem) | Completar/aceitar → vira pedido esperado no hub | `Demand` + intake universal + `nfeDistribuicaoTriagem` + `missingFields` | ✅ forte |
| 2 | **Posse** | O que chegou na doca e confere? | Buckets `incoming/awaiting/triage` do hub | Confirmar chegada (conferir) | `hubCargoReadModel` (stock/incoming/awaiting/triage), `confirmArrival` | ✅ forte |
| 3 | **Consolidação** | O que está pronto para viajar e para onde? | **Fila por rota/hub destino** com peso/m³ acumulados e idade | Montar manifesto a partir da fila | `stock` do hubCargo + `Manifest` multi-pedido + validação de capacidade | ⚠️ **fila por destino não existe — é o G1, o maior buraco da roda** |
| 4 | **Viagem** | Esta viagem está pronta, saiu, onde parou? | Manifestos por status (preparar → sair → em rota → descarregar por parada) | Avançar o ciclo (ofertar/aceitar/sair/concluir parada) | `manifestLifecycle` + `ManifestStop` + readiness + Carta Frete + CIOT/MDF-e | ✅ forte (UI de 2.204 linhas — reorganizar, não reescrever) |
| 5 | **Distribuição** | O que chegou aqui e precisa sair para entrega? | Pedidos descarregados aguardando última perna | Montar manifesto DELIVERY / entrega direta | Mesmo motor da 3–4 com `tripType DELIVERY` | ⚠️ funciona, mas sem fila própria (depende do G1) |
| 6 | **Entrega** | O que comprova esta entrega? | Entregas do dia por viagem/parada | Registrar tentativa com evidência | `DeliveryAttempt` (resultado, qtde, evidência, idempotência, próxima ação) | ✅ forte |
| 7 | **Fechamento** | O que falta encerrar (fiscal, acerto, fatura)? | Encerramentos pendentes + acertos Hub/Link + faturas | Resolver pendência | `FiscalClosureTask` best-effort + `SystemAlert` + `Payable`/`Invoice` + acerto por viagem | ✅ médio (exceções espalhadas) |
| 8 | **Aprendizado** | O que o planejado vs realizado ensinou? | Viagens concluídas sem revisão de margem | Revisar margem/ocupação/prazo | `ConsolidationFeedback` + custos por perna + Carta Frete real | ⚠️ dados existem, estação não existe |

**Papéis × estações** (quem vê o quê — os workspaces já existem nas rotas):

- **/hub** → estações 2, 3, 5 (e 6 quando o hub entrega). É a superfície principal (decisão #16 do Norte — correta).
- **/gestao (torre)** → estações 1, 4 (supervisão), 7, 8 + painel transversal de exceções.
- **/link** → estação 4 na visão do transportador (aceitar, sair, paradas, comprovar, ganhos).
- **/client** → espelho de rastreamento (não é estação da roda; não ganha investimento agora).

---

## 3. Onde a roda quebra hoje (diagnóstico de fechamento do ciclo)

Cruzando as estações com a auditoria, a roda abre em **4 pontos** — e são exatamente eles que causam a sensação de "complexo, não fecha":

1. **2→3 e 3→4 (o coração):** pedido em estoque não cai em fila por destino; o operador decide de cabeça o que agrupa com o quê. Sem fila, a consolidação é artesanal e a estação 3 não tem tela possível — por isso a montagem de viagem parece complexa. **Correção = G1** (cobertura rica + fila de consolidação por rota/hub destino).
2. **4→5:** descarga no hub destino não abre automaticamente a fila de distribuição — reaproveita o fluxo manual. Corrige-se de graça quando o G1 existir (a fila de distribuição é a mesma fila, com rota local).
3. **7 (exceções espalhadas):** divergência de descarga, falha de entrega, encerramento fiscal pendente e acerto vivem em lugares diferentes (`SystemAlert`, telas próprias). A roda exige **uma fila única de exceções** com dono/prazo/desfecho — promoção do `SystemAlert` a estação transversal.
4. **8 (roda sem retorno):** planejado vs realizado existe nos dados (utilização, margem, Carta Frete), mas ninguém fecha o laço — `ConsolidationFeedback` está pronto para receber e quase nada o alimenta. Sem a estação 8, a roda gira mas **não aprende**, e o orquestrador v0 nunca terá base.

Repare: **nenhuma quebra é de backend.** As quatro são "o dado existe, a estação não". Isso confirma sua leitura — o redesenho é de processo/superfície, não de motor.

---

## 4. Método de redesenho (como fazer, em ordem)

### Passo 1 — Caminhar a roda (1 semana, sem escrever código)
Pegue **3 remessas reais** (uma normal, uma com divergência, uma com falha de entrega) e caminhe cada uma pelas 8 estações **dentro do OMNI**, anotando em cada passagem: em que tela estava, quantos cliques/minutos, onde teve que *saber* o próximo passo em vez de o sistema dizer, onde saiu do sistema (WhatsApp, papel, Lonngren, planilha). Cada saída do sistema e cada "tive que saber" é um defeito de desenho numerado. **Entrega: a lista de defeitos da roda, por estação.**

### Passo 2 — Blueprint de cada estação (1 página por estação)
Para cada uma das 8: a pergunta, a fila (critério de ordenação e de entrada/saída), a ação primária, o evento emitido, as exceções possíveis e para onde escalam. Uma página, sem mockup bonito — contrato, não pintura. **Entrega: 8 blueprints + o mapa papel×estação.**

### Passo 3 — Reconstruir estação por estação, na ordem da dor
Ordem recomendada (dor × dependência):

1. **Estação 3 (Consolidação)** = G1 no backend (cobertura rica + fila por rota/destino) + a primeira tela nova do padrão "fila + ação primária". É o coração econômico e o maior defeito.
2. **Estação 7-transversal (fila única de exceções)** = promover `SystemAlert` a fila com dono/prazo/desfecho; barato e devolve controle imediato.
3. **Estação 4 (Viagem)** = reorganizar `ManifestDetailPage` (2.204 linhas) em passos do ciclo de vida — extração de componentes, não reescrita.
4. **Estação 5 (Distribuição)** = a fila do G1 aplicada ao hub destino.
5. **Estação 8 (Aprendizado)** = painel planejado×realizado por viagem alimentando `ConsolidationFeedback`.

Regras do passo 3: uma estação por vez, entra em uso real antes da próxima começar; nenhum componente novo > ~400 linhas; tela velha só morre quando a nova estiver em uso (sem big-bang); todo evento emitido ganha ator+porquê na passagem (G3 pago aos poucos, não como projeto separado).

### Passo 4 — Medir o fechamento da roda
A métrica do redesenho é uma só: **% de remessas que percorrem as 8 estações sem sair do sistema e sem "tive que saber"** (touchless de navegação, precursor do touchless de decisão). Medida a cada caminhada mensal da roda, com as mesmas 3 categorias de remessa.

---

## 5. Critério para retomar o trabalho amanhã

O bloqueio ("parei porque está complexo") se resolve com escopo de um passo: **a próxima entrega é só o Passo 1 — caminhar 3 remessas e numerar os defeitos.** Não é código, não é refatoração, não exige decidir arquitetura. Com a lista de defeitos na mão, os blueprints do Passo 2 praticamente se escrevem sozinhos, e o Passo 3 já tem ordem definida.

O que **não** entra enquanto os Passos 1–3 não terminarem: CRM/Opportunity novo, chat/LLM config, identidade visual, telas de cliente, qualquer rota nova de negócio. A roda fecha primeiro; cresce depois.
