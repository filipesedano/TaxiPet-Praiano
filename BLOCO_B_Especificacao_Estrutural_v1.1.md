# 🚕🐾 BLOCO B — Especificação Estrutural v1.1
## Táxi Pet Praiano — Arquitetura Offline-First

> **Status:** Especificação arquitetural — aguardando aprovação final  
> **Versão anterior:** [BLOCO B v1.0](BLOCO_B_Especificacao_Estrutural_v1.0.md)  
> **Registro de correções:** [BLOCO B v1.0 → v1.1](BLOCO_B_Registro_Correcoes_v1.0_v1.1.md)  
> **Regra:** Nenhuma alteração de código até aprovação formal deste documento  
> **Sequência de leitura:** B-01 → B-02 → B-05 → B-03 → B-06 → B-08 → B-04 → B-07  

---

## 📜 Histórico de Versões

| Versão | Data | Descrição |
|--------|------|-----------|
| v1.0 | 16/08/2026 | Especificação inicial elaborada e publicada |
| v1.1 | 16/08/2026 | Aplicação de 11 correções arquiteturais (B-FIX-001 a B-FIX-011) + decisão DEC-001 |

---

## 📋 Índice

- [B-01 — Modelo SQLite Completo](#b-01--modelo-sqlite-completo)
- [B-02 — SyncQueue: Estados e Ciclo de Vida](#b-02--syncqueue-estados-e-ciclo-de-vida)
- [B-03 — Modelo de Conflitos](#b-03--modelo-de-conflitos)
- [B-04 — Contrato do SyncEngine](#b-04--contrato-do-syncengine)
- [B-05 — Versionamento e base_version](#b-05--versionamento-e-base_version)
- [B-06 — Matriz de Conflitos por Entidade](#b-06--matriz-de-conflitos-por-entidade)
- [B-07 — Fluxos Offline → Online → Sync](#b-07--fluxos-offline--online--sync)
- [B-08 — Autoridade e Regras de Resolução](#b-08--autoridade-e-regras-de-resolução)

---

# B-01 — Modelo SQLite Completo

## 1.1 Princípios do Schema

| Princípio | Descrição |
|-----------|-----------|
| **UUID v4 para identidade** | Toda entidade de domínio possui `id` UUID v4, imutável, gerado no dispositivo ou recebido do servidor |
| **ULID para operações/eventos** | `operation_id`, `audit_id` e `device_id` usam ULID para ordenação temporal implícita |
| **Soft delete universal** | Toda tabela de domínio possui `deleted_at` (nullable datetime); nunca DELETE físico |
| **Timestamps obrigatórios** | `created_at` e `updated_at` em toda tabela de domínio e operacional |
| **Versionamento numérico** | `version` (int, NOT NULL, default 1) em toda entidade versionável |
| **Foreign keys habilitadas** | SQLite com `PRAGMA foreign_keys = ON` |
| **Transações compostas** | Operações que afetam múltiplas tabelas devem usar `BEGIN TRANSACTION … COMMIT/ROLLBACK` |
| **Índices por padrões de acesso** | Todo caminho de consulta crítico possui índice adequado, definido a partir dos padrões de acesso do aplicativo. Índices compostos são preferidos quando a consulta sempre filtra por múltiplas colunas |
| **Valores monetários em centavos** | Toda representação de valor monetário usa `INTEGER` (centavos). Conversão para decimal ocorre apenas na camada de apresentação |

## 1.2 Tabelas de Domínio

### 1.2.1 `users`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | Identidade do usuário |
| `name` | TEXT | NOT NULL | Nome completo |
| `email` | TEXT | UNIQUE, NOT NULL | E-mail |
| `phone` | TEXT | | Telefone |
| `role` | TEXT | NOT NULL | ADMIN, DRIVER, TUTOR |
| `active` | INTEGER (boolean) | DEFAULT 1 | Conta ativa |
| `created_at` | TEXT (ISO 8601) | NOT NULL | |
| `updated_at` | TEXT (ISO 8601) | NOT NULL | |
| `deleted_at` | TEXT (ISO 8601) | | Soft delete |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `email`, `role`, `deleted_at`

---

### 1.2.2 `tutors`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | |
| `user_id` | TEXT (UUID) | FOREIGN KEY → users.id | Vinculação opcional a conta |
| `name` | TEXT | NOT NULL | |
| `phone` | TEXT | NOT NULL | |
| `email` | TEXT | | |
| `notes` | TEXT | | Observações |
| `created_at` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |
| `deleted_at` | TEXT | | |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `user_id`, `phone`, `deleted_at`

---

### 1.2.3 `addresses`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | |
| `tutor_id` | TEXT (UUID) | FOREIGN KEY → tutors.id | |
| `label` | TEXT | | Casa, Trabalho, etc. |
| `street` | TEXT | NOT NULL | |
| `number` | TEXT | | |
| `complement` | TEXT | | |
| `neighborhood` | TEXT | NOT NULL | |
| `city` | TEXT | NOT NULL | |
| `state` | TEXT | NOT NULL | |
| `zip_code` | TEXT | | |
| `latitude` | REAL | | |
| `longitude` | REAL | | |
| `is_default` | INTEGER | DEFAULT 0 | Endereço padrão do tutor |
| `created_at` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |
| `deleted_at` | TEXT | | |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `tutor_id`, `is_default`, `deleted_at`

---

### 1.2.4 `pets`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | |
| `tutor_id` | TEXT (UUID) | FOREIGN KEY → tutors.id | |
| `name` | TEXT | NOT NULL | |
| `species` | TEXT | NOT NULL | cachorro, gato, etc. |
| `breed` | TEXT | | |
| `size` | TEXT | NOT NULL | pequeno, medio, grande |
| `weight` | REAL | | |
| `special_needs` | TEXT | | Necessidades especiais |
| `notes` | TEXT | | |
| `created_at` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |
| `deleted_at` | TEXT | | |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `tutor_id`, `deleted_at`

---

### 1.2.5 `vehicles`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | |
| `driver_id` | TEXT (UUID) | FOREIGN KEY → users.id | |
| `model` | TEXT | NOT NULL | |
| `plate` | TEXT | UNIQUE, NOT NULL | Placa |
| `color` | TEXT | | |
| `year` | INTEGER | | |
| `km_current` | REAL | DEFAULT 0 | Quilometragem atual |
| `active` | INTEGER | DEFAULT 1 | |
| `created_at` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |
| `deleted_at` | TEXT | | |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `driver_id`, `plate`, `active`, `deleted_at`

---

### 1.2.6 `clinics`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | |
| `name` | TEXT | NOT NULL | |
| `phone` | TEXT | | |
| `address_id` | TEXT (UUID) | FOREIGN KEY → addresses.id | Endereço da clínica |
| `active` | INTEGER | DEFAULT 1 | |
| `created_at` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |
| `deleted_at` | TEXT | | |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `address_id`, `active`, `deleted_at`

> **Nota:** `address_id` possui índice porque é utilizado em JOIN com `addresses` em caminhos de consulta críticos.

---

### 1.2.7 `rides` — Entidade Central

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | |
| `tutor_id` | TEXT (UUID) | FOREIGN KEY → tutors.id, NOT NULL | |
| `pet_id` | TEXT (UUID) | FOREIGN KEY → pets.id, NOT NULL | |
| `driver_id` | TEXT (UUID) | FOREIGN KEY → users.id | Motorista atribuído |
| `vehicle_id` | TEXT (UUID) | FOREIGN KEY → vehicles.id | Veículo usado |
| `origin_address_id` | TEXT (UUID) | FOREIGN KEY → addresses.id, NOT NULL | |
| `destination_address_id` | TEXT (UUID) | FOREIGN KEY → addresses.id, NOT NULL | |
| `clinic_id` | TEXT (UUID) | FOREIGN KEY → clinics.id | Clínica (se aplicável) |
| `scheduled_at` | TEXT (ISO 8601) | NOT NULL | Data/hora agendada |
| `type` | TEXT | NOT NULL | ida, volta, ida_e_volta, emergencia |
| `status` | TEXT | NOT NULL | Estado atual da corrida |
| `value_cents` | INTEGER | | Valor cobrado em centavos |
| `notes` | TEXT | | Observações |
| `km_start` | REAL | | KM no início |
| `km_end` | REAL | | KM no final |
| `delivery_code` | TEXT | | Código de entrega (4-6 dígitos) |
| `sync_status` | TEXT | DEFAULT 'synced' | **Cache derivado para UI** — ver nota abaixo |
| `created_at` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |
| `deleted_at` | TEXT | | |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `tutor_id`, `driver_id`, `status`, `scheduled_at`, `sync_status`, `deleted_at`, `version`

**Constraints de domínio:**
- `status` deve ser um valor válido de `RideStatus` (9 estados oficiais)
- `type` deve ser um valor válido de `RideType` (7 tipos oficiais)
- `km_end` ≥ `km_start` (quando ambos preenchidos)
- `value_cents` ≥ 0

> **Nota sobre `sync_status`:** Este campo é um **cache derivado (computed)** atualizado exclusivamente pelo `SyncEngine` após processamento da fila. Ele existe para otimização de consultas da UI e NÃO é autoridade operacional. A autoridade continua sendo `sync_queue.status`.

---

### 1.2.8 `ride_status_history`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (ULID) | PRIMARY KEY | Ordenação temporal implícita |
| `ride_id` | TEXT (UUID) | FOREIGN KEY → rides.id, NOT NULL | |
| `from_status` | TEXT | NOT NULL | Estado anterior |
| `to_status` | TEXT | NOT NULL | Novo estado |
| `reason` | TEXT | | Motivo da mudança |
| `actor_id` | TEXT (UUID) | FOREIGN KEY → users.id | Quem alterou |
| `device_id` | TEXT (ULID) | NOT NULL | Dispositivo que registrou |
| `operation_id` | TEXT (ULID) | NOT NULL | Vinculação com sync_queue |
| `created_at` | TEXT | NOT NULL | |

**Índices:** `ride_id`, `actor_id`, `device_id`, `operation_id`, `created_at`

**Regra:** NUNCA sofre soft delete. É um log imutável.

---

### 1.2.9 `financial_entries`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | |
| `ride_id` | TEXT (UUID) | FOREIGN KEY → rides.id | Vinculação opcional |
| `type` | TEXT | NOT NULL | receita, despesa |
| `category` | TEXT | NOT NULL | corrida, combustivel, manutencao, higiene, outro |
| `description` | TEXT | NOT NULL | |
| `amount_cents` | INTEGER | NOT NULL | Valor em centavos (positivo receita, negativo despesa) |
| `date` | TEXT | NOT NULL | Data do lançamento |
| `payment_method` | TEXT | | dinheiro, pix, cartao, etc. |
| `receipt_url` | TEXT | | Comprovante |
| `created_at` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |
| `deleted_at` | TEXT | | |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `ride_id`, `type`, `category`, `date`, `deleted_at`

> **Nota:** `amount_cents` representa o valor em centavos. Ex: R$ 25,50 → 2550.

---

### 1.2.10 `maintenance`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | |
| `vehicle_id` | TEXT (UUID) | FOREIGN KEY → vehicles.id, NOT NULL | |
| `type` | TEXT | NOT NULL | troca_oleo, revisao, pneu, etc. |
| `description` | TEXT | NOT NULL | |
| `cost_cents` | INTEGER | | Custo em centavos |
| `km_at_maintenance` | REAL | | KM no momento |
| `provider` | TEXT | | Oficina/fornecedor |
| `date` | TEXT | NOT NULL | |
| `created_at` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |
| `deleted_at` | TEXT | | |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `vehicle_id`, `date`, `deleted_at`

---

### 1.2.11 `fuel`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | |
| `vehicle_id` | TEXT (UUID) | FOREIGN KEY → vehicles.id, NOT NULL | |
| `liters` | REAL | NOT NULL | |
| `price_per_liter_cents` | INTEGER | NOT NULL | Preço por litro em centavos |
| `total_cost_cents` | INTEGER | NOT NULL | Custo total em centavos |
| `km_at_refuel` | REAL | NOT NULL | |
| `station_name` | TEXT | | |
| `date` | TEXT | NOT NULL | |
| `created_at` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |
| `deleted_at` | TEXT | | |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `vehicle_id`, `date`, `deleted_at`

---

### 1.2.12 `hygiene`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | |
| `vehicle_id` | TEXT (UUID) | FOREIGN KEY → vehicles.id, NOT NULL | |
| `type` | TEXT | NOT NULL | limpeza, desinfeccao, troca_capa |
| `cost_cents` | INTEGER | | Custo em centavos |
| `km_at_cleaning` | REAL | | |
| `date` | TEXT | NOT NULL | |
| `created_at` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |
| `deleted_at` | TEXT | | |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `vehicle_id`, `date`, `deleted_at`

---

### 1.2.13 `proofs` (comprovantes/fotos)

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (UUID) | PRIMARY KEY | |
| `ride_id` | TEXT (UUID) | FOREIGN KEY → rides.id | |
| `financial_entry_id` | TEXT (UUID) | FOREIGN KEY → financial_entries.id | |
| `type` | TEXT | NOT NULL | foto_pet, assinatura, comprovante, documento |
| `file_path` | TEXT | NOT NULL | Caminho local do arquivo |
| `file_hash` | TEXT | NOT NULL | Hash SHA-256 para integridade |
| `uploaded` | INTEGER | DEFAULT 0 | Já enviado para servidor? |
| `created_at` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |
| `deleted_at` | TEXT | | |
| `version` | INTEGER | DEFAULT 1, NOT NULL | |

**Índices:** `ride_id`, `financial_entry_id`, `uploaded`, `deleted_at`

---

## 1.3 Tabelas Operacionais

### 1.3.1 `sync_queue`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | INTEGER | PRIMARY KEY, AUTOINCREMENT | Ordem de inserção |
| `operation_id` | TEXT (ULID) | UNIQUE, NOT NULL | Idempotência |
| `group_id` | TEXT (ULID) | | Agrupa operações compostas |
| `entity_type` | TEXT | NOT NULL | RIDE, FINANCIAL, PET, etc. |
| `entity_id` | TEXT (UUID) | NOT NULL | |
| `operation_type` | TEXT | NOT NULL | CREATE, UPDATE, SOFT_DELETE |
| `payload` | TEXT (JSON) | NOT NULL | Delta ou objeto completo |
| `base_version` | INTEGER | NOT NULL | Versão da entidade no momento da alteração |
| `created_at` | TEXT | NOT NULL | |
| `attempt_count` | INTEGER | DEFAULT 0 | |
| `last_attempt_at` | TEXT | | |
| `next_attempt_at` | TEXT | | Calculado pelo backoff |
| `in_flight_since` | TEXT | NULL | Timestamp de início do envio |
| `status` | TEXT | DEFAULT 'pending' | pending, in_flight, failed, conflict, permanent_error, resolved |
| `last_error` | TEXT | | Mensagem/JSON do último erro |
| `resolved_at` | TEXT | | |

**Índices:** `operation_id`, `entity_type`, `entity_id`, `status`, `next_attempt_at`, `group_id`

> **Nota sobre `in_flight_since`:** Usado para detectar operações "stale" após crash. Se `status = 'in_flight'` e `in_flight_since` > 5 minutos no passado, a operação é resetada para `pending` no `SyncEngine.initialize()`.

---

### 1.3.2 `sync_conflicts`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | INTEGER | PRIMARY KEY, AUTOINCREMENT | |
| `operation_id` | TEXT (ULID) | FOREIGN KEY → sync_queue.operation_id, NOT NULL | |
| `entity_type` | TEXT | NOT NULL | |
| `entity_id` | TEXT (UUID) | NOT NULL | |
| `local_version` | INTEGER | NOT NULL | Versão local no momento do conflito |
| `server_version` | INTEGER | NOT NULL | Versão no servidor |
| `local_payload` | TEXT (JSON) | NOT NULL | Payload local |
| `server_payload` | TEXT (JSON) | NOT NULL | Payload do servidor |
| `conflict_type` | TEXT | NOT NULL | **Protocolo:** version_mismatch, state_invalid, permission_denied, merge_failed |
| `conflict_cause` | TEXT | | **Semântica:** concurrent_edit, stale_version, unknown (nullable) |
| `resolution_strategy` | TEXT | | **Como resolveu:** auto_merge, server_wins, local_wins, domain_rule, manual |
| `resolution_status` | TEXT | DEFAULT 'open' | **Estágio:** open, resolved, escalated |
| `resolution` | TEXT (JSON) | | Detalhes do resultado |
| `resolved_by` | TEXT (UUID) | FOREIGN KEY → users.id | |
| `resolved_at` | TEXT | | |
| `created_at` | TEXT | NOT NULL | |

**Índices:** `operation_id`, `entity_id`, `resolution_status`, `conflict_type`, `created_at`

> **Nota v1.1:** `conflict_type` e `conflict_cause` estão separados. `resolution_strategy` e `resolution_status` são campos independentes.

---

### 1.3.3 `audit_log`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `id` | TEXT (ULID) | PRIMARY KEY | |
| `actor_id` | TEXT (UUID) | FOREIGN KEY → users.id | Quem executou |
| `entity_type` | TEXT | NOT NULL | |
| `entity_id` | TEXT (UUID) | NOT NULL | |
| `action` | TEXT | NOT NULL | CREATE, UPDATE, SOFT_DELETE, STATUS_CHANGE, SYNC, LOGIN, etc. |
| `old_value` | TEXT (JSON) | | Estado anterior |
| `new_value` | TEXT (JSON) | | Novo estado |
| `device_id` | TEXT (ULID) | NOT NULL | |
| `operation_id` | TEXT (ULID) | | Vinculação com sync_queue |
| `ip_address` | TEXT | | IP (quando disponível) |
| `created_at` | TEXT | NOT NULL | |

**Índices:** `actor_id`, `entity_id`, `action`, `device_id`, `operation_id`, `created_at`

**Regra:** NUNCA soft delete. NUNCA alterado após inserção. Imutável.

---

## 1.4 Tabela de Controle

### 1.4.1 `app_metadata`

| Campo | Tipo | Constraints | Descrição |
|-------|------|-------------|-----------|
| `key` | TEXT | PRIMARY KEY | |
| `value` | TEXT | NOT NULL | |
| `updated_at` | TEXT | NOT NULL | |

**Registros esperados:**

| `key` | `value` | Descrição |
|-------|---------|-----------|
| `database_version` | `"3"` | Versão do schema (migrations) |
| `device_id` | `"01J…"` | ULID único por instalação |
| `installation_id` | `"uuid…"` | UUID da instalação do app |
| `last_sync_cursor` | `"cursor_token_123"` | Token opaco do servidor |
| `last_sync_at` | `"2026-08-16T10:00:00Z"` | Timestamp da última sync (referência apenas) |
| `app_version` | `"1.2.3"` | Versão do app |
| `user_id` | `"uuid…"` | Usuário logado atual |

---

## 1.5 Migrations

### Estratégia

- **Migration-based** (não dump/restore)
- Cada migration é um arquivo numerado: `migration_001.dart`, `migration_002.dart`, etc.
- Tabela interna `_migrations` rastreia quais já foram aplicadas
- Migrations são **idempotentes** quando possível (CREATE TABLE IF NOT EXISTS)
- Nunca alterar migrations já aplicadas em produção

### Exemplo de pipeline

```text
migration_001.dart  → Cria tabelas de domínio base
migration_002.dart  → Cria tabelas operacionais (sync_queue, audit_log)
migration_003.dart  → Adiciona índices faltantes
migration_004.dart  → Adiciona campo rides.delivery_code
```

---

# B-02 — SyncQueue: Estados e Ciclo de Vida

## 2.1 Estados da Fila

```text
                    ┌─────────────┐
                    │   PENDING   │
                    │  (inicial)  │
                    └──────┬──────┘
                           │ SyncEngine.pick()
                           ▼
                    ┌─────────────┐
                    │  IN_FLIGHT  │
                    │ (enviando)  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌─────────┐  ┌──────────┐  ┌──────────┐
        │RESOLVED │  │  FAILED  │  │ CONFLICT │
        │(sucesso)│  │ (retry)  │  │(resolver)│
        └─────────┘  └────┬─────┘  └────┬─────┘
                          │             │
                          │ retry       │ resolvido
                          ▼             ▼
                    ┌─────────────┐  ┌─────────┐
                    │   PENDING   │  │RESOLVED │
                    │(reagendado) │  │         │
                    └─────────────┘  └─────────┘
                           │
                    max attempts
                           ▼
                    ┌─────────────┐
                    │PERMANENT_ERR│
                    │ (não retry) │
                    └─────────────┘
```

## 2.2 Transições de Estado

| De | Para | Gatilho | Regras |
|----|------|---------|--------|
| `pending` | `in_flight` | SyncEngine inicia envio | Atualiza `last_attempt_at`, incrementa `attempt_count`, define `in_flight_since = NOW()` |
| `in_flight` | `resolved` | Servidor confirma (2xx) | `in_flight_since = NULL`; remove da fila ou marca como resolved |
| `in_flight` | `failed` | Erro retornável (5xx, timeout) | `in_flight_since = NULL`; calcula `next_attempt_at` via backoff |
| `in_flight` | `conflict` | Servidor retorna 409 | `in_flight_since = NULL`; cria entrada em `sync_conflicts` |
| `in_flight` | `permanent_error` | Erro não retornável (4xx, validação) | `in_flight_since = NULL`; não faz retry; requer intervenção |
| `failed` | `pending` | Retry scheduler | Quando `now() >= next_attempt_at` |
| `failed` | `permanent_error` | `attempt_count >= max_attempts` | Max = 10 (configurável) |
| `conflict` | `resolved` | ConflictResolver resolve | Atualiza `sync_conflicts.resolution_status` |

## 2.3 Proteção `in_flight` e Recovery

### Problema

Se o app for encerrado (crash, bateria, usuário força fechamento) durante o envio de uma operação `in_flight`, a operação pode ficar "perdida" na fila.

### Solução

1. **Timestamp explícito**: toda operação `in_flight` tem `in_flight_since` (timestamp)
2. **Stale detection**: ao iniciar o SyncEngine, verifica operações `in_flight` com `in_flight_since > 5 minutos`
3. **Auto-recovery**: operações stale são resetadas para `pending`
4. **Timeout de rede**: toda requisição HTTP tem timeout <= 30s

```text
SyncEngine.initialize()
  ↓
Verifica in_flight stale
  ↓
┌─────────────────┐
│ in_flight_since │
│   > 5 min?      │
└────────┬────────┘
    SIM  │  NÃO
     ↓   │   ↓
 status  │  mantém
= pending │  in_flight
in_flight│
_since   │
= NULL   │
```

## 2.4 Operações Compostas (`group_id`)

### Regra Fundamental v1.1

> **Dentro de um mesmo `group_id`, uma entidade só pode possuir no máximo uma operação de mutação efetiva (`CREATE`, `UPDATE`, `SOFT_DELETE`).**

### Cenário

Concluir uma corrida envolve:
1. Atualizar `rides.status` → `concluida`
2. Atualizar `rides.km_end`
3. Criar `financial_entries` (receita)
4. Criar `ride_status_history`

### Solução: Payload Consolidado

Em vez de duas operações `UPDATE` para `rides`, consolidamos em uma:

```text
operation_id: 01JABC...
entity_type:  RIDE
entity_id:    ride-123
operation:    UPDATE
payload: {
  "status": "concluida",
  "km_end": 45230,
  "updated_at": "2026-08-16T10:00:00Z"
}
base_version: 5
group_id: 01JXYZ...
```

### Semântica de `group_id`

- Operações com mesmo `group_id` devem ser processadas pelo servidor como uma **unidade atômica**
- Se uma falhar, o servidor deve rejeitar o grupo inteiro
- O retry deve reenviar o grupo inteiro
- A idempotência (`operation_id`) protege cada operação individualmente

### Exemplo Completo de Grupo

```text
GROUP 01JXYZ:

Op 1:
  operation_id: 01JABC...
  entity_type:  RIDE
  entity_id:    ride-123
  operation:    UPDATE
  payload:      {status: concluida, km_end: 45230}
  base_version: 5

Op 2:
  operation_id: 01JDEF...
  entity_type:  FINANCIAL
  entity_id:    (novo UUID)
  operation:    CREATE
  payload:      {type: receita, amount_cents: 2550, ...}
  base_version: N/A (nova entidade)

Op 3:
  operation_id: 01JGHI...
  entity_type:  RIDE_STATUS_HISTORY
  entity_id:    (novo ULID)
  operation:    CREATE
  payload:      {ride_id: ride-123, from_status: em_andamento, to_status: concluida}
  base_version: N/A (nova entidade)
```

## 2.5 Compactação da Fila

### Quando aplicar

Se houver múltiplas operações `pending` para a **mesma entidade** e **mesmo tipo** (ex: 3 `UPDATE` seguidos em `rides`), pode-se consolidar em uma única operação.

### Regras de segurança

| Pode consolidar? | Condição |
|------------------|----------|
| ✅ Sim | Apenas `UPDATE` + `UPDATE` na mesma entidade, nenhuma `CREATE` ou `SOFT_DELETE` no meio |
| ❌ Não | Há `CREATE` ou `SOFT_DELETE` para a mesma entidade na fila |
| ❌ Não | Operações têm `group_id` diferentes |
| ❌ Não | A primeira operação já está `in_flight` ou `failed` |

### Algoritmo de compactação v1.1

> **A compactação preserva o `base_version` da operação mais antiga da sequência, pois ela representa a pré-condição original sobre a qual toda a transformação foi construída.**

Além disso:

> **A compactação só é válida enquanto todas as operações participantes forem derivadas da mesma cadeia causal local e não houver evidência de alteração remota entre a primeira operação e a compactação.**

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

## 2.6 `audit_context` na SyncOperation

### Estrutura

Cada `SyncOperation` pode conter um `audit_context` que será extraído pelo servidor para gerar o registro de auditoria no lado do servidor:

```text
SyncOperation
├── operation_id
├── entity_type
├── entity_id
├── operation_type
├── payload
├── base_version
├── group_id
└── audit_context: AuditContext

AuditContext
├── actor_id: UUID
├── device_id: ULID
├── action: String
├── old_value: JSON
├── new_value: JSON
└── timestamp: ISO 8601
```

### Fluxo

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

---

# B-03 — Modelo de Conflitos

## 3.1 Tipos de Conflito (Protocolo)

| Tipo | Descrição | Detecção |
|------|-----------|----------|
| `version_mismatch` | `base_version` enviado ≠ `version` atual no servidor | Servidor compara versões |
| `state_invalid` | A alteração proposta viola `RideStateMachine` | Servidor valida transição |
| `permission_denied` | O `actor_id` não tem permissão para a operação | Servidor consulta `AuthorizationPolicy` |
| `merge_failed` | Campos diferentes foram alterados local e no servidor, mas o merge automático falhou | Servidor tenta merge, falha |

## 3.2 Causas de Conflito (Semântica)

O campo `conflict_cause` explica **por que** um `version_mismatch` ocorreu:

| Causa | Descrição |
|-------|-----------|
| `concurrent_edit` | Dois ou mais dispositivos editaram a mesma entidade offline |
| `stale_version` | O dispositivo estava muito desatualizado (não sincronizava há muito tempo) |
| `unknown` | Causa não determinada |

> **Nota:** `concurrent_edit` não é um `conflict_type` independente. É uma `cause` do tipo `version_mismatch`.

## 3.3 Detecção de Conflito

### Fluxo no Servidor

```text
Recebe operação
  ↓
Carrega entidade atual (versão servidor)
  ↓
base_version == server_version?
  ┌────────┴────────┐
  SIM               NÃO
   │                 │
   ▼                 ▼
aplica           version_mismatch
validações       conflict_cause = concurrent_edit | stale_version
   │                 │
   ▼                 ▼
RideStateMachine?  registra
  ┌────┴────┐      sync_conflicts
  válido   inválido
   │         │
   ▼         ▼
 aplica    state_invalid
   │         │
   ▼         ▼
Authorization? registra
  ┌────┴────┐  sync_conflicts
 permitido negado
   │         │
   ▼         ▼
 aplica    permission_denied
   │         │
   ▼         ▼
 retorna   registra
  200 OK   sync_conflicts
```

## 3.4 Resolução de Conflito

### Estratégias de Resolução

| Estratégia | Quando usar | Implementação |
|------------|-------------|---------------|
| `auto_merge` | Campos alterados são disjuntos (ex: local mudou `notes`, servidor mudou `status`) | Servidor faz merge dos payloads |
| `server_wins` | Regra de domínio define que servidor tem prioridade | Sobrescreve local com servidor |
| `local_wins` | Regra de domínio define que local tem prioridade | Reenvia operação com versão atualizada |
| `domain_rule` | `RideStateMachine` ou `AuthorizationPolicy` decide | Executa regra específica de domínio |
| `manual` | Nenhuma regra automática resolve | Notifica usuário/administrador |

### Hierarquia de Resolução

```text
1. Verifica se é auto_merge (campos disjuntos)
      ↓ SIM → resolve automaticamente
      ↓ NÃO
2. Verifica domain_rule (StateMachine/Policy)
      ↓ resolve → aplica regra
      ↓ não resolve
3. Verifica regra por entidade (matriz B-06)
      ↓ resolve → aplica regra
      ↓ não resolve
4. Escalona para manual
      ↓ notifica administrador
```

## 3.5 Conflitos e `RideStateMachine`

### Regra Fundamental

> **A `RideStateMachine` é consultada DURANTE a resolução de conflito, não substituída por ela.**

### Exemplo

```text
Servidor:  Ride #123  status=confirmada  version=11
           Administrador cancelou

Local:     Ride #123  status=confirmada  version=10
           Motorista iniciou deslocamento
           → quer mudar para motoristaACaminho
```

**Conflito detectado:** `version_mismatch` (10 ≠ 11), cause=`concurrent_edit`

**Resolução:**
1. Servidor carrega estado atual: `cancelada`
2. Consulta `RideStateMachine`: `cancelada` → `motoristaACaminho`? ❌ INVÁLIDO
3. Resultado: `state_invalid` (detectado durante validação, mas classificado como resolução `domain_rule`)
4. Ação: rejeita operação local, notifica motorista

---

# B-04 — Contrato do SyncEngine

## 4.1 Responsabilidades

O `SyncEngine` é um **coordenador**. Ele NÃO contém regras de negócio.

| Responsabilidade | Descrição |
|------------------|-----------|
| **Orquestração** | Decide quando sincronizar, em que ordem, com que batch size |
| **Transporte** | Envia/recebe dados da API |
| **Estado da fila** | Gerencia transições de estado da `sync_queue` |
| **Retry** | Implementa backoff exponencial |
| **Detecção** | Identifica quando há conflito (via resposta do servidor) |
| **Escalonamento** | Delega resolução para `ConflictResolver` apropriado |

## 4.2 O que o SyncEngine NÃO faz

| Não faz | Quem faz |
|---------|----------|
| Decidir se uma transição de estado é válida | `RideStateMachine` |
| Decidir se um usuário pode alterar algo | `AuthorizationPolicy` |
| Fazer merge de payloads específicos de entidade | `RideConflictResolver`, `FinancialConflictResolver`, etc. |
| Validar regras de negócio | Domínio / Use Cases |

> **DEC-001:** O `SyncEngine` não possui autoridade para decidir regras de negócio. Ele coordena sincronização, detecta conflitos e delega sua resolução aos mecanismos de domínio apropriados.

## 4.3 Interface Conceitual

```text
SyncEngine
│
├── initialize()
│   └── Verifica in_flight stale, reseta para pending
│
├── enqueue(SyncOperation operation)
│   └── Insere na sync_queue, retorna operation_id
│
├── enqueueGroup(List<SyncOperation> operations)
│   └── Gera group_id, insere todas com mesmo group_id
│
├── sync({bool force = false})
│   └── Fluxo principal de sincronização
│
├── syncBatch(int limit)
│   └── Processa até N operações pendentes
│
├── retryFailed()
│   └── Reprocessa operações failed com next_attempt_at expirado
│
├── resolveConflict(String conflictId, Resolution resolution)
│   └── Aplica resolução manual ou automática
│
├── getPendingCount() → int
│
├── getConflictCount() → int
│
├── getSyncStatus() → SyncStatus
│   └── idle, syncing, error, conflicts_pending
│
└── observeSyncStatus() → Stream<SyncStatus>
```

## 4.4 SyncOperation (DTO de entrada)

```text
SyncOperation
├── operation_id: ULID (opcional, gera se omitido)
├── entity_type: String
├── entity_id: UUID
├── operation_type: CREATE | UPDATE | SOFT_DELETE
├── payload: Map<String, dynamic>
├── base_version: int
├── group_id: ULID? (opcional)
└── audit_context: AuditContext? (opcional)
```

## 4.5 SyncResult (DTO de resposta)

```text
SyncResult
├── operation_id: ULID
├── status: success | conflict | error | rejected
├── server_version: int? (se success)
├── conflict_id: String? (se conflict)
├── error_code: String? (se error/rejected)
└── error_message: String?
```

## 4.6 ConflictResolver (Interface)

```text
abstract class ConflictResolver {
  String get entityType;

  ConflictResolution resolve({
    required Entity localEntity,
    required Entity serverEntity,
    required SyncOperation operation,
    required ConflictType type,
    required ConflictCause? cause,
  });
}

ConflictResolution
├── strategy: auto_merge | server_wins | local_wins | domain_rule | manual
├── mergedPayload: Map? (se auto_merge)
├── reason: String
└── requiresUserInput: bool
```

## 4.7 Registro de Resolvers

```text
SyncEngine registra:
  "RIDE" → RideConflictResolver
  "FINANCIAL" → FinancialConflictResolver
  "PET" → PetConflictResolver
  "TUTOR" → TutorConflictResolver
  "VEHICLE" → VehicleConflictResolver
  ...
```

---

# B-05 — Versionamento e base_version

## 5.1 Decisão: UUID para Identidade, ULID para Operações

| Conceito | Formato | Justificativa |
|----------|---------|---------------|
| `id` de entidade | UUID v4 | Identidade estável, independente de tempo |
| `operation_id` | ULID | Ordenação temporal para fila e debug |
| `device_id` | ULID | Ordenação temporal para auditoria |
| `audit_id` | ULID | Ordenação temporal para histórico |

## 5.2 Semântica de `version`

### Regras

1. `version` é um **inteiro positivo** que começa em 1
2. Cada `UPDATE` ou `SOFT_DELETE` incrementa `version` em 1
3. `CREATE` define `version = 1`
4. O servidor é a única autoridade para definir a `version` definitiva
5. O dispositivo local pode ter `version` "provisória" até sincronizar

### Ciclo de Vida da Versão

```text
Criação local:
  version = 1 (provisória)
  ↓
Sync CREATE
  ↓
Servidor confirma
  ↓
version = 1 (confirmada)

Alteração local:
  version = 1 → 2 (provisória)
  base_version = 1 (para sync_queue)
  ↓
Sync UPDATE
  ↓
Servidor confirma
  ↓
version = 2 (confirmada)
```

## 5.3 `base_version` na SyncQueue

### Definição

`base_version` é a versão da entidade **no momento em que a alteração foi feita localmente**.

### Uso

```text
Servidor recebe:
  operation_type: UPDATE
  entity_id: ride-123
  base_version: 7

Servidor verifica:
  SELECT version FROM rides WHERE id = 'ride-123'
  → version = 7

Resultado: ✅ Sem conflito, aplica alteração
  → version = 8

---

Servidor recebe:
  base_version: 7

Servidor verifica:
  → version = 9

Resultado: ⚠️ Conflito! Alguém alterou entre o offline e o sync
```

## 5.4 Versionamento em Operações Compostas

### Regra v1.1

> **Uma entidade só pode possuir uma mutação efetiva por `group_id`.**

Todas as operações de um `group_id` devem ter o **mesmo `base_version`** da entidade principal (quando aplicável).

### Exemplo

```text
Ride #123: version = 5

Operação composta (concluir corrida):
  Op 1: UPDATE rides {status: concluida, km_end: 45230}  base_version: 5
  Op 2: CREATE financial_entries                          base_version: N/A (nova entidade)
  Op 3: CREATE ride_status_history                        base_version: N/A (nova entidade)
```

O servidor valida `base_version` da ride uma única vez e aplica o delta completo.

## 5.5 Versionamento e Soft Delete

```text
Soft delete local:
  UPDATE rides SET deleted_at = '2026-08-16T10:00:00Z', version = version + 1
  sync_queue: operation_type = SOFT_DELETE, base_version = N

Servidor:
  Se version == base_version:
    Aplica soft delete
    version = N + 1
  Se version != base_version:
    Conflito (pode rejeitar se entidade já foi alterada)
```

---

# B-06 — Matriz de Conflitos por Entidade

## 6.1 Princípio

Cada entidade tem uma **estratégia de resolução padrão** e **regras específicas por campo**.

## 6.2 `rides`

### Estratégia Padrão

> **`domain_rule` via `RideStateMachine` + `AuthorizationPolicy`**

### Regras por Campo/Contexto

| Campo Alterado Local | Campo Alterado Servidor | Estratégia | Justificativa |
|----------------------|------------------------|------------|---------------|
| `status` | `status` | `domain_rule` | StateMachine decide qual transição é válida |
| `status` | `notes` | `auto_merge` | Campos independentes |
| `driver_id` | `driver_id` | `server_wins` | Atribuição é decisão administrativa |
| `value_cents` | `value_cents` | `manual` | Financeiro não pode ser resolvido automaticamente |
| `km_end` | `km_end` | `manual` | Dado factual, requer verificação |
| `scheduled_at` | `scheduled_at` | `server_wins` | Agendamento é autoridade do admin |
| `notes` | `notes` | `auto_merge` (concatena) | Ambos podem adicionar informações |
| `delivery_code` | `delivery_code` | `server_wins` | Código é gerado/validado pelo servidor |

### Regra de Estado

```text
Se servidor.status = cancelada:
  Qualquer alteração local de status → REJEITADA (state_invalid)

Se servidor.status = concluida:
  Qualquer alteração local de status → REJEITADA (state_invalid)
  Alterações de km_end, value_cents → CONFLITO MANUAL
```

## 6.3 `financial_entries`

### Estratégia Padrão

> **`manual` para alterações em `amount_cents`; `auto_merge` para outros campos`**

### Regras

| Campo Alterado | Estratégia | Justificativa |
|----------------|------------|---------------|
| `amount_cents` | `manual` | Nunca sobrescrever silenciosamente valores financeiros |
| `date` | `manual` | Data contábil é sensível |
| `description` | `auto_merge` | Texto informativo |
| `category` | `server_wins` | Categorização é regra de negócio |
| `payment_method` | `auto_merge` | Se disjunto |

### Regra de Imutabilidade Parcial

```text
Se entry.type = 'receita' AND entry.ride_id IS NOT NULL:
  amount_cents só pode ser alterado pelo servidor ou admin
  motorista não pode alterar valor de receita vinculada
```

## 6.4 `users`, `tutors`, `pets`, `vehicles`

### Estratégia Padrão

> **`auto_merge` para campos disjuntos; `server_wins` para campos críticos; `manual` para exclusão**`

### Regras

| Entidade | Campo Crítico | Estratégia |
|----------|---------------|------------|
| `users` | `role`, `active` | `server_wins` |
| `tutors` | `phone` | `auto_merge` (valida duplicidade) |
| `pets` | `tutor_id` | `server_wins` |
| `vehicles` | `driver_id`, `plate` | `server_wins` |

## 6.5 `ride_status_history`

### Estratégia

> **Nunca conflita. É append-only.**

Se dois dispositivos criarem histórico para a mesma ride, ambos são válidos desde que a transição de estado seja válida.

## 6.6 `audit_log`

### Estratégia

> **Nunca conflita. É append-only e imutável.**

Não entra na sync_queue como operação enviada. É gerado localmente e replicado para o servidor via `audit_context` anexado à `SyncOperation`.

## 6.7 Resumo da Matriz

| Entidade | Padrão | Campos Críticos | Nunca Auto-merge |
|----------|--------|-----------------|------------------|
| `rides` | `domain_rule` | `status`, `driver_id`, `value_cents` | `status` com servidor |
| `financial_entries` | `manual` (amount) | `amount_cents`, `date` | `amount_cents` |
| `users` | `server_wins` (críticos) | `role`, `active` | `role` |
| `tutors` | `auto_merge` | — | — |
| `pets` | `auto_merge` | `tutor_id` | `tutor_id` |
| `vehicles` | `server_wins` (críticos) | `driver_id`, `plate` | `driver_id` |
| `ride_status_history` | `append` | — | — |
| `audit_log` | `append` | — | — |

---

# B-07 — Fluxos Offline → Online → Sync

## 7.1 Fluxo Completo: Operação Offline

```text
┌─────────┐     ┌─────────┐     ┌─────────────┐     ┌──────────┐
│  USUÁRIO │────▶│   UI    │────▶│  USE CASE   │────▶│  DOMAIN  │
└─────────┘     └─────────┘     └─────────────┘     └────┬─────┘
                                                          │
                                                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SQLITE TRANSACTION                             │
│  ┌─────────────┐  ┌──────────────────┐  ┌─────────┐  ┌────────┐ │
│  │   ENTITY    │  │ STATUS HISTORY   │  │ AUDIT   │  │ SYNC   │ │
│  │  (UPDATE)   │  │ (INSERT)         │  │ (INSERT)│  │ QUEUE  │ │
│  │             │  │                  │  │         │  │(INSERT)│ │
│  │ version++   │  │ ride_id          │  │ actor   │  │ op_id  │ │
│  │ updated_at  │  │ from→to          │  │ entity  │  │ type   │ │
│  │ sync_status │  │ actor_id         │  │ action  │  │ payload│ │
│  │ = pending   │  │ device_id        │  │ old/new │  │ base_v │ │
│  └─────────────┘  └──────────────────┘  └─────────┘  └────────┘ │
│                                                                 │
│  COMMIT (ou ROLLBACK se qualquer passo falhar)                  │
└─────────────────────────────────────────────────────────────────┘
                                                          │
                                                          ▼
                                                  ┌─────────────┐
                                                  │   RETURN    │
                                                  │  UI atualiza│
                                                  │  (offline)  │
                                                  └─────────────┘
```

## 7.2 Fluxo Completo: Sincronização

```text
┌─────────────────┐
│  CONNECTIVITY   │
│  (online detect)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  SyncEngine     │
│  .initialize()  │
│  (stale check)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  sync_queue     │
│  WHERE status = │
│  'pending'      │
│  ORDER BY       │
│  created_at ASC │
│  LIMIT batch    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│  Para cada op:  │────▶│  status =       │
│                 │     │  'in_flight'    │
│  attempt_count++│     │  in_flight_since│
│  last_attempt   │     │  = now()        │
└─────────────────┘     └────────┬────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────┐
│                         API REQUEST                          │
│  POST /sync/batch                                            │
│  Body: [{operation_id, entity_type, entity_id, operation,    │
│          payload, base_version, group_id, audit_context}]    │
└─────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
                    ┌────────────────────┐
                    │      SERVIDOR      │
                    │  Para cada op:     │
                    │  1. Verifica versão│
                    │  2. Valida domínio │
                    │  3. Aplica ou      │
                    │     rejeita        │
                    └─────────┬──────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
        ┌─────────┐    ┌──────────┐    ┌──────────┐
        │ SUCCESS │    │ CONFLICT │    │  ERROR   │
        │ 200 OK  │    │  409     │    │ 4xx/5xx  │
        └────┬────┘    └────┬─────┘    └────┬─────┘
             │              │               │
             ▼              ▼               ▼
        ┌─────────┐   ┌──────────┐   ┌──────────┐
        │ version │   │ sync_    │   │ status = │
        │ = new   │   │ conflicts│   │ 'failed' │
        │ status =│   │ (INSERT) │   │ last_err │
        │'resolved'│  │ status = │   │ = error  │
        │ remove  │   │ 'conflict'│  │ next_at  │
        │ da fila │   │          │   │ = backoff│
        └─────────┘   └──────────┘   └──────────┘
```

## 7.3 Fluxo de Retry

```text
┌─────────────────┐
│  Timer/Trigger  │
│  (a cada 30s    │
│   ou app resume)│
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│  sync_queue                 │
│  WHERE status = 'failed'    │
│  AND next_attempt_at <= now │
│  AND attempt_count < max    │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│  status = 'pending'         │
│  (reentra no fluxo normal)  │
└─────────────────────────────┘
```

## 7.4 Fluxo de Resolução de Conflito

```text
┌─────────────────┐
│  sync_conflicts │
│  WHERE          │
│  resolution_    │
│  status = 'open'│
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│  ConflictResolver.resolve()             │
│  - RideConflictResolver                 │
│  - FinancialConflictResolver            │
│  - etc.                                 │
└─────────────────────────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌─────────┐
│ AUTO  │ │ MANUAL  │
│       │ │         │
│ aplica│ │ notifica│
│ direto│ │ usuário │
└───┬───┘ └────┬────┘
    │          │
    ▼          ▼
┌─────────────────┐
│  sync_conflicts │
│  resolution_    │
│  status =       │
│  'resolved'     │
└─────────────────┘
```

## 7.5 Fluxo de Recebimento (Servidor → Dispositivo)

```text
Servidor tem alterações
  ↓
Dispositivo solicita sync
  ↓
Servidor envia delta
  ↓
Dispositivo aplica localmente
  ↓
Regras:
  - Se entidade local não foi alterada (version igual): aplica direto
  - Se entidade local foi alterada: detecta conflito
  - Se entidade local é mais nova: mantém local, registra conflito
```

### Endpoint de Pull

```text
GET /sync/changes?cursor={sync_cursor}&limit=100

Response:
{
  "cursor": "next_cursor_token",
  "has_more": true,
  "changes": [
    {
      "entity_type": "RIDE",
      "entity_id": "uuid...",
      "operation": "UPDATE",
      "payload": { ... },
      "server_version": 12,
      "server_timestamp": "2026-08-16T10:00:00Z"
    }
  ]
}
```

---

# B-08 — Autoridade e Regras de Resolução

## 8.1 Hierarquia de Autoridade

```text
┌─────────────────────────────────────────┐
│           AUTORIDADE GLOBAL             │
│              SERVIDOR                   │
│  PostgreSQL + API + Regras de Negócio   │
└─────────────────────────────────────────┘
                    │
                    │ Sincronização
                    │ (fonte de verdade)
                    ▼
┌─────────────────────────────────────────┐
│         AUTORIDADE OPERACIONAL          │
│              DISPOSITIVO                │
│  SQLite + StateMachine + Authorization  │
│  (funciona offline, submete ao servidor)│
└─────────────────────────────────────────┘
```

## 8.2 Regras de Autoridade por Cenário

### Cenário 1: Dispositivo Offline

| Decisão | Autoridade |
|---------|------------|
| Aceitar alteração local | Dispositivo (via StateMachine + Policy) |
| Persistir alteração | SQLite local |
| Enfileirar para sync | SyncQueue local |
| Garantir idempotência | operation_id local |

### Cenário 2: Sincronização Enviando (Push)

| Decisão | Autoridade |
|---------|------------|
| Validar base_version | Servidor |
| Validar transição de estado | Servidor (via StateMachine replicada) |
| Validar permissões | Servidor (via Policy replicada) |
| Definir version definitiva | Servidor |
| Aceitar ou rejeitar | Servidor |

### Cenário 3: Sincronização Recebendo (Pull)

| Decisão | Autoridade |
|---------|------------|
| Enviar delta de alterações | Servidor |
| Definir cursor de sync | Servidor |
| Aplicar alterações no dispositivo | Dispositivo (com detecção de conflito) |
| Resolver conflito de pull | Regras B-06 (domain_rule, auto_merge, etc.) |

### Cenário 4: Conflito Não Resolvido Automaticamente

| Decisão | Autoridade |
|---------|------------|
| Detectar conflito | Servidor (push) ou Dispositivo (pull) |
| Tentar auto_merge | Regras de domínio |
| Tentar domain_rule | StateMachine + Policy |
| Escalonar para manual | Sistema |
| Decisão final em manual | ADMIN (humano) |

## 8.3 Regra de Ouro

> **O dispositivo nunca sobrescreve o servidor silenciosamente.**  
> **O servidor nunca invalida dados locais sem explicação.**

## 8.4 Dois Motoristas: Regras Específicas

### Contexto

- Motorista Felipe (Device A)
- Motorista Natasha (Device B)
- Ambos podem estar offline
- Ambos podem ter a mesma corrida visível

### Regras de Autoridade por Ação

| Ação | Quem pode (offline) | Autoridade Global | Conflito se… |
|------|---------------------|-------------------|--------------|
| Iniciar deslocamento | Motorista atribuído | Servidor valida atribuição | Outro motorista tenta a mesma ação |
| Receber pet | Motorista atribuído | Servidor valida | Status já avançou no servidor |
| Iniciar corrida | Motorista atribuído | Servidor valida | — |
| Entregar pet | Motorista atribuído | Servidor valida código | — |
| Concluir corrida | Motorista atribuído | Servidor valida KM e valor | Valor diverge do esperado |
| Cancelar corrida | ADMIN ou motorista atribuído | Servidor valida | Já foi iniciada |
| Reatribuir motorista | ADMIN apenas | Servidor | — |
| Alterar valor | ADMIN apenas | Servidor | Motorista já concluiu |
| Alterar data/hora | ADMIN ou tutor | Servidor | — |

### Mecanismo de Proteção

```text
Motorista A (offline):
  Ride #123 → motoristaACaminho

Motorista B (offline):
  Ride #123 → motoristaACaminho (mesma ação)

Servidor (quando sync):
  Recebe operação de A: base_version=5, server_version=5
  → Aplica, version=6, status=motoristaACaminho

  Recebe operação de B: base_version=5, server_version=6
  → CONFLITO: version_mismatch, cause=concurrent_edit

  Resolução: verifica StateMachine
    motoristaACaminho → motoristaACaminho? (já está)
    → Operação de B é idempotente? SIM (mesmo estado)
    → RESOLVE como no-op (não cria conflito real)
```

## 8.5 Pull vs Push: Autoridade em Recebimento

### Quando o servidor envia alterações para o dispositivo

```text
Servidor: Ride #123, version=10, status=cancelada

Dispositivo local: Ride #123, version=9, status=confirmada
                  (motorista está offline, não sabe do cancelamento)

Quando online:
  Pull recebe: status=cancelada, version=10

  Dispositivo verifica:
    - Local tem alterações pendentes? SIM (sync_queue tem UPDATE)
    - base_version local = 9, servidor = 10
    → CONFLITO

  Resolução:
    - StateMachine: cancelada é terminal
    - Regra: estados terminais (cancelada, concluida) têm prioridade
    - Resultado: server_wins, descarta alteração local
    - Notifica motorista: "Esta corrida foi cancelada"
```

## 8.6 Resumo das Regras de Autoridade

| # | Regra |
|---|-------|
| 1 | Servidor é autoridade global após sincronização |
| 2 | Dispositivo é autoridade operacional durante offline |
| 3 | Estado terminal (cancelada, concluida) sempre vence |
| 4 | Atribuição de motorista é decisão administrativa |
| 5 | Valores financeiros nunca resolvem automaticamente |
| 6 | AuditLog é imutável e replicado via audit_context |
| 7 | Conflitos não resolvidos escalonam para ADMIN |
| 8 | Operações idempotentes (mesmo resultado) não geram conflito real |
| 9 | SyncEngine coordena, mas não decide regras de negócio (DEC-001) |

---

# 📎 Apêndice A — Glossário

| Termo | Definição |
|-------|-----------|
| **UUID v4** | Identificador universal único, versão 4 (aleatório), usado para identidade de entidades |
| **ULID** | Identificador lexicograficamente sortável, usado para operações e eventos |
| **Soft Delete** | Marcação lógica de exclusão (`deleted_at`) sem remoção física do registro |
| **Base Version** | Versão da entidade no momento da alteração offline, usada para detecção de conflito |
| **Sync Cursor** | Token opaco fornecido pelo servidor para paginação de alterações |
| **Operation ID** | Identificador único de uma operação de sincronização, garante idempotência |
| **Group ID** | Vincula operações compostas que devem ser processadas atomicamente |
| **State Machine** | Máquina de estados finita que define transições válidas de `RideStatus` |
| **Authorization Policy** | Conjunto de regras que define quem pode executar qual ação em qual estado |
| **Conflict Resolver** | Componente especializado em resolver conflitos para um tipo de entidade |
| **Audit Context** | Dados de auditoria anexados à SyncOperation para replicação ao servidor |
| **In Flight Since** | Timestamp que marca quando uma operação começou a ser enviada |

---

# 📎 Apêndice B — Checklist de Auditoria v1.1

Após aplicação das correções, a v1.1 deve ser submetida à seguinte auditoria antes do congelamento:

### Auditoria de consistência
- [ ] Todos os documentos B-01 a B-08 referenciam os mesmos nomes de campos, enums e estratégias
- [ ] Não há `conflict_type` ou `resolution_strategy` definidos em um documento e ausentes em outro
- [ ] Schema SQLite é consistente com os contratos Dart descritos

### Auditoria de SQLite
- [ ] Todas as tabelas de domínio possuem `id` (UUID), `created_at`, `updated_at`, `deleted_at`, `version`
- [ ] Toda foreign key utilizada em JOIN, filtro ou ordenação em caminho de consulta crítico possui índice adequado
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
- [ ] SyncEngine não contém regras de negócio (DEC-001)
- [ ] `domain_rule` delega para StateMachine + Policy
- [ ] AuditLog replicação está definida via audit_context sem ambiguidade

### Auditoria de operações compostas
- [ ] Uma entidade = uma mutação por group_id
- [ ] Compactação preserva base_version original
- [ ] Compactação só ocorre na mesma cadeia causal local

---

> **Documento elaborado em nível arquitetural.**  
> **Aguardando aprovação formal para implementação.**  
> **Nenhuma linha de código foi alterada no projeto.**

---
*Especificação BLOCO B v1.1 — Táxi Pet Praiano*  
*Versão 1.1 — 16/08/2026*
