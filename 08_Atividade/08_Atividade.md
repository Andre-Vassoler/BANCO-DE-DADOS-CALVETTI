# T08 — Junções e Agregações no MySQL (INNER JOIN, OUTER JOIN, GROUP BY e HAVING)

> Disciplina: Banco de Dados — USJT — Prof. Calvetti
> Base: material de aula "Junções e agregações — Relacionando tabelas e produzindo indicadores no MySQL"
> Ambiente de referência: MySQL 8.4 — trechos didáticos; valide sempre no ambiente da disciplina.

## Sumário

- [Caso condutor (esquema utilizado)](#caso-condutor-esquema-utilizado)
- [Conceitos-chave da aula](#conceitos-chave-da-aula)
- [Exercícios Resolvidos (revisão)](#exercícios-resolvidos-revisão)
- [Exercício Proposto 1 — INNER JOIN e autorrelacionamento](#exercício-proposto-1--inner-join-e-autorrelacionamento)
- [Exercício Proposto 2 — Junções externas e ausência de correspondência](#exercício-proposto-2--junções-externas-e-ausência-de-correspondência)
- [Exercício Proposto 3 — Agregações, GROUP BY e HAVING](#exercício-proposto-3--agregações-group-by-e-having)
- [Desafio Integrador — Oito consultas multitabelas e indicadores](#desafio-integrador--oito-consultas-multitabelas-e-indicadores)
- [Síntese da T08](#síntese-da-t08)

---

## Caso condutor (esquema utilizado)

A loja acadêmica das aulas anteriores (T05–T07) conecta clientes, pedidos, itens e produtos:

| Tabela | Chave principal | Referências |
|---|---|---|
| `cliente` | `id_cliente` | — |
| `pedido` | `id_pedido` | `id_cliente` → `cliente` |
| `item_pedido` | `id_pedido` + `id_produto` | `pedido` e `produto` |
| `produto` | `id_produto` | — |

```sql
cliente      (id_cliente, nome, email, ativo, telefone)
pedido       (id_pedido, criado_em, status, id_cliente)
item_pedido  (id_pedido, id_produto, quantidade, preco_praticado)
produto      (id_produto, nome, categoria, preco, estoque)
```

Para o autorrelacionamento, a aula usa uma tabela `empregado` com chave estrangeira recursiva (`id_supervisor` aponta para a própria tabela). Uma definição mínima compatível com os exemplos:

```sql
CREATE TABLE empregado (
    id_empregado  INT          NOT NULL AUTO_INCREMENT,
    nome          VARCHAR(100) NOT NULL,
    id_supervisor INT          NULL,          -- NULL = topo da hierarquia
    CONSTRAINT pk_empregado PRIMARY KEY (id_empregado),
    CONSTRAINT fk_empregado_supervisor
        FOREIGN KEY (id_supervisor) REFERENCES empregado (id_empregado)
);
```

Aliases de tabela adotados em todo o documento (mesma convenção do material):

| Forma | Leitura | Uso |
|---|---|---|
| `cliente AS c` | c representa CLIENTE | `c.nome` |
| `pedido AS p` | p representa PEDIDO | `p.status` |
| `item_pedido AS i` | i representa ITEM_PEDIDO | `i.quantidade` |
| `produto AS pr` | pr representa PRODUTO | `pr.preco` |
| `empregado AS e` / `empregado AS s` | empregado / supervisor | `e.nome`, `s.nome` |

---

## Conceitos-chave da aula

**Produto cartesiano.** Sem condição de junção, cada linha de uma tabela é combinada com todas as linhas da outra (`n × m` pares). `FROM cliente AS c, pedido AS p` sem `ON`/`WHERE` é sintaticamente válido, mas semanticamente errado: muitas linhas não comprovam correção.

**INNER JOIN.** Mantém somente os pares que satisfazem a condição `ON`. Um cliente com vários pedidos aparece em várias linhas; um cliente sem pedidos não aparece.

**ON × WHERE.** `ON` estabelece o vínculo entre as tabelas; `WHERE` filtra o resultado já relacionado. Em junções externas essa diferença muda o resultado (ver Exercício Proposto 2).

**Qualificação.** Colunas que existem em mais de uma tabela (ex.: `id_cliente`) devem ser prefixadas pelo alias, senão o MySQL acusa `Column 'id_cliente' in field list is ambiguous`.

**ON, USING e NATURAL JOIN.** `ON a.id = b.id_a` aceita nomes diferentes; `USING (id_cliente)` compacta nomes iguais, mas oculta de qual lado veio a coluna; `NATURAL JOIN` infere colunas homônimas e é frágil quando o schema evolui (uma coluna nova com nome repetido muda silenciosamente a junção).

**LEFT / RIGHT JOIN.** Preservam todas as linhas da tabela à esquerda / direita; quando não há par, as colunas do outro lado recebem `NULL`. `A RIGHT JOIN B` equivale a `B LEFT JOIN A`.

**Anti-join.** `LEFT JOIN ... WHERE lado_direito.chave IS NULL` localiza linhas sem correspondência.

**FULL OUTER JOIN.** O MySQL não oferece o operador nativo; simula-se com `LEFT JOIN` + `UNION ALL` + `RIGHT JOIN` filtrado (`WHERE a.id IS NULL`) para não repetir os pares.

**Funções agregadas.**

| Função | Pergunta | NULL |
|---|---|---|
| `COUNT` | Quantas linhas ou valores? | `COUNT(coluna)` ignora NULL; `COUNT(*)` conta linhas |
| `SUM` | Qual o total? | Ignora NULL |
| `AVG` | Qual a média? | Ignora NULL |
| `MIN` | Qual o menor valor? | Ignora NULL |
| `MAX` | Qual o maior valor? | Ignora NULL |

**GROUP BY / HAVING / ONLY_FULL_GROUP_BY.** `GROUP BY` cria uma unidade de análise para cada chave distinta; colunas não agregadas no `SELECT` devem identificar o grupo (o modo `ONLY_FULL_GROUP_BY`, padrão no MySQL 8, rejeita `SELECT categoria, nome, COUNT(*) ... GROUP BY categoria`). `HAVING` filtra grupos depois do cálculo das agregações.

| Cláusula | Atua sobre | Agregação |
|---|---|---|
| `WHERE` | Linhas antes dos grupos | Não |
| `GROUP BY` | Linhas aprovadas | Forma grupos |
| `HAVING` | Grupos calculados | Sim |
| `ORDER BY` | Resultado final | Sim |

**Armadilha do COUNT com LEFT JOIN.**

| Expressão | Cliente sem pedido | Interpretação |
|---|---|---|
| `COUNT(*)` | 1 | Conta a linha preservada pelo LEFT JOIN (errado!) |
| `COUNT(p.id_pedido)` | 0 | Conta somente pedidos reais |
| `COUNT(DISTINCT p.id_pedido)` | 0 | Evita repetição quando há junções posteriores (ex.: itens) |

---

## Exercícios Resolvidos (revisão)

Já vêm resolvidos no material; reproduzo-os como referência rápida.

**1. Quais produtos aparecem em cada pedido?**
```sql
SELECT i.id_pedido, pr.nome AS produto,
       i.quantidade,
       i.quantidade * i.preco_praticado AS subtotal
FROM item_pedido AS i
JOIN produto AS pr ON pr.id_produto = i.id_produto
ORDER BY i.id_pedido, pr.nome;
```

**2. Quais produtos nunca apareceram em itens de pedido?** (anti-join)
```sql
SELECT pr.id_produto, pr.nome
FROM produto AS pr
LEFT JOIN item_pedido AS i
  ON i.id_produto = pr.id_produto
WHERE i.id_produto IS NULL
ORDER BY pr.nome;
```

**3. Quais clientes têm ao menos três pedidos pagos?**
```sql
SELECT c.id_cliente, c.nome,
       COUNT(p.id_pedido) AS pedidos_pagos
FROM cliente AS c
JOIN pedido AS p ON p.id_cliente = c.id_cliente
WHERE p.status = 'PAGO'
GROUP BY c.id_cliente, c.nome
HAVING COUNT(p.id_pedido) >= 3;
```

| Critério | Cláusula | Justificativa |
|---|---|---|
| `status = 'PAGO'` | WHERE | Decide quais pedidos entram |
| cliente | GROUP BY | Define uma linha por cliente |
| `COUNT >= 3` | HAVING | Depende do total calculado |
| nome | SELECT | Descreve o grupo |

**Exemplo Resolvido 1 — clientes com ou sem pedidos**
```sql
SELECT c.nome, COUNT(p.id_pedido) AS total_pedidos
FROM cliente AS c
LEFT JOIN pedido AS p ON p.id_cliente = c.id_cliente
GROUP BY c.id_cliente, c.nome
ORDER BY total_pedidos DESC, c.nome;
```

**Exemplo Resolvido 2 — WHERE ou HAVING: produtos disponíveis em categorias relevantes**
```sql
SELECT categoria, COUNT(*) AS quantidade,
       AVG(preco) AS preco_medio
FROM produto
WHERE estoque > 0
GROUP BY categoria
HAVING COUNT(*) >= 2
ORDER BY preco_medio DESC;
```

---

## Exercício Proposto 1 — INNER JOIN e autorrelacionamento

### a) Pedidos com nome do cliente e status

```sql
SELECT p.id_pedido,
       p.criado_em,
       c.nome   AS cliente,
       p.status
FROM pedido AS p
INNER JOIN cliente AS c
    ON c.id_cliente = p.id_cliente
ORDER BY p.criado_em DESC, p.id_pedido;
```

**Interpretação:** cada linha do resultado é um pedido (unidade de análise = pedido). Como todo pedido tem exatamente um cliente (FK obrigatória), o número de linhas é igual ao número de pedidos — se aparecer um número maior, há algo errado na condição `ON`.

### b) Produtos e subtotais de cada item

```sql
SELECT i.id_pedido,
       pr.nome                          AS produto,
       i.quantidade,
       i.preco_praticado,
       i.quantidade * i.preco_praticado AS subtotal
FROM item_pedido AS i
INNER JOIN produto AS pr
    ON pr.id_produto = i.id_produto
ORDER BY i.id_pedido, pr.nome;
```

**Interpretação:** uma linha por item de pedido. Usamos `i.preco_praticado` (preço no momento da venda) e não `pr.preco` (preço atual de catálogo), pois o subtotal precisa refletir o valor efetivamente cobrado.

### c) Caminho PEDIDO → ITEM_PEDIDO → PRODUTO

```sql
SELECT p.id_pedido,
       p.status,
       c.nome                           AS cliente,
       pr.nome                          AS produto,
       i.quantidade,
       i.quantidade * i.preco_praticado AS subtotal
FROM pedido AS p
INNER JOIN cliente     AS c  ON c.id_cliente  = p.id_cliente
INNER JOIN item_pedido AS i  ON i.id_pedido   = p.id_pedido
INNER JOIN produto     AS pr ON pr.id_produto = i.id_produto
ORDER BY p.id_pedido, pr.nome;
```

**Interpretação:** `ITEM_PEDIDO` é a tabela associativa que resolve o relacionamento N:N entre `PEDIDO` e `PRODUTO`; por isso o caminho obrigatoriamente passa por ela. Cada nova tabela recebe a sua própria condição de junção. A unidade de análise passa a ser **item de pedido** — um pedido com 3 produtos aparece em 3 linhas. Pedidos sem nenhum item somem do resultado, porque `INNER JOIN` só mantém pares correspondentes.

### d) Empregados e respectivos supervisores (autorrelacionamento)

```sql
SELECT e.id_empregado,
       e.nome AS empregado,
       s.nome AS supervisor
FROM empregado AS e
INNER JOIN empregado AS s
    ON s.id_empregado = e.id_supervisor
ORDER BY s.nome, e.nome;
```

**Interpretação:** a mesma tabela participa duas vezes com papéis diferentes — `e` é a ocorrência supervisionada e `s` é a ocorrência que exerce a supervisão. Os aliases distintos são obrigatórios, pois sem eles o MySQL não saberia a qual "cópia" cada coluna pertence. Com `INNER JOIN`, quem não tem supervisor (`id_supervisor IS NULL`, o topo da hierarquia) não aparece; para incluí-lo, troca-se por `LEFT JOIN` (ver Desafio, consulta 5).

### e) Por que cada condição ON preserva a semântica

| Junção | Condição ON | Por que está correta |
|---|---|---|
| pedido × cliente | `c.id_cliente = p.id_cliente` | Liga a FK `pedido.id_cliente` à PK de `cliente`: "este pedido foi feito por este cliente". |
| item × pedido | `i.id_pedido = p.id_pedido` | Liga cada item ao pedido ao qual pertence (parte da PK composta de `item_pedido`). |
| item × produto | `pr.id_produto = i.id_produto` | Liga cada item ao produto vendido (outra parte da PK composta). |
| empregado × empregado | `s.id_empregado = e.id_supervisor` | Liga a FK recursiva do empregado à PK de quem o supervisiona. A ordem importa: `e.id_empregado = s.id_supervisor` inverteria os papéis. |

Em todos os casos o `ON` compara **chave estrangeira com a chave primária que ela referencia**. Uma condição ausente ou incompleta geraria produto cartesiano (linhas demais); uma condição entre colunas erradas (ex.: `p.id_pedido = c.id_cliente`) executaria sem erro, mas produziria pares sem significado no domínio.

---

## Exercício Proposto 2 — Junções externas e ausência de correspondência

### a) Todos os clientes, inclusive os que não possuem pedidos

```sql
SELECT c.id_cliente,
       c.nome,
       p.id_pedido,
       p.status
FROM cliente AS c
LEFT JOIN pedido AS p
    ON p.id_cliente = c.id_cliente
ORDER BY c.nome, p.id_pedido;
```

**Interpretação:** `cliente` está à esquerda, então nenhum cliente pode desaparecer. Clientes com vários pedidos aparecem em várias linhas; clientes sem pedidos aparecem uma vez, com `p.id_pedido` e `p.status` = `NULL`.

### b) Produtos nunca vendidos

```sql
SELECT pr.id_produto,
       pr.nome,
       pr.categoria,
       pr.estoque
FROM produto AS pr
LEFT JOIN item_pedido AS i
    ON i.id_produto = pr.id_produto
WHERE i.id_produto IS NULL
ORDER BY pr.categoria, pr.nome;
```

**Interpretação:** padrão **anti-join**. O `LEFT JOIN` preserva todos os produtos; o `WHERE i.id_produto IS NULL` mantém apenas os que não encontraram nenhum item correspondente — ou seja, nunca apareceram em um pedido. O teste deve ser feito em uma coluna do lado direito que **nunca** é nula quando há par (de preferência a própria chave da junção).

### c) RIGHT JOIN reescrito como LEFT JOIN

```sql
-- Versão com RIGHT JOIN (preserva PRODUTO, que está à direita):
SELECT pr.nome, i.id_pedido, i.quantidade
FROM item_pedido AS i
RIGHT JOIN produto AS pr
    ON pr.id_produto = i.id_produto;

-- Versão equivalente com LEFT JOIN (basta trocar os lados):
SELECT pr.nome, i.id_pedido, i.quantidade
FROM produto AS pr
LEFT JOIN item_pedido AS i
    ON i.id_produto = pr.id_produto;
```

**Interpretação:** `ITEM RIGHT JOIN PRODUTO` preserva `PRODUTO`; a forma equivalente é `PRODUTO LEFT JOIN ITEM`. Os dois retornam exatamente o mesmo conjunto de linhas. Por convenção, prefere-se o `LEFT JOIN`, porque a tabela cujas linhas não podem sumir fica escrita primeiro, o que facilita a leitura.

### d) Simulação de junção externa completa (FULL OUTER JOIN) em duas tabelas

```sql
-- Parte 1: todos os clientes, com ou sem pedido (inclui os pares)
SELECT c.id_cliente, c.nome, p.id_pedido, p.status
FROM cliente AS c
LEFT JOIN pedido AS p
    ON p.id_cliente = c.id_cliente

UNION ALL

-- Parte 2: somente pedidos que NÃO encontraram cliente
SELECT c.id_cliente, c.nome, p.id_pedido, p.status
FROM cliente AS c
RIGHT JOIN pedido AS p
    ON p.id_cliente = c.id_cliente
WHERE c.id_cliente IS NULL;
```

| Parte | Preserva | Como evita repetição |
|---|---|---|
| `LEFT JOIN` | Todas as linhas da esquerda (clientes) | Já inclui todos os pares |
| `RIGHT JOIN` + `WHERE c.id_cliente IS NULL` | Somente as linhas da direita sem par | O filtro descarta os pares que a parte 1 já trouxe |
| `UNION ALL` | Combina os dois resultados | Não elimina duplicatas legítimas (ao contrário de `UNION`) |

**Interpretação:** o MySQL não tem `FULL OUTER JOIN` nativo. O resultado final contém (1) pares cliente–pedido, (2) clientes sem pedido e (3) pedidos sem cliente. No nosso esquema a parte 2 deve retornar **vazio**, porque `pedido.id_cliente` é FK obrigatória — e esse resultado vazio é justamente a confirmação de que a integridade referencial está funcionando. Usa-se `UNION ALL` (e não `UNION`) porque o filtro já garante que as partes não se sobrepõem; `UNION` faria um trabalho extra de deduplicação e poderia descartar linhas idênticas legítimas.

### e) Por que um filtro no WHERE pode eliminar linhas preservadas

```sql
-- ERRADO para a pergunta "todos os clientes e seus pedidos pagos":
SELECT c.nome, p.id_pedido
FROM cliente AS c
LEFT JOIN pedido AS p
    ON p.id_cliente = c.id_cliente
WHERE p.status = 'PAGO';

-- CORRETO: o filtro sobre a tabela da direita vai no ON
SELECT c.nome, p.id_pedido
FROM cliente AS c
LEFT JOIN pedido AS p
    ON p.id_cliente = c.id_cliente
   AND p.status = 'PAGO';
```

**Explicação:** o `LEFT JOIN` é executado primeiro e preserva os clientes sem pedido pago, preenchendo as colunas de `pedido` com `NULL`. Em seguida, o `WHERE` avalia `p.status = 'PAGO'` linha a linha; para as linhas preservadas, isso vira `NULL = 'PAGO'`, cujo resultado é `UNKNOWN` (lógica de três valores vista na T07), e o `WHERE` só mantém condições `TRUE`. Assim, as linhas preservadas são descartadas e o `LEFT JOIN` passa a se comportar como um `INNER JOIN`. Colocando a condição no `ON`, ela participa da **decisão de quais pares formar**: o cliente continua preservado e simplesmente fica sem par quando não tem pedido pago. Regra prática: filtros sobre a tabela **preservada** (esquerda) podem ir no `WHERE`; filtros sobre a tabela **opcional** (direita) devem ir no `ON`, exceto quando o objetivo é justamente testar ausência com `IS NULL`.

---

## Exercício Proposto 3 — Agregações, GROUP BY e HAVING

### a) Quantidade, menor, maior e média de preços por categoria

```sql
SELECT categoria,
       COUNT(*)            AS qtd_produtos,
       MIN(preco)          AS menor_preco,
       MAX(preco)          AS maior_preco,
       ROUND(AVG(preco),2) AS preco_medio
FROM produto
GROUP BY categoria
ORDER BY categoria;
```

**Interpretação:** uma linha por categoria. `categoria` é a única coluna não agregada e identifica o grupo, então a consulta é válida em `ONLY_FULL_GROUP_BY`. `ROUND(AVG(preco), 2)` evita médias com muitas casas decimais.

### b) Pedidos por cliente, incluindo zero

```sql
SELECT c.id_cliente,
       c.nome,
       COUNT(p.id_pedido) AS total_pedidos
FROM cliente AS c
LEFT JOIN pedido AS p
    ON p.id_cliente = c.id_cliente
GROUP BY c.id_cliente, c.nome
ORDER BY total_pedidos DESC, c.nome;
```

**Interpretação:** o `LEFT JOIN` garante que clientes sem pedido existam no resultado, e `COUNT(p.id_pedido)` — e **não** `COUNT(*)` — garante que eles apareçam com **0**. Com `COUNT(*)` o cliente sem pedido apareceria com 1, porque a linha preservada pelo `LEFT JOIN` seria contada.

### c) Apenas clientes com pelo menos dois pedidos

```sql
SELECT c.id_cliente,
       c.nome,
       COUNT(p.id_pedido) AS total_pedidos
FROM cliente AS c
INNER JOIN pedido AS p
    ON p.id_cliente = c.id_cliente
GROUP BY c.id_cliente, c.nome
HAVING COUNT(p.id_pedido) >= 2
ORDER BY total_pedidos DESC, c.nome;
```

**Interpretação:** aqui `INNER JOIN` é suficiente, pois clientes sem pedido nunca atenderiam a `>= 2`. A condição vai no `HAVING` porque depende de um valor calculado por grupo; colocá-la no `WHERE` geraria erro (`Invalid use of group function`), já que o `WHERE` atua antes da formação dos grupos. *(O MySQL também aceita `HAVING total_pedidos >= 2`, usando o alias.)*

### d) Valor total por pedido usando ITEM_PEDIDO

```sql
SELECT p.id_pedido,
       p.status,
       COUNT(i.id_produto)                             AS qtd_itens,
       COALESCE(SUM(i.quantidade * i.preco_praticado), 0) AS valor_total
FROM pedido AS p
LEFT JOIN item_pedido AS i
    ON i.id_pedido = p.id_pedido
GROUP BY p.id_pedido, p.status
ORDER BY valor_total DESC, p.id_pedido;
```

**Interpretação:** o valor do pedido não é armazenado — ele é derivado da soma dos subtotais dos itens, usando o preço praticado. Usei `LEFT JOIN` para que um eventual pedido ainda sem itens apareça; nesse caso `SUM` sobre nenhum valor devolve `NULL`, e `COALESCE(..., 0)` o converte em 0, que é o significado correto de negócio ("pedido sem valor"). Se a pergunta fosse só sobre pedidos com itens, um `INNER JOIN` sem `COALESCE` bastaria.

### e) Justificativa de cada uso de WHERE, GROUP BY e HAVING

| Consulta | WHERE | GROUP BY | HAVING |
|---|---|---|---|
| (a) | Não usado: todas as linhas de produto participam. | `categoria` — a pergunta pede um indicador por categoria. | Não usado: nenhum grupo é descartado. |
| (b) | Não usado: um filtro sobre `p` no WHERE eliminaria os clientes com zero. | `c.id_cliente, c.nome` — uma linha por cliente; `id_cliente` garante unicidade mesmo com nomes repetidos, e `nome` descreve o grupo. | Não usado. |
| (c) | Não usado (poderia filtrar linhas antes, ex.: `p.status = 'PAGO'`). | Por cliente. | `COUNT(p.id_pedido) >= 2` — critério sobre o total calculado, só disponível depois do agrupamento. |
| (d) | Não usado: todo item compõe o valor. | `p.id_pedido, p.status` — unidade de análise = pedido. | Não usado (poderia ser `HAVING valor_total > 100` para pedidos acima de um valor). |

Resumo do raciocínio: **WHERE** decide quais *linhas* entram, **GROUP BY** define a *unidade de análise* e **HAVING** decide quais *grupos calculados* saem.

---

## Desafio Integrador — Oito consultas multitabelas e indicadores

> Para cada consulta: pergunta, SQL, expectativa e interpretação. Todas as colunas estão qualificadas com alias de tabela.

### Consulta 1 — INNER JOIN com qualificação completa

**Pergunta:** Quais pedidos pagos foram feitos por clientes ativos, e por quem?

```sql
SELECT p.id_pedido,
       DATE_FORMAT(p.criado_em, '%d/%m/%Y') AS data_pedido,
       c.id_cliente,
       c.nome                               AS cliente,
       c.email
FROM pedido AS p
INNER JOIN cliente AS c
    ON c.id_cliente = p.id_cliente
WHERE p.status = 'PAGO'
  AND c.ativo = TRUE
ORDER BY p.criado_em DESC, p.id_pedido;
```

**Expectativa:** uma linha por pedido pago cujo cliente está ativo; nunca mais linhas do que pedidos existentes.
**Interpretação:** `ON` estabelece o vínculo pedido–cliente; `WHERE` aplica os dois filtros de negócio sobre o resultado já relacionado. Como é `INNER JOIN`, filtrar no `WHERE` é seguro (não há linhas preservadas a perder).

---

### Consulta 2 — INNER JOIN em três tabelas (caminho N:N)

**Pergunta:** Quais produtos da categoria LIVRO foram vendidos, em quais pedidos e com qual subtotal?

```sql
SELECT p.id_pedido,
       c.nome                           AS cliente,
       pr.nome                          AS produto,
       i.quantidade,
       i.preco_praticado,
       i.quantidade * i.preco_praticado AS subtotal
FROM item_pedido AS i
INNER JOIN pedido  AS p  ON p.id_pedido   = i.id_pedido
INNER JOIN cliente AS c  ON c.id_cliente  = p.id_cliente
INNER JOIN produto AS pr ON pr.id_produto = i.id_produto
WHERE pr.categoria = 'LIVRO'
ORDER BY p.id_pedido, pr.nome;
```

**Expectativa:** uma linha por item vendido de livro; o mesmo pedido pode aparecer várias vezes se tiver vários livros.
**Interpretação:** a unidade de análise é o **item**. Cada tabela adicionada tem sua própria condição `ON` baseada em FK → PK, o que impede combinações indevidas.

---

### Consulta 3 — Junção externa: todos os produtos e suas vendas

**Pergunta:** Para cada produto do catálogo, em quais pedidos ele foi vendido (mostrando também os que nunca venderam)?

```sql
SELECT pr.id_produto,
       pr.nome       AS produto,
       pr.categoria,
       i.id_pedido,
       i.quantidade
FROM produto AS pr
LEFT JOIN item_pedido AS i
    ON i.id_produto = pr.id_produto
ORDER BY pr.nome, i.id_pedido;
```

**Expectativa:** todos os produtos aparecem; os nunca vendidos aparecem uma vez, com `id_pedido` e `quantidade` nulos.
**Interpretação:** `produto` está à esquerda porque é a tabela cujas linhas não podem desaparecer. O `NULL` no lado direito não significa "quantidade zero" registrada, e sim **ausência de correspondência**.

---

### Consulta 4 — Junção externa com busca por ausência (anti-join)

**Pergunta:** Quais clientes ativos nunca fizeram nenhum pedido (candidatos a uma campanha de primeira compra)?

```sql
SELECT c.id_cliente,
       c.nome,
       c.email
FROM cliente AS c
LEFT JOIN pedido AS p
    ON p.id_cliente = c.id_cliente
WHERE p.id_pedido IS NULL
  AND c.ativo = TRUE
ORDER BY c.nome;
```

**Expectativa:** somente clientes ativos sem nenhum pedido; resultado vazio significa que todo cliente ativo já comprou.
**Interpretação:** `p.id_pedido IS NULL` está no `WHERE` de propósito: aqui queremos justamente as linhas preservadas sem par. Já `c.ativo = TRUE` filtra a tabela preservada, então também pode ficar no `WHERE` sem distorcer o resultado.

---

### Consulta 5 — Autorrelacionamento (incluindo o topo da hierarquia)

**Pergunta:** Qual o supervisor de cada empregado, incluindo quem não tem supervisor?

```sql
SELECT e.id_empregado,
       e.nome                                 AS empregado,
       COALESCE(s.nome, '(sem supervisor)')   AS supervisor
FROM empregado AS e
LEFT JOIN empregado AS s
    ON s.id_empregado = e.id_supervisor
ORDER BY supervisor, e.nome;
```

**Expectativa:** uma linha por empregado (mesmo total de linhas da tabela `empregado`); o topo da hierarquia aparece com "(sem supervisor)".
**Interpretação:** `e` representa o papel "supervisionado" e `s` o papel "supervisor". O `LEFT JOIN` preserva os empregados com `id_supervisor IS NULL`, que um `INNER JOIN` eliminaria. `COALESCE` apenas melhora a apresentação do `NULL`.

---

### Consulta 6 — Agregação com LEFT JOIN e contagem correta de zero

**Pergunta:** Quantos pedidos cada cliente fez e qual a data do último pedido (incluindo quem nunca comprou)?

```sql
SELECT c.id_cliente,
       c.nome,
       COUNT(p.id_pedido) AS total_pedidos,
       MAX(p.criado_em)   AS ultimo_pedido
FROM cliente AS c
LEFT JOIN pedido AS p
    ON p.id_cliente = c.id_cliente
GROUP BY c.id_cliente, c.nome
ORDER BY total_pedidos DESC, c.nome;
```

**Expectativa:** uma linha por cliente; clientes sem pedido aparecem com `total_pedidos = 0` e `ultimo_pedido = NULL`.
**Interpretação:** `COUNT(p.id_pedido)` ignora o `NULL` da linha preservada e devolve 0; `COUNT(*)` devolveria 1 (armadilha do COUNT). `MAX` também ignora NULL, por isso devolve `NULL` quando não há pedido.

---

### Consulta 7 — Agregação com LEFT JOIN em três tabelas sem inflar a contagem

**Pergunta:** Para cada cliente, quantos pedidos, quantos itens e qual o valor total comprado (incluindo zero)?

```sql
SELECT c.id_cliente,
       c.nome,
       COUNT(DISTINCT p.id_pedido)                          AS total_pedidos,
       COUNT(i.id_produto)                                  AS total_itens,
       COALESCE(SUM(i.quantidade * i.preco_praticado), 0)   AS valor_total
FROM cliente AS c
LEFT JOIN pedido      AS p ON p.id_cliente = c.id_cliente
LEFT JOIN item_pedido AS i ON i.id_pedido  = p.id_pedido
GROUP BY c.id_cliente, c.nome
ORDER BY valor_total DESC, c.nome;
```

**Expectativa:** uma linha por cliente; clientes sem pedido aparecem com 0 pedidos, 0 itens e valor 0.
**Interpretação:** depois da segunda junção, um pedido com 3 itens gera 3 linhas. Se usássemos `COUNT(p.id_pedido)`, esse pedido seria contado 3 vezes; `COUNT(DISTINCT p.id_pedido)` evita essa repetição posterior. A segunda junção também precisa ser `LEFT`: um `INNER JOIN` com `item_pedido` eliminaria novamente os clientes sem pedido.

---

### Consulta 8 — Combinação de WHERE e HAVING

**Pergunta:** Quais categorias tiveram faturamento acima de R$ 500,00 em pedidos pagos, e com quantos pedidos distintos?

```sql
SELECT pr.categoria,
       COUNT(DISTINCT p.id_pedido)              AS pedidos_pagos,
       SUM(i.quantidade)                        AS unidades_vendidas,
       SUM(i.quantidade * i.preco_praticado)    AS faturamento
FROM item_pedido AS i
INNER JOIN pedido  AS p  ON p.id_pedido   = i.id_pedido
INNER JOIN produto AS pr ON pr.id_produto = i.id_produto
WHERE p.status = 'PAGO'
GROUP BY pr.categoria
HAVING SUM(i.quantidade * i.preco_praticado) > 500
ORDER BY faturamento DESC;
```

**Expectativa:** somente categorias cujo faturamento pago passa de R$ 500; categorias abaixo disso (ou sem venda paga) não aparecem.
**Interpretação:**

| Critério | Cláusula | Justificativa |
|---|---|---|
| `p.status = 'PAGO'` | WHERE | Decide quais **linhas** (itens de pedidos pagos) entram no cálculo |
| `pr.categoria` | GROUP BY | Define uma linha por categoria |
| `SUM(...) > 500` | HAVING | Depende do total calculado por grupo |
| `faturamento DESC` | ORDER BY | Ordena o resultado final pelo indicador |

Se o filtro de status fosse para o `HAVING`, ele não faria sentido (status não é agregado nem identifica o grupo); se o filtro de faturamento fosse para o `WHERE`, o MySQL rejeitaria a consulta, pois a soma ainda não existe nesse momento.

---

## Síntese da T08

1. **Escolher tabelas:** a pergunta indica quais relações precisam participar (quais dados → em quais tabelas → como se relacionam).
2. **Declarar vínculos:** toda junção precisa de uma condição `ON` que ligue FK à PK referenciada; sem ela surge o produto cartesiano. Colunas repetidas devem ser qualificadas.
3. **Preservar ocorrências:** "somente pares?" → `INNER JOIN`; "todos da esquerda?" → `LEFT JOIN`; "ausências?" → `LEFT JOIN ... IS NULL`. Filtros sobre a tabela opcional vão no `ON`, não no `WHERE`.
4. **Definir grupos:** `GROUP BY` estabelece a unidade de análise; colunas não agregadas devem identificar o grupo; `WHERE` filtra linhas antes, `HAVING` filtra grupos depois.
5. **Interpretar indicadores:** usar `COUNT(coluna)` para contar zero corretamente após `LEFT JOIN` e `COUNT(DISTINCT ...)` quando junções posteriores multiplicam linhas.

> Uma consulta multitabela correta preserva **vínculos**, **cardinalidade** e **unidade de análise**.
