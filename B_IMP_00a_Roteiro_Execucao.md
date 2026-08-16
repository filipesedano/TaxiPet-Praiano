# 🚕🐾 B-IMP-00a — Roteiro de Execução
## Reconciliação de Enums — Táxi Pet Praiano

> **Status:** Autorizado para execução  
> **Decisões aprovadas:** D01, D02, D03  
> **Regra:** Nenhuma alteração fora do escopo B-IMP-00a  
> **Critério de conclusão:** 32/32 testes + flutter analyze limpo + git diff revisado  

---

## 📋 Decisões Aprovadas

### ✅ D01 — Remoção de `RideStatus.reagendada`

**Justificativa:** `reagendada` não pertence aos 9 estados oficiais congelados no BLOCO A. Não há consumidores fora de `ride.dart`.

**Ação:** Remover do enum duplicado em `ride.dart`. A entidade passará a usar `RideStatus` oficial de `ride_enums.dart`, que não contém `reagendada`.

**Impacto:** Nenhum consumidor externo. A entidade `Ride` perde a capacidade de assumir esse estado, o que é correto arquiteturalmente.

---

### ✅ D02 — Mapeamento `internacao` → `idaInternacao`

**Justificativa:** A UI `new_ride_page.dart` produz o valor `'internacao'`, que não existe no enum oficial `RideType` (7 tipos). Semanticamente, `internacao` no contexto de transporte representa a **ida do animal para internação**, que corresponde ao tipo oficial `idaInternacao`. O retorno da internação possui tipo próprio: `altaRetorno`.

**Ação:**
1. Em `new_ride_page.dart`, substituir `'internacao'` por `'idaInternacao'`
2. Verificar se há outros consumidores de `'internacao'` no projeto
3. Atualizar qualquer string literal ou `.byName('internacao')`

**Impacto:** A UI passa a produzir valores compatíveis com o enum oficial. Nenhuma alteração semântica no domínio — apenas alinhamento de nomenclatura.

---

### ✅ D03 — Testes da State Machine como dívida técnica documentada

**Justificativa:** Os testes atuais de `ride_state_machine_test.dart` utilizam o setter `ride.status = ...` diretamente, em vez de `ride.transitionTo(...)`. Isso significa que os 32 testes passam, mas não exercitam efetivamente as restrições da máquina de estados (AD-005 do BLOCO A).

**Ação:** NÃO corrigir agora. Registrar como dívida técnica do BLOCO A. Se futuramente for necessário, será proposta formal de alteração do BLOCO A congelado.

**Registro:**
```text
DÍVIDA TÉCNICA BLOCO A — DT-A-001
Título: Testes de RideStateMachine não exercitam transitionTo()
Impacto: 32/32 testes passam, mas validação de transições de estado não é testada
Severidade: Média
Decisão: Não corrigir durante B-IMP-00. Reavaliar em manutenção do BLOCO A.
```

---

## 🎯 Roteiro de Execução Passo a Passo

### PASSO 0 — Preparação (antes de qualquer alteração)

```bash
# 1. Certifique-se de estar na branch correta
git status

# 2. Crie uma branch para o B-IMP-00a
git checkout -b feat/B-IMP-00a-reconcilia-enums

# 3. Salve o estado atual como baseline
git add .
git commit -m "chore: baseline antes do B-IMP-00a"

# 4. Execute os testes baseline
flutter test
# → ESPERADO: 32/32 passando

flutter analyze
# → ESPERADO: 0 erros, 2 infos (existentes)

# 5. Salve os resultados
flutter test > test_baseline.txt
flutter analyze > analyze_baseline.txt
```

---

### PASSO 1 — Identificar todos os consumidores

```bash
# Execute no terminal do projeto (Git Bash / PowerShell / CMD)

grep -RniE "RideStatus|RideType" lib test

# Ou, no Windows CMD:
# findstr /S /N /I "RideStatus RideType" lib\*.dart test\*.dart
```

**Anote a lista completa de arquivos.** Você já fez isso, mas confirme que nenhum arquivo novo surgiu.

**Arquivos esperados a verificar:**
- [ ] `lib/domain/entities/ride.dart` (fonte duplicada — será alterada)
- [ ] `lib/domain/enums/ride_enums.dart` (fonte oficial — NÃO ALTERAR)
- [ ] `lib/domain/entities/ride_status_history.dart` (já usa oficial — NÃO ALTERAR)
- [ ] `lib/data/models/ride_model.dart` (usa enum de ride.dart — ALTERAR)
- [ ] `lib/core/sync/sync_engine.dart` (usa enum — VERIFICAR)
- [ ] `lib/presentation/pages/rides/new_ride_page.dart` (usa `'internacao'` — ALTERAR)
- [ ] `test/ride_state_machine_test.dart` (usa enum — VERIFICAR)
- [ ] `test/ride_repository_test.dart` (usa enum — VERIFICAR)
- [ ] `test/database_test.dart` (usa enum — VERIFICAR)
- [ ] `test/sync_engine_test.dart` (usa enum — VERIFICAR)
- [ ] `test/audit_logger_test.dart` (usa enum — VERIFICAR)

---

### PASSO 2 — Alterar `lib/domain/entities/ride.dart`

**Ação:** Remover os enums duplicados e importar a fonte oficial.

**Antes:**
```dart
// ride.dart
enum RideType { somenteIda, idaEVolta, internacao, resgate }
enum RideStatus { 
  solicitada, confirmada, motoristaACaminho, petRecebido, 
  emAndamento, petEntregue, concluida, cancelada, reagendada,
  // ... outros
}
```

**Depois:**
```dart
// ride.dart
import 'package:taxi_pet_praiano/domain/enums/ride_enums.dart';

// REMOVIDO: enum RideType { ... }
// REMOVIDO: enum RideStatus { ... }
// Agora usa RideType e RideStatus de ride_enums.dart
```

**Verificações:**
- [ ] Nenhum `enum RideType` em `ride.dart`
- [ ] Nenhum `enum RideStatus` em `ride.dart`
- [ ] Import `ride_enums.dart` presente
- [ ] Classe `Ride` usa `RideType` e `RideStatus` sem prefixo de namespace

---

### PASSO 3 — Alterar `lib/data/models/ride_model.dart`

**Ação:** Garantir que o model importe os enums da fonte oficial.

**Verifique:**
```dart
// ride_model.dart
import 'package:taxi_pet_praiano/domain/enums/ride_enums.dart';
```

Se o model referenciava `RideStatus` e `RideType` através de `ride.dart` (ex: `import 'ride.dart'`), agora ele deve:
1. Importar `ride_enums.dart` explicitamente
2. Ou, se já importa `ride.dart`, continuar funcionando porque `ride.dart` agora exporta os mesmos nomes via import

**Atenção:** Se `ride_model.dart` usava `RideStatus` como tipo de campo, a conversão `toMap()`/`fromMap()` pode precisar de ajuste se os nomes das strings mudaram.

**Verifique especialmente:**
```dart
// Se existir algo como:
RideStatus.values.byName(map['status'])
// ou
RideType.values.byName(map['type'])
```

Os nomes dos enums oficiais em `ride_enums.dart` podem usar `camelCase` ou `snake_case`. Confirme que a conversão do banco (string) para enum está alinhada.

---

### PASSO 4 — Alterar `lib/presentation/pages/rides/new_ride_page.dart`

**Ação:** Substituir `'internacao'` por `'idaInternacao'`.

**Busque:**
```bash
grep -n "internacao" lib/presentation/pages/rides/new_ride_page.dart
```

**Substitua:**
```dart
// Antes
value: 'internacao',

// Depois
value: 'idaInternacao',
```

**Verifique se há outros arquivos de UI com `'internacao'`:**
```bash
grep -Rni "internacao" lib/presentation/
```

---

### PASSO 5 — Verificar `lib/core/sync/sync_engine.dart`

**Ação:** Garantir que o SyncEngine use os enums oficiais.

**Verifique:**
```bash
grep -n "RideStatus\|RideType" lib/core/sync/sync_engine.dart
```

Se o SyncEngine importa `ride.dart` e usa `RideStatus`/`RideType`, deve continuar funcionando porque `ride.dart` agora importa de `ride_enums.dart`.

**Mas atenção:** Se o SyncEngine possui lógica como:
```dart
if (ride.status == RideStatus.reagendada) { ... }
```
Isso vai quebrar porque `reagendada` não existe mais. Verifique e remova/refatore essa lógica se existir.

---

### PASSO 6 — Verificar todos os testes

**Ação:** Buscar referências aos enums em todos os testes.

```bash
grep -RniE "RideStatus|RideType" test/
```

**Para cada ocorrência:**
- Se usa `RideStatus.reagendada` → **remover/substituir** (estado não oficial)
- Se usa `RideType.internacao` → **substituir** por `RideType.idaInternacao`
- Se usa `RideStatus` ou `RideType` de `ride.dart` → deve continuar funcionando via import

**Testes específicos a verificar:**
- [ ] `test/ride_state_machine_test.dart` — usa `RideStatus`?
- [ ] `test/ride_repository_test.dart` — cria rides com enums?
- [ ] `test/database_test.dart` — insere rides no banco?
- [ ] `test/sync_engine_test.dart` — sync de rides?

---

### PASSO 7 — Compilar e testar

```bash
# 1. Análise estática
flutter analyze

# → ESPERADO: 0 erros
# → Aceitável: infos existentes (não devem aumentar)
# → INACEITÁVEL: novos erros ou warnings

# 2. Testes
flutter test

# → ESPERADO: 32/32 passando (ou mais, se novos testes foram adicionados)
# → INACEITÁVEL: qualquer teste quebrado

# 3. Se houver erros, corrija antes de continuar
```

---

### PASSO 8 — Revisão do git diff

```bash
# Veja exatamente o que foi alterado
git diff

# Ou, para ver arquivo por arquivo
git diff --stat
git diff lib/domain/entities/ride.dart
git diff lib/data/models/ride_model.dart
git diff lib/presentation/pages/rides/new_ride_page.dart
```

**Checklist de revisão:**
- [ ] Apenas arquivos do escopo B-IMP-00a foram alterados
- [ ] Nenhuma alteração em `database_helper.dart`
- [ ] Nenhuma alteração em `sync_engine.dart` (exceto import, se necessário)
- [ ] Nenhuma alteração em `pubspec.yaml`
- [ ] `ride_enums.dart` NÃO foi alterado (fonte oficial intocável)
- [ ] Nenhum enum duplicado restante em `ride.dart`

---

### PASSO 9 — Commit e evidência

```bash
# Commit organizado
git add .
git commit -m "refactor(B-IMP-00a): unifica enums RideStatus e RideType

- Remove enums duplicados de ride.dart
- Importa fonte oficial de ride_enums.dart
- Remove estado não oficial 'reagendada' (D01)
- Mapeia 'internacao' → 'idaInternacao' na UI (D02)
- Alinha RideModel e SyncEngine aos enums oficiais

Decisões:
- D01: reagendada removido (não pertence aos 9 estados oficiais)
- D02: internacao → idaInternacao (alinhamento semântico)
- D03: testes de State Machine registrados como dívida técnica

flutter test: 32/32
flutter analyze: 0 erros"
```

---

## 📋 Checklist Final de Conclusão do B-IMP-00a

- [ ] Uma única definição de `RideStatus` (em `ride_enums.dart`)
- [ ] Uma única definição de `RideType` (em `ride_enums.dart`)
- [ ] `ride.dart` NÃO declara mais enums
- [ ] `ride.dart` importa `ride_enums.dart`
- [ ] Todos os consumidores usam a fonte oficial
- [ ] Nenhum import quebrado
- [ ] Nenhuma referência residual a `RideStatus.reagendada`
- [ ] Nenhuma referência residual a `RideType.internacao`
- [ ] `new_ride_page.dart` usa `'idaInternacao'`
- [ ] `RideModel` compatível com enums oficiais
- [ ] `SyncEngine` compatível com enums oficiais
- [ ] `flutter test` → 32/32 passando
- [ ] `flutter analyze` → 0 erros
- [ ] `git diff` revisado e aprovado
- [ ] Nenhuma alteração fora do escopo B-IMP-00a
- [ ] D03 documentada como dívida técnica

---

## 🚀 Transição para B-IMP-00b

O B-IMP-00a só está oficialmente encerrado quando **todos os itens do checklist acima estiverem marcados**.

Após conclusão:
```text
B-IMP-00a → CONCLUÍDO
     ↓
B-IMP-00b — Monetários (value → value_cents)
```

---

> **Roteiro gerado para execução local.**  
> **Aguardando execução e validação pelo desenvolvedor.**  
> **Nenhuma linha de código foi alterada remotamente.**

---
*B-IMP-00a — Roteiro de Execução*  
*Táxi Pet Praiano*  
*Versão 1.0 — 16/08/2026*
