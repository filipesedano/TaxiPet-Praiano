# 🚕🐾 BLOCO B — Registro de Correções Arquiteturais
## v1.0 → v1.1 — Táxi Pet Praiano

> **Documento base:** `BLOCO_B_Especificacao_Estrutural_v1.0.md`  
> **Documento resultante:** `BLOCO_B_Especificacao_Estrutural_v1.1.md`  
> **Status:** Correções aprovadas para aplicação  
> **Data:** 16/08/2026  
> **Regra:** Nenhuma alteração de código até aprovação formal da v1.1  

---

## 📋 Resumo Executivo

A auditoria arquitetural da **BLOCO B v1.0** identificou **11 correções necessárias** para que a especificação possa ser considerada um **contrato de implementação válido**. A arquitetura conceitual (offline-first, autoridade híbrida, versionamento, conflitos por regra de domínio) foi **aprovada na essência**. As correções tratam de **inconsistências de contrato, schema incompleto e semântica de operações**.

| Categoria | Quantidade |
|-----------|------------|
| Correções obrigatórias | 11 |
| Decisões arquiteturais adicionais | 1 |
| Documentos afetados | B-01, B-02, B-03, B-04, B-05, B-08 |

---

## 🔴 B-FIX-001 — Atomicidade de `group_id`: uma entidade, uma mutação

### Problema

O documento v1.0 permite que um mesmo `group_id` contenha múltiplas operações `UPDATE` sobre a **mesma entidade**, cada uma com seu próprio `base_version`. Isso cria uma ambiguidade no servidor: após aplicar o primeiro `UPDATE` (incrementando `version`), o segundo `UPDATE` do mesmo grupo apresentaria `base_version` obsoleto, gerando falso conflito interno.

### Exemplo do problema

```text
Ride #123 version = 5

Grupo 01JXYZ:
  Op 1: UPDATE rides status=concluida      base_version=5
  Op 2: UPDATE rides km_end=45230          base_version=5

Servidor processa Op 1:
  version 5 → 6 ✅

Servidor processa Op 2:
  base_version=5, server_version=6
  → Falso conflito de versionamento dentro do mesmo grupo ❌
```

### Correção

> **Dentro de um mesmo `group_id`, uma entidade só pode possuir no máximo uma operação de mutação efetiva (`CREATE`, `UPDATE`, `SOFT_DELETE`).**

Se o domínio exigir alteração de múltiplos campos da mesma entidade, o payload deve conter o **delta completo** em uma única operação:

```text
operation_type: UPDATE
entity_type: RIDE
entity_id: ride-123
payload: {
  "status": "concluida",
  "km_end": 45230,
  "updated_at": "2026-08-16T10:00:00Z"
}
base_version: 5
group_id: 01JXYZ...
```

### Documentos afetados
- **B-02** (SyncQueue — Operações Compostas)
- **B-05** (Versionamento — Operações Compostas)
- **B-07** (Fluxos — diagrama de operação offline)

### Checklist de aplicação
- [ ] Revisar seção "Operações Compostas (`group_id`)" em B-02
- [ ] Revisar seção "Versionamento em Operações Compostas" em B-05
- [ ] Atualizar exemplo de payload consolidado
- [ ] Atualizar diagrama de sequência em B-07

---

## 🔴 B-FIX-002 — Compactação da fila preserva `base_version` original

### Problema

O documento v1.0 afirma que a compactação da fila "atualiza `base_version` se necessário", sem definir a regra exata. Isso é perigoso porque pode mascarar a pré-condição real sobre a qual a sequência de alterações foi construída.

### Exemplo do problema

```text
version local = 5

Op1 (pending): base_version=5, payload={status: motoristaACaminho}
Op2 (pending): base_version=6, payload={notes: "Cliente avisado"}
Op3 (pending): base_version=7, payload={km_end: 45230}

Compactação incorreta:
  base_version = 7 (da última operação)
  → O servidor verifica version=5, vê base_version=7, e aplica sem conflito
  → Mas a operação original começou sobre a versão 5, e campos intermediários podem ter sido alterados no servidor
```

### Correção

> **A compactação de operações pendentes para a mesma entidade preserva o `base_version` da operação mais antiga da sequência, pois ela representa a pré-condição original sobre a qual toda a transformação foi construída.**

Algoritmo correto:

```text
Sequência de updates PENDING para mesma entidade:
  Op1: base_version=5, payload={status: motoristaACaminho}
  Op2: base_version=6, payload={notes: "Cliente avisado"}
  Op3: base_version=7, payload={km_end: 45230}

Compactação:
  operation_id = Op1.operation_id  (preserva idempotência)
  base_version = 5                  (preserva pré-condição original)
  payload = merge(Op1, Op2, Op3) = {
    status: motoristaACaminho,
    notes: "Cliente avisado",
    km_end: 45230
  }
```

### Regras de segurança da compactação (mantidas da v1.0)

| Pode compactar? | Condição |
|-----------------|----------|
| ✅ Sim | Apenas `UPDATE` + `UPDATE` na mesma entidade, sem `CREATE` ou `SOFT_DELETE` no meio |
| ❌ Não | Há `CREATE` ou `SOFT_DELETE` para a mesma entidade na fila |
| ❌ Não | Operações têm `group_id` diferentes |
| ❌ Não | A primeira operação já está `in_flight` ou `failed` |

### Documentos afetados
- **B-02** (SyncQueue — Compactação da Fila)

### Checklist de aplicação
- [ ] Revisar seção "Compactação da Fila" em B-02
- [ ] Substituir texto "atualiza base_version se necessário" pela regra explícita
- [ ] Incluir exemplo numérico de compactação com base_version preservado

---

## 🔴 B-FIX-003 — `concurrent_edit` como `cause` de `version_mismatch`

### Problema

O documento v1.0 define `concurrent_edit` como um tipo independente de conflito, mas na prática ele é sempre detectado através do mesmo mecanismo de `version_mismatch` (`base_version != server_version`). Isso cria duplicação semântica no protocolo.

### Correção

> **`concurrent_edit` deixa de ser um `conflict_type` independente e passa a ser uma `cause` (causa semântica) do tipo `version_mismatch`.**

Estrutura corrigida:

```text
conflict_type (protocolo — determina como resolver)
├── version_mismatch
│   └── cause (semântica — explica por que aconteceu)
│       ├── concurrent_edit      (dois dispositivos editaram offline)
│       ├── stale_version        (dispositivo estava muito desatualizado)
│       └── unknown              (causa não determinada)
├── state_invalid
├── permission_denied
└── merge_failed
```

### Mapeamento no schema

A tabela `sync_conflicts` deve refletir essa separação:

```text
sync_conflicts
├── conflict_type: TEXT       → version_mismatch, state_invalid, etc.
├── conflict_cause: TEXT      → concurrent_edit, stale_version, etc. (nullable)
```

### Documentos afetados
- **B-03** (Modelo de Conflitos — Tipos de Conflito)
- **B-01** (Schema — tabela `sync_conflicts`)

### Checklist de aplicação
- [ ] Revisar seção "Tipos de Conflito" em B-03
- [ ] Remover `concurrent_edit` da lista de `conflict_type`
- [ ] Adicionar `conflict_cause` ao schema de `sync_conflicts` em B-01
- [ ] Atualizar fluxo de detecção em B-03

---

## 🔴 B-FIX-004 — Separar `resolution_strategy` de `resolution_status`

### Problema

O documento v1.0 mistura **como foi resolvido** (estratégia) com **qual é o estado da resolução** (status) na tabela `sync_conflicts`. Isso cria ambiguidade e impede rastreabilidade.

### Schema v1.0 (problemático)

```text
sync_conflicts
├── status: open | resolved | escalated
├── resolution: local_wins | server_wins | merged | manual
```

O campo `resolution` mistura estratégia (`local_wins`, `server_wins`) com estado (`manual` — que na verdade é uma estratégia que requer intervenção).

### Correção

> **Separar em dois campos distintos: `resolution_strategy` (como resolveu) e `resolution_status` (em que estágio está).**

Schema v1.1:

```text
sync_conflicts
├── conflict_type: TEXT
│   → version_mismatch | state_invalid | permission_denied | merge_failed
│
├── conflict_cause: TEXT
│   → concurrent_edit | stale_version | unknown (nullable)
│
├── resolution_strategy: TEXT
│   → auto_merge | server_wins | local_wins | domain_rule | manual
│
├── resolution_status: TEXT
│   → open | resolved | escalated
│
├── resolution: TEXT (JSON)
│   → Detalhes do resultado (payload merged, justificativa, etc.)
│
├── resolved_by: UUID
│   → Usuário que resolveu (nullable)
│
├── resolved_at: TEXT (ISO 8601)
│   → Timestamp da resolução (nullable)
```

### Documentos afetados
- **B-01** (Schema — tabela `sync_conflicts`)
- **B-03** (Modelo de Conflitos — Estratégias de Resolução)
- **B-04** (Contrato do SyncEngine — `ConflictResolution`)

### Checklist de aplicação
- [ ] Revisar schema de `sync_conflicts` em B-01
- [ ] Revisar seção "Estratégias de Resolução" em B-03
- [ ] Garantir consistência entre B-01, B-03 e B-04
- [ ] Atualizar exemplo de resolução em B-07

---

## 🔴 B-FIX-005 — Adicionar `domainRule` ao contrato `ConflictResolution`

### Problema

O documento v1.0 define `domain_rule` como estratégia de resolução em B-03, mas o contrato `ConflictResolution` em B-04 não inclui essa opção. Isso quebra a cadeia de consistência entre os documentos.

### Contrato v1.0 (incompleto)

```text
ConflictResolution
├── strategy: auto_merge | server_wins | local_wins | manual
```

### Correção

> **O contrato `ConflictResolution` deve incluir `domain_rule` como estratégia válida.**

Contrato v1.1:

```dart
enum ConflictResolutionStrategy {
  autoMerge,    // Campos disjuntos, merge automático
  serverWins,   // Servidor tem prioridade
  localWins,    // Dispositivo local tem prioridade
  domainRule,   // ← NOVO: RideStateMachine / AuthorizationPolicy decidem
  manual,       // Requer intervenção humana
}

class ConflictResolution {
  final ConflictResolutionStrategy strategy;
  final Map<String, dynamic>? mergedPayload;  // Se autoMerge
  final String reason;                         // Justificativa
  final bool requiresUserInput;               // Se manual
}
```

### Quando usar `domainRule`

| Cenário | Estratégia | Decisão |
|---------|------------|---------|
| Dois motoristas tentam iniciar a mesma corrida | `domainRule` | `AuthorizationPolicy` define quem tem direito |
| Estado proposto viola `RideStateMachine` | `domainRule` | StateMachine rejeita transição inválida |
| Admin cancelou, motorista tenta iniciar | `domainRule` | Estado terminal (cancelada) bloqueia qualquer transição |

### Documentos afetados
- **B-04** (Contrato do SyncEngine — `ConflictResolution`)
- **B-03** (Modelo de Conflitos — Hierarquia de Resolução)
- **B-06** (Matriz de Conflitos — `rides` usa `domain_rule` como padrão)

### Checklist de aplicação
- [ ] Revisar `ConflictResolution` em B-04
- [ ] Adicionar `domainRule` ao enum
- [ ] Atualizar hierarquia de resolução em B-03
- [ ] Garantir que B-06 referencie `domain_rule` consistentemente

---

## 🔴 B-FIX-006 — Incorporar `audit_context` à `SyncOperation`

### Problema

O documento v1.0 afirma que `audit_log` não entra na `sync_queue`, mas também diz que é "replicado para o servidor". Isso cria uma lacuna: **como o evento de auditoria chega ao servidor?**

### Opções consideradas

| Opção | Prós | Contras |
|-------|------|---------|
| Segunda fila (`audit_outbox`) | Separação clara | Complexidade desnecessária, duplicação de mecanismo |
| Anexar à operação de sync | Simplicidade, atomicidade | Aumenta tamanho do payload |
| Servidor gerar próprio audit | Menor payload | Perde contexto do dispositivo (device_id, old_value local) |

### Correção

> **O evento de auditoria é anexado à `SyncOperation` via campo `audit_context`. O servidor extrai esse contexto ao processar a operação e persiste no seu próprio `audit_log`. O dispositivo já persistiu localmente no ato da transação SQLite.**

Estrutura corrigida:

```text
SyncOperation
├── operation_id: ULID
├── entity_type: String
├── entity_id: UUID
├── operation_type: CREATE | UPDATE | SOFT_DELETE
├── payload: Map<String, dynamic>
├── base_version: int
├── group_id: ULID?
└── audit_context: AuditContext?  // ← NOVO

AuditContext
├── actor_id: UUID          // Quem executou
├── device_id: ULID         // Em qual dispositivo
├── action: String          // CREATE, UPDATE, STATUS_CHANGE, etc.
├── old_value: JSON         // Estado anterior (parcial ou completo)
├── new_value: JSON         // Novo estado
└── timestamp: ISO 8601     // Quando ocorreu no dispositivo
```

### Fluxo completo

```text
OFFLINE (dispositivo)
  ↓
SQLite Transaction
  ├── UPDATE rides
  ├── INSERT ride_status_history
  ├── INSERT audit_log          ← local, imutável
  └── INSERT sync_queue         ← com audit_context anexado
  ↓
SYNC (envio)
  ↓
API recebe SyncOperation + audit_context
  ↓
Servidor processa operação
  ↓
Servidor extrai audit_context
  ↓
Servidor persiste no próprio audit_log
```

### Vantagem arquitetural

- **Uma única fila** de sincronização (não duplica mecanismo)
- **Atomicidade garantida**: se a operação falha, o audit não é perdido (fica na fila para retry)
- **Contexto completo preservado**: device_id, old_value, new_value, timestamp local

### Documentos afetados
- **B-02** (SyncQueue — estrutura da operação)
- **B-04** (Contrato do SyncEngine — `SyncOperation`)
- **B-08** (Autoridade — replicação de auditoria)
- **B-01** (Schema — `sync_queue.payload` deve comportar audit_context)

### Checklist de aplicação
- [ ] Adicionar `audit_context` à estrutura `SyncOperation` em B-04
- [ ] Revisar seção de AuditLog em B-02
- [ ] Atualizar B-08 para remover ambiguidade de replicação
- [ ] Garantir que `sync_queue.payload` (JSON) comporta audit_context

---

## 🔴 B-FIX-007 — Adicionar `in_flight_since` ao schema da `sync_queue`

### Problema

O documento v1.0 descreve o mecanismo de recovery de operações `in_flight` após crash usando um campo `in_flight_since`, mas esse campo **não existe no schema** da tabela `sync_queue`.

### Schema v1.0 (incompleto)

```text
sync_queue
├── created_at
├── last_attempt_at
├── next_attempt_at
└── (falta in_flight_since)
```

### Correção

> **Adicionar o campo `in_flight_since` à tabela `sync_queue`, com semântica explícita de heartbeat e recovery.**

Schema v1.1:

```text
sync_queue
├── id: INTEGER PRIMARY KEY AUTOINCREMENT
├── operation_id: TEXT UNIQUE NOT NULL
├── group_id: TEXT
├── entity_type: TEXT NOT NULL
├── entity_id: TEXT NOT NULL
├── operation_type: TEXT NOT NULL
├── payload: TEXT (JSON) NOT NULL
├── base_version: INTEGER NOT NULL
├── created_at: TEXT NOT NULL
├── attempt_count: INTEGER DEFAULT 0
├── last_attempt_at: TEXT
├── next_attempt_at: TEXT
├── in_flight_since: TEXT NULL        // ← NOVO
├── status: TEXT DEFAULT 'pending'
├── last_error: TEXT
└── resolved_at: TEXT
```

### Semântica de transição

| De | Para | `in_flight_since` |
|----|------|-------------------|
| `pending` | `in_flight` | `NOW()` |
| `in_flight` | `resolved` | `NULL` |
| `in_flight` | `failed` | `NULL` |
| `in_flight` | `conflict` | `NULL` |
| `in_flight` | `permanent_error` | `NULL` |

### Algoritmo de recovery

```text
SyncEngine.initialize()
  ↓
SELECT * FROM sync_queue
WHERE status = 'in_flight'
  AND in_flight_since < datetime('now', '-5 minutes')
  ↓
Para cada operação stale:
  status = 'pending'
  in_flight_since = NULL
  attempt_count = attempt_count + 1 (opcional)
  next_attempt_at = NOW() + backoff(attempt_count)
```

### Documentos afetados
- **B-01** (Schema — tabela `sync_queue`)
- **B-02** (SyncQueue — Proteção `in_flight`)

### Checklist de aplicação
- [ ] Adicionar `in_flight_since` ao schema em B-01
- [ ] Revisar seção "Proteção `in_flight`" em B-02
- [ ] Definir threshold de stale (recomendado: 5 minutos)
- [ ] Documentar algoritmo de recovery

---

## 🔴 B-FIX-008 — Documentar `rides.sync_status` como cache derivado

### Problema

O documento v1.0 define `rides.sync_status` (`synced`, `pending`, `conflict`) como campo da tabela `rides`, mas a autoridade operacional real está na `sync_queue`. Isso cria **duas fontes de verdade** para o mesmo conceito.

### Correção

> **`rides.sync_status` é um campo derivado (computed/cache) atualizado pelo SyncEngine após o processamento da fila. Ele existe exclusivamente para otimização de consultas da UI e NÃO é autoridade operacional. A autoridade continua sendo `sync_queue.status`.**

### Semântica

```text
rides.sync_status
├── 'synced'    → Não há operações pendentes na sync_queue para esta ride
├── 'pending'   → Existe operação pending/in_flight/failed na sync_queue
├── 'conflict'  → Existe operação com status 'conflict' na sync_queue
└── (nunca alterado diretamente por use cases — apenas pelo SyncEngine)
```

### Regra de atualização

```text
SyncEngine, após processar operação de uma ride:
  ↓
Verifica sync_queue para entity_id = ride.id
  ↓
Se há operações pendentes/failed/in_flight:
  rides.sync_status = 'pending'
Se há operações em conflito:
  rides.sync_status = 'conflict'
Se não há operações:
  rides.sync_status = 'synced'
```

### Documentos afetados
- **B-01** (Schema — tabela `rides`, campo `sync_status`)

### Checklist de aplicação
- [ ] Adicionar nota explicativa ao schema de `rides` em B-01
- [ ] Documentar que `sync_status` é cache derivado
- [ ] Referenciar `sync_queue` como autoridade operacional

---

## 🟡 B-FIX-009 — Política de índices baseada em padrões de acesso

### Problema

O documento v1.0 afirma: "Todo campo usado em WHERE, JOIN ou ORDER BY possui índice explícito". Essa regra é excessivamente rígida e pode gerar índices desnecessários que degradam performance de escrita.

### Correção

> **Todo caminho de consulta crítico deve possuir índice adequado, definido a partir dos padrões de acesso do aplicativo e validado por plano de execução. Índices compostos devem ser preferidos quando a consulta sempre filtra por múltiplas colunas.**

### Critérios para criação de índice

| Critério | Quando criar |
|----------|--------------|
| **Seletividade** | Campo com alta cardinalidade (muitos valores distintos) |
| **Frequência** | Consulta executada frequentemente na UI |
| **Cardinalidade** | Campo usado em JOIN entre tabelas |
| **Compostos** | Quando WHERE sempre inclui múltiplas colunas |
| **Tamanho** | Tabelas grandes (> 1000 registros) beneficiam mais |

### Exemplo de índice composto

```sql
-- Consulta frequente: corridas do motorista X no período Y
SELECT * FROM rides
WHERE driver_id = ? AND scheduled_at BETWEEN ? AND ?

-- Índice adequado (composto):
CREATE INDEX idx_rides_driver_scheduled
ON rides(driver_id, scheduled_at);

-- NÃO criar separadamente:
-- CREATE INDEX idx_rides_driver ON rides(driver_id);      -- desnecessário
-- CREATE INDEX idx_rides_scheduled ON rides(scheduled_at); -- menos eficiente
```

### Documentos afetados
- **B-01** (Schema — princípios do schema, seção 1.1)

### Checklist de aplicação
- [ ] Revisar princípio de índices em B-01
- [ ] Substituir regra absoluta por critérios baseados em padrões de acesso
- [ ] Revisar índices existentes para identificar oportunidades de compostos

---

## 🔴 B-FIX-010 — Índice em `clinics.address_id`

### Problema

A tabela `clinics` possui foreign key `address_id → addresses.id`, mas os índices declarados são apenas `active` e `deleted_at`. Se houver consulta ou JOIN por endereço, a performance será degradada.

### Correção

> **Adicionar índice em `clinics.address_id` se houver qualquer consulta que filtre ou faça JOIN por endereço.**

```sql
CREATE INDEX idx_clinics_address ON clinics(address_id);
```

### Documentos afetados
- **B-01** (Schema — tabela `clinics`)

### Checklist de aplicação
- [ ] Adicionar `idx_clinics_address` aos índices de `clinics` em B-01
- [ ] Verificar se outras tabelas com FK possuem índice correspondente

---

## 🔴 B-FIX-011 — Valores monetários como `INTEGER` em centavos

### Problema

Os campos `financial_entries.amount` e `rides.value` estão definidos como `REAL` no schema SQLite. Para um aplicativo com módulo financeiro, isso introduz risco de imprecisão de ponto flutuante.

### Correção

> **Toda representação de valor monetário no SQLite usa `INTEGER` representando centavos. A conversão para formato decimal ocorre apenas na camada de apresentação (UI).**

### Mapeamento

| Valor real | Representação SQLite | Exibição UI |
|------------|----------------------|-------------|
| R$ 25,50 | 2550 | `2550 / 100 = 25.50` |
| R$ 100,00 | 10000 | `10000 / 100 = 100.00` |
| R$ 0,99 | 99 | `99 / 100 = 0.99` |

### Schema corrigido

```text
financial_entries
├── amount_cents: INTEGER NOT NULL   // ← era REAL

rides
├── value_cents: INTEGER             // ← era REAL
```

### Regras de implementação

1. **Camada de domínio**: trabalha com `int` (centavos)
2. **Camada de apresentação**: converte para `double` ou `Decimal` apenas para exibição
3. **API**: pode usar `number` (JSON) ou `string` (recomendado para evitar problemas de parse)
4. **Servidor**: deve validar que valores são inteiros positivos (ou negativos para despesas)

### Documentos afetados
- **B-01** (Schema — tabelas `financial_entries` e `rides`)
- **B-06** (Matriz de Conflitos — `financial_entries` usa `manual` para `amount_cents`)

### Checklist de aplicação
- [ ] Renomear `amount` → `amount_cents` em `financial_entries`
- [ ] Renomear `value` → `value_cents` em `rides`
- [ ] Atualizar constraints (≥ 0 para receitas, qualquer sinal para despesas)
- [ ] Revisar B-06 para refletir `amount_cents`
- [ ] Documentar regra de conversão centavos ↔ decimal

---

## 🟢 DEC-001 — Decisão arquitetural adicional: SyncEngine não decide regras de negócio

### Decisão

> **O `SyncEngine` não possui autoridade para decidir regras de negócio. Ele coordena sincronização, detecta conflitos e delega sua resolução aos mecanismos de domínio apropriados (`RideStateMachine`, `AuthorizationPolicy`, `ConflictResolver`).**

### Justificativa

A separação de responsabilidades do BLOCO A (`RideStateMachine` + `AuthorizationPolicy`) só tem valor se for respeitada no BLOCO B. Se o SyncEngine começar a conter regras do tipo "cancelada sempre vence", estaremos duplicando lógica de domínio no mecanismo de infraestrutura.

### Hierarquia de decisão

```text
SyncEngine (coordena)
  ↓
  ├── detecta conflito (via resposta do servidor)
  ├── identifica tipo de conflito
  ├── delega para ConflictResolver apropriado
  └── aplica resultado da resolução

RideConflictResolver (decide)
  ↓
  ├── consulta RideStateMachine (transição válida?)
  ├── consulta AuthorizationPolicy (permissão?)
  ├── aplica regra de domínio
  └── retorna ConflictResolution

Servidor (autoridade global)
  ↓
  ├── valida base_version
  ├── valida transição via StateMachine
  ├── valida permissão via Policy
  └── aceita ou rejeita
```

### Documentos afetados
- **B-04** (Contrato do SyncEngine — responsabilidades)
- **B-08** (Autoridade — hierarquia)

### Checklist de aplicação
- [ ] Documentar explicitamente em B-04 que SyncEngine não contém regras de negócio
- [ ] Referenciar B-08 para hierarquia completa
- [ ] Garantir que `RideConflictResolver` consulta StateMachine + Policy

---

## 📎 Apêndice A — Mapeamento de correções por documento

| Documento | Correções aplicáveis |
|-----------|---------------------|
| **B-01** | B-FIX-003, B-FIX-004, B-FIX-007, B-FIX-008, B-FIX-009, B-FIX-010, B-FIX-011 |
| **B-02** | B-FIX-001, B-FIX-002, B-FIX-006, B-FIX-007 |
| **B-03** | B-FIX-003, B-FIX-004, B-FIX-005 |
| **B-04** | B-FIX-004, B-FIX-005, B-FIX-006, DEC-001 |
| **B-05** | B-FIX-001 |
| **B-06** | B-FIX-005, B-FIX-011 |
| **B-08** | B-FIX-006, DEC-001 |

---

## 📎 Apêndice B — Checklist de auditoria v1.1

Após aplicação das correções, a v1.1 deve ser submetida à seguinte auditoria antes do congelamento:

### Auditoria de consistência
- [ ] Todos os documentos B-01 a B-08 referenciam os mesmos nomes de campos, enums e estratégias
- [ ] Não há `conflict_type` ou `resolution_strategy` definidos em um documento e ausentes em outro
- [ ] Schema SQLite é consistente com os contratos Dart descritos

### Auditoria de SQLite
- [ ] Todas as tabelas de domínio possuem `id` (UUID), `created_at`, `updated_at`, `deleted_at`, `version`
- [ ] Todas as foreign keys possuem índice correspondente
- [ ] Campos monetários usam `INTEGER` (centavos)
- [ ] `sync_queue` possui `in_flight_since`
- [ ] `audit_log` não possui `deleted_at`

### Auditoria de SyncEngine
- [ ] `SyncOperation` inclui `audit_context`
- [ ] `ConflictResolution` inclui `domainRule`
- [ ] Estados da fila estão completos e com transições definidas
- [ ] Mecanismo de recovery `in_flight` está documentado

### Auditoria de conflitos
- [ ] `concurrent_edit` não é `conflict_type` independente
- [ ] `resolution_strategy` e `resolution_status` estão separados
- [ ] Matriz B-06 cobre todas as entidades de domínio
- [ ] Regra "estados terminais vencem" está documentada

### Auditoria de autoridade
- [ ] Hierarquia servidor → dispositivo está clara
- [ ] SyncEngine não contém regras de negócio
- [ ] `domain_rule` delega para StateMachine + Policy
- [ ] AuditLog replicação está definida sem ambiguidade

---

> **Registro de Correções elaborado em nível arquitetural.**  
> **Aguardando aplicação no documento BLOCO B v1.0 para geração da v1.1.**  
> **Nenhuma linha de código foi alterada no projeto.**

---
*Registro de Correções — BLOCO B v1.0 → v1.1*  
*Táxi Pet Praiano*  
*Versão 1.0 — 16/08/2026*
