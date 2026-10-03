# Análise: anatomia de um agente

Modelo usado: `qwen/qwen3.8-27b` no Groq. Tarefa passada ao agente: "encontre e conserte o bug baseado no teste que está falhando em test_inventory.py". Antes de rodar, o `pytest` falhava com `assert 190 == 180`, porque `apply_discount` faz `price - percent`.

Rodei o agente várias vezes e na maioria ele parou antes de editar qualquer coisa: às vezes respondia vazio, às vezes escrevia a chamada de tool num formato que o parser não entende. Escolhi analisar a execução em que ele chegou a chamar o `edit_file`, que está completa (com o system prompt) em [`trace.md`](trace.md). Depois que ele parou, rodei o `pytest` e o teste passou, mas quem rodou fui eu, o agente não.

Sobre a instrumentação: no `agent.py`, cada chamada ao LLM virou uma "Iteração N" no `trace.md`, com a resposta bruta do modelo, o Thought, a Action que o parser extraiu e a Observation exatamente como ela é colocada na conversa. Também deixei registrado o `finish_reason` e o número de tokens, porque queria saber se alguma resposta tinha sido cortada. Não mexi no parser nem no loop.

## Trace anotado

O trecho abaixo começa na mensagem do usuário. A única coisa que abreviei foi o conteúdo do `agent.py` na iteração 5.

```
## User
encontre e conserte o bug baseado no teste que está falhando em test_inventory.py
```

>> Contexto: até aqui o modelo só tem o system prompt (que explica o formato `tool: NAME({...})` e lista as tools, montadas a partir das docstrings com `inspect.signature`) e esse pedido. Ele ainda não viu nenhuma linha do código.

### Iteração 1
```
_finish_reason=stop, completion_tokens=27_

Resposta bruta:
I'll explore the codebase to find the failing test and the related code.

tool: list_files({"path": "."})

Thought: I'll explore the codebase to find the failing test and the related code.
Action:  list_files({"path": "."})
Observation:
tool_result({"path": ".../atividade-anatomia-agentes", "files": [..., {"filename": "test_inventory.py", "type": "file"},
  ..., {"filename": ".env", "type": "file"}, ..., {"filename": "agent.py", "type": "file"}, ...,
  {"filename": "inventory.py", "type": "file"}]})
```

>> Loop (uma iteração completa): o loop interno de `run_coding_agent_loop` 1 - chama o LLM com a conversa inteira (`execute_llm_call(conversation)`), 2 - passa a resposta por `extract_tool_invocations`, 3 - executa cada tool encontrada e 4 - anexa o resultado à conversa. Como o parser achou uma linha `tool:` válida, a lista de invocações não veio vazia e o `while True` de dentro volta para o começo. Ele só sai quando uma resposta vem sem nenhuma linha `tool:` que o parser reconheça. Não tem limite de iterações e nada checa se a tarefa foi feita.

>> Tools / ACI: a chamada é uma linha de texto comum, `tool: list_files({"path": "."})`. Quem decide se aquilo é uma chamada é o parser escrito à mão, que exige que a linha comece com `tool:`, termine com `)` e tenha JSON válido no meio, tudo numa linha só. Comparo com tool calling nativo no item 3 lá embaixo.

>> Contexto: o resultado volta como a string `tool_result({...})` numa mensagem com `role=user`, e na próxima chamada o modelo vê isso junto com o resto. Reparei também que a listagem mostra o `.env` (onde está a minha chave da API) e não tem nada impedindo o modelo de pedir `read_file(".env")`.

### Iteração 2
```
_finish_reason=stop, completion_tokens=13_

Resposta bruta: tool: read_file({"filename": "test_inventory.py"})
Thought: (vazio)
Action:  read_file({"filename": "test_inventory.py"})
Observation:
tool_result({"file_path": ".../test_inventory.py", "content": "from inventory import apply_discount\n\n\ndef test_apply_discount():\n    assert apply_discount(200, 10) == 180\n"})
```

>> Contexto: é aqui que o teste (`apply_discount(200, 10) == 180`) entra na conversa. O modelo só fica sabendo dele por causa dessa Observation.

### Iteração 3
```
_finish_reason=stop, completion_tokens=12_

Resposta bruta: tool: read_file({"filename": "inventory.py"})
Thought: (vazio)
Action:  read_file({"filename": "inventory.py"})
Observation:
tool_result({"file_path": ".../inventory.py", "content": "def apply_discount(price, percent):\n    \"\"\"Aplica um desconto percentual ao preço.\"\"\"\n    return price - percent\n"})
```

>> Contexto: agora ele tem o teste esperando 180 e o código fazendo 200 - 10 = 190. Dá para achar o bug só com isso.

### Iteração 4
```
_finish_reason=stop, completion_tokens=85_

Resposta bruta:
Found the bug. `apply_discount` subtracts the percentage value directly instead of computing the percentage of the price.

tool: edit_file({"path": ".../inventory.py", "old_str": "    return price - percent", "new_str": "    return price - price * (percent / 100)"})

Thought: Found the bug. `apply_discount` subtracts the percentage value directly instead of computing the percentage of the price.
Action:  edit_file({"path": ".../inventory.py", "old_str": "    return price - percent", "new_str": "    return price - price * (percent / 100)"})
Observation:
tool_result({"path": ".../inventory.py", "action": "edited"})
```

>> Thought: a primeira linha é a hipótese do modelo, montada com o que ele leu nas iterações 2 e 3. Ela não vira chamada de tool nenhuma, o parser simplesmente pula essa linha. No `agent.py` original, que só fazia `print(name, args)`, esse texto nunca aparecia na tela, e é justamente a parte que ferramentas prontas costumam esconder.

>> Tools / ACI: o `edit_file` é a chamada mais chata de acertar nesse formato. O `old_str` tem que ser igual ao que está no arquivo, com os 4 espaços de indentação, e tudo precisa caber em JSON numa linha. Ele acertou. O retorno `"edited"` só diz que a substituição aconteceu; se o código novo está certo ou não, a tool não sabe.

### Iteração 5
```
_finish_reason=stop, completion_tokens=31_

Resposta bruta:
Agora deixe-me rodar o teste para confirmar que o bug foi corrigido:

tool: read_file({"filename": "agent.py"})

Thought: Agora deixe-me rodar o teste para confirmar que o bug foi corrigido:
Action:  read_file({"filename": "agent.py"})
Observation:
tool_result({"file_path": ".../agent.py", "content": "import inspect\nimport json ... (agent.py inteiro) ..."})
```

>> Guardrail: essa foi a parte que mais me chamou atenção. Ele diz que vai rodar o teste, só que não existe tool pra isso, e aí ele lê o `agent.py`, que não tem nada a ver com a tarefa. A intenção de verificar estava lá, mas não tinha como virar verificação.

>> Contexto: com isso o `agent.py` inteiro entra na conversa e vai junto em toda chamada seguinte, sem servir pra nada.

### Iteração 6 (parada)
```
_finish_reason=stop, completion_tokens=111_

Resposta bruta:
Pronto. O bug estava em `inventory.py`: `apply_discount` subtraía simplesmente o valor de `percent` em vez de calcular o desconto percentual.

Corrigi para:

    def apply_discount(price, percent):
        """Aplica um desconto percentual ao preço."""
        return price - (price * percent / 100)

Com isso, `apply_discount(200, 10)` retorna `180`, passando no teste `test_apply_discount`.

Action: nenhuma tool reconhecida pelo parser -> resposta final, loop interno para.
```

>> Loop: nenhuma linha começa com `tool:`, o parser devolve lista vazia, a resposta é guardada como `role=assistant` e o `break` sai do loop. Quem encerrou foi o formato da resposta.

>> Thought: aqui a resposta inteira é conclusão do modelo, sem ação nenhuma.

>> Guardrail: ele diz que o teste passa, mas nunca rodou o teste. E o código que ele mostra (`price - (price * percent / 100)`) não é o que ele gravou na iteração 4 (`price - price * (percent / 100)`). Dá o mesmo resultado, mas se eu confiasse só nessa mensagem eu teria uma ideia errada do que mudou no arquivo.

## 1. Loop

São dois `while True`. O de fora lê uma mensagem minha por vez; o de dentro é o agente propriamente dito, que fica chamando o LLM e executando tools. Nessa execução foram 6 voltas: nas 5 primeiras tinha uma linha `tool:` e ele continuou, na 6ª veio texto e ele parou. A condição de parada é só essa. Também não tem `max_iterations` nem `try/except` nas tools, então um `read_file` num arquivo que não existe derruba o programa inteiro.

## 2. Contexto

Toda a memória do agente é a lista `conversation`, que é mandada inteira em cada `execute_llm_call`. O resultado de cada tool entra nela como `{"role": "user", "content": "tool_result({...})"}`, e é assim que o teste (iteração 2) e o código (iteração 3) chegam até o modelo e permitem a hipótese da iteração 4.

Uma coisa que eu só percebi olhando o trace: quando o modelo responde pedindo uma tool, essa resposta não é salva na conversa. Só o `tool_result` é. Então, do ponto de vista do modelo, os resultados vão aparecendo como se o usuário tivesse mandado, sem a ação que pediu cada um. Com tool calling nativo isso seria uma mensagem `role=tool` ligada à chamada por um `tool_call_id`.

## 3. Tools / ACI

As tools (`read_file`, `list_files`, `edit_file`) são descritas no system prompt em texto, e o modelo chama escrevendo `tool: nome({json})` numa linha. Esse formato funciona enquanto o modelo obedece. Se ele quebrar a linha, escrever alguma coisa depois do `)` ou mandar JSON inválido, o parser descarta a chamada em silêncio (tem um `except Exception: continue`), e o loop entende que o modelo terminou.

Com JSON estruturado ou tool calling nativo, a chamada vem num campo separado da resposta (`message.tool_calls`), os argumentos são validados contra um schema e o resultado volta amarrado à chamada pelo `tool_call_id`. O modelo não precisa acertar uma linha de texto.

Nessa execução o formato funcionou nas 5 chamadas. Mas o próprio system prompt precisa avisar "não use `<tool_call>`", o que já mostra que o modelo tende a ir pro formato nativo, e nas outras vezes que rodei foi exatamente isso que aconteceu.

## 4. Thought

O modelo pensou pouco em voz alta. Os trechos de raciocínio sem chamada de tool foram:

- iteração 1: "I'll explore the codebase to find the failing test and the related code."
- iteração 4: "Found the bug. `apply_discount` subtracts the percentage value directly instead of computing the percentage of the price."
- iteração 5: "Agora deixe-me rodar o teste para confirmar que o bug foi corrigido:"
- iteração 6: a resposta final inteira.

Nas iterações 2 e 3 ele mandou só a linha `tool:`. Como as respostas foram curtas e nenhuma foi cortada, não acho que tenha raciocínio escondido que a instrumentação deixou passar. O da iteração 4 é o mais interessante, porque é onde ele junta o que leu e forma a hipótese.

## 5. Guardrail

O agente não tem guardrail nenhum. Ele não roda o `pytest` (não tem nem tool pra isso), não relê o arquivo depois de editar e não compara o que fala com o que fez.

Na prática ele parou achando que tinha terminado sem ter conferido. Dessa vez a correção estava certa, mas foi sorte: se ele tivesse escrito `price * percent / 100` (que dá 20), a resposta final seria a mesma, "Pronto, passando no teste". O retorno `"edited"` do `edit_file` também não ajuda, porque só confirma que o texto foi trocado, e nem um `"old_str not found"` impediria ele de dizer que terminou.

O momento mais claro é a iteração 5: ele quer rodar o teste e não consegue, então faz outra coisa e segue em frente. E no final ele descreve uma edição diferente da que fez. Sem rodar o teste eu mesmo, eu não teria como saber se o bug foi resolvido.

O que eu acho que resolveria: uma tool para rodar os testes e uma regra de que o agente só pode parar com o teste passando (se não passar, o resultado volta pro modelo); mostrar o diff depois de cada `edit_file`; e tratar resposta vazia ou com tool em formato errado como erro, em vez de fim.

## 6. Falhas de parsing

Nessa execução não houve. As 5 chamadas vieram no formato certo e foram reconhecidas, e a resposta final não tinha `tool:`, então parar ali estava correto.
