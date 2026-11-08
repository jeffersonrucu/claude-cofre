# Entregar a credencial pra uma ferramenta

O `cofre-run <entrada> <comando...>` põe `COFRE_USER`, `COFRE_PASS` e `COFRE_URL` no ambiente de um comando só. O trabalho do Claude é fazer a ferramenta ler dali sem o valor aparecer em lugar nenhum.

## Como a ferramenta lê

Do mais seguro pro menos seguro:

1. **Stdin**, quando a ferramenta aceita:
   `cofre-run registro sh -c 'printf %s "$COFRE_PASS" | docker login -u "$COFRE_USER" --password-stdin'`
2. **Variável que a própria ferramenta conhece**, renomeada dentro do processo:
   `cofre-run banco sh -c 'PGPASSWORD="$COFRE_PASS" psql -U "$COFRE_USER" -h db -c "select 1"'`
3. **Código que lê o ambiente** (Node, Python, Go, PHP), quando o Claude escreve o script ou o teste: `process.env.COFRE_PASS`, `os.environ["COFRE_PASS"]`, `getenv('COFRE_PASS')`.
4. **Argumento**, só se não houver outro jeito: `-p"$COFRE_PASS"`. Fica visível no `ps` enquanto roda.

Sempre `sh -c '...'` com **aspas simples**. Com aspas duplas a variável expande na shell do Claude, onde está vazia.

## O que costuma vazar

Antes de rodar, confira se a ferramenta grava ou imprime o que recebe:

| Onde vaza | Exemplo | O que fazer |
|---|---|---|
| Gravação da execução | Trace, vídeo e screenshot de teste de navegador; gravação de terminal | Desligar no passo que usa a senha |
| Log detalhado | `curl -v` mostra o header `Authorization`; `--debug`, `-vvv`, `DEBUG=*` | Não usar modo verboso com credencial |
| Mensagem de erro | Exceção que imprime a URL com usuário e senha, ou o ambiente inteiro | Montar a URL dentro do processo; não logar `env` |
| Arquivo gerado | Sessão salva, cookie, token, `.netrc`, config com senha | Pôr no `.gitignore`; não ler nem imprimir; apagar se for temporário |
| Saída do comando | `cat wp-config.php`, `env`, `set`, `docker inspect` | Não rodar; `grep` só o nome da chave |

## Sessão em vez de senha

Quando a ferramenta permite, a senha entra uma vez e o resto usa a sessão ou o token gerado: `storageState` em teste de navegador, cookie salvo, token de API. O arquivo da sessão vale como senha: mesmas regras da tabela acima.

## Navegador do usuário

Pelas ferramentas de navegador (Claude in Chrome), o Claude **nunca digita senha**: o valor entraria no contexto. O usuário faz o login com a extensão KeePassXC-Browser e o Claude usa a sessão aberta. Entradas usadas só assim não precisam ficar no grupo exposto.
