# 📝 Commit messages - Conventional Commits + SemVer

Мы используем **Conventional Commits** для понятной истории и автогенерации релизов/CHANGELOG.

## Формат

```

<type>(<scope>): <summary>

\[body]

\[footer]

```

- **type**: `feat` | `fix` | `docs` | `refactor` | `perf` | `test` | `build` | `ci` | `chore` | `revert`
- **scope** (опционально): область изменения - `api`, `web`, `contracts`, `infra`, `docs`, `devops`, `db` и т.п.
- **summary**: коротко и по делу (повелительное наклонение).

### Примеры

```
feat(api): add rate limiting for POST /orders
fix(contracts): correct role check in withdraw()
docs: clarify local setup with uv
ci: run solidity tests on PR to main
refactor(web): extract useAuth() hook
```

### BREAKING CHANGES

Для ломающих изменений добавляй блок в теле или футере:

```

feat(api)!: switch auth to JWT

BREAKING CHANGE: OAuth tokens are no longer supported; migrate to JWT by 2025-10-01.

```

`!` после типа также помечает breaking-change.

### Ссылки на задачи и PR

Во футере:

```

Refs: PROJ-123
Closes: #456

```

### SemVer и релизы

- `feat` → **minor**
- `fix` → **patch**
- `BREAKING CHANGE` → **major**
- Остальные типы не изменяют версию по SemVer (если отдельно не настроено).

### Гайдлайны

- Один коммит - одна логическая единица.
- Пишите **на английском** (предпочтительно) или на русском единообразно.
- Не используем «update», «misc» и прочие расплывчатые описания.
