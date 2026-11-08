# Como esta máquina está montada

Pop!_OS 24.04, COSMIC, KeePassXC 2.7.6.

| Peça | Onde |
|---|---|
| Cofre | `~/cofre/*.kdbx`, banco `agent`. O grupo exposto é o que aparece no `cofre-lista` |
| Scripts | `~/.local/bin/cofre-abrir`, `cofre-lista`, `cofre-ssh` e `cofre-run`. Fonte em `scripts/` desta skill (edite lá e copie pro `~/.local/bin`). `cofre-lista` e `cofre-abrir` precisam de `jq` |
| Secret Service | KeePassXC. O gnome-keyring roda só com `pkcs11` |
| Override do keyring | `~/.config/systemd/user/gnome-keyring-daemon.service.d/override.conf` |
| Ativação D-Bus | `~/.local/share/dbus-1/services/org.freedesktop.secrets.service`, apontando pra `/usr/bin/keepassxc` |
| Autostart escondido | `~/.config/autostart/gnome-keyring-secrets.desktop` com `Hidden=true` |
| PAM | `auto_start` removido de `pam_gnome_keyring.so` em `/etc/pam.d/cosmic-greeter`. Original em `/var/backups/cosmic-greeter.antes-cofre`. Sem isso, o login religa os segredos do gnome-keyring |
| SSH agent | `gcr-ssh-agent` em `/run/user/1000/gcr/ssh` |
| Deny | `~/.claude/settings.json`, em `permissions.deny` |

## Detalhes do cofre-ssh que não devem ser "corrigidos"

- O `secret-tool search` manda os atributos pro stderr e o segredo pro stdout. Os scripts leem com `2>&1 >/dev/null`, então a senha nem passa pelo `sed`.
- Senha vazia na entrada: usa a chave SSH pelo agent. Com senha: `sshpass -e`, que lê de `SSHPASS`, nunca do argumento.
- `StrictHostKeyChecking=accept-new` grava a chave do servidor na primeira conexão e recusa se ela mudar.

## Problemas comuns

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| `cofre-lista: nada exposto` | Nenhum grupo exposto | *Banco de dados › Configurações do banco de dados › Integração com o Serviço Secreto*: escolher o grupo |
| `cofre travado` ou `não existe ou cofre travado` | KeePassXC travado | O usuário destrava. É o interruptor funcionando |
| `cofre-abrir: ... Integração com o Serviço Secreto está desligada` | `[FdoSecrets] Enabled=false` no `keepassxc.ini` | *Ferramentas › Configurações › Integração com o Serviço Secreto*. Não edite o `.ini` com o app aberto: ele sobrescreve ao sair |
| `cofre-abrir: o KeePassXC não registrou o Serviço Secreto` | Outro dono do nome | `busctl --user status org.freedesktop.secrets \| grep Comm` |
| Algo trava ~25s e dá `Connection timed out` | Chamada direta ao `org.freedesktop.secrets` sem dono: a ativação D-Bus roda `keepassxc`, que só foca a instância aberta | Sempre checar com `NameHasOwner` antes (o `cofre-abrir` faz isso) |
| Entrada não aparece | Fora do grupo exposto | Mover a entrada |
| `cofre-ssh` sai com código 5 ou "Permission denied" | Senha do cofre diferente da do servidor | O usuário gera senha nova e troca no servidor |
| KeePassXC avisa que outro serviço secreto está rodando | gnome-keyring com `secrets` | Conferir o PAM; `systemctl --user restart gnome-keyring-daemon` libera o nome na hora |
| Chrome perdeu senhas ou logins | A chave dele estava no gnome-keyring | A sincronização da Conta Google restaura; a chave nova fica no grupo exposto |
| `ssh-add -l` vazio | SSH Agent desligado ou entrada sem a opção | *Ferramentas › Configurações › Agente SSH* e aba *Agente SSH* da entrada |

## Reverter tudo

Apagar override, serviço D-Bus e autostart; `sudo cp /var/backups/cosmic-greeter.antes-cofre /etc/pam.d/cosmic-greeter`; `systemctl --user daemon-reload`; sair e entrar na sessão.
