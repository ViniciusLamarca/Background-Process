# Arquitetura — supervisor local de trabalhos assíncronos

**Estado:** desenho aprovado para orientar um spike; implementação ainda não aprovada.  
**Atualizado:** 15/10/2026

## Objetivo e limites

O Codex deve poder iniciar ou registrar um trabalho demorado, receber um identificador e continuar outro trabalho. Um supervisor local acompanha o trabalho fora do modelo. Quando há uma mudança relevante ou o trabalho termina, o processo tenta retomar a mesma thread do Codex com um resumo curto e referências aos outputs.

O problema que motivou o projeto veio do Kaggle: o agente iniciava um notebook e então fazia polling repetido para descobrir quando terminava. Isso mantinha o modelo ocupado, consumia tokens e limites e bloqueava a atenção do agente enquanto não havia novidade. O supervisor transfere essa espera para um processo local que não chama o modelo periodicamente.

Kaggle é o primeiro provedor em foco. O produto deve permanecer aberto a outras integrações, como serviços de execução remota, CLIs e processos locais longos. O núcleo padroniza identidade, associação com a thread, estados, persistência, recuperação, entrega, limites operacionais e observabilidade. Adaptadores traduzem os contratos externos sem presumir que todos os provedores têm as mesmas capacidades.

O Codex precisa permanecer aberto durante o monitoramento. A persistência permite recuperar jobs caso o processo local reinicie; ela não implica que o monitor continue após o Codex fechar. Ao iniciar novamente, o supervisor reconcilia jobs que ainda não estejam em estado terminal.

Não estão no escopo: alterar Kaggle ou outros provedores, depender de webhooks não documentados, criar conversa paralela, usar heartbeat de LLM para consultar status, executar comandos arbitrários fornecidos pelo modelo, cancelar trabalho remoto sem contrato confiável, ler dados privados não solicitados, ou enviar credenciais de negociação.

## Componentes e dependências

```text
MCP (transporte local)
  └─ casos de uso: RegistrarTrabalho, ConsultarTrabalho, Reconciliar, EntregarEvento
       ├─ domínio: TrabalhoMonitorado, estados e invariantes
       ├─ porta BackgroundWorkPort
       │    ├─ adaptador Kaggle CLI/API (primeiro)
       │    └─ adaptadores futuros (serviços, CLI, processos locais)
       ├─ porta WorkRepository / CompletionOutbox
       │    └─ adaptador SQLite
       └─ porta CodexConversationPort
            └─ adaptador Codex SDK ou app-server (decidir pelo spike)

Supervisor local
  ├─ coordena consultas com backoff limitado e jitter
  ├─ reconcilia estados remotos e locais
  └─ entrega conclusões pendentes sem concorrência na thread
```

O domínio e os casos de uso não importam MCP, SQLite, processos do sistema operacional, SDKs de provedores nem tipos do Codex. Adaptadores implementam portas pertencentes à aplicação. O transporte MCP valida e converte entradas e chama casos de uso; regras de estado ficam no domínio/aplicação.

### Contrato para integrações

Cada adaptador representa um tipo de trabalho e expõe somente capacidades que consegue cumprir com segurança. Identidade/correlação e observação de estado são requisitos para qualquer trabalho monitorado; submissão, cancelamento, coleta de outputs e consulta de logs são capacidades opcionais explícitas. Um processo local preexistente pode ser registrado e observado sem o supervisor iniciá-lo. Não assumir semântica uniforme de cancelamento, idempotência ou retenção entre provedores.

Antes de adicionar um adaptador, documentar método de autenticação, limites de frequência, timeout, IDs confiáveis, estados terminais, comportamento após falha/desconexão, capacidades suportadas e quais dados retornam ao Codex. Adaptadores passam por uma suíte de contrato comum para invariantes do núcleo e testes específicos para semântica do provedor. Isso mantém integrações substituíveis sem introduzir um sistema de plugins dinâmico no MVP.

É um monólito local com módulos explícitos, não um conjunto de microsserviços. O trabalho principal é I/O de rede e coordenação; não há evidência de que o caminho quente exija código Go. Uma futura peça Go só entra após perfil e benchmark reproduzível demonstrarem gargalo relevante, com contrato de processo/IPC estreito e substituível.

## Fluxo principal

1. O agente chama uma ferramenta adequada ao adaptador disponível, por exemplo `submit_kaggle_job` inicialmente, com recurso/escopo permitido e contexto mínimo da continuação.
2. A aplicação valida o pedido, associa-o à thread atual e grava `submitting` antes da chamada externa.
3. O adaptador correspondente inicia ou registra o trabalho usando operação e argumentos fixos, sem shell genérico. Após confirmação, o repositório grava o identificador externo e o estado conhecido.
4. O MCP devolve `job_id` sem esperar pelo kernel. O modelo não precisa continuar ativo para o supervisor consultar Kaggle.
5. O supervisor observa status pelo mecanismo disponível no adaptador; quando precisar consultar, usa espera adaptativa, limite de frequência e jitter. Consultas falhas são limitadas; erros persistentes viram estado visível, não repetição infinita.
6. Ao detectar conclusão, uma única transação SQLite grava o estado terminal e um registro de outbox pendente.
7. O adaptador Codex verifica a associação do job à thread original e tenta enfileirar uma continuação segura. Se a thread estiver ocupada, a conclusão permanece pendente até haver um ponto suportado para continuar.
8. Quando a entrega for confirmada, a outbox é marcada como entregue. Uma repetição local do evento não deve iniciar deliberadamente uma segunda continuação.

### Retomada do Codex — gate técnico

As interfaces oficiais Codex SDK e app-server documentam iniciar/retomar threads por ID e controlar turnos. Isso não comprova que um MCP local consiga obter o ID da thread desktop que o chamou nem que retomar essa thread por outro processo atualize a mesma conversa visível. O spike precisa testar o fluxo no Codex desktop real, incluindo um turno ativo e o comportamento visual.

O adaptador deve esconder a escolha entre SDK e app-server. O spike compara as opções oficiais e seleciona a que prova: identidade da thread, continuação visível no mesmo chat, serialização segura de turnos e autenticação compatível com a conta Codex usada, sem introduzir chave de API/cobrança não aprovada. Não substituir o requisito por thread paralela.

Se o Codex não fornecer uma forma suportada de entregar à mesma thread, o spike deve parar e documentar a limitação. Notificação do sistema operacional pode complementar o fluxo, mas não conta como retomada.

## Estados e invariantes

Separar estado do job remoto de estado da entrega evita misturar falha Kaggle com falha Codex.

| Máquina | Estados sugeridos | Regra |
|---|---|---|
| Job | `created`, `submitting`, `queued`, `running`, `succeeded`, `failed`, `submission_unknown`, `needs_attention` | `submission_unknown` evita reenviar às cegas quando a resposta da submissão se perde. Estado terminal só vem de evidência remota correlacionada. |
| Entrega | `pending`, `dispatching`, `delivered`, `needs_attention` | Entrega só se refere à conversa autorizada e não é apagada antes da confirmação. |

Invariantes:

- `job_id` é estável e único; eventos usam identificador de deduplicação.
- Cada job mantém kernel, run remoto, thread de origem e timestamps correlacionados.
- Execuções concorrentes para o mesmo kernel são serializadas enquanto o status remoto não identificar cada run inequivocamente.
- Estado remoto desconhecido/ambíguo não é convertido em sucesso nem em falha inferida.
- O serviço não repete uma submissão externa se não puder demonstrar que a repetição é segura.
- Entrega por SDK/app-server pode não oferecer garantia de exactly-once. Na janela em que Codex aceitou a continuação mas a confirmação local se perdeu, reconciliar antes de repetir; se não houver reconciliação suportada, usar `needs_attention` em vez de duplicar turno.

## Patterns e razão de uso

| Pattern | Aplicação | Por que aqui |
|---|---|---|
| Ports and Adapters (hexagonal), em escala pequena | Kaggle e futuras integrações, SQLite e Codex ficam atrás de portas da aplicação | Permite testar workflows sem acoplar domínio a fornecedores e adicionar adaptadores conforme a necessidade real. |
| Application Service / Use Case | Submissão, reconciliação, consulta e entrega | Dá um lugar explícito a cada workflow transacional e mantém ferramentas MCP finas. |
| State machine explícita | Ciclo de vida do job e ciclo de vida da entrega | Torna falhas, reinícios e estados ambíguos observáveis, evitando booleans contraditórios. |
| Transactional outbox | Resultado terminal e evento de conclusão na mesma transação SQLite | Evita perder conclusão entre gravar o estado e acordar o controlador. A entrega externa continua sujeita a confirmação/reconciliação. |
| Idempotência e deduplicação | `job_id`, evento e entrega | O monitor pode reiniciar/reconciliar; duplicatas não devem causar novo kernel ou turno sem verificação. |
| Adapter / Strategy somente onde varia | Provedores de trabalho e Codex SDK/app-server | Isola contratos externos. Não criar plugin framework geral nem abstração Go prematura. |

Não adotar inicialmente Redis, Kafka, serviço remoto, event bus distribuído, CQRS ou arquitetura de microsserviços: uma máquina local e um consumidor de outbox não justificam esses custos operacionais.

## Persistência, polling e operação

SQLite é suficiente para um processo por usuário, com transações locais e inspeção simples. O banco contém somente metadados necessários à recuperação e correlação; não guardar credenciais de provedores, outputs completos ou conteúdo desnecessário de conversa. Definir retenção configurável antes de implementar cleanup.

O polling é do supervisor, nunca uma chamada recorrente ao modelo. Usar limites por integração, timeout, backoff exponencial com jitter e teto; classificar falhas transitórias, autenticação expirada, trabalho terminal e estado ambíguo. Logs estruturados devem conter `work_id`, tipo de integração/evento, duração e erro classificado, sem tokens, headers de autorização ou dados pessoais.

### Robustez operacional

- Executar como um único supervisor local por usuário do Windows, enquanto o Codex estiver aberto; vários clientes MCP devem anexar-se ao mesmo supervisor e nunca iniciar loops de polling concorrentes. A forma de ciclo de vida e IPC será validada no spike do Codex.
- Usar trava de instância e handshake de saúde para evitar duas instâncias ativas; o canal local deve aceitar somente o usuário Windows atual. O protocolo concreto, como named pipe, será escolhido durante implementação.
- Persistir cada transição antes de confirmar comandos ao agente. Na inicialização, reconciliar trabalhos ativos e eventos pendentes; manter outbox transacional e reivindicação de trabalho atômica.
- Aplicar timeout, backoff limitado com jitter, teto de frequência, limite de concorrência e pausa/circuit breaker por adaptador, evitando tempestades quando um provedor falha ou limita consultas.
- Fazer shutdown gracioso e liberar recursos; se o processo morrer abruptamente, leases/heartbeats vencidos permitem recuperar a observação sem duas instâncias assumirem o mesmo trabalho.
- Expor estados acionáveis: supervisor indisponível, autenticação expirada, trabalho parado além do prazo, fila acumulada, lease vencido e entrega Codex pendente/falha.
- Persistir migrações SQLite versionadas e proteger o banco com permissões do usuário atual. Lock local não oferece garantia distribuída se o projeto evoluir para múltiplos hosts.

Os valores de concorrência, timeout e frequência serão configuráveis por integração com defaults conservadores e verificados com testes determinísticos. O supervisor não deve depender de polling do próprio Codex para se recuperar.

Se o processo cair ou Codex fechar, o SQLite mantém estado e outbox. No próximo início, reconciliar o estado remoto antes de qualquer nova submissão ou entrega. Se o kernel/run não puder ser correlacionado, deixar para atenção manual.

## Modelo de ameaças e critérios de segurança

### Escopo, atores e ativos

- **Atores:** usuário local autorizado, agente Codex, supervisor local, CLIs/serviços externos e conteúdo produzido por trabalhos monitorados.
- **Ativos:** credenciais que cada integração já usa localmente; identidade/correlação do trabalho; associação trabalho↔thread; metadados e outputs mínimos; capacidade de inserir uma mensagem em uma conversa Codex.
- **Fronteiras:** Codex↔MCP/IPC local; supervisor↔adaptador/CLI/serviço; supervisor↔SQLite; supervisor↔Codex SDK/app-server; saídas externas não confiáveis↔agente.
- **Fluxo:** thread chama MCP → aplicação persiste → adaptador inicia/registra trabalho → supervisor observa → terminal + outbox em SQLite → adaptador tenta retomar a thread associada.

| Ameaça / prioridade | Controle e camada | Critério comportamental | Risco residual |
|---|---|---|---|
| Entrada adulterada executar comando arbitrário ou acessar recurso fora do escopo (**Alta**) | Schema tipado, allowlist, canonicalização de paths e spawn com executável/argumentos fixos; sem shell genérico | Entradas inválidas, traversal e metacaracteres são rejeitados sem alterar o comando executado | Integrações ainda operam recursos da conta/host já autorizados |
| Credencial de qualquer integração aparecer em banco, log, erro ou prompt (**Alta**) | Reutilizar auth suportada pelo provedor/CLI; segredos fora do domínio, persistência e logs; filtrar saída de processo | Credenciais sentinela não aparecem no banco, logs, erros nem resumo | Malware sob a conta Windows pode acessar configurações externas ao serviço |
| Status de um trabalho ser associado ao trabalho errado (**Alta**) | Persistir ID externo e provedor; validar transições; serializar por chave de recurso quando a correlação não for inequívoca | Submissões concorrentes/replays não cruzam resultados; estado desconhecido vira `needs_attention` | Limitações/mudanças no contrato de um provedor podem impedir correlação precisa |
| Conclusão injetar mensagem em thread errada ou criar turno duplicado (**Alta**) | Associação persistente verificada; adaptador Codex restrito; outbox, dedupe e reconciliação antes de retry | Evento sem vínculo válido é recusado; crash na janela de envio não causa retry cego | Malware sob mesmo usuário local; API pode não expor chave de idempotência/estado suficiente |
| Conteúdo Kaggle instruir o agente a executar ações não solicitadas (**Média**) | Tratar logs e artefatos como dados externos; resumo limitado a status, run e paths; não seguir instruções em output | Fixture adversarial continua tratada como conteúdo; resumo não a promove a instrução | O agente ainda pode inspecionar artefatos explicitamente pedidos pelo usuário |
| Falha/reinício perder conclusão ou provocar envio duplicado (**Média**) | Transação terminal+outbox, recuperação no startup, estados separados e reconciliação externa | Reinício antes/depois de terminal entrega ou sinaliza atenção sem perder registro; replay não duplica deliberadamente | Exactly-once externo depende do contrato Codex e pode ser impossível; ambiguidade vira atenção manual |
| Adaptador/CLI travar, throttling ou autenticação expirar (**Média**) | Timeout, backoff limitado, classificação de erro, limite por adaptador e estado operacional visível | Erros transitórios param no teto definido; auth expirada pede ação; não existe loop infinito | O trabalho externo pode continuar mesmo sem monitor local |
| Dois supervisores observarem ou entregarem o mesmo trabalho simultaneamente (**Média**) | Trava de instância, leases persistidos e reivindicação transacional; IPC autenticado ao usuário Windows atual | Duas inicializações simultâneas não criam observadores concorrentes nem duplicam entrega | Exclusão mútua local não cobre múltiplos hosts ou cópias de banco em rede |

### Segurança fora do escopo

O serviço não recebe chaves Binance, não aprova ordens, não altera privacidade Kaggle e não amplia permissões de rede do kernel. O comprometimento total da conta local Windows/Kaggle não pode ser mitigado por isolamento dentro deste processo local; limitar exposição e nunca persistir credenciais reduz o impacto do serviço.

## Observabilidade e privacidade

Emitir eventos estruturados como `job_submitted`, `poll_failed`, `job_terminal`, `completion_queued`, `completion_delivered` e `needs_attention`. Incluir correlação, tipo de erro e duração; excluir conteúdo completo de notebooks, segredos, cabeçalhos e prompt desnecessário. Resultado ao agente deve ser compacto: estado, kernel/run, duração e caminhos de outputs explicitamente permitidos.

## Estratégia de verificação futura

Os testes obrigatórios na implementação devem se concentrar no comportamento:

1. Submissão/registro válido retorna `work_id` sem esperar a conclusão; entrada inválida não inicia processo externo.
2. Adaptadores conformes seguem backoff limitado; autenticação expirada e status ambíguo aparecem explicitamente.
3. Trabalhos simultâneos no mesmo recurso não associam status ao trabalho errado.
4. Reinício recupera jobs/outbox e não reenvia uma submissão desconhecida às cegas.
5. Conclusão e outbox são atômicas; repetição e crash-window não geram turno duplicado silenciosamente.
6. A integração real prova thread desktop original, turno concorrente, entrega enfileirada e uso da autenticação aprovada.
7. Segredos sentinela não vazam para SQLite, logs, erros nem mensagens ao modelo.

## Plano por gates

1. **Spike bloqueante:** obter thread ID do chamador MCP e retomar a conversa desktop original via SDK/app-server; provar fila quando ativa; verificar autenticação/limites. Nenhum serviço completo antes deste resultado.
2. **Contrato do núcleo:** definir identidade de trabalho, capacidades de observação, estado/retenção SQLite, timeouts e comportamento de falha/entrega sem homogeneizar capacidades opcionais.
3. **Primeiro adaptador:** validar correlação, limites, autenticação e recuperação para Kaggle, mantendo o contrato comum coberto por testes de conformidade.
4. **MVP robusto:** implementar MCP, máquina de estados, supervisor, SQLite/outbox, ciclo de vida único/IPC local e adaptador Codex escolhido.
5. **Integrações seguintes:** adicionar adaptadores sob demanda com avaliação de ameaça, limites operacionais e testes específicos; validar trabalho local e provedores remotos.
6. **Validação real autorizada:** job de teste, concorrência, reinício, estados ambíguos/replay e confirmação de ausência de polling LLM.

## Fontes primárias

- [Codex SDK](https://developers.openai.com/codex/codex-sdk) e [Codex como plataforma](https://developers.openai.com/blog/codex-as-a-platform): interfaces oficiais para iniciar, retomar e acompanhar trabalho Codex; app-server oferece controle de lifecycle. Não garantem, por si, que toda thread do desktop possa ser anexada por processo externo.
- [Codex app-server](https://developers.openai.com/siwc/token-sharing-open-source/codex-app-server): exemplo oficial de `thread/resume` e turnos por ID; a integração específica com uma thread já aberta no desktop é uma hipótese a validar.
- [Documentação de plugins/hook](https://developers.openai.com/plugins/build/plugins): hooks são eventos de ciclo de vida, não um callback arbitrário de processo externo.
- [MCP Events](https://developers.openai.com/plugins/build/mcp-events): não é presumido como mecanismo de retorno nesta instalação local do Codex.
- [Referência de kernels do Kaggle CLI](https://github.com/Kaggle/kaggle-cli/blob/main/skills/references/kernels.md): operações de execução, status e logs; sem webhook de conclusão documentado na referência consultada.
- [TypeScript MCP SDK](https://github.com/modelcontextprotocol/typescript-sdk) e [Go MCP SDK](https://github.com/modelcontextprotocol/go-sdk): SDKs oficiais de referência para eventual implementação do transporte MCP.
