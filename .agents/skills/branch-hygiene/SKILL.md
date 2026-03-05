---
name: branch-hygiene
description: >
  Branch hygiene for WhisPaste. Invoke AUTOMATICALLY at the end of every
  completed task/feature, before release tags, or when the user asks about
  branch cleanup. Ensures no stale, merged, or orphaned branches accumulate
  locally or on the remote.
metadata:
  scope: project
  domains: [git, workflow]
  triggers:
    - "branch cleanup"
    - "branch hygiene"
    - "stale branches"
    - "delete branch"
    - "branch aufräumen"
    - "Branches löschen"
    - "task complete"
    - "feature done"
    - "before release"
---

# Branch Hygiene Skill — WhisPaste

## Philosophie

**Branches sind Arbeitskopien, keine Archive.** Jeder Branch, der gemergt oder aufgegeben wurde, wird gelöscht. `main` ist der einzige langlebige Branch.

---

## ⚡ WANN dieser Skill automatisch greift

Dieser Skill wird IMMER ausgeführt, wenn:

1. Ein Feature oder Bugfix als „fertig" deklariert wird
2. Vor dem Erstellen eines Release-Tags (vor `versioning` Skill)
3. Der User explizit nach Branch-Cleanup fragt
4. Zu Beginn einer neuen Session (kurzer Check)
5. Nach einem erfolgreichen Merge in `main`

---

## Branch-Typen und Konventionen

| Prefix | Zweck | Lebensdauer | Beispiel |
|--------|-------|-------------|----------|
| `anvil/` | Anvil-Task-Branch (Medium/Large) | Bis Merge in `main` | `anvil/fix-login-crash` |
| `feature/` | Manuelle Feature-Branches | Bis Merge in `main` | `feature/dark-mode` |
| `hotfix/` | Dringende Fixes | Bis Merge in `main` | `hotfix/crash-on-start` |
| `main` | Hauptbranch | Permanent | — |

**Verbotene Branch-Namen:**
- ❌ Keine generischen Namen (`test`, `temp`, `wip`, `fix`)
- ❌ Keine Datum-basierten Namen (`2026-03-05`)
- ❌ Keine duplizierten Prefixes (`feature/feature/...`)

---

## Cleanup-Checkliste (Step-by-Step)

### Schritt 1 — Status erfassen

```powershell
# Aktuelle Situation erfassen
git rev-parse --abbrev-ref HEAD          # Aktueller Branch
git --no-pager branch                    # Alle lokalen Branches
git --no-pager branch -r                 # Alle Remote-Branches
git --no-pager branch --merged main      # Lokal gemergte Branches
git --no-pager branch -r --merged main   # Remote gemergte Branches
```

### Schritt 2 — Lokale gemergte Branches löschen

```powershell
# Alle in main gemergten lokalen Branches auflisten (ohne main selbst)
git --no-pager branch --merged main | Select-String -NotMatch '^\*|main'

# Einzeln löschen (sicher — nur bereits gemergte)
git branch -d <branch-name>
```

**Regeln:**
- Nur Branches löschen, die in `main` gemergt sind (`-d`, nicht `-D`)
- Vor dem Löschen prüfen: Ist der Branch wirklich vollständig gemergt?
- Bei Zweifel: `git log main..<branch>` — wenn leer, ist alles gemergt

### Schritt 3 — Remote-Branches prüfen

```powershell
# Stale Remote-Referenzen bereinigen
git fetch --prune

# Remote gemergte Branches anzeigen
git --no-pager branch -r --merged main | Select-String -NotMatch 'main'
```

**Remote-Löschung:**
- Remote-Branches werden automatisch vom CI-Workflow gelöscht (`.github/workflows/branch-hygiene.yml`)
- Der Workflow läuft bei jedem Push auf `main` und täglich um 03:17 UTC
- Bei Bedarf manuell via GitHub Actions „Run workflow" auslösen
- Manuelle Löschung nur wenn CI nicht verfügbar: `git push origin --delete <branch>`

### Schritt 4 — Verwaiste Branches identifizieren

```powershell
# Branches, die NICHT in main gemergt sind (potentiell verwaist)
git --no-pager branch --no-merged main

# Alter des letzten Commits auf einem Branch prüfen
git --no-pager log -1 --format="%ci %s" <branch-name>
```

**Entscheidungsmatrix für nicht-gemergte Branches:**

| Letzter Commit | Offener PR? | Aktion |
|----------------|-------------|--------|
| < 7 Tage | Ja | Behalten |
| < 7 Tage | Nein | User fragen |
| 7–30 Tage | Ja | User fragen |
| 7–30 Tage | Nein | Löschen (nach Bestätigung) |
| > 30 Tage | Egal | Löschen (nach Bestätigung) |

**Vor dem Löschen nicht-gemergter Branches:**
1. `ask_user` mit dem Branch-Namen, letztem Commit-Datum und Zusammenfassung
2. Nur löschen wenn explizit bestätigt
3. Bei Unsicherheit: Branch lokal taggen bevor gelöscht wird (`git tag archive/<branch> <branch>`)

---

## CI-Integration

### Automatischer Remote-Cleanup

Die Datei `.github/workflows/branch-hygiene.yml` übernimmt das automatische Löschen von Remote-Branches:

- **Trigger:** Push auf `main`, täglicher Cron (03:17 UTC), manuell
- **Verhalten:** Löscht alle Remote-Branches außer `main`
- **Dry-Run:** Manuell auslösbar mit `dry_run: true` — zeigt was gelöscht würde
- **Schutz:** `main` ist immer geschützt und wird nie gelöscht

### GitHub Repository-Einstellungen

Empfohlene Einstellung auf GitHub:
- ✅ **Automatically delete head branches** (Settings → General → Pull Requests)
  - Löscht Feature-Branches automatisch nach Merge eines PRs

---

## Integration mit Anvil-Workflow

### Branch-Erstellung (Anvil Step 0b)

Anvil erstellt Branches automatisch für Medium/Large Tasks:
```powershell
git checkout -b anvil/<task-id>
```

### Branch-Löschung (nach Anvil Step 8 — Commit)

Nach erfolgreichem Commit und Push:
1. Wenn auf einem `anvil/*` Branch: zurück zu `main` wechseln
2. Branch lokal löschen: `git branch -d anvil/<task-id>`
3. Remote wird vom CI-Workflow gelöscht

```powershell
# Nach erfolgreichem Merge
git checkout main
git pull origin main
git branch -d anvil/<task-id>
git fetch --prune
```

---

## Schnell-Befehle

```powershell
# Kompletter lokaler Cleanup (nur gemergte Branches)
git checkout main
git pull origin main
git fetch --prune
git branch --merged main | Select-String -NotMatch '^\*|main' | ForEach-Object { git branch -d $_.ToString().Trim() }

# Status-Übersicht
git --no-pager branch -a -v

# Einzelnen Branch sicher löschen
git branch -d <name>          # Nur wenn gemergt (sicher)
git branch -D <name>          # Force-Delete (VORSICHT — nur nach Bestätigung)
git push origin --delete <name>  # Remote löschen (nur wenn CI nicht verfügbar)
```

---

## Absolute Verbote

1. **NIEMALS** `main` löschen
2. **NIEMALS** nicht-gemergte Branches ohne `ask_user`-Bestätigung löschen
3. **NIEMALS** `git branch -D` (Force-Delete) ohne explizite User-Bestätigung
4. **NIEMALS** Remote-Branches löschen, die offene PRs haben
5. **NIEMALS** Branches anderer Entwickler löschen (in Team-Szenarien)

---

## Checkliste — Vor „Session Ende"

- [ ] Bin ich auf `main`? Falls nicht: wurde der Feature-Branch gemergt?
- [ ] Gibt es lokale Branches, die in `main` gemergt sind? → Löschen
- [ ] `git fetch --prune` ausgeführt? → Stale Remote-Referenzen bereinigt
- [ ] Gibt es verwaiste Branches > 30 Tage? → User fragen oder löschen
