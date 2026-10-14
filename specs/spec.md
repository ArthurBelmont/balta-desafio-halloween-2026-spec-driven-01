# Gerador de Senhas Fortes

> Esta especificação descreve **o quê** e **por quê**. Nenhuma decisão de tecnologia entra aqui; o **como** está em `plan.md`.
> Projeto lúdico de aprendizado de Spec-Driven Development. Não usar em produção.

## Problema

Criar senhas realmente fortes à mão é trabalhoso e propenso a erro: as pessoas tendem a escolher senhas curtas, previsíveis ou com espaços e padrões fracos. Além disso, quem gera uma senha precisa de uma forma simples de recuperá-la depois, sem depender de memorizá-la ou anotá-la em outro lugar.

## Objetivo

Oferecer um serviço que:

1. gera senhas fortes que seguem sempre o mesmo padrão de qualidade;
2. guarda cada senha gerada e devolve um identificador único para consultá-la depois.

## Usuários

- **Consumidor do serviço:** pessoa ou sistema que precisa de uma senha forte e quer poder recuperá-la mais tarde usando apenas o identificador recebido.

## Histórias

- **H-01** Como consumidor, quero gerar uma senha forte para não ter que inventar uma.
- **H-02** Como consumidor, quero receber um identificador junto com a senha para conseguir recuperá-la depois.
- **H-03** Como consumidor, quero consultar uma senha a partir do seu identificador para reutilizá-la quando precisar.
- **H-04** Como consumidor, quero ser informado de forma clara quando o pedido for inválido ou a senha não existir, para saber o que corrigir.

## Requisitos funcionais

- **RF-01** O sistema deve permitir gerar uma nova senha forte.
- **RF-02** O sistema deve permitir que o consumidor informe, opcionalmente, o tamanho desejado da senha; quando não informado, usa o tamanho mínimo.
- **RF-03** Toda senha gerada deve ser armazenada de forma persistente, de modo que continue disponível após o serviço ser reiniciado.
- **RF-04** Ao gerar uma senha, o sistema deve devolver a senha e o identificador único (GUID) associado a ela.
- **RF-05** O sistema deve permitir consultar uma senha previamente gerada a partir do seu identificador.
- **RF-06** Ao consultar, o sistema deve devolver a senha associada ao identificador informado.
- **RF-07** Quando o pedido for inválido, o sistema deve recusá-lo com uma explicação do problema que identifique qual campo ou regra falhou.
- **RF-08** Quando o identificador consultado não corresponder a nenhuma senha, o sistema deve informar que ela não foi encontrada.

## Regras de negócio

- **RN-01** Toda senha gerada tem **no mínimo 16 caracteres**.
- **RN-02** Toda senha gerada contém **ao menos um caractere especial**.
- **RN-03** Nenhuma senha gerada contém **espaços em branco** (espaço, tabulação ou quebra de linha).
- **RN-04** Toda senha gerada contém ao menos uma letra maiúscula, uma letra minúscula e ao menos um número (dígito de 0 a 9), além do caractere especial.
- **RN-05** Cada senha gerada recebe um identificador único, que nunca se repete entre senhas.
- **RN-06** Uma senha armazenada não é alterada nem removida pelo serviço.
- **RN-07** Gerar duas senhas em pedidos diferentes deve produzir senhas diferentes, com identificadores diferentes.
- **RN-08** O tamanho máximo aceito para uma senha é **128 caracteres**.
- **RN-09** As senhas geradas não podem ser previsíveis: a escolha de cada caractere deve usar uma fonte de aleatoriedade adequada para uso em segurança.
- **RN-10** Os caracteres especiais usados são apenas os do conjunto `! @ # $ % ^ & * - _ + = ?`. Aspas, barras, crases, sinais de maior/menor, chaves, colchetes, parênteses, vírgula, ponto, dois-pontos e ponto e vírgula nunca são usados.

## Casos de borda

- **B-01** Tamanho solicitado menor que 16: o pedido é recusado (RN-01).
- **B-02** Tamanho solicitado maior que 128: o pedido é recusado (RN-08).
- **B-03** Tamanho solicitado que não é um número inteiro (texto, decimal, negativo, zero): o pedido é recusado.
- **B-04** Tamanho exatamente 16 e exatamente 128: são aceitos.
- **B-05** Identificador consultado em formato inválido (não é um GUID): o pedido é recusado como inválido, e não como "não encontrado".
- **B-06** Identificador em formato válido, mas inexistente: informa que a senha não foi encontrada.
- **B-07** Identificador vazio ou ausente: o pedido é recusado como inválido.
- **B-08** Senha de tamanho mínimo (16) ainda deve conter todos os tipos de caractere exigidos (RN-02 e RN-04).
- **B-09** Consulta repetida ao mesmo identificador devolve sempre a mesma senha.
- **B-10** Caracteres repetidos, inclusive em sequência (ex.: `aaaa`), são permitidos na senha.

## Fora de escopo

- Criptografia ou proteção da senha armazenada.
- Autenticação, autorização ou controle de quem pode consultar uma senha.
- Alteração, expiração ou exclusão de senhas.
- Listagem de todas as senhas armazenadas.
- Personalização do conjunto de caracteres ou das regras de composição.
- Uso em produção.

## Critérios de aceite

- **CA-01** Dado um pedido de geração sem tamanho, a senha devolvida tem pelo menos 16 caracteres, contém ao menos um caractere especial, uma maiúscula, uma minúscula e um número, e não contém espaços em branco (RF-01, RN-01 a RN-04).
- **CA-02** Dado um pedido de geração com tamanho entre 16 e 128, a senha devolvida tem exatamente esse tamanho (RF-02, B-04).
- **CA-03** Dado um pedido de geração com tamanho fora do intervalo ou inválido, o pedido é recusado com explicação que identifica o campo inválido (RF-07, B-01 a B-03).
- **CA-04** Dada uma senha recém-gerada, a resposta traz o identificador único junto com a senha (RF-04, RN-05).
- **CA-05** Dado o identificador de uma senha gerada, a consulta devolve exatamente a mesma senha, inclusive após o serviço ser reiniciado e em consultas repetidas (RF-03, RF-05, RF-06, B-09).
- **CA-06** Dado um identificador válido mas inexistente, a consulta informa que a senha não foi encontrada (RF-08, B-06).
- **CA-07** Dado um identificador em formato inválido ou vazio, a consulta é recusada como pedido inválido (RF-07, B-05, B-07).
- **CA-08** Dados dois pedidos de geração consecutivos, as senhas e os identificadores devolvidos são diferentes (RN-05, RN-07).
- **CA-09** Dada qualquer senha gerada, seus caracteres especiais pertencem apenas ao conjunto permitido (RN-10).
- **CA-10** Dadas muitas senhas geradas em sequência, nenhuma se repete e a distribuição dos caracteres não apresenta padrão previsível (RN-07, RN-09).
