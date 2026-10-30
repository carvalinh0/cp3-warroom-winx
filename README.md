**Grupo (nome da equipe plantonista):** Clube das Winx

**Turma:** 2CCPH **Repo:** `cp6-warroom-winx`

**Integrantes (nome + RM):**

| Nome | RM |
|---|---|
| Felipe Souza Carvalho | RM564779 |
| Riquelme Santos da Mata | RM565053 |
| Luiz Henrique Macedo Graça | RM564704 |

**Convenção de commits** (1 commit por rodada; relâmpagos podem ir junto com a
rodada seguinte):

```
decisao: R1 - opcao C (<resumo da justificativa em uma frase>)
decisao: R2 - opcao A (<resumo em uma frase>)
pos-mortem: relatorio de incidente da madrugada
```

---

## 📁 Dossiê técnico do FiapBank (MVP em produção)

**Stack:** Java 17 + Spring Boot + Spring Data JPA + Oracle. API com endpoints em
`/api/contas` e `/api/transferencias` (cenário visto desde a Aula 13).

**Contrato e regras de negócio que o banco prometeu aos clientes e aos reguladores:**

| Regra | Como deve ser |
|---|---|
| Transferência **PIX** | taxa **R$ 0,00** |
| Transferência **TED** | taxa fixa **R$ 5,00** |
| Saldo | **nunca fica negativo**: transferência/saque sem saldo é recusado com erro claro |
| Número de conta | **sequencial e único** (1001, 1002, 1003...), gerado pelo sistema |
| Extrato de transferência | grava **quem pagou** e **quem recebeu**, na ordem certa |
| CPF | **dado sensível**: nunca aparece nas respostas da API |
| Consultas ao banco | sempre parametrizadas, e cada operação **usa e libera** a conexão |
| Suíte de testes | roda antes de todo deploy; **verde** é pré-requisito pra subir |

**Como escrever a justificativa:** nomeie o **mecanismo técnico** em jogo (o
conceito das Aulas 11 a 15 que explica o incidente) e o **trade-off** (velocidade ×
segurança × faturamento × dívida). "Porque é mais seguro" não é justificativa.

---

# 📝 REGISTRO DE DECISÕES

> A cada incidente, o professor dita o título (ex.: "Rodada 1"). **Copiem o modelo
> abaixo, colem no fim desta seção** e preencham com o rascunho feito em aula, junto
> com o **placar do grupo** após a consequência. Commitem 1 commit por rodada até
> 23h59 do dia.
>
> **Eventos relâmpago:** registrem apenas se o grupo for **afetado** (o professor
> chama quem for; não ser chamado é bom sinal).

**Modelo (copiar para cada decisão):**

```
## <título ditado pelo professor>

**Tipo:** ( rodada / relâmpago ) · **Voto:** ( A / B / C / D )

**Justificativa:**

<2 a 3 linhas: mecanismo técnico + trade-off>

**Placar do grupo após esta decisão:** 🔥 __ · 💰 R$ __ mil · 🧹 __
```

*(as decisões entram aqui, na ordem em que a madrugada as trouxer; placar inicial:
🔥 7 · 💰 0 · 🧹 2)*

---

## 🔎 O caminho do MEU grupo (preencher na 3ª aula, quando o mapa for revelado)

Uma linha por decisão registrada acima (usem os títulos ditados em aula):

| # | Decisão | Nossa letra | Consequência que ELA teria tido |
|---|---|---|---|
| 1 | Corrigir + escrever o teste (assertThrows) antes de subir | C | Demora 40 min a mais (8 clientes afetados no total), mas sobe com o assertThrows protegendo a invariante: a suíte ganha um teste honesto.  |
| 2 | Usar a derived query findByTitular (Spring Data JPA) | D | O caminho de arquitetura: usa a abstração certa (o repository já tinha a query pronta). Demora ~50 min, mais que A |
| 3 |  Tirar o endpoint de busca do ar | B | Busca fora do ar durante a BF: clientes não encontram as próprias contas, SAC explode. Injeção morta por ausência |
| 4 | Corrigir + reprocessar os extratos antigos | B | Corrige a gravação E reprocessa o histórico (~2h): os comprovantes já emitidos voltam certos; integridade total |
| 5 | Fail-fast: exceção clara na factory (tipo inválido) | A | A factory rejeita na hora com "Tipo de conta inválido: UNIVERSITARIA": o erro nasce e morre no mesmo lugar (~30 min) |
| 6 | Investigar agora e perder a janela de deploy | B | Perde a janela: as correções da noite só entram depois da abertura. MAS a investigação revela a verdade: o código é que está errado, o contrato diz R$ 5 |

---

# 📋 PÓS-MORTEM · relatório de incidente (montar em sala na 3ª aula)

> Rascunho em aula e commit final `pos-mortem:` até 23h59 do dia da 3ª aula.

## 1. Linha do tempo da madrugada

- 19h12 — Rodada 1: a API aceita transferências mesmo sem saldo; três clientes aparecem no vermelho e o contrato do banco quebra na primeira hora.
- 20h10 — evento relâmpago A: a busca por titular usa comparação por referência e volta vazia para nomes iguais, mesmo quando a conta existe.
- 21h02 — Rodada 2: tentativa de SQL injection via `OR '1'='1'`, confirmando que a query estava montada com concatenação e sem parâmetros.
- 22h40 — evento relâmpago B: o sistema deixa conexões JDBC abertas sem liberar e a base começa a degradar sob carga.
- 22h50 — Rodada 3: contas "gêmeas" são criadas com número duplicado, quebrando a promessa de sequência única e comprometendo a auditoria.
- 00h47 — Rodada 4: o extrato grava pagador e recebedor trocados, embora os saldos estejam corretos; o comprovante vira um erro de integridade de dados.
- 01h30 — Rodada 5: a `ContaFactory` retorna `null` para tipos desconhecidos e a exceção explode longe da causa real, sem mensagem útil para o cliente.
- 02h20 — evento relâmpago C: o cadastro aceita objetos incompletos, criando estados inválidos que só explodem mais tarde.
- 03h20 — evento relâmpago D: o teste passa sem `assert`, dando uma falsa sensação de segurança e mascarando regressões na suíte.
- 05h00 — go/no-go: a equipe precisa decidir se a janela de deploy vale a correção ou se a causa raiz deve ser resolvida antes de seguir.

## 2. Causa raiz de 2 incidentes (aula + mecanismo técnico)

**Incidente 1:** Transferências sem saldo aceitas e extrato com dados invertidos

Aula/mecanismo: Aula 13/14 — a invariante de negócio foi contornada no serviço; a exceção de saldo insuficiente era capturada e o fluxo continuava, e o extrato foi gravado com origem/destino trocados. O trade-off foi “correr para publicar” em vez de preservar a regra de negócio e a integridade do histórico.

**Incidente 2:** Busca vulnerável e acesso ao banco sem controle de conexão

Aula/mecanismo: Aula 11/12 — o código montava SQL com `Statement` + concatenação, comparava `String` com `==` e não liberava a conexão depois da consulta. O trade-off foi a velocidade de um hotfix funcional contra segurança, consistência e operação estável do banco.

## 3. O que faríamos diferente (2 rodadas + por quê)

- Na Rodada 1, teríamos escolhido C e escrito o `assertThrows` antes de subir: a invariante “saldo nunca pode ficar negativo” é um contrato do negócio e precisa de teste para não voltar em produção.
- Na Rodada 2, teríamos usado a camada correta do acesso a dados (`PreparedStatement`/`findByTitular` + `try-with-resources`) em vez de improvisar no endpoint: isso bloqueia injeção e aumenta a previsibilidade do banco sem sacrificar a operação.

## 4. A maior lição da equipe

A maior lição foi que velocidade sem invariante vira dívida. Em produção, um código “que compila” e “que sobe” não vale nada se a regra do negócio, a segurança do banco e o teste de regressão forem ignorados — o problema não está só no bug, está no mecanismo que deixou a falha escapar.
