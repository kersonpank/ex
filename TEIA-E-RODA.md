# TSM EX — Teia e Roda: o modelo mental congelado

> Documento curto e inegociável. Todo time de desenvolvimento lê isto antes de tocar no OMNI.
> Se um plano técnico tratar a operação como "A→B com um veículo", ele está errado — devolver.

---

## 1. O que a operação NÃO é

Não é Uber/ponto-a-ponto: um pedido, um veículo, uma perna, decisão única na origem.
Nenhuma tela, entidade ou fluxo pode assumir "pedido → viagem → destino".

## 2. O que a operação É

Uma **malha (teia)** de nós conectados por capacidades, onde a jornada de cada pedido é
construída **salto a salto**, e uma transferência **deriva outras transferências**:

```
GRU ══════ tronco (30+ pedidos) ══════▶ IMPERATRIZ
                                            │ chegada dispara novas decisões
                                            ├─ regional ▶ AÇAILÂNDIA (manifesto novo, pedidos novos)
                                            ├─ regional ▶ SÃO MATEUS
                                            └─ entrega local (última perna)
```

O pedido GRU→Açailândia compartilha o tronco com pedidos de outros destinos; em Imperatriz
entra em nova fila e embarca em manifesto diferente, possivelmente de outro parceiro.
Cascata de pernas até a entrega. (No código: `OrderLeg` N:N com `Manifest`, `sequence`,
pernas emergem na operação — o modelo já suporta.)

## 3. A analogia operacional: roteamento de pacotes

A teia descreve a estrutura; o roteamento descreve o funcionamento:

| Operação | Roteamento | OMNI |
|---|---|---|
| Hub/CD/parceiro (ponto da teia) | Roteador | `Hub` |
| Motorista/carreta/agregado/provedor (fio) | Enlace com capacidade, custo e janela | `Link` |
| Pedido | Pacote (sabe onde está + destino final) | `Order` |
| Manifesto | Quadro: muitos pacotes num mesmo enlace | `Manifest` multi-pedido |
| Cobertura/rotas/prazos | Tabela de roteamento | `Cobertura` rica (G1) |
| Transferência derivada | Próximo salto decidido em cada nó | Fila de consolidação por destino, por hub |

**O princípio que simplifica o OS:** nenhum nó precisa conhecer o caminho inteiro.
Cada hub responde apenas: *"dado o destino final deste pedido, qual o melhor próximo
salto a partir daqui, agora?"* Sem otimizador global no início — decisão local por regras
(orquestrador v0), com o pedido carregando só posição atual + destino.

**Consequências diretas:**
1. **A fila de consolidação (G1) é a decisão de próximo salto materializada.** Sem ela, o
   roteador é a cabeça do operador.
2. **Expandir a teia = adicionar linhas na tabela de roteamento** (novo hub/parceiro com
   cobertura e prazos), nunca redesenhar fluxo.
3. **O ativo que não se copia é a tabela de roteamento viva**: quais enlaces cumprem, em que
   prazo, a que custo, trecho a trecho (`Link.score`, histórico por rota, planejado×realizado).
4. **Entrega direta (um salto) é caso particular da teia**, não o padrão — a exceção cabe no
   modelo, o contrário não.

## 4. Teia × Roda: os dois eixos do mesmo organismo

- **Teia = espaço.** O mapa de nós e capacidades. Cresce ligando pontos.
- **Roda = tempo.** O ciclo que gira **dentro de cada nó**: receber → consolidar →
  despachar → comprovar (RODA-OPERACIONAL.md).

Todo nó da teia — CD próprio em GRU ou parceiro em São Mateus — roda **a mesma roda, nas
mesmas estações, com as mesmas telas**. Nó novo = mesma roda em outro ponto. É isso que
torna a expansão barata: a teia cresce sem criar fluxo novo.

## 5. Teste de conformidade para qualquer plano técnico

Antes de aprovar implementação, verificar:

1. O pedido conhece apenas posição atual + destino final (nunca "sua viagem")? 
2. A decisão de destino/agrupamento é tomada **no nó onde o pedido está**?
3. A chegada de um manifesto **alimenta automaticamente** as filas do nó de chegada?
4. O desenho funciona igual com 3 hubs e com 300 (nada hard-coded por rota)?
5. Um salto direto (coleta→entrega) passa pelo mesmo modelo como caso de 1 perna?

Cinco "sim" = conforme a teia. Qualquer "não" = voltar ao desenho.
