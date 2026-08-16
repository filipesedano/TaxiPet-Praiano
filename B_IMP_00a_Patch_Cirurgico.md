# 🚕🐾 B-IMP-00a — Patch Cirúrgico de Desbloqueio
## Reconciliação de Enums — Arquivos Alterados

> **Status:** Patch pronto para aplicação  
> **Escopo:** 3 arquivos apenas  
> **Regra:** Nenhuma alteração fora destes 3 arquivos  
> **Critério de sucesso:** `flutter test` = 32/32 + `flutter analyze` = 0 erros  

---

## 📋 Arquivos Alterados

| # | Arquivo | Alteração |
|---|---------|-----------|
| 1 | `lib/domain/entities/ride.dart` | Remove enums duplicados, importa oficial, corrige StateMachine |
| 2 | `lib/data/models/ride_model.dart` | Adiciona import explícito do enum oficial |
| 3 | `lib/presentation/pages/rides/new_ride_page.dart` | Alinha strings da UI aos nomes oficiais do enum |

---

## 🔧 Arquivo 1 — `lib/domain/entities/ride.dart`

### Ação: REESCREVER completamente

O arquivo working tree declara enums duplicados e contém `reagendada` (não oficial). Deve ser reescrito para importar a fonte oficial e usar apenas os 9 estados congelados.

**Conteúdo final:**

```dart
import 'package:taxi_pet_praiano/domain/enums/ride_enums.dart';

/// Representa uma corrida do Táxi Pet Praiano.
///
/// Entidade de domínio da corrida.
/// AD-005: controla as transições válidas de estado.
class Ride {
  Ride({
    required this.id,
    required this.tutorId,
    required this.petId,
    required this.origin,
    required this.destination,
    required this.scheduledAt,
    required this.type,
    required this.createdAt,
    required this.updatedAt,
    RideStatus status = RideStatus.agendada,
    this.version = 1,
  }) : _status = status;

  final String id;
  final String tutorId;
  final String petId;
  final String origin;
  final String destination;
  final DateTime scheduledAt;
  final RideType type;
  final DateTime createdAt;
  DateTime updatedAt;

  RideStatus _status;
  int version;

  RideStatus get status => _status;

  /// Setter de compatibilidade com o modelo atual de testes.
  ///
  /// A aplicação deve preferencialmente utilizar [transitionTo]
  /// para respeitar as regras da máquina de estados AD-005.
  set status(RideStatus value) {
    _status = value;
  }

  /// Altera o estado da corrida respeitando as regras do AD-005.
  ///
  /// Retorna [true] quando a transição é válida.
  /// Retorna [false] quando a transição não é permitida.
  bool transitionTo(RideStatus nextStatus) {
    if (!_isValidTransition(_status, nextStatus)) {
      return false;
    }

    _status = nextStatus;
    updatedAt = DateTime.now();
    version++;
    return true;
  }

  bool _isValidTransition(
    RideStatus current,
    RideStatus next,
  ) {
    switch (current) {
      case RideStatus.agendada:
        return next == RideStatus.confirmada ||
            next == RideStatus.cancelada ||
            next == RideStatus.noShow;

      case RideStatus.confirmada:
        return next == RideStatus.motoristaACaminho ||
            next == RideStatus.cancelada ||
            next == RideStatus.noShow;

      case RideStatus.motoristaACaminho:
        return next == RideStatus.petRecebido ||
            next == RideStatus.cancelada;

      case RideStatus.petRecebido:
        return next == RideStatus.emAndamento ||
            next == RideStatus.cancelada;

      case RideStatus.emAndamento:
        return next == RideStatus.entregue ||
            next == RideStatus.cancelada;

      case RideStatus.entregue:
        return next == RideStatus.concluida;

      case RideStatus.concluida:
      case RideStatus.cancelada:
      case RideStatus.noShow:
        return false;
    }
  }
}
```

**Mudanças em relação ao working tree:**
- ✅ Removido `enum RideType { ... }`
- ✅ Removido `enum RideStatus { ... }`
- ✅ Adicionado `import 'package:taxi_pet_praiano/domain/enums/ride_enums.dart';`
- ✅ Removido `next == RideStatus.reagendada` de `agendada` e `confirmada`
- ✅ Removido `case RideStatus.reagendada:` do switch
- ✅ Mantido `transitionTo()`, setter, `_isValidTransition()`
- ✅ Mantidos todos os campos e construtor (compatível com testes)

---

## 🔧 Arquivo 2 — `lib/data/models/ride_model.dart`

### Ação: ADICIONAR import explícito

O `RideModel` usa `RideType` e `RideStatus` mas importa apenas `ride.dart`. Quando `ride.dart` perder os enums, o `RideModel` precisa importar a fonte oficial explicitamente.

**Alteração:** Adicionar uma linha no topo do arquivo.

**Antes:**
```dart
import 'package:taxi_pet_praiano/domain/entities/ride.dart';
```

**Depois:**
```dart
import 'package:taxi_pet_praiano/domain/entities/ride.dart';
import 'package:taxi_pet_praiano/domain/enums/ride_enums.dart';
```

**Restante do arquivo:** INALTERADO.

---

## 🔧 Arquivo 3 — `lib/presentation/pages/rides/new_ride_page.dart`

### Ação: Alinhar strings ao enum oficial

A UI usa strings literais que não correspondem aos nomes dos enums oficiais. A correção mínima é alinhar os `value` do dropdown.

**Alteração:** Substituir os 4 `DropdownMenuItem`.

**Antes:**
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

**Depois:**
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

**Notas:**
- `'ida'` → `'somenteIda'` (nome oficial do enum)
- `'volta'` → `'idaEVolta'` (nome oficial do enum)
- `'internacao'` → `'idaInternacao'` (nome oficial do enum)
- `'resgate'` → `'resgate'` (já estava correto)
- Texto da UI alterado de `'Volta'` para `'Ida e Volta'` para refletir semanticamente o tipo oficial

**Restante do arquivo:** INALTERADO.

---

## 📋 Checklist de Aplicação

- [ ] Copiar o conteúdo do Arquivo 1 para `lib/domain/entities/ride.dart`
- [ ] Adicionar a linha de import no Arquivo 2
- [ ] Substituir os 4 DropdownMenuItem no Arquivo 3
- [ ] Executar `flutter analyze` → esperado: 0 erros
- [ ] Executar `flutter test` → esperado: 32/32 passando
- [ ] Executar `git diff --stat` → esperado: apenas 3 arquivos modificados
- [ ] Revisar `git diff` linha a linha
- [ ] Commit com mensagem descritiva

---

## ⚠️ Atenções

1. **Não altere `ride_enums.dart`** — é a fonte oficial congelada no BLOCO A.
2. **Não altere testes** — eles usam `RideStatus.agendada` e `RideType.somenteIda`, que continuam disponíveis via import de `ride.dart` → `ride_enums.dart`.
3. **Não altere `database_helper.dart`** — fora do escopo B-IMP-00a.
4. **Não altere `sync_engine.dart`** — fora do escopo B-IMP-00a.
5. **Se `flutter analyze` reportar erros** em arquivos não listados aqui, pare e reporte antes de continuar.

---

## 🎯 Critério de Conclusão

```text
flutter test
→ 32/32 passando

flutter analyze
→ 0 erros
→ infos ≤ 2 (existentes)

git diff --stat
→ 3 arquivos modificados
→ lib/domain/entities/ride.dart
→ lib/data/models/ride_model.dart
→ lib/presentation/pages/rides/new_ride_page.dart
```

---

> **Patch gerado em nível de instrução de implementação.**  
> **Aguardando aplicação e validação pelo desenvolvedor.**  
> **Nenhuma linha de código foi alterada remotamente.**

---
*B-IMP-00a — Patch Cirúrgico de Desbloqueio*  
*Táxi Pet Praiano*  
*Versão 1.0 — 16/08/2026*
