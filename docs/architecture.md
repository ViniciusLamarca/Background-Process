# Arquitetura — monitor local de execuções Kaggle

**Estado:** desenho aprovado para orientar um spike; implementação ainda não aprovada.  
**Atualizado:** 15/10/2026

## Objetivo e limites

O Codex deve poder iniciar uma execução Kaggle, receber um identificador e continuar outro trabalho. Um processo local acompanha o job fora do modelo. Quando o job termina, o processo tenta retomar a mesma thread do Codex com um resumo curto e referências aos outputs.

O Codex precisa permanecer aberto durante o monitoramento. A persistência permite recuperar jobs caso o processo local reinicie; ela não implica que o monitor continue após o Codex fechar. Ao iniciar novamente, o supervisor reconcilia jobs que ainda não estejam em estado terminal.

Não estão no escopo: alterar Kaggle, depender de webhook Kaggle, criar conversa paralela, usar heartbeat de LLM para consultar status, cancelar remotamente kernels sem contrato confiável, ler dados privados não solicitados, ou enviar credenciais de negociação.

## Componentes e dependências

```text
MCP (transporte local)
  └─ casos de uso: SubmeterJob, ConsultarJob, ReconciliarJobs, EntregarConclusão
       ├─ domínio: JobKaggle, estados e invariantes
       ├─ porta KaggleExecutionPort
       │    └─ adaptador Kaggle CLI/API
       ├─ porta JobRepository / CompletionOutbox
       │    └─ adaptador SQLite
       └─ porta CodexConversationPort
            └─ adaptador Codex SDK ou app-server (decidir pelo spike)

Supervisor local
  ├─ coordena consultas com backoff limitado e jitter
  ├─ reconcilia estados remotos e locais
  └─ entrega conclusões pendentes sem concorrência na thread
```

O domínio e os casos de uso não importam MCP, SQLite, processos do sistema operacional, Kaggle SDK nem tipos do Codex. Adaptadores implementam portas pertencentes à aplicação. O transporte MCP valida e converte entradas e chama casos de uso; regras de estado ficam no domínio/aplicação.

É um monólito local com módulos explícitos, não um conjunto de microsserviços. O trabalho principal é I/O de rede e coordenação; não há evidência de que o caminho quente exija código Go. Uma futura peça Go só entra após perfil e benchmark reproduzível demonstrarem gargalo relevante, com contrato de processo/IPC estreito e substituível.

## Fluxo principal

1. O agente chama `submit_kaggle_job` com kernel permitido, diretório/revisão permitidos e contexto mínimo da continuação.
2. A aplicação valida o pedido, associa-o à thread atual e grava `submitting` antes da chamada externa.
3. O adaptador Kaggle envia o run usando operação e argumentos fixos, sem shell genérico. Após confirmação, o repositório grava o identificador remoto e estado `queued` ou `running`.
4. O MCP devolve `job_id` sem esperar pelo kernel. O modelo não precisa continuar ativo para o supervisor consultar Kaggle.
5. O supervisor consulta status com espera adaptativa, limite de frequência e jitter. Consultas falhas são limitadas; erros persistentes viram estado visível, não repetição infinita.
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
| Ports and Adapters (hexagonal), em escala pequena | Kaggle CLI/API, SQLite e Codex ficam atrás de portas da aplicação | Permite testar workflows sem acoplar domínio a fornecedores e trocar o método de retomada após o spike. |
| Application Service / Use Case | Submissão, reconciliação, consulta e entrega | Dá um lugar explícito a cada workflow transacional e mantém ferramentas MCP finas. |
| State machine explícita | Ciclo de vida do job e ciclo de vida da entrega | Torna falhas, reinícios e estados ambíguos observáveis, evitando booleans contraditórios. |
| Transactional outbox | Resultado terminal e evento de conclusão na mesma transação SQLite | Evita perder conclusão entre gravar o estado e acordar o controlador. A entrega externa continua sujeita a confirmação/reconciliação. |
| Idempotência e deduplicação | `job_id`, evento e entrega | O monitor pode reiniciar/reconciliar; duplicatas não devem causar novo kernel ou turno sem verificação. |
| Adapter / Strategy somente onde varia | Kaggle CLI e Codex SDK/app-server | Isola contratos externos. Não criar plugin framework geral nem abstração Go prematura. |

Não adotar inicialmente Redis, Kafka, serviço remoto, event bus distribuído, CQRS ou arquitetura de microsserviços: uma máquina local e um consumidor de outbox não justificam esses custos operacionais.

## Persistência, polling e operação

SQLite é suficiente para um processo por usuário, com transações locais e inspeção simples. O banco contém somente metadados necessários à recuperação e correlação; não guardar token Kaggle, output completo do notebook ou conteúdo desnecessário de conversa. Definir retenção configurável antes de implementar cleanup.

O polling é do supervisor, nunca uma chamada recorrente ao modelo. Usar limites de frequência, timeout, backoff exponencial com jitter e teto; classificar falhas transitórias, autenticação expirada, job terminal e status ambíguo. Logs estruturados devem conter `job_id`, tipo de evento, duração e erro classificado, sem tokens, headers de autorização ou dados pessoais.

Se o processo cair ou Codex fechar, o SQLite mantém estado e outbox. No próximo início, reconciliar o estado remoto antes de qualquer nova submissão ou entrega. Se o kernel/run não puder ser correlacionado, deixar para atenção manual.

## Modelo de ameaças e critérios de segurança

### Escopo, atores e ativos

- **Atores:** usuário local autorizado, agente Codex, processo monitor local, CLI Kaggle, serviço Kaggle e conteúdo produzido pelo notebook.
- **Ativos:** credencial Kaggle já configurada localmente; kernel/run; associação job↔thread; histórico/outputs mínimos; capacidade de inserir uma mensagem em uma conversa Codex.
- **Fronteiras:** Codex↔MCP local; MCP/supervisor↔CLI/Kaggle; supervisor↔SQLite; supervisor↔Codex SDK/app-server; saída Kaggle não confiável↔agente.
- **Fluxo:** thread chama MCP → aplicação persiste → adaptador chama Kaggle → supervisor consulta status → terminal + outbox em SQLite → adaptador tenta retomar a thread associada.

| Ameaça / prioridade | Controle e camada | Critério comportamental | Risco residual |
|---|---|---|---|
| Entrada adulterada executar comando arbitrário ou selecionar kernel/diretório fora do escopo (**Alta**) | Schema tipado, allowlist, path canonicalization e spawn com executável/argumentos fixos; sem shell genérico | Entradas inválidas e traversal são rejeitados; teste de contrato verifica que metacaracteres não alteram comando | Kaggle ainda executa código enviado pelo usuário no ambiente escolhido |
| Token Kaggle aparecer em banco, log, erro ou prompt (**Alta**) | Reutilizar auth suportada pela CLI; segredos fora do domínio, persistência e logs; filtrar saída de processo | Fakes de CLI com token sentinela confirmam ausência em banco/logs/erro/resumo | Malware sob a conta Windows pode acessar arquivos de configuração fora do serviço |
| Status de um run ser associado a outro kernel/job (**Alta**) | Persistir identificador remoto; serializar por kernel até correlação inequívoca; validar transições no domínio | Submissões concorrentes/replays não cruzam resultados; estados desconhecidos viram `needs_attention` | Limitações ou mudanças no contrato Kaggle podem impedir correlação precisa |
| Conclusão injetar mensagem em thread errada ou criar turno duplicado (**Alta**) | Associação persistente verificada; adaptador Codex restrito; outbox, dedupe e reconciliação antes de retry | Evento sem vínculo válido é recusado; crash na janela de envio não causa retry cego | Malware sob mesmo usuário local; API pode não expor chave de idempotência/estado suficiente |
| Conteúdo Kaggle instruir o agente a executar ações não solicitadas (**Média**) | Tratar logs e artefatos como dados externos; resumo limitado a status, run e paths; não seguir instruções em output | Fixture adversarial continua tratada como conteúdo; resumo não a promove a instrução | O agente ainda pode inspecionar artefatos explicitamente pedidos pelo usuário |
| Falha/reinício perder conclusão ou provocar envio duplicado (**Média**) | Transação terminal+outbox, recuperação no startup, estados separados e reconciliação externa | Reinício antes/depois de terminal entrega ou sinaliza atenção sem perder registro; replay não duplica deliberadamente | Exactly-once externo depende do contrato Codex e pode ser impossível; ambiguidade vira atenção manual |
| CLI travar, throttling ou autenticação expirar (**Média**) | Timeout, backoff limitado, classificação de erro e estado operacional visível | Erros transitórios param no teto definido; auth expirada pede ação; não existe loop infinito | O job remoto pode continuar mesmo sem monitor local |

### Segurança fora do escopo

O serviço não recebe chaves Binance, não aprova ordens, não altera privacidade Kaggle e não amplia permissões de rede do kernel. O comprometimento total da conta local Windows/Kaggle não pode ser mitigado por isolamento dentro deste processo local; limitar exposição e nunca persistir credenciais reduz o impacto do serviço.

## Observabilidade e privacidade

Emitir eventos estruturados como `job_submitted`, `poll_failed`, `job_terminal`, `completion_queued`, `completion_delivered` e `needs_attention`. Incluir correlação, tipo de erro e duração; excluir conteúdo completo de notebooks, segredos, cabeçalhos e prompt desnecessário. Resultado ao agente deve ser compacto: estado, kernel/run, duração e caminhos de outputs explicitamente permitidos.

## Estratégia de verificação futura

Os testes obrigatórios na implementação devem se concentrar no comportamento:

1. Submissão válida retorna `job_id` sem esperar o kernel; entrada inválida não inicia processo externo.
2. Falha de rede segue backoff limitado; autenticação expirada e status ambíguo aparecem explicitamente.
3. Execuções simultâneas no mesmo kernel não associam status ao job errado.
4. Reinício recupera jobs/outbox e não reenvia uma submissão desconhecida às cegas.
5. Conclusão e outbox são atômicas; repetição e crash-window não geram turno duplicado silenciosamente.
6. A integração real prova thread desktop original, turno concorrente, entrega enfileirada e uso da autenticação aprovada.
7. Segredos sentinela não vazam para SQLite, logs, erros nem mensagens ao modelo.

## Plano por gates

1. **Spike bloqueante:** obter thread ID do chamador MCP e retomar a conversa desktop original via SDK/app-server; provar fila quando ativa; verificar autenticação/limites. Nenhum serviço completo antes deste resultado.
2. **Contrato:** definir esquema de solicitação, status Kaggle realmente correlacionável, estado/retention SQLite, política de timeout e comportamento de falha/entrega.
3. **MVP modular:** implementar MCP, máquina de estados, adaptador Kaggle, supervisor, SQLite/outbox e adaptador Codex escolhido.
4. **Validação real autorizada:** job de teste, reinício do monitor, estados ambíguos/replay e confirmação de ausência de polling LLM.

## Fontes primárias

- [Codex SDK](https://developers.openai.com/codex/codex-sdk) e [Codex como plataforma](https://developers.openai.com/blog/codex-as-a-platform): interfaces oficiais para iniciar, retomar e acompanhar trabalho Codex; app-server oferece controle de lifecycle. Não garantem, por si, que toda thread do desktop possa ser anexada por processo externo.
- [Codex app-server](https://developers.openai.com/siwc/token-sharing-open-source/codex-app-server): exemplo oficial de `thread/resume` e turnos por ID; a integração específica com uma thread já aberta no desktop é uma hipótese a validar.
- [Documentação de plugins/hook](https://developers.openai.com/plugins/build/plugins): hooks são eventos de ciclo de vida, não um callback arbitrário de processo externo.
- [MCP Events](https://developers.openai.com/plugins/build/mcp-events): não é presumido como mecanismo de retorno nesta instalação local do Codex.
- [Referência de kernels do Kaggle CLI](https://github.com/Kaggle/kaggle-cli/blob/main/skills/references/kernels.md): operações de execução, status e logs; sem webhook de conclusão documentado na referência consultada.
- [TypeScript MCP SDK](https://github.com/modelcontextprotocol/typescript-sdk) e [Go MCP SDK](https://github.com/modelcontextprotocol/go-sdk): SDKs oficiais de referência para eventual implementação do transporte MCP.
