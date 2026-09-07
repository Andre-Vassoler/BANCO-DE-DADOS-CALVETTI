# T04 — Modelo Lógico Relacional (Banco de Dados — Prof. Calvetti)

> Respostas dos exercícios de fixação da aula *"Modelo lógico relacional: do significado do DER à estrutura das relações"*.

---

## Sumário

- [Exercício 1 — Reconheça os elementos de uma relação](#exercício-1--reconheça-os-elementos-de-uma-relação)
- [Exercício 2 — Encontre as chaves candidatas](#exercício-2--encontre-as-chaves-candidatas)
- [Nível 1 — Domine o vocabulário relacional](#nível-1--domine-o-vocabulário-relacional)
- [Nível 2 — Classifique chaves](#nível-2--classifique-chaves)
- [Nível 3 — Mapeie um relacionamento 1:N](#nível-3--mapeie-um-relacionamento-1n)
- [Nível 4 — Mapeie um relacionamento 1:1](#nível-4--mapeie-um-relacionamento-11)
- [Nível 5 — Transforme um N:N com atributos](#nível-5--transforme-um-nn-com-atributos)
- [Desafio final — Converta integralmente um DER](#desafio-final--converta-integralmente-um-der)
- [Autoavaliação — Recuperação ativa](#autoavaliação--recuperação-ativa)

---

## Exercício 1 — Reconheça os elementos de uma relação

**Enunciado:** Considere `PEDIDO(id_pedido, data_hora, status, id_cliente)` com 250 tuplas.

1. Qual é o grau?
2. Qual é a cardinalidade da instância?
3. Cite um atributo e proponha seu domínio.
4. Diferencie o esquema da instância.

### Resposta

1. **Grau = 4** — a relação possui quatro atributos: `id_pedido`, `data_hora`, `status`, `id_cliente`.
2. **Cardinalidade = 250** — é a quantidade de tuplas presentes na instância atual.
3. Atributo: `status` → **domínio**: conjunto controlado `{ABERTO, PAGO, ENVIADO, CANCELADO}`.
4. **Esquema** é a estrutura estável: o nome da relação e seus atributos (`PEDIDO(id_pedido, data_hora, status, id_cliente)`), que define a *forma* dos fatos permitidos. **Instância** é o conjunto de tuplas existentes em um instante específico (as 250 linhas atuais); ela pode mudar a qualquer momento sem que o esquema seja alterado.

---

## Exercício 2 — Encontre as chaves candidatas

**Enunciado:** `USUÁRIO(id_usuario, email, cpf, nome)`, com `email` e `cpf` exclusivos (únicos).

1. Liste superchaves possíveis.
2. Identifique as chaves candidatas.
3. Escolha uma chave primária e justifique.
4. Indique as chaves alternativas e suas restrições.

### Resposta

1. **Superchaves possíveis** (qualquer conjunto que contenha pelo menos um atributo que já garanta unicidade):
   `{id_usuario}`, `{email}`, `{cpf}`, `{id_usuario, nome}`, `{id_usuario, email}`, `{email, cpf}`, `{id_usuario, email, cpf, nome}`, entre outras combinações.
2. **Chaves candidatas** (superchaves mínimas — nenhum atributo pode ser removido sem perder a unicidade):
   `{id_usuario}`, `{email}`, `{cpf}`.
3. **Chave primária escolhida: `id_usuario`.** Justificativa: é um identificador substituto (surrogate), estável e sem significado de negócio — não muda ao longo do tempo, ao contrário de `email` (o usuário pode trocar) e evita expor um dado sensível como o `cpf` como referência interna.
4. **Chaves alternativas:** `email` e `cpf`. Ambas devem receber a restrição `UNIQUE` no modelo físico, já que continuam sendo candidatas não escolhidas como PK.

---

## Nível 1 — Domine o vocabulário relacional

**Enunciado:** `PRODUTO(id_produto, nome, preço)` com 80 tuplas.

1. Identifique o esquema e a instância.
2. Determine o grau e a cardinalidade.
3. Proponha o domínio lógico de `preço`.
4. Dê um exemplo de tupla válida.

### Resposta

1. **Esquema:** `PRODUTO(id_produto, nome, preço)`. **Instância:** o conjunto das 80 tuplas atualmente armazenadas.
2. **Grau = 3** (três atributos); **cardinalidade = 80** (tuplas na instância).
3. **Domínio de `preço`:** número decimal não negativo, representando valor monetário em reais (ex.: `DECIMAL(10,2)`, `preço ≥ 0`).
4. **Exemplo de tupla válida:** `(12, "Mouse óptico", 49.90)`.

---

## Nível 2 — Classifique chaves

**Enunciado:** `ALUNO(id_aluno, matrícula, email, nome)`, com `matrícula` e `email` exclusivos.

1. Liste as chaves candidatas.
2. Escolha a chave primária.
3. Indique as alternativas.
4. Dê uma superchave que não seja candidata.

### Resposta

1. **Chaves candidatas:** `{id_aluno}`, `{matrícula}`, `{email}`.
2. **Chave primária:** `id_aluno` — identificador interno estável, independente de mudanças de matrícula ou e-mail.
3. **Chaves alternativas:** `matrícula` e `email`, ambas com restrição `UNIQUE`.
4. **Superchave que não é candidata:** `{id_aluno, nome}` — contém `id_aluno`, que já garante unicidade sozinho; `nome` é um atributo excedente, então o conjunto não é mínimo.

---

## Nível 3 — Mapeie um relacionamento 1:N

**Enunciado:** Um `CURSO` oferece muitas `TURMAS`; cada `TURMA` pertence a exatamente um `CURSO`.

1. Crie as duas relações.
2. Defina PKs e FK.
3. Indique a nulidade da FK.
4. Escreva a leitura preservada pelo esquema.

### Resposta

```
CURSO(id_curso [PK], nome)
TURMA(id_turma [PK], ..., id_curso [FK → CURSO])
```

- **PKs:** `id_curso` em `CURSO`; `id_turma` em `TURMA`.
- **FK:** `id_curso` em `TURMA`, referenciando `CURSO.id_curso` — a FK fica sempre no lado N.
- **Nulidade:** `id_curso` deve ser **`NOT NULL`**, pois toda turma pertence obrigatoriamente a exatamente um curso (participação total do lado N).
- **Leitura preservada:** cada turma referencia exatamente um curso; um mesmo curso pode ser referenciado por muitas turmas.

---

## Nível 4 — Mapeie um relacionamento 1:1

**Enunciado:** Uma `PESSOA` pode possuir no máximo uma `CARTEIRA_FUNCIONAL`; toda carteira pertence a uma pessoa.

1. Escolha onde colocar a FK.
2. Defina a restrição que preserva 1:1.
3. Discuta a nulidade.
4. Compare com a alternativa de fundir relações.

### Resposta

```
PESSOA(id_pessoa [PK], nome)
CARTEIRA_FUNCIONAL(id_carteira [PK], ..., id_pessoa [FK/UQ → PESSOA])
```

- **FK:** colocada no lado dependente, `CARTEIRA_FUNCIONAL`, pois é o lado com participação total (toda carteira pertence a uma pessoa).
- **Restrição que preserva o 1:1:** `UNIQUE` sobre `id_pessoa` em `CARTEIRA_FUNCIONAL` — sem essa restrição, o esquema permitiria 1:N (uma pessoa com várias carteiras).
- **Nulidade:** `id_pessoa` é `NOT NULL` em `CARTEIRA_FUNCIONAL` (toda carteira exige uma pessoa); já em `PESSOA` não há FK — a participação de pessoa no relacionamento é opcional (nem toda pessoa possui carteira).
- **Alternativa de fusão:** se **ambos** os lados tivessem participação total (toda pessoa tem exatamente uma carteira, e vice-versa) e não houvesse razão de negócio para separar os dados, as duas relações poderiam ser fundidas em uma só: `PESSOA(id_pessoa, nome, numero_carteira, ...)`.

---

## Nível 5 — Transforme um N:N com atributos

**Enunciado:** `AUTOR` escreve `LIVRO`; o vínculo registra `papel_autoral`, `percentual` e `data_assinatura`.

1. Crie a relação associativa.
2. Defina PK e FKs.
3. Associe os atributos ao fato correto.
4. Explique como permitir mais de um contrato por par autor–livro.

### Resposta

```
AUTOR(id_autor [PK], nome)
LIVRO(id_livro [PK], titulo)
AUTORIA(id_autor [PK/FK → AUTOR], id_livro [PK/FK → LIVRO],
        papel_autoral, percentual, data_assinatura)
```

- **PK e FKs:** `id_autor` e `id_livro` juntos formam a PK composta de `AUTORIA`; cada um também é FK para sua relação de origem.
- **Atributos do vínculo:** `papel_autoral`, `percentual` e `data_assinatura` descrevem o *contrato* entre autor e livro — não são propriedades nem de `AUTOR` nem de `LIVRO` isoladamente, por isso pertencem à relação associativa `AUTORIA`.
- **Permitir mais de um contrato por par autor–livro:** a PK `(id_autor, id_livro)` sozinha não permite repetição. É necessário acrescentar um atributo discriminador à chave, por exemplo `numero_contrato`:

```
AUTORIA(id_autor [PK/FK → AUTOR], id_livro [PK/FK → LIVRO],
        numero_contrato [PK], papel_autoral, percentual, data_assinatura)
```

---

## Desafio final — Converta integralmente um DER

**Enunciado:** Clínica: `PACIENTE` agenda `CONSULTA` com `MÉDICO`; cada consulta registra `data_hora`, `estado` e `procedimentos` realizados.

1. Modele as relações `PACIENTE`, `MÉDICO` e `CONSULTA`.
2. Trate `CONSULTA` como fato com identidade e duas FKs.
3. Modele `PROCEDIMENTO` e a associação N:N com `CONSULTA`.
4. Defina chaves, nulidade, domínios e regras de integridade.
5. Produza um dicionário inicial para cinco atributos críticos.

### Resposta

```
PACIENTE(id_paciente [PK], nome)

MEDICO(id_medico [PK], nome, especialidade)

CONSULTA(id_consulta [PK], data_hora, estado,
         id_paciente [FK → PACIENTE],
         id_medico   [FK → MEDICO])

PROCEDIMENTO(id_procedimento [PK], nome, descricao)

CONSULTA_PROCEDIMENTO(id_consulta     [PK/FK → CONSULTA],
                       id_procedimento [PK/FK → PROCEDIMENTO],
                       observacao)
```

**Regras de integridade:**

- `CONSULTA.id_paciente` e `CONSULTA.id_medico` são **`NOT NULL`** — toda consulta exige exatamente um paciente e um médico existentes (integridade referencial + de entidade).
- `estado` segue domínio controlado: `{AGENDADA, REALIZADA, CANCELADA}`.
- `(id_consulta, id_procedimento)` em `CONSULTA_PROCEDIMENTO` distingue cada procedimento realizado numa consulta (evita repetição indevida).
- `id_procedimento` em `CONSULTA_PROCEDIMENTO` deve referenciar um procedimento existente (integridade referencial).

**Dicionário de dados inicial (5 atributos críticos):**

| Atributo | Relação | Domínio | Nulidade | Descrição |
|---|---|---|---|---|
| `id_consulta` | CONSULTA | Inteiro, gerado automaticamente | NOT NULL (PK) | Identificador único da consulta |
| `data_hora` | CONSULTA | DATETIME | NOT NULL | Data e hora agendada da consulta |
| `estado` | CONSULTA | ENUM {AGENDADA, REALIZADA, CANCELADA} | NOT NULL | Situação atual da consulta |
| `id_paciente` | CONSULTA | Inteiro (FK → PACIENTE) | NOT NULL | Paciente que agendou a consulta |
| `id_medico` | CONSULTA | Inteiro (FK → MEDICO) | NOT NULL | Médico responsável pela consulta |

---

## Autoavaliação — Recuperação ativa

1. **Qual é a diferença entre grau e cardinalidade da relação?**
   *Grau* é o número de atributos (colunas) do esquema da relação — é fixo enquanto o esquema não muda. *Cardinalidade* é o número de tuplas (linhas) presentes na instância em um dado momento — varia conforme dados são inseridos ou removidos.

2. **Por que toda candidata é superchave, mas nem toda superchave é candidata?**
   Toda chave candidata é, por definição, uma superchave *mínima*: nenhum de seus atributos pode ser removido sem perder a capacidade de identificar unicamente cada tupla. Uma superchave qualquer pode conter atributos excedentes (desnecessários) e ainda assim garantir unicidade — nesse caso ela deixa de ser mínima e, portanto, não é candidata.

3. **Como uma FK preserva um relacionamento 1:N?**
   A chave estrangeira é posicionada na relação do lado "muitos" (N), referenciando a chave primária da relação do lado "um". Assim, cada tupla do lado N aponta para exatamente uma tupla do lado 1, mas uma mesma tupla do lado 1 pode ser referenciada por diversas tuplas do lado N, pois não há restrição de unicidade sobre a FK.

4. **Qual restrição diferencia 1:1 de 1:N no esquema?**
   A restrição `UNIQUE` sobre a coluna da chave estrangeira. No 1:1, além da FK, aplica-se `UNIQUE` (garantindo que cada valor referenciado apareça no máximo uma vez). No 1:N, a FK não tem essa restrição de unicidade, permitindo repetição de valores.

5. **Por que um relacionamento N:N exige uma relação associativa?**
   Porque nenhuma das duas relações originais pode conter a FK do relacionamento sem violar a forma tabular (geraria listas ou repetição de valores em uma célula) e sem perder os atributos próprios do vínculo. A relação associativa recebe as chaves das duas entidades como FKs — que juntas normalmente formam a PK — permitindo representar múltiplas combinações e preservar atributos específicos do relacionamento.

---

*Fonte: T04 — Modelo lógico relacional, Prof. Calvetti — Banco de Dados, USJT 2026.2.*
