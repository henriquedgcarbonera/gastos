# Controle de Gastos — Contexto do Projeto

> Documento de continuidade. Guarde na mesma pasta do `gastos-interativo.html`.
> Ao começar uma nova sessão com o Claude, envie este arquivo junto com o HTML
> (ou aponte a pasta no Claude Code / Cowork) para retomar de onde parou.
>
> Reconstruído em 08/09/2026 a partir dos comentários e do código do próprio HTML,
> após a perda do histórico da sessão original (app corrompido/reinstalado).

## O que é

App de fluxo financeiro pessoal de Henrique e Juliana, em **um único arquivo HTML**
(`gastos-interativo.html`, ~160 KB). Substituiu uma planilha original (abas
"Custos" e coluna "Cartão"). Roda direto no navegador, sem servidor.

**Stack:** HTML/CSS/JS puro · Chart.js 4.4 (CDN) · Firebase Realtime Database
(compat 10.13) para sincronização entre os dois computadores · localStorage
(`gastosApp_v1`) como armazenamento local · API do Banco Central (série 4391)
para CDI mensal real.

**Visual:** tema escuro (`--bg #0f1420`, painéis `#171e2e`, acento verde `#4fd1a5`,
azul `#5b8def`, vermelho `#f0616b`, amarelo `#f2b94b`). Chart.defaults ajustado
para texto claro (senão a legenda fica ilegível no fundo escuro).

## Abas (estado atual)

| Aba | Função |
|---|---|
| Dashboard | KPIs, evolução mensal (últimos 12m / desde o início + projeção de +6m até fim do financiamento em 2061), pizza de composição, detalhamento por categoria com drill-down nos itens |
| Lançamentos (Cartão) | Consumo avulso mensal do cartão + formulário rápido de compra parcelada (calcula parcela = total ÷ qtd automaticamente). **Editar** (v08c): botão nas tabelas de Lançamentos e Parcelas carrega a compra no mesmo formulário (`iniciarEdicaoLan`), salva no lugar (mesmo id) e atualiza item da Reforma vinculado |
| Recorrentes | Itens fixos "eternos" (mensalidades, assinaturas), modelo "vale até mudar" com histórico de reajustes |
| Prazo Definido | Despesas temporárias com mês de fim (Sociedade, Consórcio, Ozzy, Aluguel, Financiamento…) |
| Parcelas (Cartão) | Parcelamentos; panorama de quanto fecha por mês; `qtdParcelas` é a única fonte de verdade |
| Reforma do Apto | Itens a comprar / já lançados, com vínculo ao pagamento (parcelado → Parcelas; à vista/boleto → Prazo Definido); mostra "2/10", valor já pago, editar/desfazer lançamento |
| Aplicações | Saldo base, saldo real de fechamento mensal, CDI, salários mensais + salário de referência para projeção, evolução mês a mês (projetado × atualizado × real) |

Header: configurar sincronização · restaurar backup automático · exportar/importar backup JSON.

## Modelo de dados (`STATE`)

- `recorrentes[]` — `{id, nome, grupo, historico:[{mes, valor}]}`
- `recorrentesTemporarios[]` — igual + `mesFim`
- `parcelas[]` — `{id, nome, valorParcela, qtdParcelas, mesInicio, categoria, dataCompra…}`
- `lancamentos[]` — legado; hoje todo lançamento vira parcela (migração unificou os dois sistemas)
- `cartaoAvulso` — histórico mês a mês do consumo avulso; meses sem valor usam base **R$ 3.000**
- `aptoPlanejado[]` — itens da reforma, com `parcelaId`/vínculo ao lançamento
- `investimentos` — `{saldoInicial, mesInicial (jan/2026), salarioRefHenrique, salarioRefJuliana}` + saldo real por mês
- `cdiMensal` — histórico `[{mes, valor}]` vindo do BCB
- `_migrou*` flags — migrações já aplicadas (não reexecutar); `_syncTs` — carimbo p/ sync; `_cdiAtualizadoEm`

Padrão central: **histórico de pontos de mudança** `[{mes, valor}]` + `getValorNoMes()` ("vale até mudar"),
reaproveitado em recorrentes, CDI e salário de referência. Salário real e cartão avulso são editados
mês a mês (valor exato, sem reajuste).

## Engine de cálculo — regras definidas

- `computeMes(mes)` soma: recorrentes + prazo definido + parcelas ativas + cartão avulso (+ reforma via vínculos).
- `faltaOuSobraMes` = salários (Henrique + Juliana) − custo do mês. **Só o que falta sai das aplicações;
  sobra não é aportada automaticamente** (decisão do Henrique).
- **Saldo projetado** (`computeSaldoInvestido`): parte do saldo base de jan/2026, aplica CDI mês a mês,
  desconta só o que faltou. Usa **sempre o salário de referência** → estável, só muda se mudar a referência/base.
- **Saldo atualizado** (`computeSaldoAtualizado`): igual, mas reparte do **saldo real mais recente lançado**,
  para a projeção futura não acumular erro desde o começo.
- **Saldo real**: lançado manualmente no fechamento de cada mês (soma do "Patrimônio final" das duas contas);
  jan–jul/2026 já preenchidos via migração. Serve para conferir projeção × realidade.
- CDI: meses passados com valor real do BCB; futuros repetem o último conhecido. Atualiza sozinho ao abrir.

## Migrações já feitas (histórico das decisões)

1. Sociedade → prazo definido, reajuste a cada 2 meses, fim 05/2028 (v2 completou o histórico).
2. Consórcio → prazo definido, fim 07/2032.
3. Ozzy Parcela e Ozzy Reforma → prazo definido, fim no mês em que zeram na planilha.
4. Aluguel/AP encerra 08/2026; a partir daí **Financiamento**: R$ 8.000 caindo R$ 15/mês até 07/2061
   (histórico longo é resumido numa linha só — tags mês a mês confundiam).
5. Saldo real das aplicações jan–jul/2026 lançado; jan/2026 = base das projeções.
6. Lançamentos e Parcelas unificados (lançamento = parcela).
7. Cadeiras + Banquetas (Rosa) juntadas numa parcela só (10x R$ 701).
8. Recorrente que ganha data de fim vira automaticamente prazo definido.

## Bugs corrigidos que não podem voltar

- **Perda de lançamentos por sync:** a busca do CDI (assíncrona) salvava/empurrava um STATE local
  desatualizado antes de puxar o Firebase. Correção: `initFirebaseSync()` retorna Promise e tudo que
  pode disparar `saveState()` espera ela.
- Evento remoto atrasado sobrescrevia edição local mais nova. Correção: só aceita remoto se `_syncTs`
  for **realmente mais novo** (não apenas diferente).
- Firebase "esquece" arrays vazios → `sanitizeState()` garante todos os campos.
- Backup automático a cada envio em `gastosApp/historico` (últimos 50), restaurável pelo botão.
- Itens da reforma apontando para parcelas inexistentes (efeito colateral do bug de sync) — corrigido por migração.
- **(08/09/2026) Lançamentos sumindo ao reabrir o arquivo** (ex: cartão set/26 R$ 5.400 → base R$ 3.000).
  Causa: migrações rodavam ANTES do merge com o Firebase e chamavam `saveState()`, que carimbava
  `_syncTs = agora`; o local (vindo do SEED, sem os dados) passava a "ser mais novo" e sobrescrevia o Firebase.
  Correção: (1) migrações agora vivem em `aplicarMigracoes()` e rodam só no `.finally` de `initFirebaseSync()`
  e após aplicar estado remoto; (2) `saveStateInterno()` salva/envia sem mexer em `_syncTs` — usado por
  migrações e pela atualização automática do CDI; (3) `_syncTs` passou a significar "última edição do usuário".
  **Regra daqui pra frente:** qualquer gravação automática do app usa `saveStateInterno()`; só ação do usuário usa `saveState()`.
- **(08/09/2026, v2026-09-08b) Após a correção acima, um F5 ainda perdeu ago/26.** Causa não confirmada
  (suspeita: outro dispositivo/aba rodando versão antiga do HTML, ou abertura em navegador com localStorage do SEED).
  Defesas adicionadas: (1) na abertura, se o local estiver marcado como mais novo que a nuvem, o app **pergunta**
  antes de sobrescrever — padrão é usar a nuvem; (2) backups em `gastosApp/historico` agora têm chave `Date.now()`
  e `meta` {motivo, dispositivo, versão, resumo} — o diálogo "Restaurar backup" mostra quem enviou cada estado;
  (3) `APP_VERSAO` + id do dispositivo aparecem no cabeçalho; (4) restaurar/importar backup passa por
  `sanitizeState` + `aplicarMigracoes`. Formato do snapshot mudou para `{meta, estado}` (restore lê os dois formatos).
  **Ao investigar nova perda:** abrir "Restaurar backup automático" e ler motivo/dispositivo dos últimos envios.
  **Todo dispositivo (inclusive o da Juliana) precisa usar a mesma versão do HTML.**

## Pendências / próximos passos

- Login com Google **confirmado funcionando em produção** em 08/09/2026 (Henrique testou, F5 manteve o
  registro). Falta a Juliana confirmar no computador dela com `renderjuliana@gmail.com`.
- Decidir se o arquivo vira `index.html` (facilitaria a URL, mas não é urgente).
- (adicionar aqui conforme as sessões avançarem)

## Segurança — login obrigatório (decisão 08/09/2026, v2026-09-08f)

- **Problema encontrado:** o Realtime Database `gastos-mensais-3e39a` estava com `.read`/`.write` liberados
  pra qualquer pessoa — confirmado lendo dados reais (salários, saldo) via API pública sem nenhuma credencial.
  O app nunca teve autenticação, só o `firebaseConfig` (que é público por natureza, fica embutido no HTML).
- **Correção implementada no HTML:** tela de login com **Google Sign-In** (Firebase Authentication) cobrindo
  todo o app — nada renderiza nem sincroniza antes do login (`initAuthGate()`, perto do fim do script, usa
  `signInWithPopup(new firebase.auth.GoogleAuthProvider())`). Header ganhou botão "Sair". `iniciarApp()`
  isola o que antes rodava direto no carregamento (renderAll + initFirebaseSync) e só é chamado depois que
  `onAuthStateChanged` confirma um usuário.
- **Contas autorizadas:** `henriquedgcarbonera@gmail.com` (Henrique) e `renderjuliana@gmail.com` (Juliana).
- **Passos que só vocês podem fazer no Firebase Console** (projeto `gastos-mensais-3e39a`):
  1. Authentication → Sign-in method → **Google já foi ativado** (feito em 08/09/2026, projeto OAuth
     `88537286531`).
  2. Authentication → Settings → **Authorized domains** → adicionar `henriquedgcarbonera.github.io`
     (sem isso o popup do Google falha com `auth/unauthorized-domain` fora do localhost).
  3. Realtime Database → Regras → colar:
     ```json
     {
       "rules": {
         ".read": false,
         ".write": false,
         "gastosApp": {
           ".read": "auth != null && (auth.token.email === 'henriquedgcarbonera@gmail.com' || auth.token.email === 'renderjuliana@gmail.com')",
           ".write": "auth != null && (auth.token.email === 'henriquedgcarbonera@gmail.com' || auth.token.email === 'renderjuliana@gmail.com')"
         }
       }
     }
     ```
- **Enquanto o domínio não estiver autorizado, não faça `git push` desta versão** — o botão "Entrar com
  Google" falharia no app publicado (funciona em localhost, onde já é autorizado por padrão).

## Aba Teste — removida (08/09/2026, v2026-09-08g)

- A aba 🧪 Teste (diagnóstico de versão/dispositivo/sincronização, `renderTeste()`, `STATE.testes`) cumpriu
  seu papel: confirmou que o bug de perda de lançamentos ao reabrir não voltou, e que o login com Google
  funciona (F5 mantém sessão e dados). Removida do HTML, nav e JS (`renderAll` não chama mais `renderTeste`).
  `sanitizeState` continua normalizando a chave `testes` como array (histórico antigo no Firebase não quebra),
  mas nada mais grava ou lê ela.
- Backtest (projeção × saldo real) — escopo definido em 08/09/2026: comparar mês a mês. Implementado na
  aba **Aplicações**, dentro de "Evolução mês a mês": a tabela já comparava saldo projetado × saldo real
  (coluna "Real − projetado"); adicionei o **% de desvio** ao lado do valor em R$ e um resumo acima da
  tabela (`#backtestResumo`) com desvio médio e o mês de maior desvio, calculado só sobre os meses em que
  saldo projetado e saldo real lançado coexistem.

## Hospedagem e versionamento (decisão 08/09/2026)

- Problema: cópias locais do HTML em vários caminhos/dispositivos → versões diferentes rodando ao mesmo tempo, F5 não traz versão nova.
- Decisão: publicar o app em **GitHub Pages** (URL única); atualizar = subir o arquivo novo no repositório. Ambos os computadores usam a URL.
- `APP_VERSAO` (formato `AAAA-MM-DDx`) é gravado em `gastosApp/versaoPublicada` no Firebase pela cópia mais nova que conectar;
  cópias mais antigas mostram aviso vermelho no cabeçalho (`#avisoVersao`). **Toda entrega nova deve incrementar `APP_VERSAO`.**
- Ao migrar para a URL, o localStorage começa vazio nesse origin → sem `_syncTs` local → nuvem vence (comportamento desejado).

## Publicação — estado em 08/09/2026 (fim da sessão no chat)

- Repositório: `github.com/henriquedgcarbonera/gastos` (público). App em `https://henriquedgcarbonera.github.io/gastos/gastos-interativo.html`.
- **Publicado hoje: v2026-09-08d.** A v2026-09-08e (aba 🧪 Teste) foi gerada mas NÃO chegou ao repositório — o upload manual
  reenviou o 08d ("0 file changed"). O arquivo 08e é o `gastos-interativo.html` desta pasta. **Primeira tarefa no Claude Code: commit + push dele.**
- Bug de perda de lançamentos (cartão ago/set-26) ainda **não confirmado como resolvido**: hipótese principal era cópia antiga em outro
  dispositivo; com a URL única e o aviso de versão, essa hipótese fica eliminada. Roteiro de teste está na aba Teste (registrar → F5 → outro PC).
- Pendências decididas: (1) remover o `SEED` embutido do HTML (dados pessoais em repositório público; tudo já está no Firebase — o
  `sanitizeState` precisa passar a partir de um objeto vazio com as chaves); (2) revisar regras de leitura/escrita do Realtime Database
  `gastos-mensais-3e39a`; (3) backtest projeção × saldo real (escopo a definir); (4) decidir se o arquivo vira `index.html`.

## Como trabalhar com o Claude neste projeto

- Enviar `gastos-interativo.html` + este `CONTEXTO-projeto-gastos.md` no início da sessão.
- **Fluxo oficial (a partir de 08/09/2026): Claude Code** na pasta do projeto (que é um clone do repositório `gastos`).
  Este arquivo vive na raiz como `CLAUDE.md` e é lido automaticamente. Toda melhoria termina com: incrementar `APP_VERSAO`,
  atualizar este arquivo, `git commit` + `git push` → Henrique só dá F5 na URL. Nunca entregar HTML para download.
- Toda mudança de regra de negócio: registrar aqui e em comentário no código (foi isso que permitiu
  reconstruir o contexto).
- Nunca reexecutar migrações (`_migrou*`); criar migração nova com flag própria.
- Exportar backup JSON antes de mudanças estruturais no STATE.
