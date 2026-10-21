# strands — Strands Agents em Sea

Um framework de agentes orientado pelo modelo, no estilo do [Strands Agents](https://github.com/strands-agents/harness-sdk). É escrito inteiramente em [Sea](../sea_vm) e usa a [Maritaca AI](https://www.maritaca.ai) (modelos Sabiá) como provedor padrão.

Você escreve as instruções e as ferramentas. O modelo decide os passos: chama ferramentas, recebe os resultados e continua até ter a resposta.

```sea
local strands = require("../strands/strands")

local agent = strands.agent({
    system_prompt = "Você é um assistente prestativo. Responda em português.",
    tools = [strands.vended.calculator, strands.vended.current_time],
})?

local r = agent:invoke("Quanto é a raiz quadrada de 1764? E que horas são em Brasília?")?
print(r.text)
```

## Instalação

O projeto usa o binário `sea` e a biblioteca `http` do checkout do sea_vm ao lado (`~/projects/sea_vm`). Se as bibliotecas estiverem em outro lugar, mude o caminho em `lib/deps.sea`. A biblioteca `http` precisa estar compilada:

```sh
make -C ~/projects/sea_vm
make -C ~/projects/sea_vm/libs/http
make -C ~/projects/sea_vm/libs/ssl
export MARITACA_API_KEY=...        # https://plataforma.maritaca.ai
```

| Comando | O que faz |
| --- | --- |
| `bin/strands chat` | Conversa no terminal com o agente completo (harness) |
| `bin/strands run "pergunta"` | Uma pergunta, com a resposta na saída padrão |
| `~/projects/sea_vm/bin/sea test tests/` | A bateria de testes, que roda sem chave e sem rede |
| `~/projects/sea_vm/bin/sea examples/01_ola.sea` | Os exemplos; sem a chave, usam respostas roteirizadas |

## Conceitos

| | |
| --- | --- |
| **Agente** | `strands.agent({...})`: o modelo, as instruções, as ferramentas e o laço que os liga |
| **Ferramenta** | `strands.tool({name, description, inputs, fn})`: uma função que o modelo pode chamar |
| **Modelo** | Um provedor: `strands.models.maritaca()`, `openai()` (qualquer API compatível) ou `scripted()` (testes) |
| **Hooks** | Callbacks em cada passo: antes e depois da invocação, da chamada ao modelo e de cada ferramenta |
| **Conversa** | Gerenciadores que mantêm o histórico dentro do contexto do modelo |
| **Sessão** | Guarda a conversa e o estado entre execuções |
| **Multiagente** | Agentes como ferramentas, `strands.swarm` e `strands.graph_builder` |
| **MCP** | Ferramentas de servidores Model Context Protocol |
| **Servidor** | Um agente por HTTP: JSON, streaming SSE e o protocolo A2A |
| **Harness** | `strands.harness()`: um agente pronto, com ferramentas, sessão e prompt |

## O agente

```sea
local agent = strands.agent({
    model = strands.models.maritaca({model_id = "sabia-4"}),   -- o padrão
    system_prompt = "...",
    tools = [...],                     -- ferramentas, clientes MCP, agentes como ferramenta
    name = "assistente", description = "...", agent_id = "default",
    callback_handler = fn,             -- padrão: imprime o texto enquanto chega; false: silencioso
    conversation_manager = strands.conversation.sliding_window({window_size = 40}),
    hooks = [provedor],
    session_manager = strands.session.file({session_id = "..."}),
    state = {chave = "valor"},
    tool_executor = "concurrent",      -- ou "sequential"
    max_turns = 20,                    -- chamadas ao modelo por invocação
    max_total_tokens = 50000,
    retry = {max_attempts = 6, initial_delay_ms = 4000, max_delay_ms = 240000},
})?
```

### Invocação

`agent:invoke(prompt, opts)` devolve o resultado ou um erro.

O `prompt` pode ser:
- um texto;
- blocos de conteúdo (texto, imagem, documento);
- uma lista de mensagens;
- `nil`, para continuar a conversa atual;
- depois de uma interrupção, as respostas a ela.

Opções:

| Opção | Efeito |
| --- | --- |
| `state` | O `invocation_state`, que chega aos hooks e às ferramentas |
| `callback_handler` | Um handler extra só para esta invocação |
| `max_turns`, `max_total_tokens` | Limites só para esta invocação |
| `structured_output` | Um schema que a resposta final precisa seguir |

Campos do resultado:

| Campo | Conteúdo |
| --- | --- |
| `text` | O texto da resposta final |
| `stop_reason` | Por que parou: `end_turn`, `limit_turns`, `limit_total_tokens`, `cancelled` ou `interrupt` |
| `message` | A última mensagem do assistente |
| `reasoning` | O texto de raciocínio, quando o modelo envia (`sabia-4-thinking`) |
| `usage` | Tokens de entrada, de saída e o total |
| `metrics` | Ciclos, latência, uso de cada ferramenta e um trace de spans |
| `interrupts` | As interrupções pendentes, quando `stop_reason` é `interrupt` |
| `structured_output` | O valor validado, quando a invocação pediu saída estruturada |

Erros:
- Os erros são valores, como em todo o Sea.
- `strands.error_kind(e)` diz o tipo: `throttled`, `context_window_overflow`, `model`, `max_tokens`.
- Limite de requisições (HTTP 429 e 5xx) é tentado de novo com espera crescente.
- Estouro de contexto faz o gerenciador de conversa reduzir o histórico e tentar outra vez.

Outros métodos do agente:

| Método | O que faz |
| --- | --- |
| `agent:stream(prompt)` | Um canal com os eventos: `for e in agent:stream("oi") do ... end` |
| `agent:cancel()` | Para no próximo passo; pode ser chamado de outra goroutine |
| `agent:structured_output(schema, prompt)` | Extrai dados validados contra o schema |
| `agent.tool.nome({...})` | Chama uma ferramenta diretamente; a chamada fica registrada no histórico |
| `agent:as_tool()` | O agente vira ferramenta de outro agente |
| `agent:spawn({...})` | Uma cópia com as mesmas configurações e a conversa vazia |
| `agent.messages` | O histórico, no formato do Strands/Bedrock Converse |
| `agent.state` | Dados chave/valor que acompanham o agente e não vão para o modelo: `get`, `set`, `delete` |

### Mensagens

Cada mensagem é `{role, content = [bloco, ...]}`, e cada bloco tem um destes campos:
- `text`;
- `toolUse = {toolUseId, name, input}`;
- `toolResult = {toolUseId, status, content}`;
- `reasoningContent`;
- `image`;
- `document`.

O módulo `strands.messages` tem funções para montar e ler essas estruturas.

## Ferramentas

```sea
local clima = strands.tool({
    name = "clima",
    description = "Previsão do tempo de uma cidade",
    inputs = {
        cidade = {type = "string", description = "Nome da cidade"},
        dias = {type = "integer", minimum = 1, maximum = 7, default = 1},
        unidade = {type = "string", enum = ["c", "f"], optional = true},
    },
    fn = function(input, ctx)
        return {cidade = input.cidade, previsao = "sol"}
    end,
})?
```

**Parâmetros:** `inputs` gera o JSON Schema. Um parâmetro é obrigatório, a menos que tenha `default` ou `optional = true`. Também dá para passar `input_schema` com um JSON Schema completo.

**Validação:** a entrada é validada antes da chamada (tipos, `enum`, limites, `pattern`, campos obrigatórios), e os padrões são preenchidos. Uma entrada inválida volta para o modelo como erro, para ele corrigir.

**O que `fn` pode devolver:**
- um texto;
- qualquer valor JSON;
- um resultado completo `{status, content}`;
- um valor de erro.

Um `panic` dentro da ferramenta também vira resultado de erro.

**O contexto `ctx`:**
- `ctx.tool_use`: a chamada;
- `ctx.agent`: o agente;
- `ctx.invocation_state`: o estado da invocação;
- `ctx.stop()`: encerra o laço depois desta rodada de ferramentas.

### Ferramentas prontas (`strands.vended`)

| Ferramenta | O que faz |
| --- | --- |
| `calculator` | Expressões matemáticas, com um avaliador próprio que não executa código |
| `current_time` | Data e hora, em qualquer fuso horário |
| `environment` | Lê variáveis de ambiente, mascarando as que parecem segredos |
| `sleep` | Espera alguns segundos |
| `shell` | Roda comandos e devolve o código de saída, stdout e stderr |
| `http_request` | Requisições HTTP |
| `file_read` | Lê arquivos: trecho de linhas, busca, glob, estatísticas, listagem |
| `file_write` | Escreve arquivos |
| `editor` | Edita arquivos: `view`, `create`, `str_replace`, `insert`, `undo_edit` |
| `think` | Raciocina em vários ciclos antes de responder |

`shell`, `file_write` e `editor` pedem confirmação no terminal, a menos que `BYPASS_TOOL_CONSENT=true` ou que a ferramenta tenha sido criada com `confirm`.

As fábricas configuram o comportamento:
- `shell_tool({cwd, timeout_ms, confirm})`;
- `file_read_tool({root})`, que limita os caminhos a um diretório;
- `http_request_tool({allowed_hosts})`.

## Modelos

```sea
strands.models.maritaca({model_id = "sabia-4", params = {temperature = 0.3, max_tokens = 2000}})
strands.models.openai({base_url = "http://localhost:11434/v1", api_key = "ollama", model_id = "llama3.1"})
strands.models.scripted(["resposta 1", {tool = "calculator", input = {expression = "2+2"}}])
```

**Maritaca:**
- **Modelos:** `sabia-4` (o padrão), `sabiazinho-4` (o mais rápido e barato) e `sabia-4-thinking` (com raciocínio, contexto de 1M tokens).
- **Inferência no Brasil:** acrescente `-br-sp` ao nome do modelo.
- **Variáveis de ambiente:** `MARITACA_API_KEY`, `MARITACA_MODEL`, `MARITACA_BASE_URL`.

O provedor fala a API Chat Completions da OpenAI, com streaming SSE e `tool_calls`. Por isso serve também para OpenAI, Ollama, vLLM, Groq e outros.

**Provedor próprio:** basta um objeto com `stream(request, emit)`, que emite os eventos do Strands (`messageStart`, `contentBlockDelta`...). A especificação está em `lib/stream.sea`.

## Hooks

```sea
agent:add_hook("BeforeToolCallEvent", function(e)
    if e.tool_use.name == "shell" then e.cancel_tool = "o shell está desligado" end
end)
```

**Eventos:**

| Evento | Quando |
| --- | --- |
| `AgentInitializedEvent` | O agente foi criado |
| `BeforeInvocationEvent`, `AfterInvocationEvent` | Início e fim de uma invocação |
| `MessageAddedEvent` | Uma mensagem entrou no histórico |
| `BeforeModelCallEvent`, `AfterModelCallEvent` | Antes e depois de cada chamada ao modelo |
| `BeforeToolCallEvent`, `AfterToolCallEvent` | Antes e depois de cada ferramenta |
| `BeforeNodeCallEvent`, `AfterNodeCallEvent` | Antes e depois de cada nó, em swarm e grafo |

**Campos que um hook pode alterar:**

| Evento | Campo | Efeito |
| --- | --- | --- |
| `BeforeToolCallEvent` | `tool_use` | Muda o nome ou a entrada da chamada |
| `BeforeToolCallEvent` | `selected_tool` | Troca a ferramenta que vai rodar |
| `BeforeToolCallEvent` | `cancel_tool` | Cancela a chamada (`true` ou uma mensagem) |
| `AfterToolCallEvent` | `result` | Substitui o resultado |
| `AfterToolCallEvent` | `retry` | Roda a ferramenta de novo |
| `AfterModelCallEvent` | `retry` | Chama o modelo de novo |
| `AfterInvocationEvent` | `resume` | Começa outra invocação com esta entrada |

Um provedor de hooks é um objeto com `register_hooks(registry)`, passado em `hooks = [...]`.

**Interrupções (humano no circuito):** dentro de `BeforeToolCallEvent`, `e:interrupt(nome, motivo)` para o agente:
1. O resultado volta com `stop_reason = "interrupt"` e a lista `interrupts`.
2. Você chama `agent:invoke([{interruptResponse = {interruptId, response}}])`.
3. Na retomada, a mesma chamada de `e:interrupt` devolve a resposta.

O exemplo `03_hooks.sea` mostra o fluxo completo.

## Conversa e sessões

**Gerenciadores de conversa:**

| Gerenciador | Comportamento |
| --- | --- |
| `strands.conversation.sliding_window({window_size = 40})` | O padrão. Mantém as últimas mensagens sem separar uma chamada de ferramenta do seu resultado. No estouro de contexto, primeiro encurta resultados grandes de ferramentas |
| `strands.conversation.summarizing({summary_ratio = 0.3, preserve_recent_messages = 10})` | Troca as mensagens antigas por um resumo feito pelo próprio modelo |
| `strands.conversation.null()` | Não mexe no histórico |

**Sessões:**

| Sessão | Onde guarda |
| --- | --- |
| `strands.session.file({session_id, storage_dir})` | Em disco, no layout do `FileSessionManager` do Strands |
| `strands.session.memory({session_id})` | Na memória do processo |
| `strands.session.repository(repo, opts)` | Em qualquer armazenamento com `read`, `write`, `delete` e `list` |

Uma sessão guarda as mensagens, o `agent.state` e o estado do gerenciador de conversa de cada `agent_id`.

## Multiagente

```sea
-- agentes como ferramentas
local orquestrador = strands.agent({tools = [especialista:as_tool()]})

-- swarm: passam a tarefa adiante com handoff_to_agent
local time_ = strands.swarm({agents = [pesquisador, redator], max_handoffs = 10})?
local r = time_:invoke("tarefa")      -- r.status, r.text, r.node_history

-- grafo: dependências, arestas condicionais, nós em paralelo
local b = strands.graph_builder()
b:add_node(coleta, "coleta")
b:add_node(analise, "analise")
b:add_edge("coleta", "analise", function(estado) return true end)
local r2 = b:build()?:invoke("tarefa")
```

- **Nós do grafo:** um nó pode ser um agente, um swarm, outro grafo ou uma função `fn(entrada, estado)`.
- **Composição:** swarms e grafos também viram ferramentas (`:as_tool()`).
- **Limites:** `max_handoffs`, `max_iterations` e timeouts por nó e por execução.

## MCP

```sea
local fs_server = strands.mcp.client({command = ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/tmp"]})
local remoto = strands.mcp.client({url = "https://exemplo.com/mcp", headers = {Authorization = "Bearer ..."}})
local agent = strands.agent({tools = [fs_server, remoto]})
-- ...
fs_server:stop()
```

- **Transportes:** stdio (um processo filho) e Streamable HTTP.
- **Opções:** `prefix` (prefixo nos nomes das ferramentas), `allowed`, `rejected`.
- **Além das ferramentas:** `list_prompts`, `get_prompt`, `list_resources`, `read_resource`.
- **Encerramento:** chame `stop()` no fim. Um servidor stdio aberto mantém o programa vivo.

## Servidor HTTP e A2A

```sea
local srv = strands.server.serve(function(session_id)
    return strands.agent({...})            -- um agente por sessão
end, {port = 9000, name = "atendente", api_key = "segredo"})?
srv.wait()
```

| Rota | O que faz |
| --- | --- |
| `GET /health` | Verificação de saúde |
| `POST /invoke {prompt, session_id}` | O resultado em JSON |
| `POST /stream` | O resultado em Server-Sent Events: pedaços de texto, ferramentas e o resultado final |
| `GET /.well-known/agent.json` | O cartão do agente no protocolo A2A |
| `POST /` | A2A em JSON-RPC: `message/send` e `message/stream` |

O cliente A2A é `strands.server.a2a_agent(url)`. Ele tem `:invoke(texto)`, `:card()` e `:as_tool()`.

## Harness e CLI

`strands.harness({workdir, model, extra_tools, session_id, summarize, confirm})` monta um agente completo:
- as ferramentas de arquivos, editor, shell e HTTP, restritas ao `workdir`;
- calculadora, hora e `think`;
- uma sessão em `<workdir>/.strands/sessions`;
- um prompt de sistema que descreve o ambiente.

A CLI `bin/strands` usa esse agente:

```sh
bin/strands chat                         # conversa (/help, /reset, /tools, /model, /usage, /exit)
bin/strands run "resuma o README.md"     # uma pergunta
bin/strands chat --model sabiazinho-4 --mcp "npx -y @modelcontextprotocol/server-memory" --yes
```

## Estrutura

| Caminho | Conteúdo |
| --- | --- |
| `strands.sea` | O ponto de entrada |
| `cli.sea`, `bin/strands` | A linha de comando |
| `lib/agent.sea` | O agente e o laço de eventos |
| `lib/tools.sea`, `lib/messages.sea`, `lib/stream.sea` | Ferramentas, mensagens e eventos de streaming |
| `lib/hooks.sea`, `lib/conversation.sea`, `lib/session.sea` | Hooks, conversa e sessões |
| `lib/state.sea`, `lib/metrics.sea`, `lib/handlers.sea` | Estado, métricas e callback handlers |
| `lib/models/` | Os provedores: maritaca, openai, scripted |
| `lib/multiagent/` | Swarm e grafo |
| `lib/mcp.sea` | O cliente MCP |
| `lib/server.sea` | O servidor HTTP/A2A |
| `lib/harness.sea` | O harness |
| `lib/tools/` | As ferramentas prontas |
| `tests/` | Os testes (`sea test tests/`) |
| `examples/` | Os exemplos numerados |

## Diferenças em relação ao Strands em Python

- **Chamada do agente:** em Sea um objeto não é chamável, então `agent("...")` vira `agent:invoke("...")`.
- **Interrupções:** `e:interrupt()` devolve `nil` na primeira passagem em vez de lançar exceção. O hook deve retornar logo em seguida.
- **Erros:** são valores com um tipo no começo da mensagem (`[throttled] ...`), não exceções.
- **Streaming:** usa callbacks e canais (`agent:stream`) em vez de iteradores assíncronos.
- **Fora do escopo:** telemetria OpenTelemetry e streaming bidirecional de áudio. As métricas e os traces ficam em `result.metrics`.
