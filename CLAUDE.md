# Controle de Gastos — Contexto do Projeto (versão pública)

> **Este arquivo é público** (repositório no GitHub e site no GitHub Pages). Aqui entram só a arquitetura, as regras do
> app e as regras de trabalho. **Nunca** coloque aqui valores, nomes de terceiros, contratos, CPF, senhas, e-mails ou
> qualquer dado pessoal/financeiro. Esse detalhe vive em `CLAUDE.local.md` (contexto vivo) e `HISTORICO-PRIVADO.md`
> (histórico completo com valores), ambos **locais e no `.gitignore`**; o Claude Code os lê quando a sessão é aberta
> nesta pasta.

## O que é

App de fluxo financeiro pessoal de um casal, em **um único arquivo HTML** (`gastos-interativo.html`). Substituiu uma
planilha. Roda direto no navegador, sem servidor.

**Stack:** HTML/CSS/JS puro · Chart.js 4.4 (CDN) · Firebase Realtime Database (compat 10.13) para sincronização entre os
computadores · localStorage (`gastosApp_v1`) como armazenamento local · API do Banco Central (série 4391) para o CDI real.

**Visual:** tema escuro (`--bg #0f1420`, painéis `#171e2e`, acento verde `#4fd1a5`, azul `#5b8def`, vermelho `#f0616b`,
amarelo `#f2b94b`). Chart.defaults com texto claro (senão a legenda some no fundo escuro).

## Abas (estado atual)

| Aba | Função |
|---|---|
| Dashboard | KPIs, evolução mensal (últimos 12m / desde o início + projeção até o fim do financiamento), pizza de composição, detalhamento por categoria com drill-down nos itens |
| Lançamentos (Cartão) | Consumo avulso mensal do cartão + formulário rápido de compra parcelada (parcela = total ÷ qtd). **Editar** carrega a compra no mesmo formulário (`iniciarEdicaoLan`) e atualiza o item da Reforma vinculado. **Acompanhamento do consumo:** checkpoints no painel "Consumo mensal do cartão" — registra a soma das faturas numa data, desconta assinaturas e parcelas e compara com o orçamento de comparação (`CONSUMO_CARTAO_BASE`); o consumo do último checkpoint vai pro consumo do mês e pro Dashboard |
| Recorrentes | Itens fixos "eternos" (mensalidades, assinaturas), modelo "vale até mudar" com histórico de reajustes |
| Prazo Definido | Despesas temporárias com mês de fim |
| Parcelas (Cartão) | Parcelamentos; panorama de quanto fecha por mês; `qtdParcelas` é a única fonte de verdade |
| Reforma do Apto | Itens a comprar / já lançados, com vínculo ao pagamento (parcelado → Parcelas; à vista/boleto → Prazo Definido); mostra "2/10", valor já pago, editar/desfazer lançamento |
| Aplicações | Saldo base, saldo real de fechamento mensal, CDI, salários mensais + salário de referência, evolução mês a mês (projetado × atualizado × real). **Segunda análise:** painel abaixo com o gráfico "sobra do salário vira aporte" (linha sem × com aporte, barras de sobra/falta dos meses projetados, cards de resumo), horizonte próprio (padrão +5 anos), dois botões de base — **Real** (parte do último saldo real lançado) e **Projetado** (parte do saldo base e do salário de referência, sem usar o saldo real: visão mais pessimista, igual à linha "Saldo aplicado (projetado)" do gráfico de cima) — e **Entradas extraordinárias previstas** (tabela + formulário; ★ amarela = suposição, verde = confirmada, tooltip com a descrição; **Editar** reaproveita o formulário para mudar mês, descrição, valor e tipo; **chips** acima dos cards ligam/desligam cada entrada no gráfico e nos cards, com atalhos "todas / só confirmadas / nenhuma") |

Header: configurar sincronização · restaurar backup automático · exportar/importar backup JSON.

## Modelo de dados (`STATE`)

- `recorrentes[]` — `{id, nome, grupo, historico:[{mes, valor}]}`
- `recorrentesTemporarios[]` — igual + `mesFim`
- `parcelas[]` — `{id, descricao, valorParcela, qtdParcelas, mesInicio, categoria, dataCompra…}`
- `lancamentos[]` — legado; todo lançamento vira parcela
- `cartaoAvulso` — histórico mês a mês do consumo avulso; meses sem valor usam o orçamento base
- `cartaoCheckpoints[]` — `{id, mes, data (AAAA-MM-DD), valor, tipo:'fatura'|'consumo'}`. Consumo real = valor informado
  (soma das faturas) − assinaturas do grupo "Assinaturas/Cartão" − parcelas do mês (`deducoesCartaoMes`). `mes` = mês da
  fatura. Ao registrar/excluir um checkpoint, o consumo real do checkpoint mais recente do mês é gravado em
  `cartaoAvulso` (`sincronizarCheckpointComConsumo`); consumo negativo também é gravado (fatura incompleta ou estorno).
- `aptoPlanejado[]` — itens da reforma, com vínculo (`lancamentoRef`) ao lançamento
- `investimentos` — `{saldoInicial, mesInicial, salarioRefHenrique, salarioRefJuliana}` + `saldoReal[]` por mês
- `entradasPrevistas[]` — `{id, mes, valor, descricao, tipo:'suposicao'|'confirmada'}`: entradas extraordinárias na
  reserva, fora do salário, valores líquidos de IR. Entram **só na segunda análise** (`entradasPrevistasMes()` dentro de
  `computeSaldoMemo(...,aporta=true)`), nunca no gráfico principal nem no "Saldo atualizado". Mês com saldo real
  lançado ignora a entrada (o real já a inclui). **A seleção de quais entradas entram no gráfico NÃO fica nos dados**: são
  os ids desmarcados em `localStorage` (`gastosApp_entradasDesmarcadas`), por navegador, pra explorar cenários sem gravar
  nada na nuvem (cada pessoa pode ter a sua); `entradaAtiva()` filtra em `entradasPrevistasMes()`
- `cdiMensal` — histórico `[{mes, valor}]` vindo do BCB
- `_migrou*` flags — ajustes já aplicados (não reexecutar); `_syncTs` — carimbo p/ sync; `_cdiAtualizadoEm`

Padrão central: **histórico de pontos de mudança** `[{mes, valor}]` + `getValorNoMes()` ("vale até mudar"), reaproveitado em
recorrentes, CDI e salário de referência. Salário real e cartão avulso são editados mês a mês (valor exato).

## Engine de cálculo — regras definidas

- `computeMes(mes)` soma: recorrentes + prazo definido + parcelas ativas + cartão avulso (+ reforma via vínculos).
- `faltaOuSobraMes` = salários − custo do mês. **Só o que falta sai das aplicações; a sobra não é aportada
  automaticamente** (decisão do dono). O gráfico principal e a tabela seguem assim. A **segunda análise**
  (`computeSaldoMemo(mes, cache, aporta, modo)` e `renderInvestimentosAporte`; `modo` = `'real'` ou `'projetado'`) mostra o cenário alternativo em que a sobra
  entra na reserva no fim do mês, com as mesmas premissas (âncora no último saldo real, `taxaCdiMes`, `faltaOuSobraMes`);
  com `aporta=false` reproduz exatamente o "Saldo atualizado" (modo `real`) ou o "Saldo aplicado (projetado)" (modo
  `projetado`, que também aplica o fluxo do mês-base ao saldo base) — diferença zero, conferida. Sobra e entradas só contam nos meses depois do último saldo real.
  Horizontes longos só dão ordem de grandeza (custos e salário parados em valor de hoje, juros compostos).
- **Saldo projetado** (`computeSaldoInvestido`): parte do saldo base, aplica CDI mês a mês, desconta só o que faltou, usando
  **sempre o salário de referência** (estável; só muda se mudar a referência/base).
- **Saldo atualizado** (`computeSaldoAtualizado`): igual, mas parte do **saldo real mais recente lançado**.
- **Saldo real**: lançado manualmente no fechamento de cada mês; serve para conferir projeção × realidade.
- **CDI:** a projeção usa um **CDI fixo conservador em todos os meses** (`CDI_PROJECAO_PCT` / `taxaCdiMes()`), abaixo do
  CDI recente de propósito. O CDI real do BCB continua sendo buscado e aparece só como informação; **nunca** usar o valor
  do mês corrente (acumulado parcial) como referência. Pra mudar a premissa, editar só `CDI_PROJECAO_PCT`.

## Migrações e correções de dados

- `aplicarMigracoes()` é só um **gancho** (hoje vazio). Roda **depois** da sincronização com o Firebase escolher o estado
  mais novo e sempre grava com `saveStateInterno()` (nunca `saveState()`, pra não competir com dados do usuário). Cada
  migração futura precisa de flag `_migrou*` própria e nunca é reexecutada.
- **Código público não leva valores pessoais.** Até 25/11/2026 as correções de dados iam como migração dentro do HTML;
  isso foi removido (já estavam aplicadas nos dados) e o histórico completo ficou em `HISTORICO-PRIVADO.md`. **As
  correções de dados agora são feitas direto nos dados** (fora do código) — ver "Como trabalhar".
- Restaurar um backup **antigo** não reaplica correções posteriores; usar backup recente ou refazer a correção.
- Tipos de ajuste já feitos (detalhe privado): reclassificação de itens entre recorrente/prazo definido/parcela,
  conferência mensal das faturas, cronograma de pagamentos de contratos, financiamento pelo demonstrativo do banco,
  correção de nomes e vínculos da Reforma.

## Bugs corrigidos que não podem voltar

- **Perda de lançamentos por sync:** a busca do CDI (assíncrona) salvava/empurrava um STATE local desatualizado antes de
  puxar o Firebase. `initFirebaseSync()` retorna Promise e tudo que pode disparar `saveState()` espera ela.
- Evento remoto atrasado sobrescrevia edição local mais nova → só aceita remoto se `_syncTs` for **realmente mais novo**.
- Firebase "esquece" arrays vazios → `sanitizeState()` garante todos os campos (inclui `entradasPrevistas`).
- Backup automático a cada envio em `gastosApp/historico` (últimos 50), com `meta` {motivo, dispositivo, versão, resumo},
  restaurável pelo botão; restaurar/importar backup passa por `sanitizeState` + `aplicarMigracoes`.
- Itens da reforma apontando para parcelas inexistentes (efeito do bug de sync) — vínculos precisam sempre de conferência.
- **Lançamentos sumindo ao reabrir:** migrações rodavam antes do merge com o Firebase e carimbavam `_syncTs = agora`.
  **Regra:** gravação automática usa `saveStateInterno()`; só ação do usuário usa `saveState()`; `_syncTs` = "última edição
  do usuário".
- Na abertura, se o local estiver marcado como mais novo que a nuvem, o app **pergunta** (padrão = nuvem).
  **Todo dispositivo precisa usar a mesma versão do HTML** (`APP_VERSAO` + id do dispositivo no cabeçalho).
- **Projeção de saldo pessimista por CDI parcial:** repetir o último CDI da série (mês corrente parcial) nos meses futuros
  subestimava o saldo. Correção: `taxaCdiMes()` (CDI fixo).

## Segurança e privacidade (decisões)

- **Login obrigatório** (Firebase Authentication, Google Sign-In) cobre todo o app; nada renderiza nem sincroniza antes do
  login (`initAuthGate()`, `iniciarApp()`). O Realtime Database tem regras que só liberam leitura/escrita
  (`gastosApp`) para as duas contas autorizadas (e-mails no `HISTORICO-PRIVADO.md`):
  ```json
  { "rules": { ".read": false, ".write": false,
    "gastosApp": {
      ".read":  "auth != null && (auth.token.email === 'EMAIL_1' || auth.token.email === 'EMAIL_2')",
      ".write": "auth != null && (auth.token.email === 'EMAIL_1' || auth.token.email === 'EMAIL_2')" } } }
  ```
  O domínio do GitHub Pages precisa estar em Authentication → Settings → Authorized domains.
- O `firebaseConfig` no HTML é público por natureza; quem protege os dados são o login e as regras acima (verificado:
  leitura sem login devolve 401).
- **O repositório e o site são públicos.** Por isso: nada de dado pessoal no código nem aqui; `CLAUDE.local.md`,
  `HISTORICO-PRIVADO.md`, PDFs de fatura, backups JSON e `.privacidade.txt` ficam no `.gitignore`. Um **hook de commit**
  local bloqueia arquivos proibidos, CPF e os termos listados em `.privacidade.txt` (lista local, também ignorada). Antes
  de qualquer publicação: `git diff --cached` e conferir.

## Hospedagem e versionamento

- Publicado no **GitHub Pages** (URL única). Atualizar = commit + push; o Pages guarda a página por ~10 min, então depois
  de publicar vale **Ctrl+F5** e conferir `APP_VERSAO` no cabeçalho. **Toda entrega nova deve incrementar `APP_VERSAO`**
  (formato `AAAA-MM-DDx`): ela é gravada em `gastosApp/versaoPublicada` e cópias antigas mostram aviso vermelho.
- Ao abrir numa URL nova o localStorage começa vazio → sem `_syncTs` local → a nuvem vence (comportamento desejado).

## Como trabalhar com o Claude neste projeto

- Claude Code na pasta do projeto (clone do repositório). `CLAUDE.md` (este) + `CLAUDE.local.md` são lidos
  automaticamente; novas análises e dados pessoais vão **só** para o `CLAUDE.local.md`.
- Toda mudança de código termina com: incrementar `APP_VERSAO`, atualizar este arquivo se a regra mudou, testar (carregar
  um estado de teste no app isolado e conferir números e render), `git commit` + `git push`.
- **Rotina mensal:** faturas fechadas dos dois cartões + estado dos dados + valores do portal de contratos → conferir
  linha a linha → listar divergências → o dono aprova → aplicar a correção **nos dados** (não em código) → conferir de
  novo. **Checkpoint do mês = soma das faturas dos dois cartões.**
- **Regras do dono:** (1) só deduzir a que item uma compra corresponde quando o valor é **exato** (ou centavos de
  arredondamento); diferença maior não é evidência — conferir com lista de compras/nota e, na dúvida, perguntar; (2) quando
  uma frase e a planilha/portal divergirem, **o portal manda** — mostrar a contagem antes de decidir; (3) descrição de
  lançamento usa o **nome do item**, nunca o da loja; (4) estimativas **conservadoras**; (5) o gráfico principal não
  aporta a sobra.
- Exportar backup (ou baixar o estado) antes de mudanças estruturais no STATE; nunca reexecutar `_migrou*`.
