# 🚕🐾 B-IMP-00 — Plano de Reconciliação da Fase 1
## Táxi Pet Praiano — Preparação para Implementação do BLOCO B v1.1

> **Status:** Plano de execução — aguardando aprovação  
> **Pré-requisito:** BLOCO A 🔒 congelado + BLOCO B v1.1 🔒 congelado  
> **Objetivo:** Alinhar o código legado da Fase 1 ao contrato do BLOCO B v1.1, sem quebrar os 32 testes existentes  
> **Regra:** Nenhuma alteração de código do BLOCO B (migrations, SyncEngine, entidades de sync) até o B-IMP-00 ser concluído e validado  

---

## 📋 Índice

1. [Inventário do Código Atual](#1-inventário-do-código-atual)
2. [Gap Analysis: Fase 1 × BLOCO B v1.1](#2-gap-analysis-fase-1--bloco-b-v11)
3. [Plano de Correções (B-IMP-00a a B-IMP-00f)](#3-plano-de-correções-b-imp-00a-a-b-imp-00f)
4. [Critérios de Aceitação](#4-critérios-de-aceitação)
5. [Transição para B-IMP-01](#5-transição-para-b-imp-01)

---

## 1. Inventário do Código Atual

### 1.1 Estrutura de Pastas

```text
lib/
├── core/
│   ├── audit/              → infraestrutura de auditoria (reaproveitável)
│   ├── constants/          → constantes do app
│   ├── di/                 → injeção de dependência (get_it)
│   ├── errors/             → exceções customizadas
│   ├── network/            → camada HTTP (dio)
│   ├── security/           → hash, criptografia
│   ├── sync/               → SyncEngine Fase 1 (será evoluído)
│   └── utils/              → utilitários
├── data/
│   ├── local/
│   │   └── database_helper.dart   → SQLite Fase 1 (será evoluído)
│   ├── models/
│   │   └── sync_operation_model.dart   → modelo de sync Fase 1 (será substituído)
│   └── repositories/
│       └── ride_repository_impl.dart   → repository existente (manter/adaptar)
├── domain/
│   ├── entities/
│   │   ├── ride.dart              → ⚠️ contém enums duplicados
│   │   └── sync_operation.dart    → entidade de sync Fase 1 (será substituída)
│   ├── enums/
│   │   └── ride_enums.dart        → ✅ fonte oficial (manter)
│   ├── repositories/
│   │   └── ride_repository.dart   → contrato do repository (manter/adaptar)
│   └── usecases/
└── presentation/
```

### 1.2 Dependências (pubspec.yaml)

| Pacote | Versão | Uso | Status BLOCO B |
|--------|--------|-----|----------------|
| `sqflite` | ^2.3.0 | SQLite local | ✅ Manter |
| `sqflite_common_ffi` | ^2.3.0 | SQLite desktop/teste | ✅ Manter |
| `uuid` | ^4.3.0 | UUID v4 | ✅ Manter |
| `dio` | ^5.4.0 | HTTP client | ✅ Manter |
| `connectivity_plus` | ^5.0.0 | Detecção de rede | ✅ Manter |
| `crypto` | ^3.0.0 | Hash SHA-256 | ✅ Manter |
| `get_it` | ^7.6.0 | DI | ✅ Manter |
| `flutter_bloc` | ^8.1.0 | State management | ✅ Manter |
| `ulid` | — | ULID para operações | ⏳ Verificar/Adicionar |

> **Nota:** Verificar se `ulid` ou equivalente já está disponível. Se não, adicionar `ulid: ^2.0.0` ou implementar utilitário interno.

### 1.3 Testes Existentes

```text
test/
├── audit_logger_test.dart
├── database_test.dart
├── ride_repository_test.dart
├── ride_state_machine_test.dart
├── sync_engine_test.dart
└── uuid_helper_test.dart
```

**Baseline:** 32/32 testes passando, `flutter analyze` = 0 erros, 2 infos.

---

## 2. Gap Analysis: Fase 1 × BLOCO B v1.1

### 2.1 Matriz de Gaps

| # | Área | Fase 1 (Atual) | BLOCO B v1.1 (Congelado) | Severidade |
|---|------|----------------|--------------------------|------------|
| GAP-01 | **Enums duplicados** | `ride.dart` contém `RideStatus` e `RideType` próprios, diferentes de `ride_enums.dart` | Enum único oficial em `ride_enums.dart` | 🔴 CRÍTICO |
| GAP-02 | **Valores monetários** | `REAL` (`value`, `amount`, `cost`, etc.) | `INTEGER` em centavos (`value_cents`, `amount_cents`, etc.) | 🔴 CRÍTICO |
| GAP-03 | **Tabela de sync** | `sync_operations` (campos Fase 1) | `sync_queue` (campos B-02) | 🔴 CRÍTICO |
| GAP-04 | **Estados de sync** | `pending, syncing, synced, error, conflict` | `pending, in_flight, resolved, failed, conflict, permanent_error` | 🔴 CRÍTICO |
| GAP-05 | **`base_version`** | ❌ Ausente | ✅ Obrigatório em toda operação de sync | 🔴 CRÍTICO |
| GAP-06 | **`group_id`** | ❌ Ausente | ✅ Obrigatório para operações compostas | 🔴 CRÍTICO |
| GAP-07 | **`in_flight_since`** | ❌ Ausente | ✅ Campo do schema para recovery pós-crash | 🔴 CRÍTICO |
| GAP-08 | **`audit_context`** | ❌ Ausente | ✅ Anexado à SyncOperation | 🔴 CRÍTICO |
| GAP-09 | **`version` em entidades** | Presente em `rides`, ausente em outras | Presente em todas as entidades versionáveis | 🟠 ALTO |
| GAP-10 | **`sync_conflicts`** | ❌ Ausente | ✅ Tabela de registro de conflitos | 🟠 ALTO |
| GAP-11 | **`audit_log` imutável** | Parcial (infra existe) | Schema formalizado em B-01 | 🟡 MÉDIO |
| GAP-12 | **`app_metadata`** | ❌ Ausente | ✅ Tabela de controle para cursor, device_id | 🟡 MÉDIO |
| GAP-13 | **ULID** | ❌ Ausente | `operation_id`, `device_id`, `audit_id` | 🟡 MÉDIO |
| GAP-14 | **`Ride` ↔ `RideModel`** | Conversão pode não refletir `value_cents` | Deve converter centavos ↔ decimal | 🟠 ALTO |
| GAP-15 | **Soft delete universal** | Parcial | Todas as tabelas de domínio | 🟡 MÉDIO |
| GAP-16 | **Índices** | Básicos | Por padrões de acesso (B-01) | 🟡 MÉDIO |
| GAP-17 | **Migrations** | `DatabaseHelper` cria tudo em `onCreate` | Migration-based com `_migrations` | 🟡 MÉDIO |

### 2.2 Análise por Arquivo

#### `lib/domain/entities/ride.dart`

| Problema | Impacto | Correção |
|----------|---------|----------|
| Contém `enum RideStatus` próprio | Divergência com `ride_enums.dart` | **Remover** enums de `ride.dart`, importar de `ride_enums.dart` |
| Contém `enum RideType` próprio | Divergência com `ride_enums.dart` | **Remover** enums de `ride.dart`, importar de `ride_enums.dart` |
| Possui `reagendada` (não oficial) | Estado não documentado no BLOCO A | **Remover** ou mapear para estado oficial |
| Campo `value` (double?) | Conflita com `value_cents` (int) | **Renomear** para `valueCents` (int) |

#### `lib/domain/enums/ride_enums.dart`

| Situação | Ação |
|----------|------|
| Fonte oficial de `RideStatus` (9 estados) | ✅ Manter intacto |
| Fonte oficial de `RideType` (7 tipos) | ✅ Manter intacto |
| Centralizado conforme BLOCO A | ✅ Manter intacto |

#### `lib/data/local/database_helper.dart`

| Problema | Impacto | Correção |
|----------|---------|----------|
| Schema Fase 1 com `value REAL` | Não atende B-01 | **Migration** para `value_cents INTEGER` |
| Schema Fase 1 com `amount REAL` | Não atende B-01 | **Migration** para `amount_cents INTEGER` |
| `sync_operations` em vez de `sync_queue` | Não atende B-02 | **Migration** renomear/recriar tabela |
| Ausência de `sync_conflicts` | Não atende B-03 | **Migration** criar tabela |
| Ausência de `app_metadata` | Não atende B-01 | **Migration** criar tabela |
| `onCreate` monolítico | Difícil versionar | **Refatorar** para migration-based |
| `version` ausente em tabelas secundárias | Não atende B-01 | **Migration** adicionar campo |

#### `lib/core/sync/sync_engine.dart`

| Problema | Impacto | Correção |
|----------|---------|----------|
| Estados `syncing, synced` | Não atende B-02 | **Renomear** para `in_flight, resolved` |
| `resolveConflict(acceptLocal)` | Decide regra de negócio | **Refatorar** para delegar a `ConflictResolver` |
| Ausência de `base_version` | Não atende B-05 | **Adicionar** ao fluxo de sync |
| Ausência de `group_id` | Não atende B-02 | **Adicionar** ao fluxo de sync |
| Ausência de `in_flight_since` | Não atende B-02 | **Adicionar** recovery pós-crash |
| Ausência de backoff exponencial | Não atende B-02 | **Implementar** retry com backoff |

#### `lib/data/models/sync_operation_model.dart`

| Problema | Impacto | Correção |
|----------|---------|----------|
| Modelo Fase 1 | Não atende B-04 | **Substituir** por modelo B-04 |
| Campos diferentes | Incompatibilidade de schema | **Alinhar** com `sync_queue` B-01 |

#### `lib/domain/entities/sync_operation.dart`

| Problema | Impacto | Correção |
|----------|---------|----------|
| Entidade Fase 1 | Não atende B-04 | **Substituir** por entidade B-04 |
| Sem `audit_context` | Não atende B-02/B-08 | **Adicionar** campo |

---

## 3. Plano de Correções (B-IMP-00a a B-IMP-00f)

> **Regra de execução:** Cada sub-etapa é implementada, testada (32/32), e analisada (0 erros) antes de avançar para a próxima.

---

### B-IMP-00a — Unificação dos Enums

**Objetivo:** Eliminar duplicidade de `RideStatus` e `RideType`.

**Ações:**

1. **Remover** de `lib/domain/entities/ride.dart`:
   - `enum RideStatus { ... }`
   - `enum RideType { ... }`

2. **Adicionar** em `lib/domain/entities/ride.dart`:
   ```dart
   import 'package:taxi_pet_praiano/domain/enums/ride_enums.dart';
   ```

3. **Verificar** todos os arquivos que importavam `ride.dart` para usar os enums:
   - `lib/core/sync/sync_engine.dart`
   - `lib/data/repositories/ride_repository_impl.dart`
   - `lib/presentation/...` (telas que usam os enums)
   - Testes

4. **Mapear** `reagendada` (se usado em algum lugar) para estado oficial equivalente ou remover.

5. **Executar** `flutter test` → deve manter 32/32.

6. **Executar** `flutter analyze` → deve manter 0 erros.

**Risco:** Médio. Se algum arquivo referencia `RideStatus` de `ride.dart` sem import explícito, pode quebrar.

**Mitigação:** Busca global por `RideStatus` e `RideType` no projeto antes de remover.

---

### B-IMP-00b — Migração de Valores Monetários para Centavos

**Objetivo:** Converter todos os campos monetários de `REAL` para `INTEGER` centavos.

**Tabelas afetadas:**

| Tabela | Campo Atual | Campo Novo | Tipo Novo |
|--------|-------------|------------|-----------|
| `rides` | `value` | `value_cents` | `INTEGER` |
| `financial_entries` | `amount` | `amount_cents` | `INTEGER` |
| `fuel` | `unit_price` | `price_per_liter_cents` | `INTEGER` |
| `fuel` | `total_value` | `total_cost_cents` | `INTEGER` |
| `maintenance` | `cost` | `cost_cents` | `INTEGER` |
| `hygiene` | `value` | `cost_cents` | `INTEGER` |

**Ações:**

1. **Criar migration** que:
   - Adiciona novas colunas `_cents`
   - Converte valores existentes: `REAL * 100 → INTEGER`
   - Remove colunas antigas
   - Renomeia novas colunas (ou mantém nomes novos)

2. **Atualizar `Ride` entity:**
   - `double value` → `int valueCents`
   - Getter `double get value => valueCents / 100.0` (apenas para UI)

3. **Atualizar `RideModel`:**
   - Conversão `toMap()` / `fromMap()` para usar `value_cents`

4. **Atualizar testes** que criam rides com valores:
   - `value: 25.50` → `valueCents: 2550`

5. **Executar** `flutter test` → 32/32.

6. **Executar** `flutter analyze` → 0 erros.

**Risco:** Alto. Conversão de dados existentes pode perder precisão.

**Mitigação:** Testar migration com banco populado. Validar que `25.50 * 100 = 2550` (não 2549 ou 2551 por arredondamento).

---

### B-IMP-00c — Adição de `version` em Entidades Secundárias

**Objetivo:** Garantir que todas as entidades versionáveis possuam campo `version`.

**Entidades a verificar:**

| Entidade | Tem `version`? | Ação |
|----------|----------------|------|
| `rides` | ✅ Sim | Manter |
| `users` | ❓ Verificar | Adicionar se ausente |
| `tutors` | ❓ Verificar | Adicionar se ausente |
| `pets` | ❓ Verificar | Adicionar se ausente |
| `vehicles` | ❓ Verificar | Adicionar se ausente |
| `addresses` | ❓ Verificar | Adicionar se ausente |
| `clinics` | ❓ Verificar | Adicionar se ausente |
| `financial_entries` | ❓ Verificar | Adicionar se ausente |
| `fuel` | ❓ Verificar | Adicionar se ausente |
| `maintenance` | ❓ Verificar | Adicionar se ausente |
| `hygiene` | ❓ Verificar | Adicionar se ausente |
| `proofs` | ❓ Verificar | Adicionar se ausente |

**Ações:**

1. Inspecionar `database_helper.dart` para verificar campo `version` em cada CREATE TABLE.
2. Criar migration adicionando `version INTEGER NOT NULL DEFAULT 1` onde ausente.
3. Atualizar entities/models correspondentes.
4. Executar testes.

---

### B-IMP-00d — Estruturação do Sistema de Migrations

**Objetivo:** Evoluir de `onCreate` monolítico para sistema migration-based.

**Ações:**

1. **Criar** `lib/data/local/migrations/`:
   ```text
   migrations/
   ├── migration_base.dart      → abstract class Migration
   ├── migration_001.dart       → Schema Fase 1 (refatorado)
   ├── migration_002.dart       → Adiciona version em entidades secundárias
   ├── migration_003.dart       → Converte monetários para centavos
   ├── migration_004.dart       → Cria sync_queue, sync_conflicts, audit_log, app_metadata
   └── migration_registry.dart  → Lista ordenada de migrations
   ```

2. **Criar** tabela `_migrations`:
   ```sql
   CREATE TABLE _migrations (
     id INTEGER PRIMARY KEY,
     name TEXT NOT NULL,
     applied_at TEXT NOT NULL
   )
   ```

3. **Refatorar** `DatabaseHelper`:
   - `onCreate` executa todas as migrations na ordem
   - `onUpgrade` executa migrations pendentes

4. **Garantir idempotência**:
   - Migrations usam `CREATE TABLE IF NOT EXISTS`
   - Migrations de ALTER usam `PRAGMA user_version` ou verificam colunas

5. **Executar testes**.

---

### B-IMP-00e — Preparação do SyncOperation (Fase 1 → B-04)

**Objetivo:** Criar a estrutura de DTOs do BLOCO B sem ainda ativar o SyncEngine completo.

**Ações:**

1. **Criar** `lib/domain/sync/` (nova pasta):
   ```text
   domain/sync/
   ├── sync_operation.dart      → Entidade B-04
   ├── sync_result.dart         → DTO B-04
   ├── audit_context.dart       → DTO B-02
   ├── conflict_resolution.dart → DTO B-04
   └── sync_status.dart         → Enum de status da fila B-02
   ```

2. **Criar enums oficiais:**
   ```dart
   enum SyncQueueStatus { pending, inFlight, resolved, failed, conflict, permanentError }
   enum OperationType { create, update, softDelete }
   enum ConflictType { versionMismatch, stateInvalid, permissionDenied, mergeFailed }
   enum ConflictCause { concurrentEdit, staleVersion, unknown }
   enum ResolutionStrategy { autoMerge, serverWins, localWins, domainRule, manual }
   enum ResolutionStatus { open, resolved, escalated }
   ```

3. **NÃO substituir** ainda o `sync_operation.dart` existente (Fase 1).
   - Criar novos arquivos com nomes distintos ou em pasta diferente
   - O SyncEngine Fase 1 continua funcionando

4. **Executar testes** (ainda 32/32, pois nada foi conectado).

---

### B-IMP-00f — Validação Final da Fase 1 Reconciliada

**Objetivo:** Garantir que todas as correções foram aplicadas e o projeto está pronto para receber o BLOCO B.

**Checklist:**

- [ ] `flutter test` → 32/32 passando
- [ ] `flutter analyze` → 0 erros, 0 warnings (infos aceitáveis)
- [ ] `ride.dart` NÃO contém enums duplicados
- [ ] `ride_enums.dart` é a única fonte de `RideStatus` e `RideType`
- [ ] Todos os campos monetários usam `INTEGER` (centavos)
- [ ] Todas as entidades versionáveis possuem `version`
- [ ] Sistema de migrations está funcionando
- [ ] Banco de teste pode ser criado do zero via migrations
- [ ] Banco existente pode ser migrado via `onUpgrade`
- [ ] Novos DTOs do BLOCO B existem mas NÃO estão conectados ao fluxo
- [ ] SyncEngine Fase 1 continua operacional

---

## 4. Critérios de Aceitação

### 4.1 Antes de iniciar B-IMP-00

```text
flutter test
→ 32/32 passando

flutter analyze
→ 0 erros
→ 2 infos (existentes, não bloqueantes)
```

### 4.2 Após cada sub-etapa (B-IMP-00a a 00f)

```text
flutter test
→ 32/32 passando (ou mais, se novos testes foram adicionados)

flutter analyze
→ 0 erros
→ infos ≤ 2 (ou justificadas)
```

### 4.3 Após B-IMP-00 completo

```text
flutter test
→ 32/32 passando (mínimo)
→ Novos testes de migration passando

flutter analyze
→ 0 erros
→ infos justificadas

Evidência registrada em:
- B-IMP-00-RESULTADO.md
```

---

## 5. Transição para B-IMP-01

### 5.1 Condição de Transição

A transição para **B-IMP-01 — SQLite Schema + Migrations B-01** só ocorre quando:

1. ✅ B-IMP-00 completo e validado
2. ✅ 32/32 testes passando (mínimo)
3. ✅ `flutter analyze` limpo
4. ✅ Documentação de evidência publicada

### 5.2 O que muda na transição

| Aspecto | B-IMP-00 | B-IMP-01 |
|---------|----------|----------|
| Foco | Corrigir Fase 1 | Implementar BLOCO B |
| Migrations | Estruturar sistema | Criar schema B-01 completo |
| Sync | Manter Fase 1 funcional | Iniciar evolução para B-02/B-04 |
| Entidades | Corrigir existentes | Criar novas (SyncQueue, etc.) |
| Testes | Manter 32/32 | 32/32 + novos testes B |

### 5.3 Primeira tarefa do B-IMP-01

```text
B-IMP-01a — Criar migration_005.dart
├── Cria tabelas operacionais do B-01:
│   ├── sync_queue
│   ├── sync_conflicts
│   └── app_metadata
├── Cria índices operacionais
└── Valida: flutter test + flutter analyze
```

---

## 📎 Apêndice — Inventário Detalhado do Schema Atual

> **Nota:** Este apêndice deve ser preenchido com a inspeção real do `database_helper.dart`.

### Tabelas existentes (Fase 1)

| # | Tabela | Status | Ação B-IMP-00 |
|---|--------|--------|---------------|
| 1 | `users` | 🟡 Verificar `version` | Adicionar `version` se ausente |
| 2 | `tutors` | 🟡 Verificar `version` | Adicionar `version` se ausente |
| 3 | `addresses` | 🟡 Verificar `version` | Adicionar `version` se ausente |
| 4 | `pets` | 🟡 Verificar `version` | Adicionar `version` se ausente |
| 5 | `vehicles` | 🟡 Verificar `version` | Adicionar `version` se ausente |
| 6 | `clinics` | 🟡 Verificar `version` | Adicionar `version` se ausente |
| 7 | `rides` | 🟡 `value REAL` → `value_cents INTEGER` | Migration B-IMP-00b |
| 8 | `ride_status_history` | 🟢 Imutável | Manter |
| 9 | `financial_entries` | 🟡 `amount REAL` → `amount_cents INTEGER` | Migration B-IMP-00b |
| 10 | `maintenance` | 🟡 Verificar `version` + `cost` → `cost_cents` | Migration B-IMP-00b/c |
| 11 | `fuel` | 🟡 Verificar `version` + preços → `_cents` | Migration B-IMP-00b/c |
| 12 | `hygiene` | 🟡 Verificar `version` + `value` → `cost_cents` | Migration B-IMP-00b/c |
| 13 | `proofs` | 🟡 Verificar `version` | Adicionar `version` se ausente |
| 14 | `sync_operations` | 🔴 Estrutura Fase 1 | Será substituída por `sync_queue` em B-IMP-01 |
| 15 | `_migrations` | ❌ Não existe | Criar em B-IMP-00d |
| 16 | `sync_conflicts` | ❌ Não existe | Criar em B-IMP-01 |
| 17 | `audit_log` | 🟡 Parcial | Alinhar schema com B-01 em B-IMP-01 |
| 18 | `app_metadata` | ❌ Não existe | Criar em B-IMP-01 |

---

> **Documento elaborado em nível de planejamento de implementação.**  
> **Aguardando aprovação formal para execução.**  
> **Nenhuma linha de código do BLOCO B foi alterada.**

---
*B-IMP-00 — Plano de Reconciliação da Fase 1*  
*Táxi Pet Praiano*  
*Versão 1.0 — 16/08/2026*
