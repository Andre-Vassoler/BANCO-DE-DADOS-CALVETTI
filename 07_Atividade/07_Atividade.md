# T07 — Consultas Simples no MySQL (SELECT, WHERE, ORDER BY, LIMIT e Funções)

> Disciplina: Banco de Dados — USJT — Prof. Calvetti
> Base: material de aula "Consultas Simples no MySQL" (SELECT, projeção, filtros, ordenação, limites e funções)
> Ambiente de referência: MySQL 8.4 — trechos didáticos; valide sempre no ambiente da disciplina.

## Sumário

- [Caso condutor (esquema utilizado)](#caso-condutor-esquema-utilizado)
- [Exercícios Resolvidos (revisão)](#exercícios-resolvidos-revisão)
- [Exercício Proposto 1 — Projeção, aliases, expressões e DISTINCT](#exercício-proposto-1--projeção-aliases-expressões-e-distinct)
- [Exercício Proposto 2 — Filtros, padrões, intervalos e NULL](#exercício-proposto-2--filtros-padrões-intervalos-e-null)
- [Exercício Proposto 3 — Ordenação, limites e funções do MySQL](#exercício-proposto-3--ordenação-limites-e-funções-do-mysql)
- [Desafio Integrador — Seis consultas simples documentadas](#desafio-integrador--seis-consultas-simples-documentadas)
- [Síntese da T07](#síntese-da-t07)

---

## Caso condutor (esquema utilizado)

Todos os exercícios usam o mesmo cenário de uma loja acadêmica, já apresentado nas aulas anteriores (T05/T06):

```sql
cliente      (id_cliente, nome, email, ativo, telefone)
produto      (id_produto, nome, categoria, preco, estoque)
pedido       (id_pedido, criado_em, status, id_cliente)
item_pedido  (id_pedido, id_produto, quantidade, preco_praticado)
```

Perguntas de negócio associadas a cada tabela (conforme o material):

| Tabela | Pergunta típica |
|---|---|
| `cliente` | Quem está ativo? Quais domínios de e-mail aparecem? |
| `produto` | Quais produtos atendem ao orçamento? |
| `pedido` | Quais pedidos são recentes? |
| `item_pedido` | Quais linhas têm maior valor? |

---

## Exercícios Resolvidos (revisão)

**1. Quais categorias existem no catálogo?**
```sql
SELECT DISTINCT categoria
FROM produto
ORDER BY categoria;
```

**2. Quais categorias distintas têm produtos em estoque?**
```sql
SELECT DISTINCT categoria
FROM produto
WHERE estoque > 0
ORDER BY categoria;
```

**3. Quais produtos entre 20 e 100 contêm "SQL" no nome?**
```sql
SELECT nome, preco
FROM produto
WHERE preco BETWEEN 20 AND 100
  AND nome LIKE '%SQL%'
ORDER BY preco ASC, nome ASC;
```
*(`BETWEEN` inclui os dois limites: 20 e 100 entram no resultado.)*

**4. Quais são os cinco pedidos mais recentes e há quantos dias ocorreram?**
```sql
SELECT id_pedido, criado_em,
       DATEDIFF(CURDATE(), DATE(criado_em)) AS dias_decorridos
FROM pedido
ORDER BY criado_em DESC, id_pedido DESC
LIMIT 5;
```

**5. Clientes ativos sem telefone cadastrado**
```sql
SELECT id_cliente, nome, email
FROM cliente
WHERE ativo = TRUE
  AND telefone IS NULL
ORDER BY nome;
```
*(Nunca comparar com `telefone = NULL`; o correto é `IS NULL`.)*

---

## Exercício Proposto 1 — Projeção, aliases, expressões e DISTINCT

### a) Nome, preço e preço com desconto de 12%

```sql
SELECT nome,
       preco,
       preco * 0.88 AS preco_com_desconto
FROM produto;
```

**Interpretação:** a coluna `preco` permanece a original (para conferência); `preco_com_desconto` é uma coluna calculada, criada multiplicando o preço por `0.88` (ou seja, aplicando 12% de desconto: `100% - 12% = 88%`). Como o valor não existe fisicamente na tabela, ele **precisa de um alias** (`AS preco_com_desconto`) para ficar legível no resultado — sem o alias, a coluna apareceria com o nome da própria expressão.

### b) Combinações distintas de categoria e situação de estoque

Primeiro criamos uma expressão que classifica a "situação de estoque" (com/sem estoque) e depois aplicamos `DISTINCT` sobre o par categoria + situação:

```sql
SELECT DISTINCT
       categoria,
       CASE
           WHEN estoque > 0 THEN 'COM ESTOQUE'
           ELSE 'SEM ESTOQUE'
       END AS situacao_estoque
FROM produto
ORDER BY categoria, situacao_estoque;
```

> Observação: `CASE` é uma expressão condicional simples (não é agregação nem subconsulta), permitida no escopo da T07 — ela apenas transforma o valor de cada linha, como `UPPER()` ou `ROUND()` fariam.

**O que o `DISTINCT` considera duplicado aqui:** `DISTINCT` avalia a **combinação completa** das colunas listadas no `SELECT` (neste caso, o par `categoria` + `situacao_estoque`), e não cada coluna isoladamente. Ou seja, duas linhas só são tratadas como duplicadas — e uma delas descartada — se **todas** as colunas selecionadas tiverem o mesmo valor ao mesmo tempo. Por isso é perfeitamente possível ver a mesma categoria aparecer duas vezes no resultado (uma linha "COM ESTOQUE" e outra "SEM ESTOQUE"), porque o par completo é diferente em cada caso.

### c) Documentação de três linhas do resultado (pergunta / consulta / interpretação)

| # | Pergunta de negócio | Consulta (resumida) | Interpretação de uma linha do resultado |
|---|---|---|---|
| 1 | Qual seria o preço de um produto de R$ 50,00 com 12% de desconto? | item (a) | Linha `nome='Caderno Universitário', preco=50.00, preco_com_desconto=44.00` → o cliente pagaria R$ 44,00, uma redução de R$ 6,00 (12% de 50). |
| 2 | Existem categorias que possuem produtos totalmente em falta? | item (b) | Linha `categoria='PAPELARIA', situacao_estoque='SEM ESTOQUE'` → existe ao menos um produto de Papelaria com `estoque = 0`; a categoria não está "zerada" por completo, pois pode haver outra linha `('PAPELARIA','COM ESTOQUE')` também no resultado. |
| 3 | O desconto de 12% pode gerar valores com muitas casas decimais? | item (a) | Linha `preco=19.90, preco_com_desconto=17.512` → o resultado bruto tem 3 casas; para exibição ao cliente seria recomendável envolver a expressão em `ROUND(preco * 0.88, 2)`. |

---

## Exercício Proposto 2 — Filtros, padrões, intervalos e NULL

### a) Clientes ativos cujo nome começa com A ou B

```sql
SELECT id_cliente, nome, email
FROM cliente
WHERE ativo = TRUE
  AND (nome LIKE 'A%' OR nome LIKE 'B%')
ORDER BY nome;
```

*`LIKE 'A%'` localiza nomes que **começam** com "A" (o `%` representa zero ou mais caracteres após o "A"); o mesmo vale para `'B%'`. Os parênteses em `(nome LIKE 'A%' OR nome LIKE 'B%')` são obrigatórios aqui — sem eles, como `AND` tem precedência sobre `OR`, o MySQL avaliaria `(ativo = TRUE AND nome LIKE 'A%') OR (nome LIKE 'B%')`, trazendo também clientes **inativos** cujo nome começa com B.*

### b) Produtos de categorias selecionadas com preço entre 10 e 80

```sql
SELECT nome, categoria, preco
FROM produto
WHERE categoria IN ('LIVRO', 'PAPELARIA')
  AND preco BETWEEN 10 AND 80
ORDER BY categoria, preco;
```

*`IN ('LIVRO','PAPELARIA')` substitui `categoria = 'LIVRO' OR categoria = 'PAPELARIA'` de forma mais legível quando há uma lista explícita de valores. `BETWEEN 10 AND 80` inclui as duas extremidades (produtos de exatamente R$ 10,00 ou R$ 80,00 entram no resultado).*

### c) Clientes sem telefone e com e-mail informado

```sql
SELECT id_cliente, nome, email
FROM cliente
WHERE telefone IS NULL
  AND email IS NOT NULL
ORDER BY nome;
```

*Ausência de valor é testada sempre com `IS NULL` / `IS NOT NULL`, nunca com `=` ou `<>`, porque `NULL` representa "desconhecido", não um valor comparável.*

### d) Consulta combinando AND e OR (com parênteses)

**Pergunta:** *"Quais pedidos estão abertos ou pagos, e foram criados a partir de 01/08/2026?"*

```sql
SELECT id_pedido, status, criado_em, id_cliente
FROM pedido
WHERE (status = 'ABERTO' OR status = 'PAGO')
  AND criado_em >= '2026-08-01'
ORDER BY criado_em DESC;
```

*Como `AND` é avaliado antes de `OR` pela precedência lógica padrão, os parênteses em `(status = 'ABERTO' OR status = 'PAGO')` são essenciais para declarar a intenção correta: "(aberto OU pago) E recente" — sem eles, o resultado poderia incluir pedidos `'ABERTO'` de qualquer data, misturado com pedidos `'PAGO'` apenas recentes.*

### e) Provocando o erro conceitual `= NULL`

```sql
-- Consulta com o erro conceitual (NÃO retorna as linhas esperadas):
SELECT id_cliente, nome, telefone
FROM cliente
WHERE telefone = NULL;

-- Forma correta:
SELECT id_cliente, nome, telefone
FROM cliente
WHERE telefone IS NULL;
```

**Explicação do resultado:** a primeira consulta **sempre retorna um conjunto vazio**, mesmo que existam clientes com `telefone` nulo na tabela. Isso acontece porque, no MySQL (como em qualquer SGBD que siga a lógica de três valores do SQL), toda comparação que envolve `NULL` — inclusive `NULL = NULL` — produz o resultado lógico `UNKNOWN`, e não `TRUE` nem `FALSE`. Como a cláusula `WHERE` só mantém as linhas cuja condição é avaliada como `TRUE`, uma condição `UNKNOWN` é descartada, exatamente como se fosse `FALSE`. `NULL` representa "ausência ou desconhecimento de valor", e por definição não é igual (nem diferente) a nada — nem a si mesmo. Por isso existe o operador dedicado `IS NULL` (e `IS NOT NULL`), que testa diretamente a ausência de valor em vez de comparar valores.

| Condição | Resultado | Passa no WHERE? |
|---|---|---|
| `10 > 5` | TRUE | Sim |
| `10 < 5` | FALSE | Não |
| `NULL = 10` | UNKNOWN | Não |
| `NULL = NULL` | UNKNOWN | Não |
| `telefone IS NULL` | TRUE ou FALSE | Conforme o valor |

---

## Exercício Proposto 3 — Ordenação, limites e funções do MySQL

### a) Os dez produtos de maior valor em estoque (com desempate por nome)

```sql
SELECT nome,
       preco,
       estoque,
       preco * estoque AS valor_em_estoque
FROM produto
ORDER BY valor_em_estoque DESC, nome ASC
LIMIT 10;
```

**Justificativa da ordem escolhida:** o critério principal é `valor_em_estoque DESC` (preço × estoque), porque a pergunta de negócio é sobre o **valor monetário** imobilizado em cada produto, não apenas o preço unitário nem apenas a quantidade isoladamente. O `nome ASC` funciona como critério de desempate (segunda chave de ordenação): caso dois produtos tenham exatamente o mesmo `valor_em_estoque`, a ordem alfabética do nome garante um resultado determinístico e reproduzível — sem uma segunda chave, a ordem entre empates não é garantida pelo MySQL.

### b) Segunda página de resultados com LIMIT e OFFSET

```sql
-- Página 1 (produtos 1 a 10, ordenados por id):
SELECT id_produto, nome
FROM produto
ORDER BY id_produto
LIMIT 10;

-- Página 2 (produtos 11 a 20):
SELECT id_produto, nome
FROM produto
ORDER BY id_produto
LIMIT 10 OFFSET 10;
```

*`OFFSET 10` pula as 10 primeiras linhas do resultado já ordenado, e `LIMIT 10` devolve as 10 seguintes — por isso a segunda página começa no 11º produto. É indispensável manter o **mesmo `ORDER BY` determinístico** (aqui, `id_produto`, que é único) em todas as páginas; sem uma ordenação estável, `LIMIT`/`OFFSET` pode repetir ou omitir linhas entre uma página e outra, já que o SGBD não garante uma ordem implícita.*

### c) Formatação de data de pedido e cálculo de dias decorridos

```sql
SELECT id_pedido,
       DATE_FORMAT(criado_em, '%d/%m/%Y %H:%i') AS criado_em_formatado,
       DATEDIFF(CURDATE(), DATE(criado_em)) AS dias_decorridos,
       DATE_ADD(criado_em, INTERVAL 7 DAY) AS prazo_limite
FROM pedido
ORDER BY criado_em DESC
LIMIT 10;
```

*`DATE_FORMAT(criado_em, '%d/%m/%Y %H:%i')` converte o `DATETIME` armazenado para o formato brasileiro dia/mês/ano hora:minuto. `DATEDIFF(CURDATE(), DATE(criado_em))` calcula quantos dias já se passaram entre a data atual e a data do pedido (a função `DATE()` remove a parte de hora antes da subtração). `DATE_ADD(..., INTERVAL 7 DAY)` soma 7 dias à data do pedido, útil, por exemplo, para calcular um prazo de devolução ou de entrega.*

### d) CHAR_LENGTH x LENGTH com texto acentuado

```sql
SELECT nome,
       CHAR_LENGTH(nome) AS qtd_caracteres,
       LENGTH(nome)      AS qtd_bytes
FROM produto
WHERE nome LIKE '%ç%' OR nome LIKE '%ã%' OR nome LIKE '%é%'
ORDER BY nome;
```

**Comparação esperada:** para um texto sem acentuação, como `'Caderno'`, `CHAR_LENGTH` e `LENGTH` retornam o mesmo valor (7 e 7), pois cada caractere ocupa exatamente 1 byte em codificação UTF-8. Já para um texto acentuado, como `'Calculadora Científica'`, os valores divergem: `CHAR_LENGTH` conta **caracteres** (ex.: 23), enquanto `LENGTH` conta **bytes de armazenamento** (ex.: 24 ou mais), porque cada caractere acentuado (como "í") ocupa 2 bytes em UTF-8, embora continue sendo um único caractere visível. Ou seja: `CHAR_LENGTH` deve ser usado quando a regra de negócio fala em "quantidade de símbolos/caracteres" (por exemplo, limite de caracteres de um campo de nome), e `LENGTH` quando o assunto é armazenamento ou codificação.

---

## Desafio Integrador — Seis consultas simples documentadas

> Restrição do enunciado: **não usar JOIN, agregação (SUM/COUNT/AVG/...) nem subconsulta** — todas as consultas abaixo permanecem em uma única tabela, usando apenas SELECT / WHERE / ORDER BY / LIMIT / funções, preservando o escopo da T07.

### Consulta 1 — Projeção, aliases e expressão

**Pergunta:** *Qual o preço de cada produto e qual seria o preço com 15% de acréscimo (reajuste)?*

```sql
SELECT nome,
       preco AS preco_atual,
       ROUND(preco * 1.15, 2) AS preco_reajustado
FROM produto
ORDER BY nome;
```

**Resultado esperado:** uma linha por produto, com o preço atual e o preço reajustado arredondado em 2 casas decimais (ex.: `preco_atual = 45.00` → `preco_reajustado = 51.75`).

**Interpretação:** a coluna calculada usa `ROUND(..., 2)` para evitar dízimas na exibição do valor monetário; o alias `preco_reajustado` é obrigatório porque a expressão não corresponde a nenhuma coluna física da tabela.

**Teste de fronteira incluído:** um produto com `preco = 0` (se existir) resulta em `preco_reajustado = 0.00` — confirma que a expressão não gera erro nem divisão, apenas multiplica corretamente valores no limite inferior.

---

### Consulta 2 — DISTINCT sobre combinação de colunas

**Pergunta:** *Quais combinações de categoria e faixa de preço (até R$ 50 ou acima de R$ 50) existem no catálogo?*

```sql
SELECT DISTINCT
       categoria,
       CASE WHEN preco <= 50 THEN 'ATE_50' ELSE 'ACIMA_DE_50' END AS faixa_preco
FROM produto
ORDER BY categoria, faixa_preco;
```

**Resultado esperado:** uma linha para cada par único `(categoria, faixa_preco)` efetivamente presente na tabela — não necessariamente todas as combinações teóricas.

**Interpretação:** se a categoria `'PAPELARIA'` só tiver produtos até R$ 50, apenas o par `('PAPELARIA','ATE_50')` aparece; o par `('PAPELARIA','ACIMA_DE_50')` simplesmente não existe no resultado — isso ilustra que `DISTINCT` apenas remove duplicatas do que já existe, não gera combinações que não ocorrem nos dados.

---

### Consulta 3 — Filtro com padrão (LIKE) e lista (IN)

**Pergunta:** *Quais clientes ativos têm e-mail em domínios corporativos específicos (@usjt.br ou @gmail.com)?*

```sql
SELECT id_cliente, nome, email
FROM cliente
WHERE ativo = TRUE
  AND (email LIKE '%@usjt.br' OR email LIKE '%@gmail.com')
ORDER BY nome;
```

**Resultado esperado:** apenas clientes ativos cujo e-mail termina exatamente em um dos dois domínios informados.

**Interpretação:** `LIKE '%@usjt.br'` localiza qualquer texto que termine com esse domínio, independentemente do que vem antes do `@`; os parênteses garantem que o `OR` dos domínios não "escape" da condição de `ativo = TRUE`.

**Caso com ausência de dados incluído:** clientes com `email IS NULL` (se essa coluna permitir nulo) não aparecem em nenhum dos dois `LIKE`, pois qualquer comparação de padrão com `NULL` também resulta em `UNKNOWN` — coerente com a semântica de três valores do `WHERE`.

---

### Consulta 4 — Intervalo (BETWEEN) e tratamento de NULL

**Pergunta:** *Quais produtos custam entre R$ 15,00 e R$ 40,00 e não possuem categoria cadastrada?*

```sql
SELECT id_produto, nome, preco, categoria
FROM produto
WHERE preco BETWEEN 15 AND 40
  AND categoria IS NULL
ORDER BY preco;
```

**Resultado esperado:** conjunto provavelmente vazio no cenário normal da loja acadêmica (categoria costuma ser obrigatória), o que é, em si, uma resposta de negócio válida: "não há produtos nessa faixa de preço com categoria ausente".

**Interpretação:** esta consulta funciona como teste de fronteira e também como caso de ausência de dados — demonstra que `BETWEEN` (inclusive nos limites 15 e 40) pode ser combinado com `IS NULL` em um mesmo `WHERE`, e que um resultado vazio não é um erro, mas uma interpretação legítima ("nenhuma linha atende simultaneamente às duas condições").

---

### Consulta 5 — Ordenação com múltiplas chaves e LIMIT

**Pergunta:** *Quais são os cinco itens de pedido (linhas de `item_pedido`) de maior subtotal, desempatando por quantidade?*

```sql
SELECT id_pedido,
       id_produto,
       quantidade,
       preco_praticado,
       quantidade * preco_praticado AS subtotal
FROM item_pedido
ORDER BY subtotal DESC, quantidade DESC
LIMIT 5;
```

**Resultado esperado:** as cinco linhas de `item_pedido` com maior valor de `subtotal`; em caso de empate no subtotal, prevalece a linha com maior `quantidade`.

**Interpretação:** como `subtotal` é uma expressão calculada (não uma coluna física), ela só pode ser referenciada no `ORDER BY` porque o MySQL permite reutilizar o alias do `SELECT` nessa cláusula — diferente do `WHERE`, que não reconhece aliases da mesma consulta (ver "erro frequente" no Exercício Proposto 1).

**Teste de fronteira incluído:** uma linha com `quantidade = 1` e `preco_praticado` alto pode aparecer entre os cinco maiores subtotais mesmo tendo a menor quantidade possível — mostrando que o critério de ordenação é o valor total, não a quantidade isolada.

---

### Consulta 6 — Funções de data e paginação (LIMIT/OFFSET)

**Pergunta:** *Quais são os pedidos da "segunda página" (itens 6 a 10) quando ordenados do mais recente para o mais antigo, e há quantos dias cada um foi criado?*

```sql
SELECT id_pedido,
       DATE_FORMAT(criado_em, '%d/%m/%Y') AS data_pedido,
       DATEDIFF(CURDATE(), DATE(criado_em)) AS dias_decorridos
FROM pedido
ORDER BY criado_em DESC, id_pedido DESC
LIMIT 5 OFFSET 5;
```

**Resultado esperado:** exatamente os pedidos classificados na 2ª a 6ª posição (linhas 6 a 10, considerando `OFFSET 5` como "pular as 5 primeiras") da ordenação por data decrescente, com data formatada em `dd/mm/aaaa` e a quantidade de dias já passados desde a criação.

**Interpretação:** o uso de `id_pedido DESC` como segunda chave garante que, mesmo que dois pedidos tenham sido criados no mesmo instante (`criado_em` idêntico), a paginação seja **determinística** — ou seja, uma nova execução da consulta não troca a ordem das linhas nem "vaza" um pedido para a página errada, o que aconteceria se o `ORDER BY` não fosse totalmente estável.

---

## Síntese da T07
1. **SELECT** define quais colunas e expressões aparecem no resultado (incluindo colunas calculadas, sempre com alias).
2. **WHERE** mantém apenas as linhas cuja condição é avaliada como `TRUE`; qualquer comparação com `NULL` é `UNKNOWN` e é descartada — por isso `NULL` exige os operadores `IS NULL` / `IS NOT NULL`.
3. **DISTINCT** remove duplicatas considerando a combinação completa das colunas projetadas, não cada coluna isoladamente (diferente de `GROUP BY`, que ainda será visto em aulas futuras).
4. **ORDER BY** deve vir antes de `LIMIT` sempre que a posição das linhas tiver significado de negócio (ex.: "os 5 mais caros"), e precisa de critérios de desempate (chaves adicionais) para produzir uma paginação estável com `OFFSET`.
5. **Funções** (texto, numéricas e de data) transformam os valores exibidos, mas a consulta deve continuar respondendo exatamente à pergunta de negócio original — uma boa consulta é correta, legível e verificável.
