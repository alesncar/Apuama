# APUAMA

**Projeto APUAMA – Agilidade na Triagem de Solicitações Periciais.**

*Apuama*, do Tupi: "aquele que é veloz, que tem correnteza".

Triagem automática de caixas de e-mail corporativas, com apoio de um modelo
de linguagem. Em produção desde julho de 2026, rodando a cada 15 minutos,
em duas instalações independentes.

---

## O problema

Uma operação pericial recebe, numa mesma caixa compartilhada, coisas de
natureza muito diferente: convocações com prazo fatal, cobranças, dúvidas
técnicas sobre trabalhos em curso, propostas comerciais e marketing em
massa. Tudo chega misturado, e a consequência de errar não é simétrica —
deixar de ver uma convocação com prazo custa caro; deixar de ver uma
newsletter não custa nada.

A triagem manual consumia tempo da equipe todos os dias e, por depender de
alguém abrir a caixa, falhava justamente quando a caixa estava mais cheia.

Três restrições moldaram o projeto:

1. **Sigilo.** Parte das mensagens trata de processos em segredo de
   justiça. Esse conteúdo não pode sair do servidor da organização, e
   mandá-lo a um serviço externo seria falha grave.
2. **Nada pode ser perdido.** Uma classificação errada é recuperável; uma
   mensagem apagada ou escondida, não.
3. **Isolamento entre organizações.** O sistema atende duas pessoas
   jurídicas distintas, que não podem compartilhar dado nenhum.

## O que faz

- Lê as caixas por IMAP, em intervalos regulares, sem supervisão.
- Decide a categoria de cada mensagem nova entre onze classes
  operacionais, com regra de desempate explícita.
- Copia a mensagem para a pasta da categoria e sinaliza as urgentes, para
  o alerta aparecer no cliente de e-mail da equipe.
- Extrai o prazo mencionado no texto, quando houver.
- Mantém o estado de leitura sincronizado entre o original e a cópia, nos
  dois sentidos.
- Registra cada rodada em arquivo diário, com o motivo de cada decisão.

## Arquitetura

```
        caixas IMAP (varias, por instalacao)
                     |
                     v
        +------------------------------+
        |  coleta: so o que ainda nao  |
        |  tem a marca de processado   |
        +------------------------------+
                     |
                     v
   +-----------------------------------------+
   | 1. FILTRO DE SIGILO                     |
   |    processo sob segredo de justica      |--> pasta de triagem manual
   |    NUNCA sai do servidor                |    (registro discreto)
   +-----------------------------------------+
                     |
   +-----------------------------------------+
   | 2. FILTRO POR RESPONSAVEL               |
   |    caixa compartilhada entre socios     |--> ignorado, intocado
   +-----------------------------------------+
                     |
   +-----------------------------------------+
   | 3. REGRAS DETERMINISTICAS               |
   |    casos que o modelo erra e a regra    |--> categoria definida
   |    acerta; nao gastam cota de API       |
   +-----------------------------------------+
                     |
   +-----------------------------------------+
   | 4. MASCARA DE IDENTIFICADORES           |
   |    documentos, e-mails, telefones e     |
   |    inscricoes viram marcadores          |
   +-----------------------------------------+
                     |
   +-----------------------------------------+
   | 5. CLASSIFICACAO POR MODELO DE LINGUAGEM|
   |    cascata: principal -> repeticao ->   |
   |    modelo reserva, so em falha          |
   |    transitoria                          |
   +-----------------------------------------+
                     |
                     v
   +-----------------------------------------+
   | 6. ACAO NA CAIXA                        |
   |    COPIA para a pasta da categoria      |
   |    ORIGINAL fica na entrada, marcado    |
   |    nenhum EXPUNGE, nunca                |
   +-----------------------------------------+
                     |
                     v
          registro diario + sincronizacao
            de leitura bidirecional
```

### Decisões de projeto que merecem destaque

**O filtro de sigilo vem antes de tudo.** O único ponto em que o conteúdo
sai do controle da organização é a chamada ao modelo. O filtro roda antes
dela e é deliberadamente conservador: na dúvida, desvia para triagem
manual. Um falso positivo custa um minuto de trabalho humano; um falso
negativo custaria o sigilo.

**Regra determinística antes do modelo.** Certos avisos automáticos não
trazem o teor do ato no corpo da mensagem — só informam que algo foi
expedido. O modelo, vendo apenas isso, rebaixa a mensagem para uma
categoria inócua. Uma regra textual de três linhas acerta esses casos com
precisão maior que o modelo, custo zero e comportamento auditável.

**Cascata de modelos com classificação de erro.** Falhas transitórias do
provedor (sobrecarga, timeout) autorizam repetir e descer para um modelo
reserva. Falhas permanentes — chave inválida, modelo inexistente —
devolvem erro na hora, de propósito: se a reserva as cobrisse, uma
credencial revogada ficaria invisível no registro. O log identifica qual
modelo respondeu cada mensagem, o que torna mensurável a troca de modelo.

Essa cascata nasceu de um incidente real: o modelo principal passou a
recusar 20% a 30% das chamadas de um dia, por indisponibilidade do
provedor, com uso muito abaixo da cota contratada. A reserva recuperou
cerca de um terço dessas falhas.

**Nada é destruído.** O original nunca sai da caixa de entrada; o que vai
para a pasta da categoria é uma cópia. Nenhum comando de remoção
definitiva é emitido em momento algum. Quando a classificação falha, a
mensagem simplesmente não é marcada e volta na rodada seguinte — o efeito
de uma indisponibilidade é atraso, nunca perda.

**Isolamento por construção.** Cada organização tem a sua instalação, a
sua credencial e a sua tarefa agendada. Não há chave de configuração capaz
de misturar as duas: a separação é física. Uma suíte de testes compara as
duas cópias do programa e falha se qualquer constante divergir, porque o
erro mais provável na manutenção é corrigir um lado e esquecer o outro.

**Entrada hostil tratada como hostil.** O identificador de mensagem vem do
cabeçalho, isto é, de fora. Ele é validado antes de entrar em qualquer
comando do protocolo: quebras de linha, aspas e formatos fora do padrão
são rejeitados, porque um cabeçalho forjado seria injeção de comando.
Identificadores duplicados são registrados como ambíguos e não são
tocados.

## Tecnologias

| Camada | Escolha |
|---|---|
| Linguagem | Python 3, só a biblioteca padrão |
| Correio | IMAP sobre TLS |
| Classificação | API de modelo de linguagem, via HTTP |
| Configuração | arquivos de texto locais, fora do versionamento |
| Execução | agendador de tarefas do sistema operacional, a cada 15 min |
| Dependências externas | nenhuma |
| Banco de dados | nenhum; o estado vive nas marcas do próprio servidor |

A ausência de dependências é deliberada. O sistema roda numa estação de
trabalho comum, mantido por uma equipe pequena: cada biblioteca a mais
seria uma atualização a mais para administrar, e o protocolo IMAP já
oferece o armazenamento de estado necessário.

## Números

| | |
|---|---|
| Linhas de código | 1.044 |
| Linhas de teste | 373 |
| Casos de teste | 18, com 37 asserções |
| Categorias de classificação | 11, mais a de revisão manual |
| Instalações em produção | 2, independentes |
| Intervalo de execução | 15 minutos |
| Em produção desde | julho de 2026 |

Os testes não acessam caixa real, não usam rede e não chamam a API:
exercitam as funções puras contra um servidor IMAP simulado. Um deles mede
escala — a sincronização em lote faz 28 idas ao servidor onde a versão
ingênua fazia cerca de 1.200.

---

O código-fonte fica em repositório privado. Esta página descreve o projeto;
não acompanha código, credencial nem dado de produção.
