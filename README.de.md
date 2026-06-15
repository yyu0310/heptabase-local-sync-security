# Heptabase Local Sync Security

**Datenschutz zuerst, KI-sicher. Kontinuierliche lokale Synchronisierung von Heptabase zu strukturiertem Markdown.**

Sie entscheiden genau, welche Karten ein KI-Agent zu sehen bekommt. Alles andere bleibt außer Reichweite.

[English](README.md) · [繁體中文](README.zh-TW.md) · [简体中文](README.zh-CN.md) · [日本語](README.ja.md) · [한국어](README.ko.md) · [Tiếng Việt](README.vi.md) · [Español](README.es.md) · [Français](README.fr.md) · Deutsch · [العربية](README.ar.md) · [עברית](README.he.md) · [Русский](README.ru.md) · [Українська](README.uk.md)

> Inoffizielles Tool. Keine Verbindung zu oder Unterstützung durch Heptabase. Derzeit nur macOS.

---

## Warum es das gibt

Es begann mit einem einfachen Ziel: Heptabase mit Claude Code verbinden, damit ein KI-Agent meine Notizen lesen kann.

Der offizielle Weg ist Heptabases eigenes [CLI](https://github.com/heptameta/heptabase-cli-skills), das Sie in der App unter Settings, AI Features, CLI aktivieren. Es ist **fail-open**: Einmal autorisiert, kann der Agent Ihre gesamte Wissensbasis lesen. Drittanbieter-Tools wie der `heptabase-mcp`-Server funktionieren genauso. Das ist in Ordnung, wenn alles in Ihrer Wissensbasis geteilt werden kann. Aber wenn Sie vertrauliche Karten neben denen haben, die die KI nutzen soll, was die meisten Menschen tun, funktioniert dieser Weg nicht.

Das Kernproblem: Die Datenschutzmauer muss an der Grenze liegen, *was die KI lesen kann*. Diese Grenze liegt außerhalb von Heptabase, in der Art, wie Sie Notizen an den Agenten übergeben. Die eigentliche Aufgabe eines solchen Tools ist daher, **die vertraulichen Karten an einem Ort zu halten, den die KI nicht erreichen kann, und nur den Rest als KI-lesbares Markdown zu exportieren.** Die Notizen zu synchronisieren ist die einfache Hälfte.

## Der Name

Der Name gliedert sich in drei Teile, jeder beschreibt etwas anderes. `heptabase` ist die Datenquelle, `local-sync` beschreibt den Mechanismus, und `security` definiert die Rolle.

Security trägt zwei Bedeutungen. Die erste ist Schutz: Das Tool steht zwischen der KI und Ihrer Wissensbasis und nutzt einen fail-closed-Mechanismus, um zu entscheiden, welche Karten in das Sichtfeld der KI gelangen und welche vollständig blockiert werden. Keine Datenlecks, über die man sich Sorgen machen müsste. Die zweite ähnelt eher einer gewissenhaften Sekretärin: Sie organisiert Ihre Notizen still im Hintergrund, synchronisiert automatisch alle 15 Minuten an den konfigurierten Ort, damit Ihre KI-Tools sofort Kontext haben, wenn Sie sie öffnen. Den Eingang bewachen und die Routinearbeit erledigen. Das ist es, was der Name bedeutet.

## Was es tut

Heptabase Local Sync Security führt ein lokales Python-Skript aus, das Ihre Heptabase-Datenbank direkt liest und ausgewählte Karten als Markdown-Dateien an die von Ihnen gewählten Ziele schreibt. Ein `launchd`-Job führt es alle 15 Minuten aus, damit das Markdown mit Ihren Notizen synchron bleibt. Der KI-Agent liest nur den exportierten Markdown-Ordner. Er berührt die Datenbank nie.

- **Liest die lokale Live-Datenbank.** Heptabase hat Ende 2025 aufgehört, [automatische lokale Backups](https://support.heptabase.com/en/articles/11064116-how-does-auto-backup-work-in-heptabase) anzubieten, daher ist das direkte Lesen der Live-DB nun der zuverlässige Weg für kontinuierliche lokale Synchronisierung.
- **Strukturtreue Konvertierung.** Tabellen, Bullet / Todo / Toggle-Listen, verschachtelte Abschnitte und Videos werden aus Heptabases ProseMirror-Schema per Reverse Engineering gewonnen und als sauberes Markdown gerendert.
- **Beliebiges Ziel-Routing.** Jedes Whiteboard kann in seinen eigenen Ordner gelangen, einschließlich eines absoluten Pfads, der ein Board direkt in ein separates Projekt legt. Sie können mit einem KI-Agenten zusammenarbeiten, um zu entscheiden, auf welchen lokalen Pfad Python zeigen soll.

## Das fail-closed Datenschutzmodell

Eine Karte wird nur exportiert, wenn sie zu einer von zwei expliziten Erlaubnislisten passt. Standardmäßig wird nichts gelesen.

| Quelle | Regel |
|---|---|
| **`whitelist_whiteboards`** | Whiteboards, die Sie benennen. Nur Karten auf der *Oberfläche* jedes Boards werden gelesen. Sub-Whiteboards werden nicht verfolgt. Um eines einzuschließen, benennen Sie es ebenfalls. |
| **`card_map`** | Eine `Titel -> exakter Pfad`-Ebene. Diese Titel werden immer synchronisiert, und ihr Pfad hat Vorrang. |
| **`blacklist_whiteboards`** | Karten auf diesen Boards werden *bevor* Inhalte gelesen werden abgezogen. Die Blacklist schlägt die Whitelist, also wird eine Karte, die versehentlich auf zwei Boards liegt, trotzdem blockiert. |
| **Sub-Whiteboards (unbenannt)** | Eine Karte in ein Sub-Whiteboard zu verschieben, ändert ihre `whiteboard_id`, sodass ein Oberflächenscan sie nie sieht. Per Struktur ausgeschlossen, nicht durch eine Regel, die Sie sich merken müssen. |

Noch wichtiger: Jede Abfrage, die den Titel oder Inhalt einer Karte berührt, ist auf Whitelist-Whiteboard-IDs oder `card_map`-Titel beschränkt. Titel und Inhalt einer nicht gelisteten Karte werden nie in den Speicher geladen.

Zwei Designprinzipien folgen daraus. **Struktureller Ausschluss schlägt subtraktiven Ausschluss**: Eine Karte, die die Abfrage nicht erreichen kann, ist sicherer als eine, die nach dem Lesen gefiltert wird. Das Tool ist so gebaut, dass Sie sich nie fragen müssen, ob eine Karte geleakt ist, garantiert durch Struktur statt durch nachträgliche Überprüfung.

## Vergleich

| | Heptabase Local Sync Security | Offizielles Heptabase CLI | Andere Export-Tools |
|---|---|---|---|
| Datenschutzmodell | Fail-closed Erlaubnisliste | Fail-open (gesamte Wissensbasis) | Vollständiger Export |
| Kontinuierliche lokale Sync | Ja (`launchd`, 15 Min.) | Lesen auf Anfrage | Einmaliger Export |
| Liest lokale Live-DB | Ja | Variiert | Benötigt oft Backup-Datei |
| Strukturtreues Markdown | Tabellen, Listen, Abschnitte, Video | Variiert | Variiert |
| Board-spezifisches Ziel-Routing | Ja, inkl. absoluter Pfade | Nein | Nein |

Dies ist kein vollständiger Ersatz für Heptabase, und "besser als offiziell" gilt nur auf drei Achsen: kontrollierbarer Datenschutz, kontinuierliche lokale Synchronisierung und Strukturtreue. Die Zielgruppe ist bewusst eng: macOS-Nutzer, die in Heptabase leben und sich darum kümmern, was eine KI sehen kann. Wenn das auf Sie zutrifft, ist dieses Tool genau für Ihren Fall gebaut.

## Installation

Voraussetzungen: macOS, Python 3.9+, die Heptabase-Desktop-App installiert.

```bash
git clone https://github.com/yyu0310/heptabase-local-sync-security.git
cd heptabase-local-sync-security
cp config.example.json config.json
```

Bearbeiten Sie dann `config.json` (jedes Feld hat einen Inline-Kommentar, der es erklärt):

1. Bestätigen Sie, dass `db_path` auf Ihre lokale `hepta.db` zeigt.
2. Setzen Sie `base_output_dir` und `board_output_dir` auf den Ort, an dem Markdown geschrieben werden soll.
3. Listen Sie die zu exportierenden Whiteboards unter `whitelist_whiteboards` auf.
4. Fügen Sie unter `card_map` alle genauen Pfad-Überschreibungen hinzu.

Führen Sie zuerst eine Vorschau aus, die nichts schreibt:

```bash
python3 heptabase_sync.py --dry
```

Wenn der Plan stimmt, führen Sie es wirklich aus:

```bash
python3 heptabase_sync.py
```

### Automatische Synchronisierung alle 15 Minuten

```bash
cp com.example.heptasieve.plist ~/Library/LaunchAgents/
# bearbeiten Sie die kopierte Datei: setzen Sie die absoluten Pfade und bestätigen Sie Ihren python3-Pfad
launchctl load ~/Library/LaunchAgents/com.example.heptasieve.plist
```

## Mit einem KI-Agenten verwenden

Dieses Tool enthält agentenlesbare Dokumente, damit Sie es durch Gespräche mit einem KI-Coding-Agenten einrichten können, anstatt Schritte manuell zu befolgen:

- [`AGENTS.md`](AGENTS.md) und [`CLAUDE.md`](CLAUDE.md) sagen dem Agenten, wie er dieses Tool verstehen und konfigurieren soll.
- [`llms.txt`](llms.txt) ist ein Index der Dokumentation für LLMs.
- [`skills/setup-heptasieve/`](skills/setup-heptasieve/) ist ein Claude Code Skill, der die gesamte Einrichtung in einer einzigen Anfrage führt.

Richten Sie Ihren Agenten auf den exportierten Markdown-Ordner, nie auf `hepta.db`. Diese Trennung ist der eigentliche Punkt.

### Szenario 1: Claude Code Projektassistent

Sie entwickeln ein quantitatives Trading-Tool. Heptabase hat drei Whiteboards: `Trading-Strategie-Forschung`, `System-Design-Notizen`, und `Persönliche Finanzunterlagen`. Sie fügen nur die ersten zwei zu `whitelist_whiteboards` hinzu, und das Tool synchronisiert sie alle 15 Minuten nach `/projects/trading/heptabase-export/`. Sie fügen diesen Ordnerpfad zur CLAUDE.md Ihres Projekts hinzu, damit Claude Code ihn als Hintergrundkontext liest. Das Whiteboard für Finanzunterlagen ist nie in der Allowlist; Claude Code kann nicht einmal dessen Titel sehen.

### Szenario 2: Persönliche Gedächtnisschicht über Tools hinweg

Verschiedene KI-Tools teilen kein gemeinsames Gedächtnis, was Sie zwingt, bei jedem Wechsel den Kontext neu zu erklären. Synchronisieren Sie Ihre häufig verwendeten Referenznotizen, Arbeitskontext und Forschungszusammenfassungen in einen Markdown-Ordner, und jedes KI-Tool, das lokale Dateien liest, macht dort weiter, wo Sie aufgehört haben, innerhalb von Sekunden. Welche Whiteboards in die Allowlist kommen und welche gesperrt bleiben, entscheidet Ihre Konfiguration, nicht das Vertrauen in das Urteil des KI-Tools.

## Wie es funktioniert

Architekturdetails finden Sie in [`ARCHITECTURE.md`](ARCHITECTURE.md): der Datenfluss, die fail-closed-Reihenfolge innerhalb von `build_plan`, die gelesenen Datenbanktabellen und die Datenschutz-Invarianten, die beim Ändern des Codes zu bewahren sind.

## Einschränkungen und ehrliche Hinweise

- **Das Schema ist fragil.** Es hängt von Heptabases interner Datenbankstruktur ab. Ein Heptabase-Update kann es beschädigen. Es ist von Natur aus inoffiziell.
- **Die Live-DB zu lesen ist nicht offiziell genehmigt.** Es funktioniert in der Praxis gut und ist nur lesend, aber Sie sollten wissen, dass es keine unterstützte Integration ist.
- **Nur macOS.** Die Pfade und die `launchd`-Konfiguration setzen macOS voraus.

## Lizenz

[MIT](LICENSE).
