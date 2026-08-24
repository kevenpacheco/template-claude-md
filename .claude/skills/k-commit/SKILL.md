---
name: k-commit
description: Unico ponto do projeto que toca git/gh para branch, commit (Conventional Commits em portugues), push, PR e merge (apos aprovacao explicita do usuario). Qualquer fluxo que precise dessas operacoes deve chamar esta skill em vez de rodar os comandos diretamente. Use sempre que for criar um commit, abrir um PR ou mesclar algo no projeto.
---

# Skill: k-commit

## Quando usar

Use esta skill sempre que uma operacao de git/gh relacionada a branch, commit, push, PR, CI ou merge precisar acontecer no projeto — de forma isolada (ad hoc) ou chamada internamente por outro fluxo/skill que precise dessas operacoes. Nenhum outro fluxo do projeto roda `git`/`gh` para essas coisas por conta propria; todos chamam o `k-commit`.

Excecao: leituras que nao mudam estado (`git status`, `git branch --show-current`, ou uma sincronizacao de leitura antes de investigar codigo) nao passam por aqui — nao sao commit, branch nova, push, PR nem merge.

## Objetivo

Ser o unico lugar do projeto que decide **como** o git/gh e usado (regras de mensagem, seguranca, ordem das operacoes). Quem chama decide **o que**: quais arquivos, qual mensagem sugerida, qual conteudo de PR. O `k-commit` nunca inventa contexto de negocio que nao recebeu.

## Modos de uso

### 1. Isolado (ad hoc)

Ciclo completo descrito nesta skill: branch -> commits pequenos -> revisao automatizada -> push unico -> PR -> merge. Pergunte ao usuario a cada transicao relevante (ver "Regras do ciclo de shipping").

### 2. Chamado internamente por outro fluxo/skill

O chamador ja decidiu o que fazer e, quando e o caso, ja perguntou ao usuario. Tres acoes possiveis, chamadas separadamente:

**a) Garantir a branch** — o chamador informa o nome exato da branch (ex.: prefixo por tipo de mudanca + slug, calculado por quem chamou). Execute so "Atualizar `main` antes de criar nova branch" / o passo 1 do "Fluxo de decisao de branch", usando esse nome exato em vez de inventar um, e pare. Sem commit, sem pergunta ao usuario — e infraestrutura, nao decisao. Automatico mesmo que o usuario ainda nao tenha decidido comitar nada, quando os passos seguintes do fluxo do chamador precisarem de uma branch de feature de pe pra funcionar.

**b) Commit** — chamado so depois que o chamador confirmou com o usuario que e hora de comitar (ou, em fluxos que comitam uma unidade de trabalho por vez, porque comitar e parte inerente de concluir essa unidade — sem pergunta extra nesse caso). Adicione os arquivos indicados pelo chamador, redija a mensagem seguindo o padrao desta skill (regras 1-14), comite. **Nunca push, nunca PR, nunca merge nesta acao** — pare depois do commit.

O chamador pode **fixar a branch atual** como destino ("comite aqui, sem trocar de branch"). Nesse caso pule o passo 2 do "Fluxo de decisao de branch" — nao avalie se a branch combina com o conteudo, nao volte pra `main`, nao crie branch nova. E o caso do registro de achado fora do escopo: o arquivo tem contexto diferente do da branch de proposito, e trocar de branch no meio quebraria o trabalho em andamento. O passo 1 continua valendo: em branch bloqueada (`main`), crie uma branch mesmo assim — commit direto na `main` nao acontece em hipotese nenhuma.

**c) Ciclo de shipping** — chamado so quando o chamador ja decidiu, com o usuario, que e hora de subir a mudanca (ex.: no encerramento de um fluxo maior de varios commits). O chamador fornece o alvo do PR e o titulo/corpo ja montados (contexto de negocio de quem chamou, nao reconstruido aqui). Pule a pergunta "ha mais mudancas pendentes" — quem chamou ja decidiu. Rode, na ordem: revisao automatizada -> push unico -> abrir PR com o titulo/corpo recebido -> pergunta de merge -> limpeza (ver "Regras do ciclo de shipping").

## Regras obrigatorias

1. Mensagens sempre em portugues brasileiro
2. Seguir Conventional Commits: `tipo(escopo): descricao`
3. Nunca mencionar ajuda, co-participacao ou uso de LLMs, IAs ou assistentes
4. Nunca usar `Co-Authored-By` de LLMs ou IAs
5. Nunca usar `--no-verify` ou `--no-gpg-sign`
6. Nunca fazer `git add .` ou `git add -A` — adicionar apenas os arquivos relevantes pelo nome
7. Sempre verificar `git status` e `git diff` antes de commitar
8. Sempre verificar `git log --oneline -5` para manter consistencia com o historico
9. Nunca fazer commit diretamente na branch bloqueada `main` — verificar a branch atual antes de commitar e, se estiver nela, criar uma nova branch para receber as mudancas
10. Se a branch atual nao for bloqueada, avaliar se ela faz sentido para o contexto das modificacoes; se nao fizer, criar uma nova branch a partir de `main`
11. Sempre iniciar novas implementacoes a partir da branch `main`, exceto quando uma branch ja existente fizer sentido para as alteracoes em curso e esteja sincronizada com a main
12. Priorizar commits pequenos e focados para facilitar a revisao e aprovacao do PR
13. Nunca mesclar para `main` sem perguntar antes. So pode mesclar apos aprovacao explicita do usuario para aquele PR especifico — nunca merge silencioso, nem uma aprovacao anterior vale como permissao permanente
14. Antes de commitar em uma branch nao bloqueada, verificar se ela esta atualizada com `origin/main`. Se estiver desatualizada, fazer merge/rebase primeiro. Se houver conflitos, **nunca** resolver automaticamente — listar cada conflito e perguntar explicitamente ao usuario como resolver arquivo por arquivo

## Regras do ciclo de shipping

Aplica-se ao **uso isolado** e ao **modo 2c (ciclo de shipping)** chamado internamente. A diferenca entre os dois esta marcada em cada regra.

15. Push acontece **uma unica vez**, depois do ultimo commit pequeno da mudanca — nunca apos cada commit individual.
    - **Uso isolado:** como nao ha lista de tarefas que sinalize "acabou", **pergunte explicitamente** ao usuario ("sem mais mudancas pendentes, posso seguir pra push+PR?") antes de avancar. Nunca decida isso sozinho.
    - **Modo 2c:** essa pergunta ja foi feita pelo chamador antes de te chamar — pule direto pro push.
16. Antes do push, rode a revisao automatizada (ver "Revisao automatizada antes do PR") sobre o diff acumulado da branch.
17. Apos o push, abra o PR e imprima a URL no chat.
    - **Uso isolado:** alvo e `main`; titulo/corpo vem do proprio commit/contexto da conversa.
    - **Modo 2c:** alvo e titulo/corpo sao os que o chamador forneceu — nao invente conteudo de negocio aqui.
18. Apos abrir o PR, pergunte se o usuario ja quer mesclar agora e seguir o fluxo (ver "Merge e limpeza").

### Revisao automatizada antes do PR

Antes do push, revise o diff acumulado da branch (`main...HEAD`): dispare subagentes de revisao de codigo, cada um cobrindo um eixo (aderencia aos padroes/convencoes do projeto; aderencia ao pedido original; simplificacao e reuso), e resuma os achados. Rode sempre, mesmo em PR so de documentacao (spec/plan/tasks) — nao decida sozinho que um diff e "trivial demais" pra revisar. E uma passada **informativa, nao bloqueante**: mostre os achados ao usuario junto com o resumo do push/PR, mas nao pare o fluxo por causa deles — quem decide se algo precisa ser corrigido antes do PR e o usuario.

### Merge e limpeza

So pergunte sobre merge se o alvo do PR for uma branch protegida do projeto (ex.: `main`). Para outros alvos, pare no PR aberto — nada de merge automatico.

Se o usuario aprovar:

```bash
gh pr merge <numero-ou-url-do-pr> --merge --delete-branch
git checkout main
git pull origin main
```

Nao ha espera de CI proativa antes dessa tentativa — se a branch protegida exigir checks obrigatorios e eles ainda nao passaram, o comando falha. Nesse caso, reporte o motivo da recusa (a partir da saida do `gh`) e pare — sem tentar de novo automaticamente, sem contornar a protecao. Corrigir e repetir o ciclo, ou aguardar os checks, e decisao do usuario.

Se o usuario recusar: pare. O PR fica aberto e a branch intacta; a decisao de mesclar continua manual a partir daqui.

## Tipos permitidos

| Tipo       | Quando usar                                                  |
| ---------- | ------------------------------------------------------------ |
| `feat`     | Nova funcionalidade                                          |
| `fix`      | Correcao de bug                                              |
| `refactor` | Refatoracao sem alterar comportamento                        |
| `chore`    | Tarefas de manutencao, config, dependencias                  |
| `docs`     | Apenas documentacao                                          |
| `style`    | Formatacao, espacos, ponto e virgula (sem mudanca de logica) |
| `perf`     | Melhoria de performance                                      |
| `test`     | Adicao ou correcao de testes                                 |

## Formato da mensagem

```
tipo(escopo): descricao curta em portugues

Corpo opcional com mais detalhes sobre o que foi feito e por que.
```

### Regras do titulo (primeira linha)

- Maximo 72 caracteres
- Letra minuscula no inicio da descricao
- Sem ponto final
- Verbo no infinitivo (ex: "adicionar", "corrigir", "remover")
- Escopo e opcional — usar quando facilitar a compreensao (ex: `feat(api)`, `fix(banners)`)

### Regras do corpo (opcional)

- Separado do titulo por uma linha em branco
- Explicar o que e por que, nao o como
- Maximo 72 caracteres por linha

## Formato do comando

Sempre usar HEREDOC para garantir formatacao correta:

```bash
git commit -m "$(cat <<'EOF'
tipo(escopo): descricao curta

Corpo opcional.
EOF
)"
```

## Processo (comum a todos os modos, ate o commit)

1. Executar `git status` e `git branch --show-current` para ver arquivos alterados e branch atual
2. Executar `git diff` (staged e unstaged) para entender as mudancas
3. Executar `git log --oneline -5` para manter consistencia de estilo
4. Decidir a branch de destino seguindo o fluxo de decisao de branch (ou usar o nome exato recebido do chamador, no modo 2a/2b)
5. Se a branch de destino nao for bloqueada e for reutilizada: sincronizar com `origin/main` (ver "Sincronizar branch atual com `main`"). Se houver conflitos, perguntar ao usuario como resolver cada um antes de continuar
6. Adicionar apenas os arquivos relevantes com `git add <arquivo>` (os informados pelo chamador, no modo 2b)
7. Redigir mensagem seguindo o padrao (ou usar a mensagem sugerida pelo chamador, ajustando so o necessario pra bater com o padrao)
8. Criar o commit

A partir daqui, o que acontece depende do modo (ver "Modos de uso"): parar (2a/2b), ou seguir para o ciclo de shipping (isolado ou 2c).

## Fluxo de decisao de branch

Antes de commitar, seguir esta arvore de decisao:

1. **A branch atual e bloqueada (`main`)?**
   - Sim: criar uma nova branch a partir de `main` (seguindo obrigatoriamente o passo "Atualizar `main` antes de criar nova branch") com nome descritivo do contexto da mudanca (`tipo/descricao-curta`, ex.: `feat/visualizacao-condicional-relatorios`) — ou o nome exato recebido do chamador, no modo 2a/2b — e commitar nela.
   - Nao: seguir para o proximo passo.

2. **A branch atual faz sentido para o contexto das mudancas?**
   - Pulado quando o chamador fixou a branch atual como destino (modo 2b) — ver "Modos de uso".
   - Sim (mesma feature/fix em andamento): antes de commitar, executar a verificacao "Sincronizar branch atual com `main`" abaixo. So prosseguir com o commit apos a branch estar atualizada e sem conflitos.
   - Nao (contexto diferente, tema novo): voltar para `main`, atualizar (ver "Atualizar `main` antes de criar nova branch") e criar nova branch a partir dela.

### Sincronizar branch atual com `main`

**Regra obrigatoria** antes de qualquer commit em branch nao bloqueada:

```bash
git fetch origin main
git merge-base --is-ancestor origin/main HEAD || echo "DESATUALIZADA"
```

- Se a branch ja contem `origin/main` (esta atualizada): pode prosseguir para o commit.
- Se estiver desatualizada: fazer `git merge origin/main` (ou `git rebase origin/main`, se for o padrao da branch) para trazer as mudancas da `main` para a branch atual antes do commit.

**Tratamento de conflitos**:

- Se o merge/rebase gerar conflitos, **NAO resolver automaticamente**. Listar cada arquivo em conflito com `git status` e, para cada conflito encontrado, **perguntar explicitamente ao usuario como resolver** (qual lado manter, se deve combinar, etc.). Mostrar o trecho do conflito (`<<<<<<<` ... `=======` ... `>>>>>>>`) quando necessario.
- Nunca assumir a resolucao sem confirmacao. Nunca usar `-X ours`/`-X theirs` sem o usuario pedir.
- Depois de cada resolucao confirmada pelo usuario: `git add <arquivo>` e seguir para o proximo conflito.
- Apos resolver todos, finalizar com `git merge --continue` ou `git rebase --continue` (conforme o caso) e so entao prosseguir com o fluxo normal de commit.

### Atualizar `main` antes de criar nova branch

**Regra obrigatoria**: toda vez que for criar uma nova branch a partir de `main`, atualizar a `main` local com o remoto antes de ramificar:

```bash
git checkout main
git pull origin main
git checkout -b <nome-da-branch>
```

`<nome-da-branch>` e `<tipo>/<descricao>` inventado a partir do contexto da mudanca (uso isolado) ou o nome exato recebido do chamador (modo 2a/2b).

Nao pular essa etapa — ramificar a partir de uma `main` desatualizada gera conflitos desnecessarios e PRs divergentes. Se houver mudancas locais nao commitadas, usar `git stash` antes do `checkout main` e `git stash pop` apos criar a nova branch.

### Nomenclatura de branch

- `feat/<descricao>` — nova funcionalidade
- `fix/<descricao>` — correcao de bug
- `refactor/<descricao>` — refatoracao
- `chore/<descricao>` — manutencao/config
- `docs/<descricao>` — documentacao

## Exemplos

```
feat(api): adicionar endpoint de listagem de banners
```

```
fix(cupom): corrigir cupom aplicado duas vezes nas compras via site
```

```
refactor(auth): extrair validacao de token para service
```

```
docs(api): documentar endpoint de banners
```

```
chore: atualizar dependencias do composer
```
