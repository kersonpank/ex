# TSM O.S. — Plano da UI do Hub

> O plano de construção da nova superfície operacional do hub, desenhada a partir dos
> processos rotineiros (a Roda girando dentro do nó), consumindo o backend existente do OMNI.
> Incorpora as respostas do fundador (out/2026). Complementa RODA-OPERACIONAL.md e TEIA-E-RODA.md.

---

## 1. Decisões (incorporando suas respostas)

**D1 — Mobile-first, mínimo de interação.** A teia é plugada na existência das pessoas: um
ponto de coleta pode ser uma loja. Logo: telas que funcionam no celular com uma mão, ação
primária única e grande, **scan como entrada principal** (câmera lê etiqueta/QR e o sistema
deduz o contexto — receber, conferir, carregar — em vez de o operador navegar até a função).
Desktop é o mesmo app, responsivo. Métrica de UX: **toques por tarefa** (orçamento por fluxo,
medido, não opinião).

**D2 — Hub é hub.** Uma única superfície para todos os tamanhos — da loja-ponto-de-coleta ao
CD de GRU. O que varia é **o que aparece**, não o app: o backend já tem
`HubCargoCapabilities`/`HubRole`/`allowedActions` — a UI renderiza por capability. Loja vê
"Receber" e "Entregar"; CD vê tudo. Configuração fica no admin/gestão, nunca no hub.

**D3 — Nome e lugar: TSM O.S., dentro do repo OMNI, como módulo isolado.** (Decisão que você
delegou.) Repo separado duplicaria auth, API client, tipos e deploy, e atrasaria a primeira
fatia. "Não ficar lerdo" e "organizado" se garantem por regra, não por repo novo:

- módulo novo `components/os/` + rota `/os` **lazy-loaded** (code-splitting: o bundle do
  TSM O.S. não carrega nada do app legado, e vice-versa);
- **imports permitidos no módulo**: `lib/api`, `types`, primitives novos de `components/os/ui`.
  **Proibido** importar qualquer componente legado (admin/, operacao/, forms/);
- **limite de ~300 linhas por arquivo** (hard rule de review);
- orçamento de performance por fatia: primeira pintura da rota < 2s em 4G, interação < 100ms,
  listas virtualizadas acima de ~50 itens;
- a rota `/hub` atual continua intacta até a fatia equivalente do O.S. estar em uso real —
  aí a tela velha morre (flag por hub, corte gradual).

**D4 — Constrói-se a jornada, não telas soltas.** "Tudo dói porque uma coisa puxa a outra" —
exatamente por isso a entrega é em **fatias verticais do dia do hub**: cada fatia pega um
pedaço do fluxo de ponta a ponta (tela + API + evento que alimenta a próxima estação), entra
em teste na operação real, e só então vem a próxima.

**Jev (TypeSafe):** o MCP não está conectado nesta sessão cloud (a chave vive na sua máquina
— nas suas sessões locais ele funciona). O lugar certo dele no produto é mais à frente e é
ótimo: **gates tipados do orquestrador** (classificar pedido→fila, elegibilidade, risco de
prazo) com confiança calibrada e auditoria — exatamente o L2 da régua. Fica registrado como
peça do orquestrador v0, não desta fase de UI.

---

## 2. O dia do hub (os processos rotineiros que a UI serve)

Levantado do backend real (buckets do `hubCargoReadModel`, `manifestLifecycle`,
`HubArrivalSheet`, `DeliveryAttempt`) + suas respostas. A roda girando dentro do nó:

| # | Momento | O que o operador precisa | O que já existe | Gap |
|---|---|---|---|---|
| 1 | **Hoje** (abre o dia) | "O que precisa de mim agora": chegadas previstas, pendências, cargas paradas, exceções | buckets `incoming/awaiting/triage/stock`, manifests ativos | Só composição de tela |
| 2 | **Chegou caminhão** | Conferir o que desceu contra o esperado; divergência vira exceção na hora | `confirmArrival`, `HubArrivalSheet`, conciliação esperado×conferido | Conferência por volume (scan) vem na F3 |
| 3 | **Entrada avulsa** | Mercadoria chega no balcão → dar entrada em segundos → **etiquetar** | intake universal, entrada manual | **Etiquetas por volume: não existe — nasce aqui (início do G2)** |
| 4 | **Estoque** | O que tenho, há quanto tempo, para onde vai; achar carga por scan/busca | bucket `stock` + facts | Visão por destino depende do G1 |
| 5 | **Montar viagem** | Fila por destino com peso/m³ acumulados → manifesto → carregar conferindo | manifesto multi-pedido, elegibilidade, capacidade | **Fila por destino = G1 (backend pequeno, novo)** |
| 6 | **Despachar** | Docs prontos (CT-e/MDF-e/CIOT já automatizados), saída com km/lacre | `startManifest` + readiness + jobs fiscais | Só superfície |
| 7 | **Entregar local** | Rota do dia, comprovar com foto/recebedor, falha → próxima ação | `DeliveryAttempt` completo (evidência, idempotência, nextAction) | Só superfície mobile |
| 8 | **Exceções** | Uma fila: o que travou, de quem é, até quando | `SystemAlert` + exceptionReason espalhados | Promover a fila única (dono/prazo/desfecho) |

**Navegação mobile: 4 abas.** `Hoje` (momento 1) · `Cargas` (2–4) · `Viagens` (5–7) ·
`Exceções` (8). Toda tela segue o contrato de estação da RODA-OPERACIONAL.md: pergunta única,
fila, 1 ação primária, evento emitido (com ator — G3 pago aqui de graça).

**Scan-first como cola:** a etiqueta (momento 3) é o que destrava o "mínimo de interação" em
todos os outros: apontar a câmera para um volume responde "o que é isto, de quem é, para onde
vai, o que fazer agora" sem nenhuma navegação.

---

## 3. As fatias de entrega

Cada fatia: tela + API + evento, testada na operação real antes da próxima. Critério de
aceite sempre inclui o orçamento de toques.

### F1 — Shell + Hoje + Receber chegada *(backend 100% pronto)*
Shell TSM O.S. (rota lazy, abas, auth por capability) · tela **Hoje** · fluxo **chegada**:
ver o que está vindo → conferir descida (nível pedido) → divergência abre exceção.
**Aceite:** operador de Imperatriz recebe um caminhão real pelo celular; receber chegada ≤ 6
toques; divergência não deixa encerrar silenciosamente (já é regra do backend — a UI expõe).

### F2 — Entrada avulsa + Etiquetas + Estoque *(gap novo pequeno: etiqueta)*
Dar entrada no balcão em ≤ 60s · gerar/imprimir **etiqueta por volume** (código = chave do
scan; impressora térmica comum + fallback A4) · estoque com busca e scan.
**Aceite:** toda carga que entra sai etiquetada; achar qualquer carga do estoque em ≤ 3 toques
ou 1 scan. *(Backend novo: entidade Volume + etiqueta — o início do G2 da auditoria.)*

### F3 — Fila por destino + Montar viagem + Carregar conferindo *(G1 — o coração)*
Backend: cobertura rica + fila de consolidação por rota/hub destino (prompt de dev já no
formato da skill, seção 5). UI: fila com peso/m³/idade acumulados → criar manifesto a partir
dela → **carregamento com scan por volume** (custódia no 1º ponto crítico).
**Aceite:** manifesto GRU→Imperatriz montado a partir da fila em menos tempo que hoje
(baseline medida na F1); divergência de carregamento detectada antes da saída.

### F4 — Despachar + Entregas locais + Fila única de exceções
Saída com readiness/docs · rota de entrega do dia no celular com comprovação ·
`SystemAlert` promovido a fila única com dono/prazo/desfecho.
**Aceite:** uma remessa real percorre as 8 estações sem sair do sistema (a métrica da Roda).

**Fora de escopo de todas as fatias:** telas de cliente, CRM, chat/LLM config, identidade
visual, qualquer coisa do /gestao que não seja a fila de exceções.

---

## 4. Como vamos trabalhar (protocolo)

1. Eu gero o prompt de dev de cada fatia (formato da skill: contexto, escopo, fora de escopo,
   regras, modelo de dados, API, eventos, aceite, cuidados).
2. Time/sessão de dev devolve plano técnico **antes** de implementar (os 10 itens do
   protocolo §17); eu reviso e corto escopo.
3. Implementa → você testa **na operação real** (Imperatriz/GRU) → medimos toques e minutos →
   só então a próxima fatia.
4. Review automático de cada PR checa: limite de linhas, import proibido do legado, teste de
   conformidade da teia (5 perguntas do TEIA-E-RODA.md).

---

## 5. Prompt de desenvolvimento — F1 (pronto para despachar)

```markdown
# Tarefa: TSM O.S. — Shell + tela Hoje + fluxo Receber Chegada (Fatia 1)

## Contexto
O TSM EX OMNI (repo tsmex-omni) tem backend maduro: buckets de carga por hub
(server/src/services/hubCargo/hubCargoReadModel.ts), ciclo de manifesto
(manifestLifecycle.ts), confirmação de chegada (HubArrivalSheet já consome API).
A UI atual é organizada por entidade e será substituída por uma superfície nova,
TSM O.S., organizada por estação de trabalho (ver RODA-OPERACIONAL.md e
TEIA-E-RODA.md no repo kersonpank/ex). Esta é a primeira fatia.

## Objetivo
Operador de hub (qualquer tamanho) abre /os no celular e: vê o que precisa dele
hoje; recebe um caminhão conferindo a descida; divergência abre exceção. Sem
treinamento.

## Escopo
1. Módulo components/os/ + rota /os lazy-loaded (React.lazy + chunk próprio),
   navegação inferior com 4 abas (Hoje, Cargas, Viagens, Exceções — as duas
   últimas como stub "em construção").
2. Tela HOJE: chegadas previstas (manifests IN_PROGRESS com parada neste hub),
   pendências de conferência, cargas aguardando (awaiting/triage), contadores
   dos buckets. Fonte: GET hub-cargo existente + manifests. Cada card leva à ação.
3. Fluxo RECEBER: lista de manifests chegando → detalhe → conferência por pedido
   (chegou/não chegou/avaria) → confirmar → divergência cria exceção e NÃO
   permite encerrar como completo (usar regras/endpoints existentes de
   confirmArrival/arrival; não criar lógica nova de negócio no front).
4. Primitives novos em components/os/ui (Button, Card, List, Sheet, Scanner
   placeholder) — mobile-first, alvos ≥ 44px, tema claro/escuro.
5. Todos os eventos enviados incluem o usuário autenticado (ator).

## Fora de escopo
Etiquetas, scan real por câmera, fila por destino, montar viagem, entregas,
qualquer alteração de schema/backend, qualquer tela do app legado.

## Regras de negócio
- Renderização por capability (HubCargoCapabilities): sem canReceive → fluxo
  Receber oculto.
- Divergência (esperado ≠ conferido) sempre gera registro visível — nunca
  encerramento silencioso (o backend já garante; a UI deve expor, não contornar).

## Backend/API
Consumir endpoints existentes (hubCargo, manifests, arrival). Se algum dado da
tela Hoje exigir novo endpoint de leitura, propor no plano técnico antes
(read-only, sem migração).

## Critérios de aceite
- /os carrega em < 2s em 4G simulado; nenhum import de components/{admin,operacao,forms}.
- Receber uma chegada completa: ≤ 6 toques.
- Nenhum arquivo novo > 300 linhas; testes dos fluxos Hoje e Receber.
- Rota /hub legada intocada e funcionando.

## Cuidados
Não quebrar auth/rotas existentes; chunk do O.S. separado do bundle legado;
nada de estado global novo (usar o client de API existente).

## Antes de implementar, responda (protocolo §17)
Arquivos/módulos a alterar; endpoints consumidos; telas; eventos; risco de
quebrar fluxo existente; como testar ponta a ponta; menor corte implementável.
```

---

## 6. Resposta direta às suas perguntas

- **"Criamos a UI nova pensada no processo e aí consumir o backend?"** Sim — com a correção
  de que não é "a UI inteira e depois ligar": cada fatia já nasce ligada ao backend e testada
  na doca. UI sem dado real é mockup; mockup não destrava operação.
- **"Iniciar pelo hub?"** Sim. O hub é onde a roda gira — as estações 2–7 vivem nele; o
  /gestao supervisiona e o /link executa, ambos ficam melhores de graça quando o hub emite
  eventos limpos.
- **"Já suportando todos os níveis de hub?"** Sim, pela via barata: um app só, renderizado
  por capability (D2). Suportar todos os níveis nunca significa construir para todos os
  níveis — a loja-ponto-de-coleta usa 2 dos fluxos; o CD usa todos.
