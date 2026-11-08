# Cadastrar uma entrada

O usuário faz no KeePassXC, no grupo exposto, em *Nova entrada*. O Claude orienta.

| Campo | O que vai |
|---|---|
| Título | Minúsculas, hífen, sem espaço nem acento, no formato `cliente-funcao`: `clientex-stage`, `clientex-prod`, `clientex-db`. É o nome dos comandos |
| Usuário | O usuário de acesso |
| Senha | Gerada pelo app. Vazia se o servidor usa chave SSH |
| URL | Servidor: `host`, `host:porta` ou `ssh://host:porta`. Site: a URL da tela de login. **Sempre preencha**: é por ela que o Claude escolhe a entrada. **Nunca com usuário ou senha dentro** (`mysql://root:senha@db`) nem token na query: usuário vai no campo Usuário, senha no campo Senha |

Servidor com chave SSH: anexe a chave privada em *Avançado › Anexos* e, na aba *Agente SSH*, marque adicionar quando o cofre destravar e remover quando travar. A chave pública vai no `authorized_keys` do servidor.

Depois: `Ctrl+S` e conferir com `cofre-lista`.

Prefira usuário próprio pro agente (servidor, WordPress, AWS), sem admin onde não precisa.

## TOTP (código de 2FA)

Não vai no grupo exposto. Senha e código no mesmo lugar que o agente acessa anulam o segundo fator. Se o usuário quiser TOTP no KeePassXC, num cofre ou grupo pessoal: *Entradas › TOTP › Configurar TOTP* com a chave Base32 (só o trecho depois de `secret=`), ou um atributo `otp` com o link `otpauth://` inteiro. Nunca peça a chave no chat.

## Senha mestra

- **Não existe recuperação.** Esqueceu, perdeu o cofre.
- Frase-senha de 6 ou mais palavras sorteadas pelo gerador do KeePassXC (≈ 77 bits). Nunca frase inventada.
- Cópia em papel, duas, em lugares diferentes. Nunca em arquivo local, e-mail, chat ou dentro do cofre.
- O `.kdbx` pode ir pra nuvem. Teste abrindo só com a cópia guardada.
