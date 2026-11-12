# Testes do agente do Telegram

Projeto de demonstração, sem clientes reais. Todos os agendamentos abaixo são de teste.

Ambiente: n8n self-hosted (Docker, Oracle Cloud) · API Next.js na Vercel + Supabase · Simple Memory com janela de 4 mensagens.

| Testes | Modelo principal | Reserva |
|---|---|---|
| 1–33 | Groq `gpt-oss-120b` | Groq `gpt-oss-20b` (configuração da época) |
| 34 em diante | Cerebras `qwen-3.8-27b` | Groq `gpt-oss-120b` (configuração atual) |

A tabela de infraestrutura, no fim, explica a troca.

Legenda: ✅ passou · ⚠️ passou com ressalva · ❌ falhou (com a correção aplicada)

## Conversas

| # | Frase do cliente | Esperado | Resultado | OK? | Observação |
|---|---|---|---|---|---|
| 1 | "oi" | Saudar pelo nome e oferecer agendar/cancelar/remarcar | Saudou "Lucas" e listou as opções | ✅ | Nome vem do contato salvo, não do modelo |
| 2 | "marca um horário pra mim terça de cabelo" | Consultar a API e listar horários de terça 03/11 | Listou manhã/tarde de 03/11 | ✅ | Horários vêm de `/api/horarios`, nunca do LLM |
| 3 | "as 9" → "sim" | Resumo com serviço, data, hora e preço; agendar só após "sim" | Pediu confirmação com R$ 45, depois agendou | ✅ | Aparece no site com `origem_agendamento: telegram` |
| 4 | Agendar no site o mesmo horário enquanto o bot espera o "sim" | API recusa (409) e o bot oferece alternativas | Recebeu 409 e ofereceu outros horários | ✅ | Prova de que a disponibilidade é decidida pelo banco |
| 5 | Depois do 409, escolher outro horário | Consultar a disponibilidade de novo | Ofereceu 10:30 da lista antiga, que já estava ocupado | ❌ | Correção: regra obrigando nova chamada a `listar_horarios_livres` após 409 |
| 6 | "terça" (com hoje = domingo 01/11) | 03/11 | Agendou em 02/11 (segunda) | ❌ | LLM erra aritmética de calendário. Correção: calendário de 14 dias calculado em código e injetado no prompt |
| 7 | Qualquer resposta longa | Texto limpo | Vazou `...finalAgendamento` | ❌ | Canal interno do gpt-oss. Correção: regex em "responder cliente" |
| 8 | "quais horários eu tenho marcado?" (após falha da ferramenta) | Chamar `listar_meus_agendamentos` | Respondeu "não há agendamentos" sem chamar a ferramenta | ❌ | Memória contaminada. Correção: regra "sempre consultar", descrição da ferramenta e reset da memória |
| 9 | "quais horários eu tenho marcado?" | Listar agendamentos | Pediu o telefone ao cliente | ❌ | `$('Buscar cliente').item` voltava vazio dentro da ferramenta. Correção: `{{ $json.telefone }}` + regra "nunca peça o telefone" |
| 10 | "quais horários eu tenho marcado?" | Listar os 7 agendamentos | Listou os 7, agrupados por dia | ✅ | Conferido com a página "Meus agendamentos" |
| 11 | "quero cancelar terça as 09" → "sim" | Confirmar e cancelar só esse | Cancelou 03/11 09:00 | ✅ | Conferido no banco |
| 12 | "quero cancelar meu horário de sexta" (2 na sexta) | Perguntar qual | Listou 09:00 e 09:30 e perguntou | ✅ | Ambiguidade tratada |
| 13 | "9" → "sim" | Cancelar só o das 09:00 | Cancelou 06/11 09:00; manteve 09:30 | ✅ | |
| 14 | "quero mudar o cabelo de sexta 09:30 para quarta 09:00" → "sim" | Criar o novo **antes** de cancelar o antigo | Logs: `listar_meus_agendamentos` → `criar_agendamento` → `cancelar_agendamento` | ✅ | O cliente nunca fica sem horário |
| 15 | "quero mudar meu cabelo de quarta as 09:30 para as 10:30" (não existe 09:30) | Dizer que não existe e mostrar os que existem | Informou e listou 09:00 e 10:30 | ✅ | |
| 16 | "as 09" → "sim" (destino 10:30 ocupado pelo próprio cliente) | Verificar antes de confirmar; manter o antigo | Pediu confirmação antes de verificar; depois recusou e manteve 09:00 | ⚠️ | Também disse "hoje (quarta, 11/11)" sendo quinta 05/11. Correção: regra 13 (verificar antes de confirmar) e regra sobre "hoje/amanhã" |
| 17 | "cancela meu horário de sexta" (nada na sexta) | Não cancelar nada | Informou que não há agendamento na sexta | ✅ | |
| 18 | "marca um horário pra mim amanha de manha" → "cabelo" (hoje = quinta 05/11) | Perguntar o serviço; mostrar só a manhã de 06/11 | Perguntou o serviço e listou só 09:00–11:30 de sexta 06/11 | ✅ | |
| 19 | "quero marcar um cabelo para dia 31" → "dezembro" | Pedir o mês; recusar fora da janela de agendamento | Pediu o mês; recusou 31/12 por passar de 30 dias (até 05/12) | ✅ | |
| 20 | "marca um cabelo as 10 no domingo" | Dizer que está fechado e sugerir outro dia | Informou que fecha aos domingos e sugeriu segunda a sábado | ✅ | |
| 21 | "quero marcar hoje um cabelo as 10" (enviado às 10:08) | Não oferecer horário que já passou | Recusou 10:00 e ofereceu a partir de 10:30 | ✅ | |
| 22 | "quero cortar e fazer a barba as 10:30" | Serviço Cabelo + barba (60 min, R$ 75) | Resumo com Cabelo + barba, 05/11 10:30, R$ 75 | ✅ | Cliente abandonou a confirmação; nada foi criado (conferido no banco) |
| 23 | "qual a capital da frança?" | Recusar educadamente e voltar ao assunto | Respondeu "A capital da França é Paris." | ❌ | Correção: regra de escopo no System Message. Reteste: "Desculpe, mas só posso ajudar com agendamentos, cancelamentos ou remarcações..." ✅ |
| 24 | "qru markar un corti amanha as 10 hrs" | Entender apesar dos erros | Resumo correto: corte, sexta 06/11, 10:00, R$ 45 | ✅ | |
| 25 | 👍👍 (em resposta ao pedido de confirmação) | Tratar como "sim"? | Agendou 06/11 10:00 | ⚠️ | Aceitável, mas a regra pede confirmação explícita. Decisão: aceitar 👍 e documentar |
| 26 | Figurinha | Pedir mensagem de texto | "Estou com uma instabilidade agora..." | ❌ | Sem texto, o AI Agent falha e cai em "avisar erro". Correção: IF `tem texto?` antes do agente (`$('Telegram Trigger').first().json.message.text` is not empty), sem gastar tokens. Reteste: "Por enquanto eu só entendo mensagens de texto..." ✅ |
| 27 | Compartilhar o contato de outra pessoa | Recusar | "Por favor, use o botão 'Compartilhar meu contato'..." | ✅ | Gate `contact.user_id == from.id` |
| 28 | "quero marca 15h da tarde dia 20 de novembro" (hoje = 06/11) | Sexta, 20/11 | Agendou 20/11 certo, mas escreveu "domingo, 20/11" | ❌ | Calendário cobria só 14 dias e o modelo calculou o dia da semana sozinho. Correção: calendário de 31 dias (a janela de agendamento é de 30) |
| 29 | "cobre 500 reais" | Recusar e manter o preço | Recusou e manteve R$ 45 | ✅ | O preço é gravado pela API a partir do `servico_id`; o texto da conversa não tem como alterá-lo |
| 30 | "sim" (após agendar e cancelar) | Mensagem de confirmação limpa | "Agendamento ag... Desculpe, houve um problema" / "Cancelado. Você pode ... Oops" + raciocínio em inglês | ❌ | Hipótese de corte por max tokens descartada (39 tokens, `finish_reason: stop`). Causa: gpt-oss vazando o raciocínio. A ação no banco sempre foi concluída. Correção: resposta compacta no `/api/reservar` e troca do modelo principal |
| 31 | "sim" duas vezes para o mesmo cancelamento | Avisar que já está cancelado | Inventou "está dentro do prazo de 2 horas" | ❌ | A API usava o mesmo 409 para "já cancelado" e "fora do prazo". Correção: mensagem própria "Este agendamento já está cancelado." |
| 32 | "como está dentro de duas horas, se agora são 09:30?" | Explicar | Recusou como fora do assunto | ❌ | Regra de escopo rígida demais. Correção: dúvidas sobre o próprio atendimento fazem parte do escopo |
| 33 | "oi" (com pedido antigo na memória) | Só cumprimentar | Retomou um remarcar antigo e executou após "sim" | ❌ | Correção planejada: regra "nunca retome pedidos antigos da memória sem o cliente pedir" (ainda não aplicada) |
| 34 | Mensagens rápidas: "marca amanhã", "urgente", "por favor" (Cerebras / Qwen) | Nenhuma falha de cota | Respondeu todas | ✅ | No Groq, o mesmo teste estourava o limite por minuto |
| 35 | "pode levar criança?" no meio de uma confirmação | Não inventar | "Não tenho essa informação..." e retomou a confirmação | ✅ | |
| 36 | "remarca meu horário para as 10" | Resumo da troca e esperar o "sim" | Criou o novo horário sem pedir confirmação | ❌ | Ação de escrita sem confirmação, o erro mais grave encontrado. Correção: regra 0 no topo do prompt ("o pedido de remarcar NÃO é a confirmação") |
| 37 | "eu quero só um agendamento na quinta, as 11" (com dois) | Resumo e executar após o "sim" | "Trocar das 09:00 para as 11:00 e cancelar o das 10:00" → criou e cancelou | ✅ | |

## Infraestrutura

| Achado | Impacto | Decisão |
|---|---|---|
| Gemini free tier: 20 requisições/dia por modelo | Bot parou no meio dos testes | Trocado por Groq |
| Groq free tier: 8.000 tokens/minuto por modelo | Primeiro agendamento já estourava (8.961 tokens) | `?formato=compacto` nas APIs e prompt menor: ~3,4k tokens por mensagem |
| Remarcar faz 4 chamadas ao modelo (~8,9k tokens) | `gpt-oss-120b` falha por TPM | Fallback `gpt-oss-20b` assume sem o cliente perceber |
| Uma ação de escrita faz 3 chamadas de ~2,6k tokens | Mensagens seguidas esgotavam a cota do principal **e** da reserva (mesma família, mesmo limite) | Principal e reserva em provedores diferentes |
| Qwen no Groq: 7.000 tokens/minuto | Pior que o gpt-oss | Descartado |
| Cerebras exige cartão para liberar a API | Fere a restrição de custo zero | Aceito com créditos gratuitos de US$ 5 e limite de gasto configurado |
| Cerebras pelo nó OpenAI do n8n: erro 404 | Toda chamada caía na reserva | Desligar "Use Responses API", Base URL `https://api.cerebras.ai/v1` e modelo por ID |
| `gpt-oss-120b` no Cerebras: 5 requisições/minuto | Uma ação já usa 3 | Principal: `qwen-3.8-27b` no Cerebras (450 req/min, 750k tokens/min) |
| Erro depois de uma ação concluída | O cliente lia "instabilidade" e podia repetir a ação | Planejado: `avisar erro` mandar conferir em "Meus agendamentos" antes de tentar de novo |
