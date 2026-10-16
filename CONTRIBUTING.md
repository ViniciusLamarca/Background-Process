# Contribuindo

Obrigado pelo interesse no supervisor local de trabalhos assíncronos para o Codex.

## Pull requests

- Toda contribuição deve chegar por pull request; não há contribuição direta à branch `main`.
- O pull request precisa da aprovação explícita do mantenedor, além de todos os checks obrigatórios passarem.
- Uma aprovação perde validade quando novos commits alteram o conteúdo revisado; o último push também precisa ser revisado.
- Mudanças em workflows, políticas de segurança, dependências e `CODEOWNERS` recebem a mesma revisão obrigatória.
- Os checks obrigatórios incluem varredura de segredos e revisão de dependências. CodeQL está habilitado para análise de código; ações usadas nos workflows precisam ser fixadas a um SHA completo.
- Workflows de pull request usam `pull_request`, permissões somente de leitura e não recebem segredos do repositório.
- Não inclua credenciais Kaggle/Codex, dados privados, outputs pessoais ou notebooks que executem ações externas sem autorização.
- Prefira alterações pequenas, descreva o risco e como validar o comportamento.

O repositório está público, mas ainda não possui uma licença. Não presuma permissão de reutilização ou redistribuição do conteúdo.
