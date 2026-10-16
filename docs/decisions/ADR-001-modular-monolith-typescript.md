# ADR-001: monólito local modular em TypeScript

- **Estado:** aprovado para o desenho; o spike do Codex ainda é gate antes da implementação.
- **Data:** 15/10/2026
- **Decisores:** usuário e agente Codex

## Contexto

Precisamos acompanhar trabalhos demorados sem manter o modelo em polling e, quando houver resultado, retomar a mesma conversa do Codex. O problema surgiu com jobs Kaggle: consultas repetidas consumiam tokens/limites e mantinham o agente ocupado sem mudança de estado. Kaggle é a primeira integração, mas o produto deve poder observar outros provedores, CLIs e processos locais. O processo deve operar enquanto o Codex estiver aberto. A retomada da conversa desktop é a principal incerteza técnica.

## Decisão

Usar TypeScript/Node.js como runtime principal em um único serviço local modular, iniciado/gerenciado pela integração do Codex. O serviço contém servidor MCP, casos de uso, supervisor de observação, persistência SQLite e adaptadores para integrações de trabalho e Codex. Kaggle será o primeiro adaptador; novas integrações usam capacidades explícitas e contratos de conformidade.

O desenho usa ports-and-adapters em escala pequena, casos de uso explícitos, estados separados para job e entrega e outbox transacional em SQLite. O contrato `CodexConversationPort` mantém SDK/app-server intercambiáveis até o spike decidir o adaptador viável.

Go permanece permitido para um módulo isolado caso benchmark reproduzível demonstre um gargalo de desempenho que compense a comunicação entre processos e a manutenção de duas toolchains. Instalar Go é preparação do ambiente, não aprovação para introduzir Go no produto. Python fica fora da stack proposta.

## Justificativa

- TypeScript já é a integração mais sólida disponível no ambiente e há SDKs oficiais para Codex e MCP.
- Node é adequado ao caminho I/O-bound de aguardar status remoto e entregar eventos; ainda não existe evidência para dividir código por desempenho.
- Um supervisor e SQLite mantêm instalação, ciclo de vida e recuperação compatíveis com uso local enquanto o Codex estiver aberto.
- Portas e adaptadores permitem adicionar provedores de trabalho sem espalhar tipos externos pela lógica do núcleo.
- O spike reduz o risco de construir um monitor completo que não consiga atualizar a conversa original.

## Alternativas consideradas

### Go para todo o serviço

Rejeitado como stack inicial: exigiria outra toolchain/ecossistema e não atende melhor, com evidência atual, ao risco dominante de integração Codex. Go fica disponível para módulos futuros se um perfil apontar CPU/bloqueio específico.

### Python

Descartado nesta proposta pelo usuário, apesar de existir integração no projeto adjacente.

### Microsserviços e broker distribuído

Rejeitados para MVP local: acrescentam deploy, rede, autenticação, operação e recuperação sem necessidade de escala multi-host. SQLite + outbox é suficiente para um consumidor local.

### Hook como mecanismo primário de conclusão

Não escolhido: hooks documentados cobrem pontos de ciclo de vida, sem contrato demonstrado para um monitor externo acordar uma thread desktop arbitrária. A entrega primária será testada pelo adaptador oficial Codex SDK/app-server.

## Consequências

### Positivas

- Polling e espera ficam fora do modelo e não consomem turnos de acompanhamento. O problema que motivou o projeto fica registrado para orientar decisões futuras.
- Um processo local simplifica segurança, instalação e diagnóstico.
- Estados/outbox persistidos permitem recuperar falhas locais sem perder a conclusão.
- Adaptadores e portas permitem adicionar integrações e trocar peças individualmente sem limitar o produto ao Kaggle.

### Custos e limites

- Node/TypeScript segue sendo uma segunda toolchain em relação ao projeto analítico Python.
- A confiabilidade da retomada depende de uma API suportada e acessível à conversa desktop original; sem isso o projeto não cumpre o requisito central.
- Entrega externa exactly-once pode não ser garantida; em crash-window ambígua o serviço precisa reconciliar ou pedir atenção, não repetir às cegas.
- Um monitor que só vive enquanto Codex está aberto deixa de consultar Kaggle quando o host fecha; estado local é recuperado na próxima inicialização.
- Go em módulos adicionais acrescentaria contrato IPC, build/release e manutenção cross-language e só deve ser introduzido com ganho medido.

## Critérios para rever

- O spike não consegue obter ou retomar a thread original em desktop.
- Benchmark sustentado identifica gargalo CPU-bound em módulo TypeScript e uma implementação Go reduz custo de forma material, sem comprometer observabilidade/segurança.
- Uso evolui para múltiplos usuários ou hosts, tornando inadequados processo único e SQLite local.

## Referências

- [Codex SDK](https://developers.openai.com/codex/codex-sdk)
- [Codex como plataforma](https://developers.openai.com/blog/codex-as-a-platform)
- [Codex app-server](https://developers.openai.com/siwc/token-sharing-open-source/codex-app-server)
- [TypeScript MCP SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Go MCP SDK](https://github.com/modelcontextprotocol/go-sdk)
