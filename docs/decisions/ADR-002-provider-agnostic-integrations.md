# ADR-002: núcleo extensível a integrações de trabalhos assíncronos

- **Estado:** aprovado como direção arquitetural.
- **Data:** 15/10/2026
- **Decisores:** usuário e agente Codex

## Contexto

O primeiro caso de uso é monitorar uma execução Kaggle. O problema é mais amplo: sempre que o agente inicia trabalho demorado e precisa consultar repetidamente seu estado, polling dentro do chat consome tokens/limites e ocupa o agente sem produzir progresso. A ferramenta deve suportar outras integrações quando necessário, incluindo serviços remotos, CLIs e processos locais.

Há risco de resolver isso com um modelo Kaggle-específico ou, no extremo oposto, criar cedo uma abstração universal que apague diferenças importantes entre integrações.

## Decisão

Construir um núcleo de supervisão independente de provedor e integrar cada sistema por adaptador TypeScript. O núcleo é responsável por identidade e correlação, ciclo de vida durável, recuperação, limites operacionais, observabilidade, outbox e associação segura da entrega à conversa Codex de origem.

O adaptador declara suas capacidades. Identidade/correlação e observação de estado são requisitos para qualquer trabalho monitorado; iniciar, cancelar, coletar artefatos e consultar logs são capacidades opcionais explícitas. Um processo local preexistente pode ser registrado e observado sem o supervisor iniciá-lo. A primeira integração será Kaggle.

Cada adaptação documentará autenticação, semântica de estados, idempotência, limites, timeouts, comportamento em desconexão, outputs e modelo de ameaça. Um contrato comum verifica invariantes do núcleo, enquanto testes específicos verificam o provedor real. O MVP não terá carregamento dinâmico de plugins nem DSL para descrever workflows.

## Justificativa

- Preserva o caso de uso original e evita confundir “monitor de Kaggle” com o objetivo do produto.
- Permite suportar serviços remotos e processos locais por etapas, sem multiplicar serviços ou toolchains.
- Capacidades explícitas impedem o núcleo de prometer cancelamento ou retomada que um provedor não oferece.
- Conformidade comum reduz divergências em persistência, concorrência, recuperação e entrega.
- Uma fronteira enxuta mantém custos de suporte baixos e permite aprender com o primeiro adaptador antes de generalizar mais.

## Consequências

### Positivas

- A integração inicial não limita futuras integrações.
- O núcleo pode oferecer comportamento consistente mesmo quando cada provedor tem protocolos diferentes.
- Saídas e credenciais continuam nos limites de cada adaptador e não precisam vazar ao domínio.

### Custos e limites

- Cada integração exige entender e documentar seu contrato, ameaças e modos de falha.
- Um conjunto de capacidades comum aumenta um pouco o desenho inicial e precisa ser mantido pequeno.
- Nem todos os eventos, cancelamentos e outputs estarão disponíveis de todos os provedores.

## Critérios para rever

- A segunda integração demonstra que o contrato atual força duplicação substancial ou perde semântica necessária.
- Plugins dinâmicos passam a ser uma necessidade concreta de extensão por terceiros, com modelo de confiança e isolamento definidos.
- O produto passa a observar múltiplos usuários/hosts e requer coordenação distribuída além do supervisor local.
