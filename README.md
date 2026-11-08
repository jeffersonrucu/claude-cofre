# cofre

Skill do Claude Code que usa senhas guardadas no KeePassXC sem que elas passem pela conversa.

O Claude **usa** a credencial e nunca **vê**. O KeePassXC guarda, um script `cofre-*` busca e entrega direto pro processo que precisa (ssh, psql, docker login, um teste com Playwright, um script qualquer), e o Claude recebe só a saída do comando.

Eu escrevi isso porque me peguei colando senha de stage no chat "só dessa vez" e percebi que ela ficava no histórico da sessão, em `~/.claude/projects`, pra sempre. A skill é o jeito que encontrei de parar com isso sem abrir mão de deixar o agente entrar no servidor.

## Como funciona

```
você: "roda o wp plugin list no stage da loja"
  │
  ├─ Claude roda cofre-lista            → vê só título e URL de cada entrada
  ├─ escolhe "loja-stage" e avisa antes de usar
  └─ roda cofre-ssh loja-stage "wp plugin list"
        │
        └─ o script pede a senha ao KeePassXC pelo D-Bus (Secret Service)
           e passa pro sshpass por variável de ambiente
                                         → Claude recebe só a lista de plugins
```

Você não precisa dizer o nome da entrada. O Claude cruza o pedido ("o stage da loja", "o banco de teste do cliente X") com o título e a URL das entradas. Se duas baterem, tipo `-stage` e `-prod`, ele pergunta. Produção por dedução, nunca.

São três scripts:

| Script | O que faz |
|---|---|
| `cofre-lista` | Lista `título<TAB>URL` das entradas expostas. Não busca senha e tira usuário, senha e query string de dentro da URL |
| `cofre-ssh <entrada> "<comando>"` | Roda um comando remoto por SSH, com senha (via `sshpass -e`) ou com chave (via agente SSH do KeePassXC) |
| `cofre-run <entrada> <comando...>` | Roda o comando com `COFRE_USER`, `COFRE_PASS` e `COFRE_URL` no ambiente dele, e só dele |

O `cofre-run` serve pra todo o resto. A variável precisa ser lida dentro do processo, então é `sh -c` com aspas simples:

```bash
cofre-run clientex-db sh -c 'PGPASSWORD="$COFRE_PASS" psql -U "$COFRE_USER" -h db -c "select 1"'
cofre-run registro sh -c 'printf %s "$COFRE_PASS" | docker login -u "$COFRE_USER" --password-stdin'
cofre-run loja-stage npx playwright test login.spec.ts   # o teste lê process.env.COFRE_PASS
```

Com aspas duplas a variável expande na shell do Claude, onde está vazia, e o comando falha. Melhor falhar do que vazar.

## O que vem no repositório

```
cofre/
├── SKILL.md                 regras e fluxo que o Claude segue
├── scripts/                 cofre-lista, cofre-ssh, cofre-run
└── references/
    ├── ferramentas.md       como cada tipo de ferramenta lê a credencial e onde ela costuma vazar
    ├── cadastro.md          como nomear e preencher uma entrada; TOTP e senha mestra
    ├── vazamento.md         checklist pra quando uma senha aparece na conversa ou num commit
    ├── aws.md               credential_process pro AWS CLI
    └── montagem.md          como a minha máquina está montada e problemas comuns
exemplos/
└── settings-deny.json       regras de deny pro ~/.claude/settings.json
```

## Requisitos

- Linux com D-Bus. Testei no Pop!_OS 24.04 com COSMIC e KeePassXC 2.7.6
- KeePassXC com a integração do Serviço Secreto ligada
- `secret-tool` (pacote `libsecret-tools`), `jq`, `busctl` (vem com o systemd)
- `sshpass`, se algum servidor usar senha em vez de chave

No macOS e no Windows não existe Secret Service, então os scripts não funcionam do jeito que estão. A ideia vale, a implementação teria que mudar.

## Instalação

1. Copie a skill e os scripts:

   ```bash
   git clone https://github.com/joliveira-sv/cofre-skill.git
   cp -r cofre-skill/cofre ~/.claude/skills/
   install -m 755 cofre-skill/cofre/scripts/cofre-* ~/.local/bin/
   sudo apt install libsecret-tools jq sshpass
   ```

2. No KeePassXC, crie um banco só pro agente (ou um grupo separado no seu) e ligue o Serviço Secreto: *Ferramentas › Configurações › Integração com o Serviço Secreto*. Depois, em *Banco de dados › Configurações do banco de dados › Integração com o Serviço Secreto*, escolha o grupo que vai ficar exposto. Só o que estiver nesse grupo aparece pro `cofre-lista`.

3. Se o gnome-keyring estiver rodando, ele disputa o nome `org.freedesktop.secrets` com o KeePassXC e normalmente ganha. O [montagem.md](cofre/references/montagem.md) mostra como deixei ele só com `pkcs11` e como desfazer. Essa é a parte chata da instalação. Vá com calma e guarde o backup do PAM antes de mexer.

4. Junte as regras de [exemplos/settings-deny.json](exemplos/settings-deny.json) ao `permissions.deny` do seu `~/.claude/settings.json`. Elas impedem o Claude de rodar `secret-tool` e `keepassxc-cli` direto, ou de ler `.kdbx`, `~/.ssh`, `.env*` e `~/.aws/credentials`.

5. Cadastre uma entrada de teste e confira:

   ```bash
   cofre-lista
   cofre-ssh meu-servidor "hostname"
   ```

## Cadastrando entradas

O título vira o nome que os comandos usam, então vale um padrão: minúsculas, hífen, `cliente-funcao`. Por exemplo `clientex-stage`, `clientex-prod`, `clientex-db`.

Preencha a URL sempre, porque é por ela que o Claude escolhe a entrada. Pra servidor, `host`, `host:porta` ou `ssh://host:porta`. Pra site, a URL da tela de login. Nada de usuário ou senha dentro da URL (`mysql://root:senha@db`); cada coisa vai no seu campo.

Servidor com chave SSH fica com a senha em branco e a chave anexada na entrada, com o agente SSH do KeePassXC carregando ela quando o cofre destrava.

TOTP não vai no grupo exposto. Senha e segundo fator no mesmo lugar que o agente acessa é um fator só. Detalhes em [cadastro.md](cofre/references/cadastro.md).

## Limites

Isso é boa prática, não blindagem, e prefiro deixar claro:

- Com o cofre destravado, qualquer processo do seu usuário consegue pedir senha pelo D-Bus. A regra de deny barra o comando que ela nomeia (`secret-tool`), não um `python` que faça a mesma chamada. A skill manda o Claude não contornar, mas isso é instrução, não trava.
- A trava de verdade é a confirmação de acesso do KeePassXC. Ligada, cada `cofre-ssh` ou `cofre-run` abre uma janela pedindo sua aprovação. É mais lento, e é o único ponto que o agente não consegue pular sozinho.
- Travar o KeePassXC corta tudo na hora. Os scripts respondem "cofre travado" e param.
- Se a ferramenta chamada imprimir a senha (`curl -v`, trace do Playwright, mensagem de erro com a URL completa), ela volta pra conversa. O [ferramentas.md](cofre/references/ferramentas.md) lista os casos que conheço, mas a lista não é completa.

Se uma senha vazar mesmo assim, o [vazamento.md](cofre/references/vazamento.md) tem o passo a passo: trocar primeiro, procurar as cópias depois.

## Como desfazer

Apague `~/.claude/skills/cofre` e `~/.local/bin/cofre-*`, tire as regras de deny e, se mexeu no gnome-keyring, siga a seção "Reverter tudo" do [montagem.md](cofre/references/montagem.md).
