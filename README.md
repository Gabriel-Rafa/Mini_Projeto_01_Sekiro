# Mini-Projeto 01: Conselheiro Shinobi

Sistema baseado em regras para recomendar ações de combate em **Sekiro: Shadows Die Twice**.

**Autor:** Gabriel Rafá Martins Freire  
**Matrícula:** 20230145310  

## Descrição do domínio

O **Conselheiro Shinobi** é um Sistema Baseado em Conhecimento (SBC) que
recomenda **uma ação imediata de combate** em *Sekiro: Shadows Die Twice*.
O usuário informa uma situação; o motor Experta aplica regras IF–THEN,
produz fatos intermediários e explica a recomendação.

O recorte inclui estocadas, varreduras, agarres, ataques comuns e uso da
Cabaça Curativa. Mikiri é uma resposta a estocadas, saltar é uma resposta
a varreduras e defletir é uma resposta a ataques comuns. O modelo usa uma
estratégia conservadora de afastamento para agarres e estocadas sem Mikiri.
Essa última escolha é uma simplificação: estocadas também podem ser
defletidas com o tempo correto, e movimentos específicos têm exceções.

**Premissas do modelo:** um jogador vivo, um adversário e uma situação por
execução; o tipo do ataque é informado manualmente. `nenhum` significa que
não há ataque em curso, mas só `abertura_segura=True` informa tempo e espaço
para usar a cura. O limiar de 30% e as prioridades são escolhas deste
projeto, não valores oficiais do jogo. Há apenas uma ação imediata por
consulta; uma nova situação exige nova execução.

## Arquivos

- [`Mini_Projeto_01_Sekiro.ipynb`](Mini_Projeto_01_Sekiro.ipynb): implementação completa, explicações, testes e resultados de execução.
- `README.md`: documentação para o repositório GitHub.

Não há base de dados externa: os cenários sintéticos estão no notebook.

## Como executar no Google Colab

1. Acesse [Google Colab](https://colab.research.google.com/).
2. Abra o arquivo `Mini_Projeto_01_Sekiro.ipynb` pela opção de upload de notebook. Se já tiver publicado o repositório, também pode abri-lo pela opção GitHub.
3. Use um ambiente Python padrão; CPU é suficiente.
4. Execute todas as células, de cima para baixo. A primeira célula de código instala Experta.
5. Confira `TESTE 1: APROVADO`, `TESTE 2: APROVADO`, `TESTE 3: APROVADO` e o resumo dos nove cenários.
6. Na seção 11, altere `meu_cenario` para experimentar outra situação.

É necessária internet para a instalação inicial. O notebook não pede entradas interativas e não usa Drive, GPU ou arquivos auxiliares. Os resultados salvos podem ser lidos antes de executar.

### Execução local e compatibilidade

Em Jupyter ou VS Code, selecione um kernel Python e execute todas as células. A célula `%pip` instala o pacote no ambiente do kernel. Todas as células foram executadas em sequência com **IPython local**, incluindo a instalação e os testes, em **Python 3.12.14**, com **Experta 1.9.4** e **frozendict 1.2**. O notebook está preparado para Colab; a validação desta entrega não foi feita em uma sessão autenticada do Colab.

Experta usa uma dependência antiga que referencia `collections.Mapping`. O notebook cria esse alias a partir de `collections.abc.Mapping` quando necessário, antes da importação. Isso mantém a versão de `frozendict` exigida pelo pacote. Em um ambiente que já tenha carregado outra instalação de Experta, reinicie o kernel e execute desde a primeira célula.

## Dados de entrada

| Campo de `Combate` | Tipo / valores | Significado |
|---|---|---|
| `cenario` | texto não vazio | Identificador da consulta |
| `vida` | número finito: `0 < vida <= 100` | Porcentagem de vida do jogador |
| `ataque` | `estocada`, `varredura`, `agarre`, `normal`, `nenhum` | Movimento observado |
| `mikiri` | `True` ou `False` | Habilidade Mikiri desbloqueada |
| `curas` | inteiro maior ou igual a zero | Cargas restantes da Cabaça Curativa |
| `abertura_segura` | `True` ou `False` | Janela segura para se curar; só pode ser `True` com `ataque="nenhum"` |

Os campos são validados antes da declaração do fato. Uma entrada inválida gera `ValueError`, sem decisão de combate.

## Fatos e três níveis de encadeamento

| Etapa | Fato | Papel | Exemplo |
|---|---|---|---|
| Entrada | `Combate` | Dados informados | Estocada e Mikiri desbloqueado |
| Nível 1 | `Analise` | Interpretação do estado | R2 identifica ataque perigoso |
| Nível 2 | `Estrategia` | Ação candidata | R5 propõe Mikiri usando o fato de R2 |
| Nível 3 | `Recomendacao` | Decisão única | R12 seleciona a estratégia de R5 |

O caminho **R2 → R5 → R12** contém três disparos dependentes: R5 exige a
conclusão de R2, e R12 exige a conclusão de R5. Não são apenas três regras
independentes que consultam a entrada. A cura também encadeia os três níveis:
R1 e R4 produzem análises, R9 combina essas análises e R13 decide.

O campo `cenario` relaciona os fatos de uma consulta. O campo `cadeia` dos
fatos derivados armazena sua proveniência. O fato original não é alterado;
cada nível acrescenta uma conclusão com uma função distinta.

## Listagem das regras em linguagem natural

O roteiro pede pelo menos 10 regras. O projeto implementa **14**, todas com decoradores `@Rule`.

| Regra | Nível | `salience` | Regra em linguagem natural |
|---|---:|---:|---|
| R1 | 1 | 100 | SE a vida for menor ou igual a 30%, ENTÃO identificar vida baixa. |
| R2 | 1 | 99 | SE o ataque for estocada, varredura ou agarre, ENTÃO identificar ataque perigoso. |
| R3 | 1 | 98 | SE o ataque for normal, ENTÃO identificar ataque comum. |
| R4 | 1 | 97 | SE não houver ataque em curso, ENTÃO identificar ausência de ataque. |
| R5 | 2 | 90 | SE houver ataque perigoso do tipo estocada e Mikiri desbloqueado, ENTÃO propor MIKIRI como reação. |
| R6 | 2 | 89 | SE houver ataque perigoso do tipo varredura, ENTÃO propor SALTAR como reação. |
| R7 | 2 | 88 | SE houver ataque perigoso e ele for agarre, OU for estocada sem Mikiri, ENTÃO propor AFASTAR como reação. |
| R8 | 2 | 87 | SE houver ataque comum, ENTÃO propor DEFLETIR como reação. |
| R9 | 2 | 86 | SE houver vida baixa, ausência de ataque, abertura segura e ao menos uma carga de cura, ENTÃO propor CURAR. |
| R10 | 2 | 85 | SE houver vida baixa, ENTÃO propor BUSCAR_ABERTURA como alternativa de cautela. |
| R11 | 2 | 84 | SE houver ausência de ataque e NÃO houver vida baixa, ENTÃO propor OBSERVAR como cautela. |
| R12 | 3 | 30 | SE existir estratégia de reação e NÃO existir recomendação para o cenário, ENTÃO recomendar essa reação. |
| R13 | 3 | 20 | SE existir estratégia de cura e NÃO existir recomendação para o cenário, ENTÃO recomendar CURAR. |
| R14 | 3 | 10 | SE existir estratégia de cautela e NÃO existir recomendação para o cenário, ENTÃO recomendar essa cautela. |

R7 significa: ataque perigoso E (agarre OU estocada sem Mikiri). As análises e estratégias também possuem guardas `NOT` contra duplicação. No modelo, `R11` só avalia a ausência de vida baixa depois que as regras do nível 1 tiveram oportunidade de disparar.

## Resolução de conflitos

As regras de análise têm `salience` de 97 a 100; as de estratégia, de 84
a 90; as de decisão, 30, 20 e 10. Assim, a situação é analisada e todas as
estratégias aplicáveis são geradas antes da escolha final.

| Regra de decisão | Categoria | Prioridade |
|---|---|---:|
| R12 | Reagir a um ataque em curso | 30 |
| R13 | Curar em abertura segura | 20 |
| R14 | Cautela: buscar abertura ou observar | 10 |

Com vida baixa e uma estocada, R5 propõe Mikiri e R10 propõe buscar abertura.
R12 vence R14 por prioridade. Após a primeira recomendação ser declarada,
`NOT(Recomendacao(cenario=MATCH.c))` deixa de ser verdadeiro e bloqueia as
outras decisões. Com vida baixa e cura segura, R13 vence R14.

R10 gera uma alternativa mesmo quando há uma estratégia melhor. Isso é
intencional: torna a competição entre regras observável. R10 e R11 não
concorrem entre si, pois exigem condições opostas de vida baixa.

O sistema imprime as candidatas e suas prioridades, o trace completo e a
cadeia causal da vencedora. O trace inclui tudo que disparou; a explicação
causal inclui somente os fatos que sustentam a decisão, além do motivo da
prioridade. Não há ordenação de ações em Python para escolher a vencedora:
a escolha é feita pelos decoradores `@Rule` do Experta.

## Trace e explicabilidade

Cada disparo registra identificador da regra, nível, `salience`, motivo, conclusão e cadeia causal. `AS` recupera os fatos antecedentes e o método `registrar()` propaga sua proveniência. A recomendação contém a frase “foi escolhida porque ... dispararam”, seguida dos motivos e da prioridade.

No caso 1, o trace é **R1, R2, R5, R10, R12**, enquanto a cadeia causal de Mikiri é **R2 → R5 → R12**. R1 e R10 explicam a alternativa de cautela, não a escolha de Mikiri. Essa distinção permite verificar o caminho efetivo da decisão.

O método `concluir()` apresenta as candidatas depois que o Experta escolhe a regra final. Ele não implementa uma seleção paralela com `if/else`, ordenação ou pontuação externa.

## Três casos de teste principais

| Caso | Vida | Ataque | Mikiri | Curas | Abertura segura | Saída esperada | Cadeia causal |
|---|---:|---|---|---:|---|---|---|
| C1 | 20% | estocada | Sim | 2 | Não | `MIKIRI` | R2 → R5 → R12 |
| C2 | 25% | nenhum | Não | 2 | Sim | `CURAR` | R1 e R4 → R9 → R13 |
| C3 | 30% | nenhum | Não | 0 | Não | `BUSCAR_ABERTURA` | R1 → R10 → R14 |

**C1:** testa a competição entre reação e cautela. R12 tem prioridade 30 e vence R14, que tem 10. A cura não é elegível durante o ataque.

**C2:** testa a combinação de análises e a competição entre cura e cautela. R13 tem prioridade 20 e vence R14. A presença da recomendação impede a segunda decisão.

**C3:** testa o limite inclusivo de 30% e a ausência de recurso. R9 não pode recomendar cura; R11 não pode classificar a situação como observação com vida fora da faixa baixa.

As saídas esperadas estão comentadas antes dos testes no notebook. Os `assert` comparam a ação final, a cadeia causal e a sequência completa de regras. Os conflitos dos dois primeiros casos também são verificados.

### Casos complementares

| Caso | Situação | Saída esperada |
|---|---|---|
| C4 | Vida 80%, varredura | `SALTAR` |
| C5 | Vida 80%, agarre | `AFASTAR` |
| C6 | Vida 80%, estocada sem Mikiri | `AFASTAR` |
| C7 | Vida 80%, ataque normal | `DEFLETIR` |
| C8 | Vida 30,1%, nenhum ataque | `OBSERVAR` |
| C9 | Vida 20%, uma cura, nenhum ataque, sem abertura segura | `BUSCAR_ABERTURA` |

**Resultado da verificação:** nove cenários aprovados e todas as 14 regras exercitadas; uma decisão por cenário; três níveis na proveniência de cada decisão; nenhum disparo repetido. Quatro entradas inválidas também são rejeitadas corretamente. Isso valida os cenários definidos, sem medir desempenho em partidas reais.

## Exemplo de consulta

Depois de executar as definições do notebook:

```python
resultado = executar_cenario(dict(
    cenario="Meu treino",
    vida=65,
    ataque="varredura",
    mikiri=True,
    curas=1,
    abertura_segura=False,
))
```

Saída esperada: `SALTAR`, sustentada por **R2 → R6 → R12**. Os fatos ficam em `resultado["motor"].facts`; o histórico, em `resultado["trace"]`; e a explicação, em `resultado["decisao"]["explicacao"]`.

