# 🚕🐾 B-IMP-00a — Arquivos Corrigidos para Substituição
## Táxi Pet Praiano — Reconciliação de Enums

> **Status:** Arquivos prontos para substituição no projeto  
> **Data:** 16/08/2026  
> **Base:** BLOCO A 🔒 + BLOCO B v1.1 🔒  

---

## 📦 Arquivos para Download

### ✅ Arquivo completo — substituir diretamente

| Arquivo | Download | Ação |
|---------|----------|------|
| `ride.dart` | [ride.dart](sandbox:///mnt/agents/output/ride.dart) | **Substituir** `lib/domain/entities/ride.dart` |
| `app_constants.dart` | [app_constants.dart](sandbox:///mnt/agents/output/app_constants.dart) | **Substituir** `lib/core/constants/app_constants.dart` |

### 🔧 Instruções manuais — aplicar no arquivo existente

| Arquivo | Instrução | Linha |
|---------|-----------|-------|
| `ride_model.dart` | Adicionar import | Topo do arquivo |
| `new_ride_page.dart` | Substituir 4 DropdownMenuItem | Aprox. linha 117 |

---

## 🔧 Instrução 1 — `lib/data/models/ride_model.dart`

**Adicione esta linha logo abaixo do primeiro import:**

```dart
import 'package:taxi_pet_praiano/domain/enums/ride_enums.dart';
```

**Exemplo de como deve ficar:**
```dart
import 'package:taxi_pet_praiano/domain/entities/ride.dart';
import 'package:taxi_pet_praiano/domain/enums/ride_enums.dart';  // ← ADICIONAR ESTA LINHA
```

**Restante do arquivo:** NÃO ALTERAR.

---

## 🔧 Instrução 2 — `lib/presentation/pages/rides/new_ride_page.dart`

**Localize os 4 DropdownMenuItem (aproximadamente na linha 117).**

**Substitua DE:**
```dart
items: const [
  DropdownMenuItem(
    value: 'ida',
    child: Text('Ida'),
  ),
  DropdownMenuItem(
    value: 'volta',
    child: Text('Volta'),
  ),
  DropdownMenuItem(
    value: 'internacao',
    child: Text('Internação'),
  ),
  DropdownMenuItem(
    value: 'resgate',
    child: Text('Resgate'),
  ),
],
```

**PARA:**
```dart
items: const [
  DropdownMenuItem(
    value: 'somenteIda',
    child: Text('Ida'),
  ),
  DropdownMenuItem(
    value: 'idaEVolta',
    child: Text('Ida e Volta'),
  ),
  DropdownMenuItem(
    value: 'idaInternacao',
    child: Text('Internação'),
  ),
  DropdownMenuItem(
    value: 'resgate',
    child: Text('Resgate'),
  ),
],
```

**Restante do arquivo:** NÃO ALTERAR.

---

## 📋 Roteiro de Aplicação

### Passo 1 — Faça backup (opcional mas recomendado)
```cmd
git stash
```

### Passo 2 — Substitua os 2 arquivos completos
1. Baixe `ride.dart` e `app_constants.dart` acima
2. Copie para as pastas correspondentes no projeto, **substituindo os existentes**

### Passo 3 — Aplique as 2 instruções manuais
1. Abra `ride_model.dart` e adicione o import
2. Abra `new_ride_page.dart` e substitua os 4 DropdownMenuItem

### Passo 4 — Verifique os arquivos já corretos
Confirme que estes arquivos já estão corretos no seu working tree:
- `lib/domain/entities/ride_status_history.dart` → importa `ride_enums.dart`
- `test/ride_state_machine_test.dart` → NÃO importa `app_constants.dart`

Se não estiverem, aplique as mesmas correções.

### Passo 5 — Valide
```cmd
flutter analyze
flutter test
git diff --stat
```

**Esperado:**
- `flutter analyze` → 0 erros
- `flutter test` → 32/32 passando
- `git diff --stat` → 6 arquivos modificados (os 4 acima + ride_status_history + test)

### Passo 6 — Commit
```cmd
git add .
git commit -m "refactor(B-IMP-00a): centraliza enums RideStatus e RideType

- Remove enums duplicados de ride.dart e app_constants.dart
- Fonte única: domain/enums/ride_enums.dart (BLOCO A congelado)
- Corrige StateMachine: remove estado não oficial 'reagendada'
- Alinha UI: internacao → idaInternacao (D02)
- Atualiza imports transitivos para fonte oficial

flutter test: 32/32
flutter analyze: 0 erros"
```

---

## ⚠️ Arquivos que NÃO devem ser alterados

| Arquivo | Motivo |
|---------|--------|
| `lib/domain/enums/ride_enums.dart` | Fonte oficial congelada no BLOCO A |
| `lib/data/local/database_helper.dart` | Fora do escopo B-IMP-00a |
| `lib/core/sync/sync_engine.dart` | Fora do escopo B-IMP-00a |
| `pubspec.yaml` | Fora do escopo B-IMP-00a |
| `test/*.dart` (exceto ride_state_machine_test) | Fora do escopo B-IMP-00a |

---

## ✅ Checklist de Sucesso

- [ ] `ride.dart` importa `ride_enums.dart` e NÃO declara enums
- [ ] `app_constants.dart` NÃO declara `RideStatus` nem `RideType`
- [ ] `ride_model.dart` importa `ride_enums.dart` explicitamente
- [ ] `new_ride_page.dart` usa `'somenteIda'`, `'idaEVolta'`, `'idaInternacao'`
- [ ] `flutter test` → 32/32
- [ ] `flutter analyze` → 0 erros
- [ ] `git diff --stat` → apenas os 6 arquivos esperados

---

> **Arquivos gerados com base na inspeção completa do projeto.**  
> **Não foi possível acessar o ZIP de 20.7 MB diretamente.**  
> **Todos os arquivos foram reconstruídos a partir dos diffs e conteúdos fornecidos durante a auditoria.**

---
*B-IMP-00a — Arquivos Corrigidos*  
*Táxi Pet Praiano*  
*16/08/2026*
