# T06 — SQL no MySQL: DDL e DML

**Disciplina:** Banco de Dados · **Prof. Calvetti**
**Tema:** CREATE, ALTER, DROP, TRUNCATE, INSERT, UPDATE, DELETE

> Respostas aos **3 exercícios propostos** e ao **desafio integrador** do slide "Exemplo resolvido e exercícios", além da autoavaliação de fechamento. Scripts validados para a sintaxe do MySQL 8.4, seguindo o mesmo padrão de tabelas (`cliente`, `produto`, `pedido`, `item_pedido`) usado nos exemplos resolvidos da aula.

---

## Sumário

- [Exercício Proposto 1 — Crie e altere um schema acadêmico](#exercício-proposto-1--crie-e-altere-um-schema-acadêmico)
- [Exercício Proposto 2 — Povoe a base respeitando dependências](#exercício-proposto-2--povoe-a-base-respeitando-dependências)
- [Exercício Proposto 3 — Atualize e exclua com segurança](#exercício-proposto-3--atualize-e-exclua-com-segurança)
- [Desafio Integrador — Scripts DDL e DML para uma clínica](#desafio-integrador--scripts-ddl-e-dml-para-uma-clínica)
- [Autoavaliação — Recuperação ativa](#autoavaliação--recuperação-ativa)

---

## Exercício Proposto 1 — Crie e altere um schema acadêmico

**Enunciado:**
- Crie o schema `universidade` e selecione-o.
- Crie `ALUNO`, `DISCIPLINA` e `MATRICULA` com InnoDB.
- Inclua PKs, FKs, `UNIQUE` e `CHECK` necessários.
- Adicione em `DISCIPLINA` a coluna `carga_horaria` com `ALTER TABLE`.
- Inspecione a definição final e registre a ordem dos comandos.

### Solução

```sql
-- 1) Schema
CREATE DATABASE IF NOT EXISTS universidade
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

USE universidade;

-- 2) Tabelas (pai antes do filho: ALUNO e DISCIPLINA antes de MATRICULA)
CREATE TABLE aluno (
  id_aluno   INT NOT NULL AUTO_INCREMENT,
  matricula  CHAR(10) NOT NULL,
  nome       VARCHAR(120) NOT NULL,
  email      VARCHAR(254) NOT NULL,
  CONSTRAINT pk_aluno PRIMARY KEY (id_aluno),
  CONSTRAINT uq_aluno_matricula UNIQUE (matricula),
  CONSTRAINT uq_aluno_email UNIQUE (email)
) ENGINE=InnoDB;

CREATE TABLE disciplina (
  id_disciplina INT NOT NULL AUTO_INCREMENT,
  codigo        CHAR(8) NOT NULL,
  nome          VARCHAR(120) NOT NULL,
  CONSTRAINT pk_disciplina PRIMARY KEY (id_disciplina),
  CONSTRAINT uq_disciplina_codigo UNIQUE (codigo)
) ENGINE=InnoDB;

CREATE TABLE matricula (
  id_matricula   INT NOT NULL AUTO_INCREMENT,
  id_aluno       INT NOT NULL,
  id_disciplina  INT NOT NULL,
  situacao       VARCHAR(15) NOT NULL DEFAULT 'ATIVA',
  criado_em      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT pk_matricula PRIMARY KEY (id_matricula),
  CONSTRAINT uq_matricula_aluno_disciplina UNIQUE (id_aluno, id_disciplina),
  CONSTRAINT ck_matricula_situacao
    CHECK (situacao IN ('ATIVA','EM_CURSO','TRANCADA','CONCLUIDA')),
  CONSTRAINT fk_matricula_aluno
    FOREIGN KEY (id_aluno) REFERENCES aluno(id_aluno)
    ON DELETE RESTRICT ON UPDATE RESTRICT,
  CONSTRAINT fk_matricula_disciplina
    FOREIGN KEY (id_disciplina) REFERENCES disciplina(id_disciplina)
    ON DELETE RESTRICT ON UPDATE RESTRICT
) ENGINE=InnoDB;

-- 3) Evolução da estrutura
ALTER TABLE disciplina
  ADD COLUMN carga_horaria INT NOT NULL DEFAULT 60 AFTER nome;

-- 4) Inspeção
SHOW CREATE TABLE disciplina;
SHOW CREATE TABLE matricula;
SHOW INDEX FROM matricula;
```

### Ordem dos comandos registrada

| Etapa | Comando | Por quê nesta ordem |
|---|---|---|
| 1 | `CREATE DATABASE` + `USE` | Estabelece o espaço de nomes antes de qualquer objeto |
| 2 | `CREATE TABLE aluno` | Tabela pai; não depende de nenhuma outra |
| 3 | `CREATE TABLE disciplina` | Tabela pai; não depende de nenhuma outra |
| 4 | `CREATE TABLE matricula` | Tabela filha; suas FKs exigem que `aluno` e `disciplina` já existam |
| 5 | `ALTER TABLE disciplina ADD COLUMN` | Estrutura já existe; a coluna é adicionada depois, sem contradizer dados (tabela ainda vazia) |
| 6 | `SHOW CREATE TABLE` / `SHOW INDEX` | Verificação final da definição efetivamente aplicada |

---

## Exercício Proposto 2 — Povoe a base respeitando dependências

**Enunciado:**
- Insira três alunos e três disciplinas com inserção múltipla.
- Recupere ou consulte as chaves geradas.
- Insira matrículas válidas.
- Provoque uma matrícula com aluno inexistente.
- Interprete o erro e corrija o comando sem desabilitar a FK.

### Solução

```sql
-- Inserção múltipla de pais
INSERT INTO aluno (matricula, nome, email) VALUES
  ('2026A001', 'Ana Lima',   'ana@exemplo.br'),
  ('2026A002', 'Bruno Reis', 'bruno@exemplo.br'),
  ('2026A003', 'Carla Dias', 'carla@exemplo.br');

INSERT INTO disciplina (codigo, nome, carga_horaria) VALUES
  ('BD100',  'Banco de Dados',        80),
  ('ALG100', 'Algoritmos',            60),
  ('MAT100', 'Matemática Discreta',   60);

-- Consultando as chaves geradas (AUTO_INCREMENT)
SELECT id_aluno, matricula, nome
FROM aluno
WHERE matricula IN ('2026A001','2026A002','2026A003');

SELECT id_disciplina, codigo, nome
FROM disciplina
WHERE codigo IN ('BD100','ALG100','MAT100');

-- Matrículas válidas (supondo id_aluno 1-3 e id_disciplina 1-3, conforme SELECTs acima)
INSERT INTO matricula (id_aluno, id_disciplina) VALUES
  (1, 1),
  (2, 1),
  (3, 2);

-- Provocando o erro: aluno inexistente
INSERT INTO matricula (id_aluno, id_disciplina) VALUES (999, 1);
-- ERRO esperado (1452): Cannot add or update a child row: a foreign key
-- constraint fails — não existe ALUNO com id_aluno = 999.

-- Interpretação: o "pai" referenciado pela FK fk_matricula_aluno não existe.
-- Correção: usar uma chave realmente persistida (obtida no SELECT acima),
-- sem desabilitar a constraint (nunca usar SET FOREIGN_KEY_CHECKS = 0 aqui):
INSERT INTO matricula (id_aluno, id_disciplina) VALUES (1, 2);
```

---

## Exercício Proposto 3 — Atualize e exclua com segurança

**Enunciado:**
- Selecione previamente as matrículas ATIVAS de 2026.
- Atualize apenas esse conjunto para `EM_CURSO`.
- Tente excluir uma disciplina com matrículas.
- Explique o comportamento de `RESTRICT` ou `CASCADE` escolhido.
- Demonstre por que a mesma instrução sem `WHERE` seria perigosa.

### Solução

```sql
-- 1) Pré-visualização: mesmo WHERE que será usado no UPDATE
SELECT id_matricula, id_aluno, id_disciplina, situacao, criado_em
FROM matricula
WHERE situacao = 'ATIVA'
  AND YEAR(criado_em) = 2026;

-- 2) UPDATE restrito ao conjunto conferido
UPDATE matricula
SET situacao = 'EM_CURSO'
WHERE situacao = 'ATIVA'
  AND YEAR(criado_em) = 2026;

-- 3) Tentativa de excluir uma disciplina que possui matrículas
DELETE FROM disciplina WHERE id_disciplina = 1;
-- ERRO esperado (1451): Cannot delete or update a parent row: a foreign
-- key constraint fails — existe matrícula referenciando id_disciplina = 1.
```

**Por que `RESTRICT` é o comportamento correto aqui:** `fk_matricula_disciplina` foi declarada com `ON DELETE RESTRICT`. Uma matrícula é um registro acadêmico — histórico ou em curso — vinculado a uma disciplina específica; excluir a disciplina não pode apagar silenciosamente (via `CASCADE`) o vínculo de todos os alunos matriculados nela, pois isso destruiria informação de currículo cursado. O `RESTRICT` obriga que a decisão sobre essas matrículas seja tomada explicitamente antes (encerrá-las, transferi-las, ou de fato removê-las) — é o mesmo raciocínio aplicado a `PRODUTO → ITEM_PEDIDO` nos exemplos da aula.

**Por que a mesma instrução sem `WHERE` seria perigosa:**

```sql
-- PERIGOSO — NÃO EXECUTAR
UPDATE matricula
SET situacao = 'EM_CURSO';
```

Sem `WHERE`, o alcance da operação passa a ser **todas as linhas da tabela**, não apenas o conjunto conferido no `SELECT` prévio. Matrículas já `TRANCADA` ou `CONCLUIDA` — inclusive de anos anteriores a 2026 — seriam reescritas para `EM_CURSO`, corrompendo o histórico acadêmico inteiro em uma única instrução. Diferente de um `DELETE` bloqueado por `RESTRICT`, um `UPDATE` sem `WHERE` que respeita todas as constraints é aceito pelo banco sem nenhum aviso — por isso a rotina "escrever WHERE → executar SELECT → conferir contagem → executar DML → validar resultado" é indispensável antes de qualquer `UPDATE` ou `DELETE`.

---

## Desafio Integrador — Scripts DDL e DML para uma clínica

**Enunciado:**
- Crie `PACIENTE`, `MEDICO`, `CONSULTA`, `PROCEDIMENTO` e `CONSULTA_PROCEDIMENTO`.
- Evolua o schema com pelo menos dois `ALTER TABLE`.
- Povoe pais e filhos com `INSERT` simples e múltiplo.
- Inclua `UPDATE` e `DELETE` precedidos de `SELECT` equivalente.
- Provoque e documente quatro erros de integridade.
- Finalize com `SHOW CREATE TABLE` e `SHOW INDEX` das estruturas principais.

### 1) Schema e tabelas base

```sql
CREATE DATABASE IF NOT EXISTS clinica
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

USE clinica;

CREATE TABLE paciente (
  id_paciente      INT NOT NULL AUTO_INCREMENT,
  cpf              CHAR(11) NOT NULL,
  nome             VARCHAR(120) NOT NULL,
  data_nascimento  DATE NOT NULL,
  CONSTRAINT pk_paciente PRIMARY KEY (id_paciente),
  CONSTRAINT uq_paciente_cpf UNIQUE (cpf)
) ENGINE=InnoDB;

CREATE TABLE medico (
  id_medico  INT NOT NULL AUTO_INCREMENT,
  crm        CHAR(12) NOT NULL,
  nome       VARCHAR(120) NOT NULL,
  CONSTRAINT pk_medico PRIMARY KEY (id_medico),
  CONSTRAINT uq_medico_crm UNIQUE (crm)
) ENGINE=InnoDB;

CREATE TABLE procedimento (
  id_procedimento  INT NOT NULL AUTO_INCREMENT,
  nome             VARCHAR(160) NOT NULL,
  valor_padrao     DECIMAL(10,2) NOT NULL,
  CONSTRAINT pk_procedimento PRIMARY KEY (id_procedimento),
  CONSTRAINT ck_procedimento_valor CHECK (valor_padrao >= 0)
) ENGINE=InnoDB;

CREATE TABLE consulta (
  id_consulta  INT NOT NULL AUTO_INCREMENT,
  id_paciente  INT NOT NULL,
  id_medico    INT NOT NULL,
  data_hora    DATETIME NOT NULL,
  status       VARCHAR(20) NOT NULL DEFAULT 'AGENDADA',
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

### 2) Evolução do schema — pelo menos dois `ALTER TABLE`

```sql
-- ALTER #1: novo atributo em PACIENTE
ALTER TABLE paciente
  ADD COLUMN telefone VARCHAR(20) NULL AFTER nome;

-- ALTER #2: novo atributo em CONSULTA
ALTER TABLE consulta
  ADD COLUMN observacoes TEXT NULL;

-- ALTER #3 (extra): estado lógico em MEDICO, com constraint adicionada depois
ALTER TABLE medico
  ADD COLUMN ativo BOOLEAN NOT NULL DEFAULT TRUE;

ALTER TABLE medico
  ADD CONSTRAINT ck_medico_ativo CHECK (ativo IN (0,1));
```

### 3) Povoamento — pais antes de filhos, inserção simples e múltipla

```sql
-- INSERT simples
INSERT INTO paciente (cpf, nome, telefone, data_nascimento)
VALUES ('11111111111', 'Fernanda Alves', '11999990000', '1990-04-12');

-- INSERT múltiplo
INSERT INTO medico (crm, nome, ativo) VALUES
  ('CRM-SP-1111', 'Dr. João Souza',    TRUE),
  ('CRM-SP-2222', 'Dra. Marcela Lima', TRUE);

INSERT INTO procedimento (nome, valor_padrao) VALUES
  ('Consulta de rotina',   150.00),
  ('Eletrocardiograma',    120.00),
  ('Hemograma completo',    60.00);

-- Filhos: consulta depende de paciente e médico já existentes
INSERT INTO consulta (id_paciente, id_medico, data_hora)
VALUES (1, 1, '2026-10-05 09:00:00');

INSERT INTO consulta_procedimento (id_consulta, id_procedimento, valor_praticado) VALUES
  (1, 1, 150.00),
  (1, 2, 120.00);
```

### 4) `UPDATE` e `DELETE` precedidos de `SELECT` equivalente

```sql
-- Atualizar status da consulta
SELECT id_consulta, status FROM consulta WHERE id_consulta = 1;

UPDATE consulta
SET status = 'REALIZADA'
WHERE id_consulta = 1;

-- Excluir um procedimento do catálogo ainda não utilizado
SELECT id_procedimento, nome FROM procedimento WHERE id_procedimento = 3;

DELETE FROM procedimento WHERE id_procedimento = 3;
-- Sucesso: nenhum registro em consulta_procedimento referencia id_procedimento = 3.
```

### 5) Quatro erros de integridade provocados e documentados

```sql
-- 1) PRIMARY KEY duplicada
INSERT INTO medico (id_medico, crm, nome, ativo) VALUES (1, 'CRM-SP-9999', 'Dr. Outro', TRUE);
-- ERRO 1062 (Duplicate entry '1' for key 'medico.PRIMARY')
-- Diagnóstico: id_medico 1 já existe — chave primária não pode repetir.

-- 2) UNIQUE duplicada
INSERT INTO paciente (cpf, nome, data_nascimento) VALUES ('11111111111', 'Outro Nome', '2000-01-01');
-- ERRO 1062 (Duplicate entry '11111111111' for key 'paciente.uq_paciente_cpf')
-- Diagnóstico: já existe paciente cadastrado com este CPF.

-- 3) CHECK violado
INSERT INTO procedimento (nome, valor_padrao) VALUES ('Procedimento inválido', -10.00);
-- ERRO 3819 (Check constraint 'ck_procedimento_valor' is violated)
-- Diagnóstico: o predicado valor_padrao >= 0 ficou falso.

-- 4) FK órfã
INSERT INTO consulta (id_paciente, id_medico, data_hora) VALUES (1, 999, '2026-10-06 10:00:00');
-- ERRO 1452 (Cannot add or update a child row: a foreign key constraint fails)
-- Diagnóstico: não existe médico com id_medico = 999.
```

### 6) Verificação final

```sql
SHOW CREATE TABLE consulta;
SHOW CREATE TABLE consulta_procedimento;
SHOW INDEX FROM consulta;
SHOW INDEX FROM consulta_procedimento;
```

---

## Autoavaliação — Recuperação ativa

> Respostas objetivas às seis perguntas de fechamento, para conferência.

**1. Qual diferença prática existe entre DELETE, TRUNCATE e DROP?**
`DELETE` é DML: remove linhas selecionadas por um `WHERE` (ou todas, se omitido), mas preserva a tabela e pode ser controlado linha a linha pelas FKs. `TRUNCATE` é DDL: remove **todas** as linhas de uma vez, não aceita `WHERE`, exige privilégio `DROP` e pode ser impedido por referências de chave estrangeira. `DROP TABLE` remove a estrutura inteira — definição e dados — de forma definitiva; nenhuma linha ou coluna resta.

**2. Por que a ordem de criação e inserção segue as FKs?**
Porque uma `FOREIGN KEY` exige que o valor referenciado já exista na tabela pai no momento da operação. Criar ou povoar a tabela filha antes de a tabela pai existir (ou antes de a linha pai estar inserida) leva a um erro de referência — por isso o princípio "pai antes do filho" vale tanto para `CREATE TABLE` quanto para `INSERT`.

**3. Que garantias vêm de PK/UNIQUE e quais vêm de índices comuns?**
`PRIMARY KEY` garante identidade única e não nula da linha; `UNIQUE` garante que um valor não se repita (podendo aceitar múltiplos `NULL` no MySQL). Ambas criam um índice como efeito colateral da regra. Um `INDEX` comum, por outro lado, só acelera buscas e ordenação — não impõe nenhuma regra de unicidade nem substitui `PRIMARY KEY`, `UNIQUE` ou `FOREIGN KEY`.

**4. Por que AUTO_INCREMENT pode apresentar lacunas?**
Porque o contador interno avança a cada tentativa de inserção — mesmo que ela falhe, seja desfeita (`ROLLBACK`) ou a linha gerada seja excluída depois. O valor serve para **identificar** a linha, não para representar posição, quantidade ou sequência contígua; por isso não deve ser usado como numeração contábil ou contagem de registros.

**5. Como reduzir o risco de UPDATE ou DELETE sem WHERE?**
Seguindo a rotina segura do material: escrever a cláusula `WHERE` primeiro, executar um `SELECT` com exatamente esse `WHERE` para pré-visualizar o alcance, conferir a contagem/conjunto retornado, só então executar o `UPDATE`/`DELETE` equivalente, e validar o resultado depois. Um `SELECT` prévio nunca altera dados, então é a forma mais segura de confirmar o alcance antes de uma operação destrutiva.

**6. Que perguntas transformam uma mensagem de erro em diagnóstico?**
- *"Duplicate entry"* → qual valor já existe (violação de `PRIMARY KEY` ou `UNIQUE`)?
- *"Cannot add/update a child row"* → o pai referenciado existe e os tipos/índices são compatíveis (`FOREIGN KEY`)?
- *"Check constraint violated"* → qual predicado do `CHECK` ficou falso?
- *"Column cannot be null"* → o valor foi omitido ou ficou nulo, e a coluna é `NOT NULL`?
- *"Data truncated/out of range"* → o tipo da coluna e o modo SQL são adequados ao valor informado?
