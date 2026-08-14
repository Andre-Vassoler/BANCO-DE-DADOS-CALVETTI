# 📘 Banco de Dados — T01: Fundamentos de Bancos de Dados
### Exercícios de Fixação (Prof. Calvetti)

> Respostas comentadas dos exercícios propostos na Parte 6 da aula.

---

## 🔥 Exercício de Aquecimento — Diagnóstico Rápido

**Cenário:** Uma equipe controla empréstimos de equipamentos em três planilhas.

| Pergunta | Resposta |
|---|---|
| **Dois dados provavelmente repetidos** | Nome/contato da pessoa que retira o equipamento e a descrição do equipamento — ambos tendem a ser copiados em mais de uma planilha (ex: uma para controle de retirada, outra para manutenção). |
| **Inconsistência possível** | Um equipamento aparece como "disponível" em uma planilha, mas como "emprestado" em outra, porque a devolução foi registrada em apenas uma delas. |
| **Pergunta difícil de responder** | "Quais equipamentos estão atrasados agora e com quem estão?" — exige cruzar dados de pessoas, equipamentos e datas espalhados em arquivos diferentes, sem garantia de que estejam sincronizados. |

---

## 🧩 Nível 1 — Reconhecimento
**Classifique cada item: dado, informação ou metadado?**

| Item | Classificação | Justificativa |
|---|---|---|
| `"2026-08-08"` | **Dado** | É um valor isolado, sem contexto — não sabemos, por si só, a que se refere. |
| `"Foram realizados 128 atendimentos no dia."` | **Informação** | O valor 128 foi interpretado em um contexto (atendimentos, período), respondendo a uma pergunta. |
| `"data_atendimento usa o tipo DATE e não aceita nulo."` | **Metadado** | Descreve a estrutura e a regra do dado (tipo e obrigatoriedade), não o valor em si. |

---

## 🌡️ Nível 2 — Interpretação
**Cenário:** registros de temperatura coletados a cada minuto.

1. **Dois exemplos de dados brutos:**
   - `23.4°C às 14:01`

2. **Transformação em informação útil:**
   - "A temperatura média da sala entre 14h e 15h foi de 24,1°C, 2°C acima do limite recomendado."

3. **Decisão apoiada por essa informação:**
   - Acionar o ar-condicionado ou revisar a climatização do ambiente.

---

## 🗂️ Nível 3 — Diagnóstico
**Cenário:** vendas e suporte mantêm cadastros separados de clientes.

| Item | Resposta |
|---|---|
| **Redundância** | Nome, telefone e e-mail do cliente cadastrados separadamente nos dois setores. |
| **Inconsistência possível** | Cliente atualiza o telefone com o suporte, mas vendas continua com o número antigo. |
| **Impacto operacional** | A equipe de vendas tenta contato usando um dado desatualizado e perde a venda ou o atendimento. |
| **Regra centralizada proposta** | Criar um cadastro único de cliente, compartilhado pelos dois setores, com um único responsável (ou processo) autorizado a alterar os dados cadastrais — fonte única de verdade. |

---

## ⚖️ Nível 4 — Decisão
**Cenário:** associação com 80 membros, um único responsável atualiza mensalidades uma vez por mês.

- **Solução inicial:** Planilha.
- **Dois critérios que sustentam a escolha:**
  1. Baixa simultaneidade — apenas uma pessoa edita os dados.
  2. Volume pequeno e regras simples — não há relacionamentos complexos entre entidades.
- **Evento que justificaria migração:** Se o número de membros crescer muito, ou se mais de uma pessoa passar a atualizar dados ao mesmo tempo, exigindo controle de acesso e concorrência.

---

## 🏗️ Nível 5 — Aplicação
**Cenário:** aplicação conectada pelo Workbench ao MySQL.

| Elemento | Papel |
|---|---|
| **Base de dados** | O conteúdo persistente — as tabelas, registros e relacionamentos armazenados (ex: clientes, pedidos). |
| **SGBD** | O MySQL Server — o software que interpreta comandos SQL, controla acesso e gerencia a base. |
| **Workbench** | O cliente — interface usada para enviar comandos SQL e administrar conexões e objetos. |
| **InnoDB** | O mecanismo de armazenamento — responsável por gravar/ler fisicamente as páginas de dados, índices e logs, além de garantir transações e integridade referencial. |

---

## 🧱 Nível 6 — Estruturação
**Cenário:** controle de empréstimos de equipamentos.

**Entidades e identificadores:**
- `Pessoa` → `id_pessoa`
- `Equipamento` → `id_equipamento`
- `Empréstimo` → `id_emprestimo`

**Relacionamentos:**
1. `emprestimo.id_pessoa` referencia `Pessoa` (quem retirou o item).
2. `emprestimo.id_equipamento` referencia `Equipamento` (qual item foi retirado).

**Regra de integridade:**
> Um equipamento não pode ter dois empréstimos ativos ao mesmo tempo (não pode ser retirado novamente enquanto estiver marcado como emprestado).

---

## ⚙️ Nível 7 — Raciocínio Técnico
**Cenário:** um usuário executa `SHOW TABLES` no Workbench.

**Ordem do fluxo:**
`Workbench → Conexão → Servidor (MySQL) → InnoDB → Resultado`

- **Responsabilidade do servidor:** Interpretar o comando SQL recebido, autenticar o usuário e verificar seus privilégios, planejar a operação e solicitar os dados ao mecanismo de armazenamento (InnoDB), devolvendo o resultado formatado.
- **Por que um usuário pode ver menos tabelas que outro:** Os privilégios de acesso são concedidos individualmente — cada usuário só enxerga (ou consegue listar) as tabelas para as quais recebeu permissão explícita.

---

## 🔐 Nível 8 — Avaliação
**Cenário:** uma equipe pede acesso irrestrito à base de clientes para criar um relatório.

**Três perguntas antes de liberar o acesso:**
1. Qual é a finalidade exata do relatório?
2. Quais campos são realmente necessários para atendê-la?
3. Por quanto tempo esse acesso será necessário?

**Alternativa de menor privilégio:**
> Criar uma *view* somente leitura contendo apenas as colunas relevantes para o relatório, sem os campos sensíveis (ex: sem CPF, sem dados de contato pessoal).

**Como preservar utilidade e privacidade:**
> Mascarando ou omitindo os dados sensíveis e mantendo apenas os campos analíticos necessários — a equipe consegue extrair os indicadores sem ter acesso a informações que não competem à sua função.

---

## 🏥 Desafio Final — Solução Integrada
**Cenário:** uma clínica controla pacientes, consultas e pagamentos em arquivos separados.

**Quatro riscos diagnosticados:**
1. Redundância dos dados do paciente em múltiplos arquivos.
2. Inconsistência entre agenda de consultas e registros de pagamento.
3. Falta de rastreabilidade sobre quem alterou o quê e quando.
4. Acesso não controlado a dados sensíveis (prontuário, dados pessoais — risco de LGPD).

**Como um SGBD reduz cada risco:**
1. Centraliza o cadastro do paciente em uma única tabela, eliminando cópias divergentes.
2. Aplica integridade referencial entre `Paciente`, `Consulta` e `Pagamento`, garantindo consistência.
3. Registra transações e logs, permitindo auditar alterações.
4. Implementa controle de privilégios por usuário/papel, restringindo quem vê ou edita cada campo.

**Entidades e relacionamentos iniciais:**
- `Paciente` (`id_paciente`)
- `Consulta` (`id_consulta`, `consulta.id_paciente` → referencia `Paciente`)
- `Pagamento` (`id_pagamento`, `pagamento.id_consulta` → referencia `Consulta`)

**Dois controles de segurança:**
1. Autenticação individual + privilégios mínimos por papel (recepção só vê agenda, financeiro só vê pagamentos, médico vê prontuário).
2. Restrição/criptografia de acesso a dados sensíveis (prontuário médico, dados pessoais).

**Limitação que o SGBD não resolve sozinho:**
> O SGBD garante integridade técnica e controle de acesso, mas não garante, por si só, que as pessoas usem os dados de forma ética e responsável. Governança, treinamento das equipes e políticas de privacidade continuam dependendo de decisões humanas.

---

*Baseado na aula T01 — Fundamentos de Bancos de Dados, Prof. Calvetti.*
