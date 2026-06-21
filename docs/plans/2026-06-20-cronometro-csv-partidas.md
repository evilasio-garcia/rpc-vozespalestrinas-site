# Cronômetro alimentado por CSV de partidas

Data: 2026-06-20

## Objetivo
Substituir o cronômetro de `deadline` fixo por uma agenda de partidas lida de um CSV no
carregamento da página, com estados de badge (ao vivo / próxima / encerrado) e cálculo
dinâmico do prazo de entrega.

## Decisões aprovadas pelo usuário
1. **CSV** em `assets/jogos.csv`, separador `;`, com cabeçalho. Colunas:
   `data;inicio;fim;partida` — data `DD/MM/AAAA`, horas `HH:MM` (24h),
   `partida` é texto livre (exibido em maiúsculas pela UI). Alterar dados = novo commit.
2. **Contagem regressiva** sempre até o **horário de fim** da partida ativa. Ao chegar no
   fim, o cronômetro encerra (zera 00:00:00), como já era feito antes.
3. **Abertura da próxima partida**: só às **00:00 do dia seguinte ao jogo que acabou**
   (não a 00:00 da data da própria próxima partida). Ex.: jogo 19/06 fim 23:00 →
   de 23:01 a 23:59 do dia 19 fica **encerrado**; a partir de 00:00 do dia 20 monta a
   regressiva até o fim do jogo de 24/06.
4. **Badge da partida** (estados):
   - antes do início: `PRÓXIMA PARTIDA · <partida>` — cor neutra.
   - durante início→fim: `AO VIVO HOJE · <partida>` — estilo/cores atuais (amarelo + dot pulsando).
   - após o fim: `ENCERRADO · <partida>` — cor desativada/apagada.
   - a partida exibida troca no reset (quando a nova contagem abre).
5. **Prazo de entrega**: dia-limite = hoje **+ 2 dias úteis** (seg–sex), com **corte às 11:00**.
   Base = data/hora **real do visitante**. Até 11:00 hoje conta como 1º dia útil; depois,
   começa no próximo dia útil. Mantida a palavra "amanhã"; só o dia-limite é calculado.
   Atualiza em todos os pontos (hero, faixa de benefícios, fechamento, rodapé, barra fixa).
6. **Servido via HTTP** (deploy normal). `fetch` do CSV **não funciona** abrindo o
   `index.html` por duplo-clique (file://). Para testar local: `python -m http.server`.

## Implementação (index.html, componente DCLogic)
- `state = { now, matches }`. CSV lido em `componentDidMount` via `fetch` + `setState`.
- `parseMatches(txt)`: parse `;`, valida data/hora por regex (pula cabeçalho/linhas inválidas),
  ordena por início.
- `matchState(matches, now)`: define o ponteiro (maior i com activation(i) ≤ now;
  activation(0) = -∞, activation(i≥1) = 00:00 do dia seguinte ao jogo i-1) e a fase
  (`upcoming` / `live` / `ended` / `loading`). Alvo da contagem = fim da partida.
- `deliveryLimitWeekday(now)`: 2 dias úteis com corte 11:00; nomes pt-br.
- `badgePresentation(ms)`: prefixo + estilo + dot por fase.

## Verificação
28 testes em node contra o código real extraído do `index.html` — todos os exemplos do
usuário (gating 19/06→20/06, 5 casos de entrega, transições de fase) passaram (28/0).
Servido via HTTP: index.html, support.js, assets/jogos.csv e imagens retornam 200;
CSV decodifica UTF-8 corretamente.

## Atualizações (mesma data)
- **Cupom** alterado para `VOZES-COPA26` (default + fallback).
- **Clique-para-copiar + toast**: todos os 7 locais onde o cupom aparece (caixa do hero,
  faixa de benefícios, badge "−10%" de cada produto, fechamento, rodapé, barra fixa) copiam
  o cupom (`navigator.clipboard` com fallback `execCommand`) e exibem um toast por ~2,4s.
- **Botão de produto esgotado**: não navega (sem atributo `href` — React omite quando o
  valor bindado é `undefined`) e usa `cursor:not-allowed`; hover desativado. Em estoque
  permanece normal.
- **Produção**: removido o modo de validação `?demo` (`isDemo`/`demoMatches`). O runtime é
  React (via support.js): atributo bindado `undefined` é omitido; `onClick` recebe o evento.
- Verificação: 17 testes em node (limpeza do demo, botão esgotado, entrega 22/06→terça,
  estados de partida, cupom) — todos passaram.
