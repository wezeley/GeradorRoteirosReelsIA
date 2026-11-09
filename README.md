# Gerador de Roteiros de Reels com IA (n8n + Gemini)

Automação que transforma um tema de genética médica em um **rascunho completo de Reels** (roteiro, legenda, sugestões de edição e pontos para revisão médica) no tom do perfil de uma médica geneticista, e organiza tudo numa planilha para revisão.

Construído a partir de um problema real do meu trabalho: desde 2023 sou responsável pela comunicação digital da Dra. Sâmara Chebli, médica geneticista com atendimento online ([@drasamara.genetica](https://www.instagram.com/drasamara.genetica/)). Projeto publicado com autorização dela.

![Fluxo no n8n](prints/01-fluxo-n8n.png)

---

## O problema

Cada Reels do perfil passa por quatro etapas antes de ser gravado: escolher a pauta (datas de conscientização, notícias), pesquisar a condição, escrever roteiro e legenda, e revisar com a médica. Pela minha estimativa, isso leva de **3h30 a 6h do meu trabalho ativo por Reels**, e publicamos cerca de **2 Reels por semana**.

O recurso mais escasso do processo é o tempo da médica, que divide a agenda entre pacientes e família. Roteiro e legenda eram construídos em conjunto e dependiam de encaixes na agenda dela, por isso o roteiro aprovado e o vídeo gravado levavam cerca de 2 dias para chegar. A legenda, escrita depois da edição do vídeo, muitas vezes chegava perto do horário de postagem.

## A solução

Um fluxo no n8n em que eu informo o tema e o tipo de vídeo num formulário. A API do Gemini gera um rascunho no estilo do perfil, e o resultado entra numa planilha com o status **"aguardando revisão médica"**.

A médica deixa de ser coautora do texto desde o zero e passa a ser **revisora** de um rascunho que já segue a estrutura e o tom dos vídeos dela.

> **A IA não substitui a médica.** Ela gera um rascunho fiel ao estilo do perfil, e a revisão humana continua sendo a etapa que garante precisão e sensibilidade. O ganho está em revisar em vez de escrever do zero.

```mermaid
flowchart LR
    A[Formulário<br/>tema, tipo, contexto] --> B[Gemini<br/>prompt de sistema]
    B --> C[Code<br/>parse do JSON]
    C --> D[Google Sheets<br/>status: aguardando revisão]
    D --> E[Revisão da médica<br/>ajustar, aprovar, publicar]
```

## Como funciona

| Bloco | O que faz |
|---|---|
| **Formulário (n8n Form Trigger)** | Porta de entrada. Recebe `tema`, `tipo` (didático, campanha ou institucional), `contexto` opcional (data ou notícia) e `observacoes`. |
| **Gemini (Message a model)** | Envia o pedido para a API do Gemini 3.8 Flash com um prompt de sistema detalhado e exige a resposta em JSON. Tenta de novo automaticamente em caso de instabilidade da API (até 5 tentativas, 5 s de intervalo). |
| **Code (JavaScript)** | A IA devolve o JSON como texto. Este bloco remove possíveis marcações de código e faz o `JSON.parse`, separando cada parte em campos. |
| **Google Sheets (Append row)** | Grava uma linha por conteúdo: data, tema, tipo, contexto, título, roteiro, legenda, textos de tela, sugestões visuais, hashtags, pontos de revisão e status. |

![Formulário](prints/02-formulario.png)

### O que a IA entrega em cada rascunho

- **Roteiro** para ser falado, com 110 a 160 palavras (45 a 70 segundos)
- **Legenda** em três blocos, no padrão do perfil
- **Textos de tela** e **sugestões visuais** para a edição
- **Hashtags** opcionais
- **Pontos para revisão médica**: números, genes, sinais, tratamentos e expressões sensíveis que a médica precisa conferir

![Planilha com os rascunhos gerados](prints/03-planilha.png)

## Engenharia de prompt

O prompt de sistema ([ver no fluxo](gerador-de-roteiros-reels.json)) foi construído e refinado em etapas:

1. **Análise de 6 Reels reais do perfil.** Extraí os quadros dos vídeos e li as legendas embutidas para reconstruir as falas. Daí saíram três estruturas de vídeo: **didático** (uma condição), **campanha** (data de conscientização ou dor emocional) e **institucional** (o serviço e quem deve procurar).
2. **Regras de linguagem médica.** Linguagem cuidadosa ("pode", "costuma"), sem promessa de cura, sem diagnóstico, sem citar medicamentos, números apenas quando bem estabelecidos e em faixa quando a literatura usa faixa.
3. **Testes e correções.** Cada rodada de testes revelou problemas que viraram regras:

| Problema encontrado no teste | Regra adicionada |
|---|---|
| "Isso pode ser síndrome de Turner", soando como diagnóstico | Nome da condição em frase neutra e isolada |
| "A menina costuma ser muito esperta e ativa" | Proibido generalizar inteligência, comportamento ou aparência |
| "Me chama no link da bio", canal inventado | Proibido citar canais não informados |
| "Hoje, dia 21 de março", assumindo a data de publicação | Datas sem "hoje", a menos que informado |
| "Aperto no peito" num vídeo sobre câncer de mama | Proibidas expressões que lembrem sintomas da própria condição |
| No vídeo sobre atrofia muscular espinhal, a médica mudou o ângulo e acrescentou herança e triagem neonatal | Seguir o ângulo informado nas observações; explicar a herança e formas de diagnóstico precoce quando bem estabelecidas; condicionar o efeito do tratamento ("quando iniciado cedo") |

4. **Exemplos reais (few-shot).** A versão final inclui roteiros e legendas reais aprovados pela médica, com a instrução de usar apenas o tom e a estrutura, nunca copiar frases. Na comparação com um vídeo já publicado, o rascunho reproduziu a estrutura e o tamanho; a diferença ficou na criatividade das expressões, que continua sendo papel humano.

![Configuração do bloco do Gemini](prints/04-config-gemini.png)

## Revisão humana: a etapa que garante a qualidade

Nos testes, a própria IA listou bem os pontos factuais para conferência (números, genes, critérios), mas **não identificou problemas de tom**. Exemplos reais encontrados só na revisão humana:

- A generalização "muito esperta e ativa" sobre meninas com síndrome de Turner
- "Aperto no peito" num vídeo sobre câncer de mama
- "Uma pergunta sem nome": uma expressão com cara de boa, mas que não faz sentido (o que não tem nome é a doença, não a pergunta)

Por isso, todo conteúdo entra na planilha como **"aguardando revisão médica"**, e o status só muda para "aprovado" ou "publicado" depois da revisão.

## Resultado (fase de testes)

> Em uso piloto desde novembro de 2026. Os números abaixo vêm do primeiro Reels produzido com a automação (atrofia muscular espinhal) e da comparação com o processo anterior, e serão atualizados conforme novos vídeos forem produzidos.

O maior ganho não está no tempo de escrita, mas na **espera pela médica**, que era o gargalo do processo.

| | Antes | Com a automação |
|---|---|---|
| Tempo até receber roteiro aprovado e vídeo gravado | cerca de 2 dias | cerca de 6 horas |
| Legenda | escrita pela médica depois da edição do vídeo; com postagem por volta das 18h, costumava chegar depois das 17h | rascunho pronto desde o início; a médica só revisa |
| Papel da médica | coautora do texto desde o zero | revisora de um rascunho no estilo do perfil |

A escolha da pauta continua manual e leva o mesmo tempo, por isso fica fora da comparação.

**Próxima medição:** cronometrar o meu tempo ativo e o tempo de revisão da médica nos próximos Reels, para confirmar se o ganho se mantém.

## Limites e falhas encontradas

A automação depende de uma API externa, e isso tem consequências que precisei tratar no uso real.

- **Instabilidade da API.** Em novembro de 2026, durante uma sessão com a médica, a API do Gemini ficou vários minutos sem responder e terminou com erro 503 ("Service unavailable"). Aumentei as novas tentativas automáticas para 5, com 5 segundos de intervalo, o máximo que o n8n permite, mas a falha mostrou que só repetir não basta quando a API demora a responder. Naquele dia, o rascunho foi gerado no Google AI Studio com o mesmo prompt, e a saída planejada é um tempo limite com modelo reserva (ver Próximos passos).
- **Resposta fora do formato.** Mesmo com a saída em JSON exigida, modelos de linguagem podem devolver o conteúdo dentro de marcações de código. Por precaução, o bloco de JavaScript limpa o texto antes de fazer o `JSON.parse`.
- **Ângulo e conhecimento clínico.** No vídeo sobre atrofia muscular espinhal, a IA acertou a estrutura e o tom, mas a médica mudou o ângulo do vídeo e acrescentou informações que só a experiência dela trouxe, como a herança recessiva e a triagem neonatal. O prompt foi ajustado com regras gerais a partir dessa revisão (ver Engenharia de prompt), e a decisão sobre o que o vídeo diz continua sendo dela.
- **Erros de tom que a própria IA não percebe.** Como mostrado na seção de revisão humana, o modelo lista bem os pontos factuais, mas não identifica generalizações ou expressões inadequadas. Essa limitação é o motivo de a revisão médica ser obrigatória.

## Ferramentas

- **n8n** (n8n Cloud): orquestração do fluxo
- **Google Gemini API** (Gemini 3.8 Flash, plano gratuito): geração do texto
- **Google Sheets**: armazenamento e controle de status da revisão
- **JavaScript**: tratamento da resposta da IA
- **Google AI Studio**: testes e refinamento do prompt

## Como importar e usar

1. No n8n, crie um fluxo novo e importe o arquivo `gerador-de-roteiros-reels.json` (menu **…** > **Import from File**).
2. Crie uma credencial do **Google Gemini** com a sua própria chave do [Google AI Studio](https://aistudio.google.com/) e selecione-a no bloco do Gemini.
3. Crie uma planilha com estes cabeçalhos na primeira linha:
   `data | tema | tipo | contexto | titulo | roteiro | legenda | textos_de_tela | sugestoes_visuais | hashtags_sugeridas | pontos_revisao | status`
4. No bloco do Google Sheets, conecte sua conta Google, troque `COLE_AQUI_O_ID_DA_SUA_PLANILHA` pelo ID da sua planilha e `NOME_DA_ABA` pelo nome da aba.
5. Publique o fluxo e use a **Production URL** do formulário.

## Segurança

- Nenhuma chave de API está neste repositório. As credenciais ficam guardadas apenas no n8n.
- O arquivo exportado foi limpo antes da publicação: sem IDs de credenciais, sem ID da planilha original e sem o endereço do formulário de produção.
- Nenhum dado de paciente passa pelo fluxo: a entrada é apenas o tema educativo do vídeo.

## Próximos passos

- Tempo limite e modelo reserva: se o Gemini não responder a tempo, o fluxo cancela o pedido, usa um segundo modelo e registra na planilha qual modelo gerou o rascunho
- Calendário de datas de conscientização na planilha, para sugerir pautas automaticamente
- Busca de notícias recentes sobre o tema, como contexto para a IA
- Novos formatos além de Reels (carrossel e post estático)
- Gerar automaticamente o arquivo de legenda do vídeo a partir do áudio gravado

---

**Wesley Lima** · Estudante de ADS na UNIRP
[LinkedIn](https://linkedin.com/in/wezeley) · [GitHub](https://github.com/wezeley)
