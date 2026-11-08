# Credencial da AWS

A chave nunca fica em `~/.aws/credentials`. O perfil usa `credential_process`: o AWS CLI roda um script que busca a chave no cofre e entrega direto pra ele.

```ini
# ~/.aws/config
[profile projeto-agente]
region = us-east-1
credential_process = /home/SEU_USUARIO/.local/bin/cofre-aws aws-projeto-agente
```

Entrada no cofre: **Usuário** = Access Key ID, **Senha** = Secret Access Key.

O Claude roda `aws <comando> --profile <perfil>` e vê só a resposta da AWS. Nunca rode o `cofre-aws` direto: ele imprime a chave (há deny pra isso).

## Antes de usar

- O `cofre-aws` não vem nesta skill. Se ele não existir na máquina, combine com o usuário a criação antes de usar AWS.
- Se `~/.aws/credentials` ainda tiver chaves em texto puro, lembre o usuário de migrar pro cofre e apagar o arquivo.
- Nunca use perfil admin pro agente. Use um usuário IAM próprio, com permissão mínima.
