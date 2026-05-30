# Prompt-Sammlung: Sourcecode pruefen, testen und TODOs loesen (OpenSimulator)

Basis-Pfad fuer alle Prompts:

- D:/opensim-next/opensim

Hinweise zur Nutzung:

- Platzhalter wie [DATEI], [BEREICH], [TODO_TEXT] ersetzen.
- Die Prompts sind fuer Copilot/LLM-Workflows in VS Code ausgelegt.

## 1) Codeabschnitt fachlich pruefen

```text
Analysiere den Codeabschnitt in [DATEI] bei [BEREICH].
Ziel: funktionale Risiken, Nebenwirkungen, Regressionen und fehlende Randfallbehandlung finden.
Erwarte Ausgabe in dieser Reihenfolge:
1. Kritische Findings (mit Schweregrad, Grund, konkreter Stelle)
2. Mittlere Findings
3. Geringe Findings
4. Offene Annahmen/Fragen
5. Konkrete Aenderungsvorschlaege als kurze Diff-Ideen
Wenn keine Findings vorhanden sind, schreibe explizit: Keine Findings, und nenne verbleibende Testluecken.
```

## 2) Codeabschnitt auf Performance und Skalierung pruefen

```text
Pruefe [DATEI] Bereich [BEREICH] auf Performance-Probleme unter Last.
Fokussiere auf:
- unnoetige Allokationen
- teure Schleifen/N+1-Muster
- Locking/Threading-Risiken
- I/O im Hot Path
Liefere:
1. Priorisierte Engpaesse
2. Erwarteter Impact (CPU, RAM, Latenz)
3. Konkrete Optimierungen mit minimal-invasivem Ansatz
4. Messplan (welche Metriken vor/nach Aenderung)
```

## 3) Unit-Test-Plan fuer einen Codebereich

```text
Erstelle fuer [DATEI] Bereich [BEREICH] einen Unit-Test-Plan.
Erwarte:
1. Testfaelle als Tabelle: Name | Eingabe | Erwartung | Warum wichtig
2. Mock/Stubs-Bedarf
3. Edge Cases und Fehlerpfade
4. Minimale Testdaten
5. Reihenfolge der Implementierung (Quick Wins zuerst)
Nutze vorhandene Projektkonventionen im OpenSimulator-Code.
```

## 4) Integrationstest-Plan fuer Modulgrenzen

```text
Erstelle einen Integrationstest-Plan fuer [MODUL_A] mit [MODUL_B].
Kontextpfad: D:/opensim-next/opensim.
Liefere:
1. Abhaengigkeiten und Schnittstellen
2. Test-Szenarien fuer Erfolg/Fehlschlag/Timeout/Retry
3. Benoetigte Konfiguration und Test-Fixtures
4. Erwartete Logs/Signale zur Verifikation
5. Risiken bei Parallelitaet und verteilten Zustaenden
```

## 5) Alle TODOs in Datei in konkrete Tasks umwandeln

```text
Lies alle TODO-Kommentare in [DATEI] und wandle sie in umsetzbare Aufgaben um.
Format pro TODO:
1. TODO-Text (Original)
2. Problem in einem Satz
3. Loesungsansatz (technisch konkret)
4. Akzeptanzkriterien
5. Teststrategie
6. Aufwand (S/M/L)
7. Risiko (Niedrig/Mittel/Hoch)
Sortiere nach Risiko und Impact.
```

## 6) TODO zu fertiger Loesung mit Patch machen

```text
Bearbeite dieses TODO und schlage eine vollstaendige Loesung vor:
Datei: [DATEI]
Zeile/Bereich: [BEREICH]
TODO: [TODO_TEXT]

Liefere in dieser Reihenfolge:
1. Kurze Problemdefinition
2. Geplanter Loesungsweg
3. Patch (moeglichst klein, keine unnoetigen Refactorings)
4. Begruendung, warum keine Regression zu erwarten ist
5. Tests, die neu hinzukommen oder angepasst werden muessen
```

## 7) TODO-Bulk-Bearbeitung fuer mehrere Dateien

```text
Ich gebe dir eine TODO-Liste aus mehreren Dateien im OpenSimulator.
Erstelle eine priorisierte Umsetzungs-Roadmap.
Erwarte:
1. Clusterung nach Themen (Security, Permissions, Networking, Performance, Cleanup)
2. Abhaengigkeiten zwischen TODOs
3. Empfohlene Reihenfolge in 2-Wochen-Sprints
4. Risiken und Rueckfallplan pro Sprint
5. Definition of Done pro Cluster
```

## 8) Security-Fokus fuer TODOs

```text
Pruefe alle TODOs in [DATEI_ODER_ORDNER] auf Security-Relevanz.
Finde insbesondere fehlende Checks fuer:
- Berechtigungen/Authentifizierung
- Eingabevalidierung
- Privilege Escalation
- Informationsleck in Logs/Fehlermeldungen
Ausgabe:
1. Security-TODOs (priorisiert)
2. Konkrete Gegenmassnahmen
3. Negative Tests (Missbrauchsszenarien)
```

## 9) Prompt fuer Testcode-Generierung

```text
Schreibe Tests fuer [DATEI] und Bereich [BEREICH].
Anforderungen:
1. Bestehenden Stil der Testprojekte einhalten
2. Deterministische Tests (keine flaky sleeps)
3. Aussagekraeftige Assert-Meldungen
4. Erfolgspfad und Fehlerpfad abdecken
5. Nur notwendige Mocks
Gib zuerst eine kurze Teststrategie, dann den vollstaendigen Testcode.
```

## 10) Prompt fuer Review eines vorgeschlagenen TODO-Fixes

```text
Reviewe den folgenden Fix fuer ein TODO im OpenSimulator:
[PATCH_ODER_CODE]

Pruefe:
1. Korrektheit und Vollstaendigkeit
2. Seiteneffekte in angrenzenden Komponenten
3. Thread-Safety und Race Conditions
4. Logging/Fehlerbehandlung
5. Wartbarkeit
Ausgabe als Findings nach Schweregrad plus konkrete Nachbesserungen.
```

## 11) Prompt fuer schnelle Tagesroutine

```text
Arbeite diese Routine fuer den OpenSimulator-Ordner D:/opensim-next/opensim ab:
1. Nenne mir die 10 wichtigsten offenen TODOs nach Risiko x Impact.
2. Waehle die Top 3 fuer heute.
3. Gib fuer jedes Top-TODO eine minimale Loesungsskizze.
4. Erzeuge fuer jedes Top-TODO die noetigen Tests.
5. Erstelle eine kurze Checkliste fuer den Abschluss.
```

## 12) Prompt-Template mit striktem Ausgabeformat (maschinenlesbar)

```text
Analysiere TODOs in [DATEI_ODER_ORDNER] und antworte ausschliesslich als JSON.
Schema:
{
  "items": [
    {
      "file": "...",
      "line": 0,
      "todo": "...",
      "category": "security|bug|perf|refactor|test|docs",
      "severity": "low|medium|high|critical",
      "proposal": "...",
      "acceptance_criteria": ["..."],
      "tests": ["..."],
      "effort": "S|M|L"
    }
  ]
}
Keine zusaetzlichen Erlaeuterungen ausserhalb von JSON.
```
