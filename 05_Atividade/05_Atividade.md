# T05 — Modelo Físico e Integridade no MySQL

**Disciplina:** Banco de Dados · **Prof. Calvetti**
**Tema:** Schemas, tipos, constraints, integridade referencial e anomalias

> Respostas aos **4 exercícios resolvidos** (conferência) e aos **5 exercícios propostos + desafio integrador** do slide "Exercícios resolvidos e propostos". Scripts validados para a sintaxe do MySQL 8.4.

---

## Sumário

- [Exercício Resolvido 1 — Tipos (ALUNO)](#exercício-resolvido-1--tipos-aluno)
- [Exercício Resolvido 2 — Constraints (MATRICULA)](#exercício-resolvido-2--constraints-matricula)
- [Exercício Resolvido 3 — Ações referenciais (PEDIDO / ITEM_PEDIDO)](#exercício-resolvido-3--ações-referenciais-pedido--item_pedido)
- [Exercício Resolvido 4 — Anomalias (PEDIDO_COMPLETO)](#exercício-resolvido-4--anomalias-pedido_completo)
- [Exercício Proposto 1 — Reconhecimento (sistema acadêmico)](#exercício-proposto-1--reconhecimento-sistema-acadêmico)
- [Exercício Proposto 2 — Aplicação (RESERVA)](#exercício-proposto-2--aplicação-reserva)
- [Exercício Proposto 3 — Decisão (ON DELETE / ON UPDATE)](#exercício-proposto-3--decisão-on-delete--on-update)
- [Exercício Proposto 4 — Teste (violações de constraints)](#exercício-proposto-4--teste-violações-de-constraints)
- [Exercício Proposto 5 — Análise (VENDA não normalizada)](#exercício-proposto-5--análise-venda-não-normalizada)
- [Desafio Integrador — Modelo físico de uma clínica](#desafio-integrador--modelo-físico-de-uma-clínica)
- [Autoavaliação — Recuperação ativa](#autoavaliação--recuperação-ativa)

---

## Exercício Resolvido 1 — Tipos (ALUNO)

**Enunciado:** Selecione tipos para ALUNO e justifique cada decisão.

| Atributo | Solução | Justificativa |
|---|---|---|
| id_aluno | `INT AUTO_INCREMENT` | Identidade interna (chave substituta) |
| matricula | `CHAR(10)` | Código fixo; preserva zeros à esquerda |
| nome | `VARCHAR(120)` | Texto curto variável |
| email | `VARCHAR(254)` | Identificador alternativo |
| data_nascimento | `DATE` | Data sem horário |
| bolsista | `BOOLEAN` | Estado lógico |

*(Exercício já resolvido no material — reproduzido aqui apenas para referência de conferência.)*

---

## Exercício Resolvido 2 — Constraints (MATRICULA)

**Enunciado:** Transforme as regras de MATRÍCULA em constraints físicas.

```sql
CREATE TABLE matricula (
  id_matricula INT NOT NULL AUTO_INCREMENT,
  id_aluno     INT NOT NULL,
  id_turma     INT NOT NULL,
  situacao     VARCHAR(15) NOT NULL DEFAULT 'ATIVA',
  PRIMARY KEY (id_matricula),
  UNIQUE (id_aluno, id_turma),
  CHECK (situacao IN ('ATIVA','TRANCADA','CONCLUÍDA')),
  FOREIGN KEY (id_aluno) REFERENCES aluno(id_aluno),
  FOREIGN KEY (id_turma) REFERENCES turma(id_turma)
) ENGINE=InnoDB;
```

*(Já resolvido no material.)*

---

## Exercício Resolvido 3 — Ações referenciais (PEDIDO / ITEM_PEDIDO)

**Enunciado:** Justifique políticas diferentes para PEDIDO e ITEM_PEDIDO.

| Vínculo | Política | Raciocínio |
|---|---|---|
| CLIENTE → PEDIDO | `ON DELETE RESTRICT` | Pedido é histórico; não deve desaparecer com o cliente |
| PEDIDO → ITEM_PEDIDO | `ON DELETE CASCADE` | Item não possui sentido sem o pedido |
| PRODUTO → ITEM_PEDIDO | `ON DELETE RESTRICT` | Produto vendido participa do histórico |
| PKs substitutas | `ON UPDATE RESTRICT` | Identificadores devem permanecer estáveis |

*(Já resolvido no material.)*

---

## Exercício Resolvido 4 — Anomalias (PEDIDO_COMPLETO)

**Enunciado:** Diagnostique a tabela PEDIDO_COMPLETO e proponha a decomposição.

| Sintoma | Anomalia | Correção |
|---|---|---|
| nome_cliente repetido em cada item | Atualização | Manter nome somente em CLIENTE |
| produto só existe após uma venda | Inserção | Criar PRODUTO independente |
| excluir último item apaga produto | Exclusão | Separar PRODUTO de ITEM_PEDIDO |
| preço praticado muda com preço atual | Histórica | Guardar `preco_praticado` no vínculo |

*(Já resolvido no material.)*

---

## Exercício Proposto 1 — Reconhecimento (sistema acadêmico)

**Enunciado:**
- Modele `ALUNO(id, matrícula, nome, e-mail, nascimento, coeficiente, bolsista, observações)`.
- Escolha um tipo MySQL para cada atributo.
- Defina comprimentos e precisão com justificativas.
- Identifique códigos que não devem ser tratados como números.
- Diferencie dado ausente, inaplicável e obrigatório.

### Solução

| Atributo | Tipo escolhido | Justificativa |
|---|---|---|
| id_aluno | `INT AUTO_INCREMENT` | Identificador substituto interno; não carrega significado de negócio |
| matricula | `CHAR(10) NOT NULL` | Código de tamanho fixo; comprimento invariável definido pela secretaria acadêmica; **não é número**, é um rótulo — pode conter zeros à esquerda e não participa de cálculo |
| nome | `VARCHAR(120) NOT NULL` | Texto curto de comprimento variável |
| email | `VARCHAR(254) NOT NULL` | Identificador alternativo; 254 é o comprimento máximo prático de um endereço de e-mail |
| data_nascimento | `DATE NOT NULL` | Representa uma data civil, sem necessidade de horário |
| coeficiente | `DECIMAL(4,2) NOT NULL DEFAULT 0.00` | Coeficiente de rendimento acadêmico (CR) é um valor decimal exato, faixa 0.00–10.00; `DECIMAL` evita erros de arredondamento binário que `FLOAT`/`DOUBLE` introduziriam |
| bolsista | `BOOLEAN NOT NULL DEFAULT FALSE` | Estado lógico (verdadeiro/falso) |
| observacoes | `TEXT NULL` | Conteúdo livre e opcional, de tamanho imprevisível |

```sql
CREATE TABLE aluno (
  id_aluno         INT NOT NULL AUTO_INCREMENT,
  matricula        CHAR(10) NOT NULL,
  nome             VARCHAR(120) NOT NULL,
  email            VARCHAR(254) NOT NULL,
  data_nascimento  DATE NOT NULL,
  coeficiente      DECIMAL(4,2) NOT NULL DEFAULT 0.00,
  bolsista         BOOLEAN NOT NULL DEFAULT FALSE,
  observacoes      TEXT NULL,
  CONSTRAINT pk_aluno PRIMARY KEY (id_aluno),
  CONSTRAINT uq_aluno_matricula UNIQUE (matricula),
  CONSTRAINT uq_aluno_email UNIQUE (email),
  CONSTRAINT ck_aluno_coeficiente CHECK (coeficiente BETWEEN 0.00 AND 10.00)
) ENGINE=InnoDB;
```

**Códigos que não devem ser tratados como números:** `matricula` é o caso central — apesar de conter apenas dígitos, ela não é somada, multiplicada nem tem média calculada; por isso recebe `CHAR`, não `INT`. O mesmo raciocínio vale para CPF, CEP e telefone (discutidos no material): dígitos que identificam, não que quantificam.

**Dado ausente × inaplicável × obrigatório:**
- **Obrigatório** — a coluna deve sempre conter um valor válido para que a linha faça sentido no domínio (`matricula`, `nome`, `email`, `data_nascimento`): recebem `NOT NULL`.
- **Ausente** — o valor existe no domínio, mas ainda não foi informado (ex.: `observacoes` de um aluno recém-matriculado). Nesse caso a coluna é `NULL`-ável, porque a ausência é temporária ou opcional, não uma regra de negócio.
- **Inaplicável** — o atributo simplesmente não se aplica àquela linha (ex.: um campo "data de conclusão de bolsa" não faz sentido para `bolsista = FALSE`). Diferente do "ausente", aqui não é que falte informar — é que a pergunta não cabe para aquele registro. Fisicamente, ambos os casos usam `NULL`, mas a **semântica** é diferente e deve ser documentada, pois confundi-las leva a interpretações erradas em relatórios (ex.: tratar "inaplicável" como "esquecido de preencher").

---

## Exercício Proposto 2 — Aplicação (RESERVA)

**Enunciado:**
- Use `id_reserva, id_sala, id_responsavel, inicio, fim, situacao`.
- Defina PK, FKs, `NOT NULL` e `DEFAULT`.
- Imponha `CHECK` para fim posterior a início.
- Controle o vocabulário de `situacao`.
- Explique por que `UNIQUE(id_sala, inicio)` não detecta toda sobreposição temporal.

### Solução

```sql
CREATE TABLE reserva (
  id_reserva      INT NOT NULL AUTO_INCREMENT,
  id_sala         INT NOT NULL,
  id_responsavel  INT NOT NULL,
  inicio          DATETIME NOT NULL,
  fim             DATETIME NOT NULL,
  situacao        VARCHAR(20) NOT NULL DEFAULT 'CONFIRMADA',
  CONSTRAINT pk_reserva PRIMARY KEY (id_reserva),
  CONSTRAINT ck_reserva_periodo CHECK (fim > inicio),
  CONSTRAINT ck_reserva_situacao CHECK (situacao IN ('CONFIRMADA','CANCELADA','CONCLUIDA')),
  CONSTRAINT fk_reserva_sala
    FOREIGN KEY (id_sala) REFERENCES sala(id_sala)
    ON DELETE RESTRICT ON UPDATE RESTRICT,
  CONSTRAINT fk_reserva_responsavel
    FOREIGN KEY (id_responsavel) REFERENCES responsavel(id_responsavel)
    ON DELETE RESTRICT ON UPDATE RESTRICT
) ENGINE=InnoDB;
```

**Por que `UNIQUE (id_sala, inicio)` não basta:**
Essa restrição só impede que **duas reservas comecem exatamente no mesmo instante** na mesma sala. Ela não enxerga o **intervalo** entre `inicio` e `fim`. Duas reservas com horários de início diferentes ainda podem se sobrepor — por exemplo, sala 101 reservada das 10:00 às 12:00 e, em outra linha, das 11:00 às 13:00: os valores de `(id_sala, inicio)` são distintos (`10:00` ≠ `11:00`), então a `UNIQUE` aceita as duas, mas há uma hora de conflito real entre elas. Detectar sobreposição de intervalos exige lógica que compare `inicio`/`fim` par a par (uma condição do tipo `nova.inicio < existente.fim AND nova.fim > existente.inicio`), o que uma `UNIQUE` simples não expressa — é necessário um `TRIGGER`, uma verificação na aplicação, ou (em bancos que suportam) uma *exclusion constraint*, recurso que o MySQL não oferece nativamente.

---

## Exercício Proposto 3 — Decisão (ON DELETE / ON UPDATE)

**Enunciado:** Escolha `ON DELETE` e `ON UPDATE` para três vínculos, justificando pela obrigatoriedade, história e ciclo de vida.

| Vínculo | Pergunta-chave | ON DELETE | ON UPDATE | Justificativa |
|---|---|---|---|---|
| DEPARTAMENTO → EMPREGADO | O empregado pode permanecer sem departamento? | `SET NULL` | `RESTRICT` | Um departamento pode ser extinto sem que os empregados sejam demitidos; o vínculo é **opcional**, então a FK deve aceitar `NULL`. Já a alteração da PK do departamento é evento raro e deve ser bloqueada para preservar estabilidade dos identificadores |
| PROJETO → TAREFA | A tarefa possui existência independente? | `CASCADE` | `RESTRICT` | Uma tarefa não tem sentido fora do projeto que a originou — é parte composicional dele. Excluir o projeto deve excluir suas tarefas. A PK do projeto, porém, não deve ser alterada livremente |
| AUTOR → LIVRO_AUTOR | O contrato histórico pode ser apagado? | `RESTRICT` | `RESTRICT` | `LIVRO_AUTOR` registra um fato histórico (autoria de uma obra publicada); apagar o autor não pode apagar silenciosamente esse registro — a exclusão deve ser bloqueada até que o histórico seja tratado conscientemente |

```sql
-- DEPARTAMENTO → EMPREGADO
CONSTRAINT fk_empregado_departamento
  FOREIGN KEY (id_departamento) REFERENCES departamento(id_departamento)
  ON DELETE SET NULL ON UPDATE RESTRICT,

-- PROJETO → TAREFA
CONSTRAINT fk_tarefa_projeto
  FOREIGN KEY (id_projeto) REFERENCES projeto(id_projeto)
  ON DELETE CASCADE ON UPDATE RESTRICT,

-- AUTOR → LIVRO_AUTOR
CONSTRAINT fk_livroautor_autor
  FOREIGN KEY (id_autor) REFERENCES autor(id_autor)
  ON DELETE RESTRICT ON UPDATE RESTRICT
```

> Observação: para `ON DELETE SET NULL` funcionar, `id_departamento` em `EMPREGADO` precisa ser declarado **sem** `NOT NULL`.

---

## Exercício Proposto 4 — Teste (violações de constraints)

**Enunciado:** Projete violações que demonstrem a eficácia das constraints, usando o esquema `cliente` / `pedido` / `item_pedido` / `produto` construído ao longo da aula.

```sql
-- Esquema de referência (retomado do material da aula)
CREATE TABLE cliente (
  id_cliente INT NOT NULL AUTO_INCREMENT,
  nome       VARCHAR(120) NOT NULL,
  email      VARCHAR(254) NOT NULL,
  ativo      BOOLEAN NOT NULL DEFAULT TRUE,
  CONSTRAINT pk_cliente PRIMARY KEY (id_cliente),
  CONSTRAINT uq_cliente_email UNIQUE (email)
) ENGINE=InnoDB;

CREATE TABLE produto (
  id_produto   INT NOT NULL AUTO_INCREMENT,
  nome         VARCHAR(160) NOT NULL,
  preco_atual  DECIMAL(10,2) NOT NULL,
  CONSTRAINT pk_produto PRIMARY KEY (id_produto),
  CONSTRAINT ck_produto_preco CHECK (preco_atual >= 0)
) ENGINE=InnoDB;

CREATE TABLE pedido (
  id_pedido  INT NOT NULL AUTO_INCREMENT,
  id_cliente INT NOT NULL,
  status     VARCHAR(20) NOT NULL DEFAULT 'ABERTO',
  CONSTRAINT pk_pedido PRIMARY KEY (id_pedido),
  CONSTRAINT ck_pedido_status CHECK (status IN ('ABERTO','PAGO','CANCELADO')),
  CONSTRAINT fk_pedido_cliente FOREIGN KEY (id_cliente) REFERENCES cliente(id_cliente)
    ON DELETE RESTRICT ON UPDATE RESTRICT
) ENGINE=InnoDB;

CREATE TABLE item_pedido (
  id_pedido        INT NOT NULL,
  id_produto       INT NOT NULL,
  quantidade       INT NOT NULL,
  preco_praticado  DECIMAL(10,2) NOT NULL,
  CONSTRAINT pk_item PRIMARY KEY (id_pedido, id_produto),
  CONSTRAINT ck_item_qtd CHECK (quantidade > 0),
  CONSTRAINT fk_item_pedido  FOREIGN KEY (id_pedido)  REFERENCES pedido(id_pedido)  ON DELETE CASCADE,
  CONSTRAINT fk_item_produto FOREIGN KEY (id_produto) REFERENCES produto(id_produto) ON DELETE RESTRICT
) ENGINE=InnoDB;
```

### Bateria de testes

```sql
-- 1) INSERT que viola PRIMARY KEY
INSERT INTO cliente (id_cliente, nome, email) VALUES (1, 'Ana', 'ana@mail.com');
INSERT INTO cliente (id_cliente, nome, email) VALUES (1, 'Bia', 'bia@mail.com');
-- Resultado esperado: ERRO 1062 (Duplicate entry '1' for key 'cliente.PRIMARY')

-- 2) INSERT que viola UNIQUE
INSERT INTO cliente (nome, email) VALUES ('Carlos', 'ana@mail.com');
-- Resultado esperado: ERRO 1062 (Duplicate entry 'ana@mail.com' for key 'cliente.uq_cliente_email')

-- 3) INSERT que viola CHECK
INSERT INTO produto (nome, preco_atual) VALUES ('Caneta', -5.00);
-- Resultado esperado: ERRO 3819 (Check constraint 'ck_produto_preco' is violated)

-- 4) INSERT com FK órfã
INSERT INTO pedido (id_cliente) VALUES (999); -- id_cliente 999 não existe
-- Resultado esperado: ERRO 1452 (Cannot add or update a child row: a foreign key constraint fails)

-- 5) DELETE bloqueado por RESTRICT
INSERT INTO pedido (id_cliente) VALUES (1);
DELETE FROM cliente WHERE id_cliente = 1;
-- Resultado esperado: ERRO 1451 (Cannot delete or update a parent row: a foreign key constraint fails)
-- porque existe pedido dependente e a FK usa ON DELETE RESTRICT

-- 6) DELETE propagado por CASCADE
INSERT INTO produto (nome, preco_atual) VALUES ('Caderno', 20.00);
INSERT INTO item_pedido (id_pedido, id_produto, quantidade, preco_praticado) VALUES (1, 1, 2, 20.00);
DELETE FROM pedido WHERE id_pedido = 1;
-- Resultado esperado: sucesso; a linha de item_pedido (id_pedido = 1) é excluída
-- automaticamente porque fk_item_pedido usa ON DELETE CASCADE.
-- Estado final: SELECT * FROM item_pedido não retorna nenhuma linha com id_pedido = 1.
```

| Teste | Regra esperada | Resultado |
|---|---|---|
| PK duplicada | `PRIMARY KEY` | Rejeitar |
| E-mail duplicado | `UNIQUE` | Rejeitar |
| Preço negativo | `CHECK` | Rejeitar |
| FK para cliente inexistente | `FOREIGN KEY` | Rejeitar |
| Excluir cliente com pedido | `RESTRICT` | Rejeitar |
| Excluir pedido com item | `CASCADE` | Excluir o item em cascata |

---

## Exercício Proposto 5 — Análise (VENDA não normalizada)

**Enunciado:** Analise `VENDA(id_venda, data, id_cliente, nome_cliente, id_produto, nome_produto, quantidade)`. Identifique os fatos distintos misturados, demonstre uma anomalia de cada tipo e proponha a decomposição.

### Fatos distintos misturados

A relação mistura **três fatos independentes** em uma única tabela:
1. **Dados do cliente** (`id_cliente`, `nome_cliente`) — deveriam existir uma única vez por cliente.
2. **Dados do produto** (`id_produto`, `nome_produto`) — deveriam existir uma única vez por produto.
3. **O evento da venda em si** (`id_venda`, `data`, `quantidade`) — é o fato que de fato varia a cada linha.

Como `nome_cliente` e `nome_produto` são repetidos a cada nova venda, a tabela sofre redundância.

### Anomalias

- **Atualização:** se o cliente 42 mudar de nome, é preciso atualizar `nome_cliente` em **todas** as vendas desse cliente. Se uma única linha não for atualizada, a base passa a ter duas versões do nome — inconsistência.
- **Inserção:** não é possível cadastrar um produto novo (ex.: um lançamento ainda sem nenhuma venda) sem inventar uma venda fictícia, pois `id_produto`/`nome_produto` só existem dentro de uma linha de `VENDA`.
- **Exclusão:** se a única venda registrada de um cliente for excluída, os dados desse cliente (`nome_cliente`) desaparecem da base junto com o fato de venda — perde-se informação que era independente do evento.

### Decomposição proposta

```sql
CREATE TABLE cliente (
  id_cliente INT NOT NULL AUTO_INCREMENT,
  nome       VARCHAR(120) NOT NULL,
  CONSTRAINT pk_cliente PRIMARY KEY (id_cliente)
) ENGINE=InnoDB;

CREATE TABLE produto (
  id_produto INT NOT NULL AUTO_INCREMENT,
  nome       VARCHAR(160) NOT NULL,
  CONSTRAINT pk_produto PRIMARY KEY (id_produto)
) ENGINE=InnoDB;

CREATE TABLE venda (
  id_venda   INT NOT NULL AUTO_INCREMENT,
  data       DATE NOT NULL,
  id_cliente INT NOT NULL,
  CONSTRAINT pk_venda PRIMARY KEY (id_venda),
  CONSTRAINT fk_venda_cliente FOREIGN KEY (id_cliente) REFERENCES cliente(id_cliente)
    ON DELETE RESTRICT ON UPDATE RESTRICT
) ENGINE=InnoDB;

CREATE TABLE item_venda (
  id_venda   INT NOT NULL,
  id_produto INT NOT NULL,
  quantidade INT NOT NULL,
  CONSTRAINT pk_item_venda PRIMARY KEY (id_venda, id_produto),
  CONSTRAINT ck_item_venda_qtd CHECK (quantidade > 0),
  CONSTRAINT fk_itemvenda_venda   FOREIGN KEY (id_venda)   REFERENCES venda(id_venda)   ON DELETE CASCADE,
  CONSTRAINT fk_itemvenda_produto FOREIGN KEY (id_produto) REFERENCES produto(id_produto) ON DELETE RESTRICT
) ENGINE=InnoDB;
```

**Qual atributo histórico deve permanecer no item da venda?** Se a venda registra um preço praticado no momento da transação, ele deve ficar em `item_venda` (ex.: `preco_praticado DECIMAL(10,2) NOT NULL`), e não apenas referenciado via `produto.preco_atual` — do contrário, alterações futuras no preço do produto reescreveriam silenciosamente o valor histórico de vendas já concluídas (mesma lógica do Exercício Resolvido 4, atributo "preço praticado").

---

## Desafio Integrador — Modelo físico de uma clínica

**Enunciado:**
- Crie schema e tabelas PACIENTE, MÉDICO, CONSULTA, PROCEDIMENTO e CONSULTA_PROCEDIMENTO.
- Escolha tipos para identificadores, textos, data_hora, valores e estados.
- Declare PKs, FKs, UNIQUE, NOT NULL, DEFAULT e CHECK.
- Defina e justifique ON DELETE e ON UPDATE para cada vínculo.
- Planeje ao menos seis testes de violação e preveja cada resultado.
- Revise redundâncias, dados históricos e possíveis anomalias.

### Schema e DDL completo

```sql
CREATE DATABASE IF NOT EXISTS clinica
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

USE clinica;

-- ---------------------------------------------------------------
-- PACIENTE
-- ---------------------------------------------------------------
CREATE TABLE paciente (
  id_paciente       INT NOT NULL AUTO_INCREMENT,
  cpf               CHAR(11) NOT NULL,
  nome              VARCHAR(120) NOT NULL,
  data_nascimento   DATE NOT NULL,
  telefone          VARCHAR(20) NULL,
  ativo             BOOLEAN NOT NULL DEFAULT TRUE,
  CONSTRAINT pk_paciente PRIMARY KEY (id_paciente),
  CONSTRAINT uq_paciente_cpf UNIQUE (cpf)
) ENGINE=InnoDB;

-- ---------------------------------------------------------------
-- MÉDICO
-- ---------------------------------------------------------------
CREATE TABLE medico (
  id_medico  INT NOT NULL AUTO_INCREMENT,
  crm        CHAR(12) NOT NULL,
  nome       VARCHAR(120) NOT NULL,
  ativo      BOOLEAN NOT NULL DEFAULT TRUE,
  CONSTRAINT pk_medico PRIMARY KEY (id_medico),
  CONSTRAINT uq_medico_crm UNIQUE (crm)
) ENGINE=InnoDB;

-- ---------------------------------------------------------------
-- PROCEDIMENTO (catálogo — independe de haver consultas)
-- ---------------------------------------------------------------
CREATE TABLE procedimento (
  id_procedimento  INT NOT NULL AUTO_INCREMENT,
  nome             VARCHAR(160) NOT NULL,
  valor_padrao     DECIMAL(10,2) NOT NULL,
  CONSTRAINT pk_procedimento PRIMARY KEY (id_procedimento),
  CONSTRAINT ck_procedimento_valor CHECK (valor_padrao >= 0)
) ENGINE=InnoDB;

-- ---------------------------------------------------------------
-- CONSULTA
-- ---------------------------------------------------------------
CREATE TABLE consulta (
  id_consulta  INT NOT NULL AUTO_INCREMENT,
  id_paciente  INT NOT NULL,
  id_medico    INT NOT NULL,
  data_hora    DATETIME NOT NULL,
  status       VARCHAR(20) NOT NULL DEFAULT 'AGENDADA',
  observacoes  TEXT NULL,
  CONSTRAINT pk_consulta PRIMARY KEY (id_consulta),
  CONSTRAINT ck_consulta_status
    CHECK (status IN ('AGENDADA','REALIZADA','CANCELADA')),
  CONSTRAINT fk_consulta_paciente
    FOREIGN KEY (id_paciente) REFERENCES paciente(id_paciente)
    ON DELETE RESTRICT ON UPDATE RESTRICT,
  CONSTRAINT fk_consulta_medico
    FOREIGN KEY (id_medico) REFERENCES medico(id_medico)
    ON DELETE RESTRICT ON UPDATE RESTRICT
) ENGINE=InnoDB;

-- ---------------------------------------------------------------
-- CONSULTA_PROCEDIMENTO (associativa N:N, com valor histórico)
-- ---------------------------------------------------------------
CREATE TABLE consulta_procedimento (
  id_consulta       INT NOT NULL,
  id_procedimento   INT NOT NULL,
  valor_praticado   DECIMAL(10,2) NOT NULL,
  CONSTRAINT pk_consulta_procedimento PRIMARY KEY (id_consulta, id_procedimento),
  CONSTRAINT ck_cp_valor CHECK (valor_praticado >= 0),
  CONSTRAINT fk_cp_consulta
    FOREIGN KEY (id_consulta) REFERENCES consulta(id_consulta)
    ON DELETE CASCADE ON UPDATE RESTRICT,
  CONSTRAINT fk_cp_procedimento
    FOREIGN KEY (id_procedimento) REFERENCES procedimento(id_procedimento)
    ON DELETE RESTRICT ON UPDATE RESTRICT
) ENGINE=InnoDB;
```

### Tipos escolhidos — justificativa resumida

| Atributo | Tipo | Justificativa |
|---|---|---|
| `id_*` | `INT AUTO_INCREMENT` | Identificadores substitutos internos |
| `cpf`, `crm` | `CHAR(n)` | Códigos de tamanho fixo; não são números para cálculo |
| `nome`, `telefone` | `VARCHAR(n)` | Textos curtos de comprimento variável |
| `data_nascimento` | `DATE` | Data civil, sem horário |
| `data_hora` (consulta) | `DATETIME` | Instante civil de agendamento/atendimento |
| `valor_padrao`, `valor_praticado` | `DECIMAL(10,2)` | Precisão financeira exata |
| `status` | `VARCHAR(20)` + `CHECK` | Vocabulário controlado |
| `ativo` | `BOOLEAN` | Estado lógico |
| `observacoes` | `TEXT NULL` | Conteúdo livre e opcional |

### ON DELETE / ON UPDATE — justificativa por vínculo

| Vínculo | ON DELETE | ON UPDATE | Justificativa |
|---|---|---|---|
| PACIENTE → CONSULTA | `RESTRICT` | `RESTRICT` | Consulta é histórico clínico; não pode desaparecer com o cadastro do paciente |
| MÉDICO → CONSULTA | `RESTRICT` | `RESTRICT` | Consulta registra quem atendeu; é prova de atendimento, não pode ser apagada junto com o médico |
| CONSULTA → CONSULTA_PROCEDIMENTO | `CASCADE` | `RESTRICT` | O vínculo procedimento-consulta não tem sentido sem a consulta; é parte composicional dela |
| PROCEDIMENTO → CONSULTA_PROCEDIMENTO | `RESTRICT` | `RESTRICT` | Procedimento já realizado permanece no histórico mesmo que o item do catálogo seja descontinuado |

### Seis testes de violação planejados

```sql
-- 1) PRIMARY KEY duplicada
INSERT INTO paciente (id_paciente, cpf, nome, data_nascimento) VALUES (1,'11111111111','Ana','2000-01-01');
INSERT INTO paciente (id_paciente, cpf, nome, data_nascimento) VALUES (1,'22222222222','Bia','1999-05-10');
-- Esperado: ERRO 1062 — chave primária duplicada

-- 2) UNIQUE duplicada (CPF)
INSERT INTO paciente (cpf, nome, data_nascimento) VALUES ('11111111111','Carlos','1998-03-03');
-- Esperado: ERRO 1062 — uq_paciente_cpf duplicada

-- 3) CHECK violado (status inválido)
INSERT INTO consulta (id_paciente, id_medico, data_hora, status) VALUES (1,1,'2026-10-01 09:00:00','FINALIZADA');
-- Esperado: ERRO 3819 — ck_consulta_status rejeita valor fora do vocabulário

-- 4) FK órfã
INSERT INTO consulta (id_paciente, id_medico, data_hora) VALUES (1, 999, '2026-10-01 09:00:00');
-- Esperado: ERRO 1452 — id_medico 999 não existe em MEDICO

-- 5) DELETE bloqueado por RESTRICT
INSERT INTO medico (crm, nome) VALUES ('CRM-SP-1234','Dr. João');
INSERT INTO consulta (id_paciente, id_medico, data_hora) VALUES (1, 1, '2026-10-01 09:00:00');
DELETE FROM medico WHERE id_medico = 1;
-- Esperado: ERRO 1451 — existe consulta dependente; fk_consulta_medico usa RESTRICT

-- 6) DELETE propagado por CASCADE
INSERT INTO procedimento (nome, valor_padrao) VALUES ('Eletrocardiograma', 120.00);
INSERT INTO consulta_procedimento (id_consulta, id_procedimento, valor_praticado) VALUES (1, 1, 120.00);
DELETE FROM consulta WHERE id_consulta = 1;
-- Esperado: sucesso; a linha correspondente em consulta_procedimento é excluída
-- automaticamente (fk_cp_consulta ON DELETE CASCADE). SELECT * FROM
-- consulta_procedimento WHERE id_consulta = 1 não retorna nenhuma linha.
```

### Revisão de redundâncias, dados históricos e anomalias

- **Redundância evitada:** `nome` do paciente e do médico existem uma única vez em suas respectivas tabelas — `CONSULTA` e `CONSULTA_PROCEDIMENTO` referenciam por `id`, nunca duplicam o nome.
- **Dado histórico preservado:** `valor_praticado` é guardado em `CONSULTA_PROCEDIMENTO` (e não apenas lido de `procedimento.valor_padrao`), porque o valor cobrado numa consulta já realizada não deve mudar se a tabela de preços do catálogo for reajustada depois — mesma lógica do `preco_praticado` do exemplo de pedidos.
- **Anomalias evitadas pela decomposição:**
  - *Atualização*: alterar o nome de um médico atualiza uma única linha em `MEDICO`, refletindo automaticamente em todas as consultas via FK.
  - *Inserção*: é possível cadastrar um `PROCEDIMENTO` novo no catálogo sem que nenhuma consulta o tenha usado ainda.
  - *Exclusão*: excluir a única consulta de um paciente não apaga o cadastro do paciente, pois estão em tabelas independentes ligadas por FK com `RESTRICT`.

---

## Autoavaliação — Recuperação ativa

> Respostas objetivas às seis perguntas de fechamento, para conferência.

**1. Por que CPF deve ser CHAR/VARCHAR e preço deve ser DECIMAL?**
CPF é um código identificador — não participa de soma, média ou cálculo, e pode conter zeros à esquerda que um tipo numérico descartaria; por isso é texto (`CHAR`/`VARCHAR`), nunca `INT`. Preço é um valor monetário que exige precisão decimal **exata**; tipos de ponto flutuante (`FLOAT`/`DOUBLE`) representam frações em binário e introduzem erros de arredondamento — `DECIMAL(p,s)` preserva a precisão exata necessária para dinheiro.

**2. Qual diferença existe entre PRIMARY KEY, UNIQUE e NOT NULL?**
`PRIMARY KEY` identifica cada linha de forma única **e** não nula — é a identidade da tabela, e só pode haver uma por tabela. `UNIQUE` também impede valores repetidos, mas pode haver várias por tabela e (no MySQL) admite múltiplos `NULL`, servindo para identificadores alternativos (ex.: e-mail). `NOT NULL` isoladamente só proíbe ausência de valor — não impede repetição.

**3. Quando ON DELETE CASCADE é coerente — e quando é perigoso?**
É coerente quando o registro filho é parte composicional do pai e não tem sentido sem ele (ex.: item de pedido sem o pedido). É perigoso quando o filho representa um fato histórico ou independente (ex.: um pedido de um cliente): usar `CASCADE` ali apagaria silenciosamente um histórico que deveria ser preservado — o correto nesses casos é `RESTRICT`.

**4. Por que SET NULL exige uma FK anulável?**
Porque `ON DELETE SET NULL` funciona atribuindo `NULL` à coluna de chave estrangeira quando o pai é excluído. Se essa coluna tiver `NOT NULL`, a própria operação de "setar NULL" violaria a constraint de obrigatoriedade, e o banco rejeitaria a exclusão — por isso `SET NULL` só é coerente em vínculos opcionais.

**5. Como uma redundância produz anomalia de atualização?**
Quando o mesmo fato (ex.: nome de um cliente) é armazenado em múltiplas linhas ou tabelas, uma mudança nesse fato exige atualizar **todas** as cópias. Se alguma cópia não for atualizada — por falha de aplicação, concorrência ou simples esquecimento — as cópias divergem, e o banco passa a conter duas versões conflitantes do mesmo dado.

**6. Que violações você testaria antes de considerar o schema confiável?**
Pelo menos: (1) inserir uma chave primária duplicada; (2) inserir um valor duplicado numa coluna `UNIQUE`; (3) inserir um valor que viole um `CHECK` de domínio; (4) inserir uma chave estrangeira "órfã" (sem correspondência no pai); (5) tentar excluir um registro pai que tem filhos protegidos por `RESTRICT`; (6) excluir um registro pai que tem filhos ligados por `CASCADE` e conferir se a propagação ocorreu como esperado.
