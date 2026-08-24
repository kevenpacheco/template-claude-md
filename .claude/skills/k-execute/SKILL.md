---
name: k-execute
description: Etapa final do fluxo de trabalho. Executa UMA tarefa pendente por invocacao, delegando para a skill tatica indicada no frontmatter, roda lint e testes do projeto, commita (via k-commit, que revisa o diff antes de cada commit) e marca a tarefa como concluida. Achado fora do escopo vira stub via k-spec modo desvio, sem sair da tarefa. Na ultima tarefa roda a suite completa, marca o spec.md como fluxo concluido e pergunta se quer subir agora; se sim, delega ao k-commit o push, o PR e o merge. Nunca roda git/gh diretamente. Use depois de /k-task.
---

# k-execute

Etapa 4 de 4: `k-spec` → `k-plan` → `k-task` → `k-execute`.

## Quando usar

Depois que `/k-task` gravou as tarefas. Invocacao:

```
/k-execute                        # infere pela branch atual
/k-execute corrigir-crop-banner   # por slug
```

## Regra central

**Uma tarefa por invocacao.** Pegar a proxima `pendente`, concluir, commitar, parar. Nao encadeie tarefas na mesma execucao — e isso que impede o contexto de estourar e o padrao do projeto de se perder no meio do caminho.

## Procedimento

### 1. Localizar a spec

Se o slug veio por argumento, use. Senao, resolva pela branch: `git branch --show-current` → `<prefixo>/<slug>` → `<raiz-de-specs>/*-<slug>`, tentando `docs/specs/` e `documentation/specs/`.

Se o glob achar mais de um diretorio, ou a branch nao tiver slug reconhecivel, **liste e pergunte**. Nao chute.

Se estiver numa branch protegida do projeto (ex.: `main`), **pare** — a branch e criada pelo `k-plan`.

Leia o frontmatter do `spec.md` antes de escolher tarefa:

- `fluxo: pendente` — e um stub, ainda sem contrato de comportamento. **Pare** e mande rodar `/k-spec <slug>`.
- `depende_de: <slug>` presente — este fluxo ficou parado esperando outro (ver "Achado fora do escopo"). Cheque o `fluxo:` da spec apontada:
  - `concluido` ou `descartado`: a dependencia caiu. Remova o campo `depende_de:`, ache a tarefa parada por ela (`grep -l "motivo: bloqueada por <slug>" tasks/*.md`), devolva-a para `status: pendente` com `motivo:` vazio, e siga.
  - `pendente` ou `ativo`: **pare** e mostre o que falta — o fluxo do qual este depende ainda nao entrou.

### 2. Escolher a tarefa

Liste `tasks/*.md` em ordem numerica e pegue a **primeira** com `status: pendente`.

- Se houver alguma `em-andamento`, ela e a escolhida — a execucao anterior foi interrompida. Verifique o estado do working tree antes de continuar.
- Se houver alguma `bloqueada` antes da primeira `pendente`, **pare** e mostre o `motivo` do frontmatter dela. Nao pule tarefa bloqueada.
- Se nao houver nenhuma pendente, siga para "Encerramento".

Marque `status: em-andamento` antes de comecar.

### 3. Executar

- Leia `spec.md`, `plan.md` e o arquivo da tarefa **antes de tocar em arquivo**. O contexto vem dos arquivos, nao da memoria da sessao.
- Se o frontmatter tiver `skill`, invoque essa skill tatica e siga o padrao dela.
- Altere **apenas** os arquivos listados em `arquivos`. Se faltar arquivo, atualize o frontmatter da tarefa e explique o porque; nao alargue o escopo em silencio.
- Respeite o `## Nao faca` da tarefa e o `## Nao entra neste trabalho` do plano.
- TDD: o ciclo acontece **dentro** da tarefa. Para cada item de `## Ciclos TDD`, nesta ordem: escreva o teste, rode-o, confirme que falha pelo motivo certo, so entao implemente ate passar. O vermelho existe durante a execucao e **nunca vira commit**.
- A tarefa so termina quando todos os ciclos estao verdes. Tarefa nunca termina vermelha.

### 4. Gate de conclusao

Obrigatorio antes do commit. Extraia o comando de lint e de teste da documentacao do projeto (`CLAUDE.md`, `README`); se nao estiver documentado, pergunte ao usuario uma vez. Rode no ambiente que o projeto define (container, local, etc.) — nunca fora dele quando o projeto exigir um ambiente especifico.

Rode **apenas as camadas de teste afetadas** pela tarefa — a tarefa e vertical e pode tocar mais de uma. Camadas que dependem de infraestrutura externa (ex.: integracao, E2E) exigem o ambiente correspondente de pe.

**Nunca commite com teste vermelho.** Se falhar: corrija e rode de novo, no maximo **2 tentativas**. Na terceira falha, **pare**: marque `status: bloqueada`, preencha `motivo:` com uma linha, registre no corpo da tarefa a linha decisiva da saida, o diagnostico e **quais ciclos ficaram verdes**, e devolva ao usuario. Nao commite.

Antes de dar a falha por perdida, verifique se ela vem de um problema **fora do escopo desta spec**. Se vier, siga "Achado fora do escopo" — o caminho bloqueante.

Consequencia esperada: o trabalho parcial (teste e implementacao incompletos) fica **no working tree, nao comitado**. E o preco de nunca ter testsuite vermelha no historico. Diga isso ao usuario ao devolver, para ele saber onde o codigo esta.

### 5. Commitar

Toda operacao de git e responsabilidade exclusiva da skill `k-commit` — o `k-execute` nunca roda `git`/`gh` diretamente. Chame `k-commit` no **modo 2b (commit)**: ela roda a revisao automatizada antes de commitar, cuida da mensagem em Conventional Commits e das regras de branch, e para depois do commit — sem push, sem PR, sem merge. Push, PR e merge ficam com o Encerramento abaixo, depois da ultima tarefa, via `k-commit` no **modo 2c (ciclo de shipping)**.

Adicione somente os arquivos da tarefa, mais o arquivo da propria tarefa e os `docker-compose` quando o bump se aplicar. Nunca `git add .`.

### 6. Fechar a tarefa

No frontmatter do arquivo da tarefa:

- `status: concluida`
- `commit: <SHA curto>`

Isso entra no mesmo commit da tarefa.

### 7. Reportar e parar

```
Tarefa <n> concluida: <resumo em uma linha>
Commit: <tipo(escopo): descricao>
Restam <k> tarefas. Proximo passo: /k-execute
```

## Achado fora do escopo

Bug ou pendencia descoberta no caminho que **nao** faz parte da spec atual. Primeiro classifique, pela evidencia do gate da etapa 4.

### Nao bloqueante

A tarefa em andamento passa no gate apesar do achado.

1. Chame `k-spec` no **modo desvio** com o achado. Ele grava o stub (`fluxo: pendente`), commita via `k-commit` e devolve o controle.
2. **Nao corrija.** Nao vira tarefa, nao vira commit carona.
3. Continue a tarefa em andamento, **no ponto exato onde parou**.

### Bloqueante

O gate falha **por causa** do achado, nao por erro da propria tarefa.

1. Chame `k-spec` no modo desvio do mesmo jeito — o stub e gravado antes de qualquer decisao.
2. Marque a tarefa atual `status: bloqueada`, com `motivo: bloqueada por <slug-do-stub>`.
3. **Pare e apresente as duas saidas ao usuario** (`AskUserQuestion`):

   - **Absorver no escopo atual** — o achado passa a fazer parte deste fluxo. Marque o stub como `fluxo: descartado`, com `## Por que foi descartado` apontando o fluxo que o absorveu, e so entao volte ao `/k-plan` para registrar a decisao e ao `/k-task` para gerar a tarefa.
   - **Inverter a prioridade** — este fluxo fica parado. Grave `depende_de: <slug-do-stub>` no frontmatter do `spec.md` deste fluxo e pare aqui. O trabalho no stub comeca em outra janela, por `/k-spec <slug>`.

**Nao decida sozinho qual das duas.** Alargar o escopo do trabalho e decisao do usuario.

Achado que **nao** fez o gate falhar e que mesmo assim o usuario considere bloqueante nao e detectavel aqui: ele vira stub pelo caminho nao bloqueante, e a decisao acontece na conversa.

## Encerramento

Toda operacao de git/gh a partir daqui e responsabilidade exclusiva da skill `k-commit` — o `k-execute` nunca roda `git`/`gh` diretamente. O `k-execute` decide o **que** (alvo, titulo, corpo do PR, a partir do contexto de negocio de `spec.md`/`plan.md`); o `k-commit` decide o **como**.

Quando nao restar tarefa `pendente` nem `em-andamento`:

1. **Suite completa**, com o comando de teste do projeto (o mesmo identificado no gate da etapa 4), no ambiente que o projeto define:

Falhou: pare e reporte. Nao chame o `k-commit` com suite vermelha.

2. **Marcar o fluxo como concluido**: grave `fluxo: concluido` no frontmatter do `spec.md`. Aqui `concluido` significa **todas as tarefas fechadas e suite completa verde** — nao "mergeado". Este e o ultimo ponto em que da pra escrever dentro da historia do proprio fluxo: depois do merge a branch ja foi deletada, e marcar exigiria commit direto na `main`.

Commite so esse arquivo, via `k-commit` modo 2b, mensagem `docs(spec): concluir <titulo>`.

3. **Perguntar se e hora de subir**: todas as tarefas ja estao comitadas localmente (uma por uma, via `k-commit` modo 2b). Pergunte ao usuario: "quer subir agora (push + revisao + PR + merge, via `k-commit`) ou parar aqui?"

- Se nao: pare. As tarefas ficam comitadas localmente, nada e pushado. Nao ha proxima etapa `k-*` pra sugerir — a decisao de subir fica pra depois (manual, ou chamando `/k-execute` de novo).
- Se sim: continue para os passos seguintes.

4. **Perguntar o alvo do PR**, sem sugerir um default fixo — depende do fluxo de branches do projeto. Espere a resposta antes de abrir.

5. **Montar o titulo e o corpo do PR** a partir do `spec.md` e do `plan.md` — esse conteudo de negocio e responsabilidade do `k-execute`, nao do `k-commit`.

Branch `feat/`, `refactor/`, `chore/`, `docs/`:

```
Titulo: <titulo da spec>
Base: <alvo>

Spec: `<raiz-de-specs>/<ts>-<slug>/`

## O que muda

<objetivo e escopo, vindos do spec.md>

## Como testar

<testes manuais, vindos do plan.md>
```

Branch `fix/`:

```
Titulo: <titulo da spec>
Base: <alvo>

Spec: `<raiz-de-specs>/<ts>-<slug>/`

## Causa raiz

<arquivo:linha e por que o comportamento errado acontece, vindo do plan.md>

## Correcao escolhida

<opcao adotada e por que ela resolve a causa, nao o sintoma>

## Opcoes descartadas

<cada alternativa do plano e o motivo de nao ter sido escolhida>

## Regressao coberta

<teste que falhava antes e passa depois, ou a justificativa do plano para nao haver teste automatizado>

## Como testar

<testes manuais, vindos do plan.md>
```

6. **Chamar o `k-commit`** no **modo 2c (ciclo de shipping)**, passando: branch atual, alvo escolhido (passo 4) e o titulo/corpo montados (passo 5, template acima). O `k-commit` executa, nessa ordem: push unico -> abertura do PR (imprimindo a URL) -> pergunta de merge, so se o alvo for uma branch protegida do projeto (ex.: `main`) -> limpeza da branch se aprovado. A revisao automatizada ja aconteceu commit a commit no passo 5 de cada tarefa, entao o ciclo de shipping nao revisa de novo. Sem `gh` disponivel, o `k-commit` monta o corpo com o conteudo recebido e instrui a abertura manual.

7. Se este fluxo gerou stubs de desvio, liste-os:

```bash
grep -rl "^origem_spec: <slug>" <raiz-de-specs>/
```

Diga que eles entram na fila do `/k-spec` **quando este PR mergear**. Nao ofereca elabora-los agora: a spec do desvio deve ser escrita contra o codigo ja atualizado. Isso e paralelo ao passo 6 e nao depende dele.

## Restricoes

- Nunca rode `git`/`gh` diretamente para branch, commit, push, PR, CI ou merge — sempre delegue ao `k-commit` (etapa "Commitar" e passo 6 do Encerramento).
- Nao corrija achado fora do escopo, nem transforme achado em tarefa deste fluxo por conta propria. Absorver escopo e decisao do usuario.
- Nao pule o gate de teste "porque a mudanca e pequena".
- Nao commite com teste vermelho, em hipotese nenhuma. Vermelho so existe dentro da execucao da tarefa, entre escrever o teste e implementar.
- Nao execute tarefa que nao existe como arquivo em `tasks/`. Se surgir trabalho novo, volte ao `/k-task`.
- Nao pule tarefa `bloqueada`.
- Nao proponha recursos de linguagem/versao alem do que o projeto ja usa.
