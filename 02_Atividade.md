# T02 — Modelagem Conceitual: Entidades e Atributos
### Respostas dos exercícios — Prof. Calvetti

---

## Pausa de verificação 1 — Em qual nível estamos?

| Afirmação | Nível |
|---|---|
| "Pedido possui uma data de realização." | **Conceitual** — descreve o domínio, sem depender de estrutura relacional ou de SGBD |
| "PEDIDO terá uma chave estrangeira para CLIENTE." | **Lógico** — já fala de chave estrangeira, conceito do modelo relacional |
| "data_pedido será DATE e terá índice." | **Físico** — tipo de coluna e índice são decisões de implementação no SGBD |

---

## Pausa de verificação 2 — Entidade ou atributo? (loja virtual)

| Item | Classificação | Justificativa |
|---|---|---|
| CLIENTE | **Entidade** | Tem identidade própria, várias ocorrências, atributos e participa de relacionamentos |
| e-mail | **Atributo** (de CLIENTE) | Propriedade descritiva; simples se único, multivalorado se o cliente puder ter mais de um |
| PEDIDO | **Entidade** | Evento com identidade, ciclo de vida e atributos próprios (data, itens, status) |
| endereço | **Atributo composto** (de CLIENTE) | Decomponível em rua, número, cidade, UF, CEP — só vira entidade se precisar de histórico/verificação próprios |
| FORMA_PAGAMENTO | **Depende da regra** | Atributo simples se for só uma categoria (ex.: "boleto"/"cartão"); vira **entidade** se precisar guardar dados próprios (bandeira, validade, histórico de uso) |

---

## Nível 1 — Reconhecimento

| Afirmação | Nível |
|---|---|
| "ALUNO possui nome e data de ingresso." | **Conceitual** |
| "MATRÍCULA terá chave composta." | **Lógico** |
| "nome será VARCHAR(120)." | **Físico** |

---

## Nível 2 — Classificação de atributos (entidade PESSOA)

| Atributo | Classificação |
|---|---|
| nome_completo | **Simples** |
| endereço (rua, cidade, CEP) | **Composto** |
| telefones | **Multivalorado** |
| idade (a partir da data de nascimento) | **Derivado** |

---

## Nível 3 — Extração de modelo (academia)

**Narrativa:** "Uma academia cadastra alunos, planos e contratos. Cada contrato possui data de início e situação."

**Entidades candidatas:** ALUNO, PLANO, CONTRATO

**Atributos iniciais:**
- ALUNO: `id_aluno`, `nome`
- PLANO: `id_plano`, `nome_plano`, `valor`
- CONTRATO: `id_contrato`, `data_inicio`, `situação`

**Identificadores propostos:** `id_aluno`, `id_plano`, `id_contrato` (substitutos, criados pelo sistema)

**CONTRATO como entidade (não atributo):** possui atributos próprios (`data_inicio`, `situação`), identidade própria e representa um vínculo com ciclo de vida entre ALUNO e PLANO — não é uma simples característica de outra entidade. Se fosse apenas "aluno está associado a um plano" sem esses dados, poderia ser um relacionamento simples; como carrega informação e histórico, merece ser modelado como entidade (ou relacionamento atributuído) ligando ALUNO e PLANO.

---

## Nível 4 — Escolha de identificadores (plataforma de usuários)

**Identificador principal proposto:** `id_usuario` (substituto, gerado pelo sistema)

**Candidatos naturais:** e-mail, documento (ex.: CPF)

**Riscos de usar e-mail ou documento como chave principal:**
- E-mail pode ser alterado pelo usuário e não é totalmente estável
- Documento é dado sensível — expô-lo como chave pública em várias tabelas aumenta risco de privacidade
- Ambos dependem de regras/formatos externos ao sistema

**Regras de unicidade que ainda devem existir:** mesmo usando `id_usuario` como chave, manter restrições `UNIQUE` em e-mail e documento para preservar a regra de negócio (não permitir dois cadastros com o mesmo e-mail/CPF).

---

## Nível 5 — Entidade fraca (receita e etapas)

**Narrativa:** "Uma receita possui etapas numeradas. Cada etapa contém instrução e duração e não existe fora da receita."

- **Entidade forte:** RECEITA — identificador: `id_receita`
- **Entidade fraca:** ETAPA — não tem identidade própria, só existe vinculada à receita
- **Chave parcial:** `numero_etapa` (a numeração recomeça em cada receita)
- **Identificação completa:** `id_receita` + `numero_etapa`
- **Notação:** RECEITA em retângulo simples; ETAPA em retângulo duplo; relacionamento identificador (ex.: "CONTÉM") em losango duplo; `numero_etapa` em elipse com sublinhado tracejado

---

## Nível 6 — Avaliação de modelo problemático

**Modelo dado:** CLIENTE(nome como ID, telefones em uma única string, idade armazenada, endereço indivisível)

**Quatro fragilidades:**
1. **Nome como identificador** — homônimos quebram a unicidade; nome não é estável
2. **Telefones em string única** — perde atomicidade, impede múltiplos valores estruturados e dificulta consulta
3. **Idade armazenada** — fica desatualizada com o tempo; deveria ser derivada de `data_nascimento`
4. **Endereço indivisível** — impede filtrar por cidade, UF ou CEP separadamente; deveria ser composto

**Correções propostas:**
- Criar `id_cliente` como identificador substituto
- `telefone` como atributo multivalorado
- `idade` → atributo derivado de `data_nascimento`
- `endereço` → composto (rua, número, cidade, UF, CEP)

**O que depende das regras do domínio:** se o sistema precisar de histórico/verificação de cada telefone, este deixa de ser atributo multivalorado e vira entidade própria; se a idade for usada em regras legais referenciadas a uma data específica (ex.: idade no momento da compra), pode ser necessário guardar essa data de referência em vez de só derivar da data atual.

---

## Desafio final — Biblioteca

**Narrativa:** usuários, exemplares, obras, autores e empréstimos.

**Entidades e atributos:**
- **AUTOR** (forte) — `id_autor`, `nome`
- **OBRA** (forte) — `id_obra`, `título`, `ano_publicação` — relaciona-se com AUTOR pelo relacionamento **ESCREVE** (N:N, pois um autor pode escrever várias obras e uma obra pode ter mais de um autor)
- **EXEMPLAR** (**fraca**, depende de OBRA) — chave parcial `numero_exemplar`; identificação completa: `id_obra` + `numero_exemplar`
- **USUARIO** (forte) — `id_usuario`, `nome`
- **EMPRÉSTIMO** — relacionamento atributuído (ou entidade forte, se precisar de histórico/multas próprios) entre USUARIO e EXEMPLAR, com atributos `data_retirada` e `data_devolução`

**Há entidade fraca?** Sim — EXEMPLAR, pois o exemplar físico só existe vinculado a uma OBRA (que é o conceito abstrato do livro/título).

**Classificação de atributos:**
- Simples: `nome` (usuário, autor), `título` (obra)
- Derivado (opcional): "quantidade de exemplares disponíveis", calculado a partir dos empréstimos ativos

**Distinção-chave do enunciado:** OBRA é o conceito (título, autor, edição); EXEMPLAR é a cópia física específica que é emprestada. AUTOR é a entidade da pessoa; o nome do autor sozinho não seria suficiente caso duas obras tenham "autores" homônimos.

---

## Autoavaliação — Recuperação ativa

1. **Diferença entre conceitual, lógico e físico:** conceitual descreve o domínio (o que existe, independente de tecnologia); lógico traduz isso em estruturas do modelo relacional (tabelas, chaves); físico especifica a implementação concreta no SGBD (tipos, índices).
2. **Quando um atributo vira entidade:** quando precisa de identidade própria, tem atributos próprios, participa de relacionamentos relevantes ou seu histórico importa (ex.: telefone com verificação e tipo).
3. **O que torna um identificador adequado:** unicidade, não nulidade, estabilidade, minimalidade e cuidado com privacidade.
4. **Como reconhecer entidade fraca:** quando ela não tem identificador completo próprio e sua existência depende de outra entidade (ex.: ETAPA depende de RECEITA).
5. **Representação de atributos no DER de Chen:** elipse simples (atributo comum), elipse dupla (multivalorado), elipse tracejada (derivado), com sublinhado (identificador) ou sublinhado tracejado (chave parcial).
