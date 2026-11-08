# Se uma senha vazou

Apareceu na conversa, num arquivo ou num commit: trate como vazada. **Trocar primeiro, procurar depois.**

```
- [ ] 1. Trocar a credencial
- [ ] 2. Derrubar as sessões
- [ ] 3. Procurar as cópias (o usuário roda, fora do Claude)
- [ ] 4. Limpar
- [ ] 5. Conferir se alguém usou
- [ ] 6. Guardar a nova no cofre
```

1. **Trocar.** Senha nova no servidor ou no WordPress; chave SSH nova, com a antiga fora do `authorized_keys`; token de API revogado; Application Password apagada; chave da AWS desativada.
2. **Derrubar as sessões.** WordPress: perfil › "Sair de todas as outras sessões".
3. **Procurar as cópias.** O usuário roda no terminal dele, pra o valor não voltar à conversa:
   ```bash
   read -rs S                         # digita a senha antiga, sem eco
   grep -rlF -- "$S" ~/.claude .      # só nomes de arquivos
   git log --all -S "$S" --oneline    # commits que tocaram o valor
   unset S
   ```
4. **Limpar.** A sessão em `~/.claude/projects`, as linhas em `~/.claude/history.jsonl` e os arquivos. Commit já enviado: o `git filter-repo` reduz, não desfaz, porque quem clonou tem cópia.
5. **Conferir uso.** `last` e `/var/log/auth.log` no servidor, log de login do WordPress, CloudTrail na AWS.
6. **A credencial nova vai pro cofre**, não pro chat.
