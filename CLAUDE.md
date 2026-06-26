# CLAUDE.md — Flowday Project

> Diretrizes completas para o Claude Code ao trabalhar neste projeto.
> Leia este arquivo integralmente antes de criar, editar ou refatorar qualquer coisa.

---

## 1. O Projeto

**Flowday** — assistente de produtividade com IA que organiza o dia, cuida dos hábitos, monitora o bem-estar e registra o financeiro básico.

- **Tagline**: Your day, in flow
- **Conceito**: SLC — Simple, Loveable, Complete
- **Plataformas**: iOS · Android · Web · Desktop (mesmo codebase Flutter)
- **Proprietário**: Jean Vítor

### Os 5 Módulos

| # | Módulo | Diferencial |
|---|---|---|
| 1 | Agenda com IA | IA sugere o melhor horário para cada tarefa |
| 2 | Hábitos | Streak + checklist diário + XP |
| 3 | Bem-estar | Check-in de humor e ansiedade + feedback da IA |
| 4 | Recompensas e XP | Gamificação com conquistas desbloqueáveis |
| 5 | Financeiro | Registro leve + insight mensal da IA |

---

## 2. Stack Técnica — Decisões Finais

Não questionar estas decisões. São definitivas para v1.

| Camada | Tecnologia | Versão alvo |
|---|---|---|
| **UI / Framework** | Flutter | 3.x estável |
| **Linguagem** | Dart | 3.x (null safety obrigatório) |
| **State Management** | Riverpod 2.x + code generation (`@riverpod`) | ^2.6 |
| **Navegação** | GoRouter | ^14.x |
| **Backend / Auth** | Supabase (PostgreSQL) | supabase_flutter ^2.x |
| **Banco local** | Drift (SQLite) | ^2.20 |
| **Gráficos** | fl_chart | ^0.69 |
| **Fontes** | Inter / Plus Jakarta Sans | google_fonts |
| **Tema** | adaptive_theme | ^3.x |
| **Notificações** | flutter_local_notifications | ^18.x |
| **IA** | Claude API via anthropic_sdk_dart | ^0.9 |
| **Upload de imagem** | Supabase Storage + image (resize) | — |
| **CI/CD** | GitHub Actions | — |

> **Banco de dados**: PostgreSQL via Supabase. Nunca Firebase, nunca MongoDB, nunca Firestore.
> **Isar está abandonado pelo autor** — não usar, não sugerir.

---

## 3. Arquitetura

### 3.1 Padrão: Feature-First + Clean Architecture

Cada feature é **auto-contida** com 3 camadas internas:

```
lib/
├── core/
│   ├── constants/        # FlowdayColors, FlowdayRadius, strings
│   ├── theme/            # AppTheme (light + dark)
│   ├── router/           # GoRouter — app_router.dart
│   ├── errors/           # Failures, exceptions
│   └── utils/            # formatters, extensions
├── features/
│   ├── auth/
│   ├── agenda/
│   ├── habits/
│   ├── wellbeing/
│   ├── rewards/
│   ├── finance/
│   └── profile/
├── shared/
│   ├── widgets/          # componentes usados em 2+ features
│   └── services/         # AIService, NotificationService, SyncService
└── main.dart
```

Dentro de cada feature:

```
feature_name/
├── data/
│   ├── datasources/      # remote (Supabase) + local (Drift)
│   ├── models/           # DTOs com fromJson/toJson
│   └── repositories/     # implementações concretas
├── domain/
│   ├── entities/         # Dart puro — sem imports Flutter/Supabase
│   ├── repositories/     # interfaces abstratas
│   └── usecases/         # 1 usecase = 1 ação de negócio
└── presentation/
    ├── screens/
    ├── widgets/
    └── providers/        # Riverpod notifiers/providers
```

### 3.2 Regras de Dependência

```
Presentation → Domain ← Data
```

- `domain/` **nunca importa** Flutter, Supabase, Drift — apenas Dart puro
- `data/` implementa interfaces do `domain/`
- `presentation/` usa entidades do `domain/` via providers Riverpod

### 3.3 Riverpod — Padrões Obrigatórios

```dart
// SEMPRE usar code generation (@riverpod)
@riverpod
class TasksNotifier extends _$TasksNotifier {
  @override
  Future<List<Task>> build() async { ... }
}

// NUNCA usar ref.read() dentro de build()
// SEMPRE usar ref.watch() para estado reativo
// SEMPRE usar AsyncValue.when() para estados de loading/error/data
// SEMPRE usar keepAlive: true para providers globais (XP, perfil)
```

### 3.4 Nomenclatura

| Elemento | Padrão | Exemplo |
|---|---|---|
| Arquivos | `snake_case.dart` | `task_repository.dart` |
| Classes | `PascalCase` | `TaskRepositoryImpl` |
| Providers | `camelCaseProvider` | `todayTasksProvider` |
| Notifiers | `PascalCaseNotifier` | `TasksNotifier` |
| Constantes | `camelCase` em classes | `FlowdayColors.primary` |
| Rotas | kebab-case | `/home/task/new` |

---

## 4. Banco de Dados (PostgreSQL / Supabase)

### 4.1 Convenções de Schema

```sql
-- Toda tabela DEVE ter:
id         UUID    PRIMARY KEY DEFAULT gen_random_uuid()
user_id    UUID    NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE
created_at TIMESTAMPTZ DEFAULT NOW()
updated_at TIMESTAMPTZ DEFAULT NOW()  -- com trigger
synced     BOOLEAN DEFAULT TRUE       -- controle offline (FALSE = aguarda sync)

-- Toda tabela DEVE ter:
ALTER TABLE nome ENABLE ROW LEVEL SECURITY;
CREATE POLICY nome_user_policy ON nome
  FOR ALL USING (auth.uid() = user_id);
```

### 4.2 Trigger Padrão de updated_at

```sql
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

-- Aplicar em cada tabela:
CREATE TRIGGER nome_updated_at
  BEFORE UPDATE ON nome
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

### 4.3 Tabelas do Projeto

| Tabela | Módulo | Observação |
|---|---|---|
| `user_profiles` | Perfil | Extende auth.users |
| `tasks` | Agenda | Campo `ai_suggested`, `google_event_id` |
| `task_reminders` | Agenda | N notificações por tarefa |
| `habits` | Hábitos | `frequency`: daily/weekly/specific |
| `habit_completions` | Hábitos | `UNIQUE(habit_id, date)` |
| `wellbeing_checkins` | Bem-estar | `UNIQUE(user_id, date)` |
| `transactions` | Financeiro | FK para `finance_categories` |
| `finance_categories` | Financeiro | System seeds + user custom (v2) |
| `user_xp` | Recompensas | Saldo denormalizado para performance |
| `xp_transactions` | Recompensas | Log imutável de XP |
| `user_achievements` | Recompensas | `UNIQUE(user_id, achievement_key)` |
| `achievement_progress` | Recompensas | Progresso em direção a conquistas |
| `data_export_requests` | Perfil | LGPD — exportação de dados |

### 4.4 Índices Obrigatórios

Sempre criar índice em colunas usadas em `WHERE` e `ORDER BY`:

```sql
-- Padrão mínimo por tabela com user_id + date:
CREATE INDEX idx_nome_user_date ON nome(user_id, date DESC);
```

### 4.5 RLS — Nunca Esquecer

**Toda** tabela com dados de usuário **DEVE** ter RLS habilitado antes de ir a produção.
`anon key` é pública — sem RLS, qualquer usuário autenticado lê dados de todos.

---

## 5. Estratégia Offline-First

### Regra de ouro

```
Escrever: salva LOCAL (Drift) primeiro → tenta Supabase imediatamente → se offline, fila com synced=FALSE
Ler: SEMPRE do Drift (fonte de verdade local) → Supabase atualiza Drift em background
```

### Campo synced

Toda tabela Drift tem `BoolColumn get synced`. O `SyncService` varre registros com `synced = false` ao detectar conectividade.

```dart
// Fluxo padrão de escrita
Future<void> saveLocal(Entity entity) async {
  await _drift.insert(entity.toCompanion(synced: false));
  _trySyncInBackground(entity.id);
}

void _trySyncInBackground(String id) {
  unawaited(() async {
    if (await _connectivity.isOnline) {
      await _supabase.from('table').upsert(entity.toJson());
      await _drift.markSynced(id);
    }
  }());
}
```

---

## 6. Design System

### 6.1 Cores (NUNCA hardcode fora de FlowdayColors)

```dart
class FlowdayColors {
  static const primary    = Color(0xFF1D9E75); // agenda, hábitos, CTAs
  static const aiBlue     = Color(0xFF378ADD); // elementos de IA
  static const xpPurple   = Color(0xFF7F77DD); // recompensas, XP
  static const background = Color(0xFFF4FAF7); // fundo
  static const textPrimary = Color(0xFF085041); // texto
  static const surface    = Color(0xFFFFFFFF); // cards
  static const error      = Color(0xFFEF4444); // erros, saldo negativo
  static const warning    = Color(0xFFF59E0B); // atenção
}
```

### 6.2 Border Radius

```dart
class FlowdayRadius {
  static const double screen = 28.0;  // telas, bottom sheets
  static const double card   = 12.0;  // cards
  static const double button = 12.0;  // botões
  static const double pill   = 100.0; // chips, badges
}
```

### 6.3 Tipografia

- Família: **Inter** (preferencial) ou **Plus Jakarta Sans**
- Pesos permitidos: **400** (regular) e **500** (medium) APENAS
- Nunca usar Bold (700) ou Light (300)

### 6.4 Estilo Visual

- **Flat**: `elevation: 0` em todos os botões e cards
- **Sombras**: máximo `BoxShadow` com opacidade 6–8% e blur 8px — opcional
- **Ícones**: outlined (não filled)
- **Sem gradientes**
- **Espaçamento**: múltiplos de 8px (8, 16, 24, 32)

### 6.5 Regra Dark Mode

**Nunca** usar `Colors.white` ou `Colors.black` hardcodados em widgets.
Sempre usar `Theme.of(context).colorScheme.surface` e `.onSurface`.

---

## 7. Multiplataforma — Regras de Compatibilidade

O app DEVE rodar em **iOS, Android, Web e Desktop** sem branches de código na UI.

| Regra | Detalhe |
|---|---|
| Verificar suporte web antes de adicionar package | Ver aba "Platforms" no pub.dev |
| Notificações web | `flutter_local_notifications` não agenda no web — usar FCM |
| Platform checks | Abstrair em interfaces, não `if (Platform.isAndroid)` na UI |
| GoRouter | URL-based — essencial para deep links no web |
| Fonts | Declarar no `pubspec.yaml` — não carregam automaticamente no web |
| Storage local | `shared_preferences` funciona no web (localStorage) |
| Drift no web | Usa IndexedDB automaticamente — zero mudança de código |

---

## 8. Integração com IA (Claude API)

### 8.1 Regra de Segurança

**NUNCA** expor a API key no app Flutter. Em produção, todas as chamadas à IA passam por **Supabase Edge Function** como proxy.

```
Flutter → POST /functions/v1/ai-suggest (JWT do usuário)
Edge Function → Anthropic API (ANTHROPIC_API_KEY no ambiente Supabase)
Edge Function → retorna resposta para o Flutter
```

Em desenvolvimento local, usar `--dart-define=ANTHROPIC_API_KEY=...`.

### 8.2 Modelo padrão

Usar **Claude Haiku** (`claude-haiku-4-5-20251001`) para todas as features in-app (respostas curtas, baixo custo).

### 8.3 As 4 funções de IA no Flowday

| Função | Trigger | Output |
|---|---|---|
| Sugestão de horário | Criar tarefa | JSON `{"time": "HH:MM", "reason": "..."}` |
| Feedback de bem-estar | Check-in diário | Texto empático, 3–4 linhas |
| Insight financeiro | Botão no relatório mensal | Texto analítico, 4–5 linhas |
| Mensagem de conquista | Achievement desbloqueado | Texto celebratório, 2–3 linhas |

### 8.4 Tom da IA

- **Bem-estar**: empático, acolhedor, nunca clínico ou alarmista. Se ansiedade > 7, sugerir gentilmente 1 técnica simples.
- **Agenda**: direto, prático, justificativa em 1 frase.
- **Financeiro**: analítico mas acessível, sem julgamentos.
- **Conquistas**: entusiasmado, genuíno, motivador.

---

## 9. Tabela de XP

| Ação | Constante | XP |
|---|---|---|
| Completar tarefa | `complete_task` | +10 |
| Completar hábito | `complete_habit` | +15 |
| Check-in bem-estar | `wellbeing_checkin` | +5 |
| Registrar transação | `add_transaction` | +3 |
| Streak 7 dias (hábito) | `habit_streak_7` | +50 bônus |
| Streak 30 dias (hábito) | `habit_streak_30` | +200 bônus |
| Completar onboarding | `complete_onboarding` | +100 |

**Fórmula de nível**: `nivel = floor(xp_total / 500) + 1`

**Regra imutável**: XP nunca é retirado por inatividade. Só é revertido ao desmarcar um hábito no mesmo dia.

---

## 10. Roadmap Resumido

| Versão | Foco | Entregável |
|---|---|---|
| v0.1 | Setup | Projeto configurado, design system, GoRouter, ProviderScope |
| v0.2 | Onboarding | Auth real, fluxo completo de cadastro e configuração inicial |
| v0.3 | MVP Core | Agenda + Hábitos funcionais com offline-first |
| v0.4 | Loveable | XP, conquistas, bem-estar, streaks, animações |
| v0.5 | Complete | Módulo financeiro completo |
| v0.6 | IA | Claude API integrada nos 4 pontos |
| v0.7 | Polimento | Perfil, dark mode, estados vazios, performance |
| v1.0 | Lançamento | App Store + Google Play + Web deploy |

---

## 11. Comportamento Esperado do Claude

Ao receber qualquer solicitação neste projeto, o Claude deve:

### O que SEMPRE fazer

1. **Seguir Feature-First**: nunca criar arquivo fora da estrutura `features/nome/camada/`
2. **Null safety**: todo código Dart com null safety — sem `dynamic`, sem `!` sem justificativa
3. **Riverpod com code gen**: usar `@riverpod` annotation, nunca `StateProvider` manual para lógica complexa
4. **AsyncValue.when()**: sempre tratar loading, error e data nas telas
5. **Usar FlowdayColors**: nunca cores hardcodadas em widgets
6. **Offline-first**: toda escrita vai ao Drift primeiro
7. **RLS no SQL**: todo `CREATE TABLE` vem com `ENABLE ROW LEVEL SECURITY` e policy
8. **PostgreSQL**: usar UUID como PK, gen_random_uuid() como default, TIMESTAMPTZ para datas
9. **Multiplataforma**: verificar compatibilidade web de packages antes de sugerir
10. **Pequenos commits**: ao gerar código, organizar por feature — não misturar módulos no mesmo arquivo

### O que NUNCA fazer

- Usar Firebase, Firestore ou qualquer alternativa ao Supabase/PostgreSQL
- Usar Isar como banco local (abandonado pelo autor)
- Usar `setState` para lógica de negócio — isso vai para o Notifier
- Hardcodar cores RGB diretamente em widgets
- Usar peso de fonte 700 (Bold) ou 300 (Light)
- Expor `ANTHROPIC_API_KEY` em código Flutter
- Criar tabela sem RLS
- Usar `INT` como PK — sempre UUID
- Adicionar dependência sem verificar suporte a Flutter Web
- Duplicar lógica de negócio entre módulos — extrair para `shared/services/`

### Como gerar código

- **Entidade primeiro**: definir a entity em `domain/entities/` antes de qualquer widget
- **Interface antes da implementação**: criar `repository.dart` (abstract) antes de `repository_impl.dart`
- **Widget com ConsumerWidget**: toda tela usa `ConsumerWidget` ou `ConsumerStatefulWidget`
- **Separar widget grande**: se um widget ultrapassa ~80 linhas, extrair em sub-widget
- **Comentários**: apenas onde a lógica não é auto-evidente — não comentar o óbvio

---

## 12. Variáveis de Ambiente

```
# .env (nunca commitar)
SUPABASE_URL=https://seu-projeto.supabase.co
SUPABASE_ANON_KEY=eyJ...
ANTHROPIC_API_KEY=sk-ant-...  # apenas para dev local

# Injetar via --dart-define no flutter run/build
flutter run \
  --dart-define=SUPABASE_URL=$SUPABASE_URL \
  --dart-define=SUPABASE_ANON_KEY=$SUPABASE_ANON_KEY \
  --dart-define=ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY
```

Acessar no Dart:
```dart
const supabaseUrl = String.fromEnvironment('SUPABASE_URL');
const supabaseKey = String.fromEnvironment('SUPABASE_ANON_KEY');
```

---

## 13. Packages Aprovados

```yaml
dependencies:
  flutter_riverpod: ^2.6.1
  riverpod_annotation: ^2.4.1
  go_router: ^14.6.1
  supabase_flutter: ^2.8.0
  drift: ^2.20.0
  sqlite3_flutter_libs: ^0.5.0
  fl_chart: ^0.69.0
  flutter_local_notifications: ^18.0.0
  google_fonts: ^6.2.1
  adaptive_theme: ^3.6.0
  anthropic_sdk_dart: ^0.9.0
  google_sign_in: ^6.2.0
  shared_preferences: ^2.3.0
  connectivity_plus: ^6.0.0
  path_provider: ^2.1.0
  path: ^1.9.0
  image: ^4.2.0          # resize de avatar
  intl: ^0.19.0          # formatação de datas e moeda
  uuid: ^4.4.0

dev_dependencies:
  riverpod_generator: ^2.4.3
  drift_dev: ^2.20.0
  build_runner: ^2.4.9
  flutter_lints: ^4.0.0
```

Packages **não aprovados** (não adicionar sem discussão):
- `isar` — abandonado
- `hive` — sem queries relacionais, sem suporte web nativo
- `get` / `GetX` — conflita com Riverpod
- `bloc` — já decidido por Riverpod
- `provider` — substituído por Riverpod

---

## 14. Referências Internas (Obsidian Vault)

A documentação completa está em `C:\Users\Jean\Documents\Obsidian\Claude\01 - Projects\Flow\`:

- `Flowday — Visão Geral.md` — hub central
- `Produto/Flowday — Módulos e Funcionalidades.md` — spec de produto
- `Produto/Flowday — Fluxo de Telas.md` — 39 telas com diagrama Mermaid
- `Produto/Flowday — Identidade Visual.md` — design system completo
- `Roadmap/Flowday — Roadmap.md` — versões v0.1 a v1.x
- `Tech/Flowday — Arquitetura Técnica.md` — decisões técnicas
- `Produto/Módulos/Módulo 1 — Agenda com IA.md`
- `Produto/Módulos/Módulo 2 — Hábitos.md`
- `Produto/Módulos/Módulo 3 — Bem-estar.md`
- `Produto/Módulos/Módulo 4 — Recompensas e XP.md`
- `Produto/Módulos/Módulo 5 — Financeiro.md`
- `Produto/Módulos/Módulo 6 — Perfil e Configurações.md`
