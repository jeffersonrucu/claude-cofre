---
name: cofre
description: Busca credenciais no KeePassXC do usuário e entrega direto pra qualquer ferramenta (SSH, CLI, script, teste, cliente HTTP, banco) sem que a senha passe pela conversa, usando os scripts cofre-lista, cofre-ssh e cofre-run. Use quando uma tarefa precisar de login, senha, chave SSH, token ou credencial, mesmo que o usuário não diga qual entrada usar ("entra no stage", "roda isso no servidor do cliente", "conecta no banco de teste"). Use também ao cadastrar uma entrada, quando um cofre-* falhar ou quando uma senha aparecer na conversa.
---

# Cofre

O Claude **usa** a credencial e nunca **vê**. O KeePassXC guarda, um script `cofre-*` busca e entrega direto pro processo, e o Claude recebe só a saída.

## Regras

- Nunca peça senha, token, chave ou código TOTP no chat. Se o usuário colar um, siga [references/vazamento.md](references/vazamento.md).
- Nunca rode `secret-tool`, `keepassxc-cli`, `printenv`, `env` ou `aws configure export-credentials`, nem leia `*.kdbx`, `~/cofre`, `~/.ssh`, `.env*` ou `~/.aws/credentials`. As regras de deny em `~/.claude/settings.json` barram isso; não contorne com outro programa (python, busctl, cópia de arquivo). Bloqueou? Explique e peça pro usuário fazer no terminal dele.
- Nunca faça o `cofre-run` imprimir as variáveis (`echo $COFRE_PASS`, `env`, `console.log(process.env)`).
- Nunca digite senha em formulário pelas ferramentas de navegador: o valor entra no contexto. Login automatizado só por código que lê do ambiente via `cofre-run`.
- Não rode no servidor comandos que imprimem segredo (`cat wp-config.php`, `cat .env`). Precisa de um valor? `grep` só o nome da chave.
- Antes de abrir um arquivo do projeto pra achar seletor, endpoint ou config, procure segredo nele sem imprimir o valor (`grep -nE -l 'pass|secret|token|key'`). Se tiver, não abra: use só o que precisa via `grep` do trecho sem o valor. Achou um segredo em algum lugar? Não repita o valor; cite arquivo e linha e trate como vazado ([references/vazamento.md](references/vazamento.md)).

## Escolher a entrada

O usuário não precisa dizer o nome. Copie e siga:

```
- [ ] 1. Rodar cofre-lista
- [ ] 2. Cruzar título e URL com a tarefa
- [ ] 3. Decidir: uma, várias ou nenhuma
- [ ] 4. Dizer qual entrada vai usar antes de usar
```

1. `cofre-lista` imprime `título<TAB>URL` de cada entrada exposta. Lê só atributos públicos: não busca senha nem abre a confirmação do KeePassXC.
2. Cruze com o domínio ou host citado, a URL do projeto (config da ferramenta, `README`, remote do git), o nome do cliente e o ambiente (stage, prod).
3. Decida:
   - **Uma bate**: use.
   - **Várias batem** (`-stage` e `-prod`): pergunte, listando os títulos. Nunca escolha produção por dedução.
   - **Nenhuma bate**: diga o que procurou e peça o cadastro ([references/cadastro.md](references/cadastro.md)).
4. Avise antes: "Vou usar `stage-ecommerce` (stage.loja.com.br)".

Não leia `.env` nem config com segredo pra descobrir a URL.

## Comandos

| Script | Uso |
|---|---|
| `cofre-abrir` | Abre o KeePassXC se estiver fechado e espera o usuário destravar. Os outros três já chamam sozinhos |
| `cofre-lista` | Título e URL das entradas expostas |
| `cofre-ssh <entrada> "<comando>"` | Comando remoto por SSH, com senha ou chave |
| `cofre-run <entrada> <comando...>` | Roda o comando com `COFRE_USER`, `COFRE_PASS` e `COFRE_URL` só no ambiente dele |

No `cofre-run`, a variável tem que ser lida **dentro** do processo: código que leia o ambiente ou `sh -c '...'` com aspas simples. Com aspas duplas ela expande na shell do Claude, onde está vazia:

```bash
cofre-run clientex-db sh -c 'PGPASSWORD="$COFRE_PASS" psql -U "$COFRE_USER" -c "select 1"'
```

Antes de rodar, confira se a ferramenta grava ou imprime a credencial (trace, log verboso, mensagem de erro, arquivo de sessão): [references/ferramentas.md](references/ferramentas.md).

Com o KeePassXC fechado ou travado, os scripts abrem a janela e esperam até 120s (`COFRE_ESPERA`) o usuário digitar a senha mestra. Avise o usuário antes de rodar e use `timeout` de 180000 no Bash. Se a saída disser que a Integração com o Serviço Secreto está desligada, repasse o caminho do menu: o script não liga isso sozinho.

Se o usuário ligou a confirmação do KeePassXC, cada `cofre-ssh` ou `cofre-run` abre janelas pedindo aprovação. Avise antes de rodar.

## Quando ler as referências

- Como cada tipo de ferramenta lê a credencial e o que vaza: [references/ferramentas.md](references/ferramentas.md)
- Credencial da AWS: [references/aws.md](references/aws.md)
- Cadastrar entrada, nomes, senha mestra, TOTP: [references/cadastro.md](references/cadastro.md)
- Erro de `cofre-*`, KeePassXC, gnome-keyring ou como esta máquina está montada: [references/montagem.md](references/montagem.md)
- Senha apareceu na conversa, num arquivo ou num commit: [references/vazamento.md](references/vazamento.md)

## Limites

É boa prática, não blindagem. Com o cofre destravado, qualquer processo do usuário pede senha pelo D-Bus, e a regra de deny barra o comando que nomeia, não o programa. A trava que o agente não contorna é a confirmação do KeePassXC. Seja honesto com o usuário sobre isso.
