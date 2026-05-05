# Kapitel 4: ScriptEngine – YEngine (LSL/OSSL)

**Pfad:** `OpenSim/Region/ScriptEngine/`  
**Assembly:** `OpenSim.Region.ScriptEngine.Yengine.dll`

---

## Übersicht

OpenSimulator führt LSL-Skripte (Linden Scripting Language) über die **YEngine** aus – eine vollständige Compiler- und Runtime-Infrastruktur, die LSL/OSSL-Quelltext in .NET-IL kompiliert und in Micro-Threads (eigene Continuation-basierte Ausführung) betreibt.

Die YEngine ist ein Fork der früheren **XMREngine** (von Mike Rieker / DreamNation), weiterentwickelt von Melanie Thielker und weiteren Kontributoren.

### Verzeichnisstruktur

```text
ScriptEngine/
├── YEngine/          ← Compiler + Runtime
├── Shared/
│   ├── Api/          ← LSL/OSSL/MOD/AA Implementierungen
│   │   ├── Implementation/   ← C#-Klassen mit Funktionen
│   │   ├── Interface/        ← Interfaces (ILSL_Api etc.)
│   │   └── Runtime/          ← Glue-Code für Laufzeit
│   └── LSL_Types.cs  ← LSL-Typsystem
└── Interfaces/       ← IScriptEngine, IScriptModule
```

---

## 4.1 LSL-Typsystem

**Datei:** `Shared/LSL_Types.cs`

LSL kennt nur wenige Datentypen. Diese werden auf C#-Typen abgebildet:

| LSL-Typ | C#-Typ | Klasse |
| --- | --- | --- |
| `integer` | `int` | `LSL_Types.LSLInteger` |
| `float` | `double` | `LSL_Types.LSLFloat` |
| `string` | `string` | `LSL_Types.LSLString` |
| `key` | `string` (UUID-Format) | `LSL_Types.LSLString` |
| `vector` | `OpenMetaverse.Vector3` | `LSL_Types.Vector3` |
| `rotation` | `OpenMetaverse.Quaternion` | `LSL_Types.Quaternion` |
| `list` | `object[]` | `LSL_Types.list` |

Die Typen sind implizit konvertierbar und überladen Operatoren (`+`, `-`, `*`, …).

---

## 4.2 YEngine – Architektur

### Klassenstruktur

```text
Yengine                     ← IRegionModule, IScriptEngine
├── XMRInstance             ← Eine Skript-Instanz (partial class)
│   ├── XMRInstMain.cs      ← Zustandsvariablen
│   ├── XMRInstCtor.cs      ← Konstruktion und Kompilierung
│   ├── XMRInstRun.cs       ← Ausführungsloop
│   ├── XMRInstQueue.cs     ← Event-Warteschlange
│   ├── XMRInstBackend.cs   ← API-Aufruf-Dispatcher
│   ├── XMRInstMisc.cs      ← Diverse Operationen
│   └── XMRInstCapture.cs   ← Checkpoint/Restore
├── MMRScriptCompile.cs     ← Kompilier-Pipeline (Einstiegspunkt)
├── MMRScriptTokenize.cs    ← Lexer
├── MMRScriptReduce.cs      ← Parser (LR)
├── MMRScriptCodeGen.cs     ← IL-Codegenerator
├── MMRScriptObjCode.cs     ← Binär-Objektcode
└── XMRScriptThread.cs      ← Worker-Thread-Pool
```

---

## 4.3 Kompilier-Pipeline

### Schritt 1: Tokenisierung (`MMRScriptTokenize.cs`)

- LSL-Quelltext → Token-Liste
- Unterstützt LSL und YSL (erweiterte Syntax für YEngine)

### Schritt 2: Parsing/Reduktion (`MMRScriptReduce.cs`)

- LR-Parser
- Erstellt einen abstrakten Syntaxbaum (AST)

### Schritt 3: IL-Codegenerierung (`MMRScriptCodeGen.cs`)

- AST → .NET-IL-Bytecode
- Verwendet `System.Reflection.Emit`
- Jeder LSL-`state`-Handler wird zu einer C#-Methode

### Schritt 4: Typenprüfung (`MMRScriptTypeCast.cs`)

Automatische Typ-Casts gemäß LSL-Spezifikation (z. B. `integer` → `string`).

### Schritt 5: Objektcode-Persistenz (`MMRScriptObjCode.cs`, `MMRScriptObjWriter.cs`)

Kompilierte Skripte werden als `.state`-Dateien gecacht:

```bash
bin/ScriptEngines/<region-uuid>/<script-uuid>.state
```

Bei erneutem Start wird gecachter Code wiederverwendet – kein Rekompilieren.

---

## 4.4 Instanz-Zustände (`XMRInstState`)

Jede Skript-Instanz (`XMRInstance`) befindet sich in einem der folgenden Zustände:

```text
CONSTRUCT → ONSTARTQ → RUNNING → IDLE
                                ↓
                             ONSLEEPQ  (llSleep, llSetTimerEvent)
                                ↓
                             REMDFROMSLPQ → ONYIELDQ → RUNNING
                             
RUNNING → SUSPENDED   (llSetScriptState(false))
RUNNING → FINISHED → IDLE / ONSTARTQ
RUNNING → RESETTING   (llResetScript)
Any    → DISPOSED     (Skript entfernt)
```

### Queues in `Yengine`

| Queue | Inhalt |
| --- | --- |
| `m_StartQueue` | Skripte mit neuem Event, noch nicht gestartet |
| `m_YieldQueue` | Skripte bereit zur Weiterführung (nach llSleep/yield) |
| `m_SleepQueue` | Skripte, die auf einen Zeitpunkt warten (sortiert nach Zeit) |

---

## 4.5 Event-System

### Event-Dispatcher

Wenn eine Szene ein Event auslöst (z. B. Kollision), ruft sie über `IScriptEngine.PostObjectEvent()` die Engine auf.

```csharp
// Beispiel: Kollision gemeldet
engine.PostObjectEvent(localID, new EventParams(
    "collision_start",
    new object[] { new LSL_Types.LSLInteger(1) },
    detectedParams));
```

Die Engine stellt das Event in die `m_EventQueue` der betroffenen `XMRInstance`.  
Max. Größe: `MAXEVENTQUEUE = 64` Events pro Instanz.

### Script-Event-Bitmasken

Jeder `SceneObjectPart` führt eine Bitmaske der abonnierten Events. Nicht abonnierte Events werden nicht zugestellt (Performance-Optimierung):

```csharp
// Event-Bits (aus SceneObjectGroup.cs)
attach           = 0x0001
touch_start      = 0x0200
touch            = 0x0400
touch_end        = 0x0800
collision_start  = 0x0008
collision        = 0x0010
collision_end    = 0x0020
timer            = 0x0040
listen           = 0x0080
```

---

## 4.6 Micro-Thread-Modell (Checkpointing)

Das Besondere an YEngine ist, dass **kooperatives Multitasking** verwendet wird.  
Skripte teilen **freiwillig** die CPU ab – an bestimmten Yield-Punkten.

Yield-Punkte entstehen bei:

- `llSleep(n)` – Warten auf Systemzeit
- Schleifendurchläufen (automatisch alle N Iterationen)
- `llSetTimerEvent()` – Zeitgesteuerte Events
- Blockierenden API-Aufrufen

Beim Yield wird der **komplette Ausführungsstack** als Checkpoint gespeichert (`XMRInstCapture.cs`) und kann später exakt an dieser Stelle fortgesetzt werden – ohne OS-Threads zu belegen.

---

## 4.7 Script-API: Shared API

**Pfad:** `ScriptEngine/Shared/Api/Implementation/`

### `LSL_Api.cs`

Implementiert alle Standard-LSL-Funktionen (ca. 6000 Zeilen).  
Jede Funktion ist eine C#-Methode:

```csharp
public LSL_Types.LSLString llGetOwner()
{
    m_host.AddScriptLPS(1);
    return m_host.OwnerID.ToString();
}

public void llSay(int channelID, string text)
{
    m_host.AddScriptLPS(1);
    // Sendet Chat-Nachricht
    ...
}
```

Wichtige Kategorien:

| Kategorie | Beispielfunktionen |
| --- | --- |
| Kommunikation | `llSay`, `llShout`, `llWhisper`, `llInstantMessage`, `llListen` |
| Bewegung | `llSetPos`, `llMoveToTarget`, `llApplyImpulse`, `llSetVelocity` |
| Avatar | `llGetOwner`, `llAvatarOnSitTarget`, `llRequestAgentData` |
| Inventar | `llGetInventoryName`, `llRezObject`, `llGiveInventory` |
| Sensoren | `llSensor`, `llSensorRepeat`, `llGetAgentList` |
| Physik | `llSetBuoyancy`, `llSetVehicleType`, `llSetForce` |
| Zeitsteuerung | `llSleep`, `llSetTimerEvent`, `llGetTime` |
| HTTP | `llHTTPRequest`, `llCreateURL` |
| JSON | `llJsonGetValue`, `llJsonSetValue` |
| String | `llSubStringIndex`, `llGetSubString`, `llStringTrim` |
| Krypto | `llMD5String`, `llSHA1String`, `llSHA256String` |
| Gelände | `llGround`, `llGroundNormal`, `llWater` |

### `OSSL_Api.cs`

OpenSim-Erweiterungen – Funktionen beginnen mit `os`:

| Funktion | Beschreibung |
| --- | --- |
| `osGetSimulatorVersion()` | Simulatorversion als String |
| `osMessageObject(UUID, msg)` | Direktnachricht an Objekt |
| `osSetRegionSun(...)` | Sonnenposition setzen |
| `osGetAgentList(scope)` | Liste aller Avatare |
| `osNpcCreate(...)` | NPC erstellen |
| `osNpcMoveTo(npc, pos)` | NPC bewegen |
| `osDrawImage(...)` | Bild in DynamicTexture zeichnen |
| `osGetInventoryDesc(name)` | Inventarbeschreibung lesen |
| `osSetSpeed(uuid, factor)` | Avatar-Geschwindigkeit setzen |
| `osGetMapTexture()` | Karten-Textur-UUID |
| `osTeleportAgent(uuid, region, pos, look)` | Avatar teleportieren |
| `osGetNotecardLine(name, line)` | Zeile aus Notecard lesen |

OSSL-Funktionen können pro Funktion mit Threat-Level gesichert werden:

```ini
[OSSL]
Allow_osNpcCreate = true
osslMinThreatLevel = VeryLow
```

### `MOD_Api.cs`

Kommunikation zwischen Skript und C#-Modulen:

```lsl
modSendCommand("ModuleName", "command", "params");
```

### `AA_Api.cs` / `LS_Api.cs`

Erweiterte Avatar-Funktionen (Avatar-Appearance, Animationen).

---

## 4.8 Konfiguration

In `OpenSim.ini` unter `[YEngine]`:

```ini
[YEngine]
Enabled = true
ScriptBasePath = ScriptEngines
AppDomainLoading = false
DeleteScriptsOnStartup = false
CompileWithDebugInformation = false
MaxScriptEventBacklog = 64
MinTimerInterval = 0.5
SleepWhilePollCount = 0
```

Unter `[OSSL]`:

```ini
[OSSL]
osslEnable = true
osslMinThreatLevel = VeryLow
Allow_osNpcCreate = ${Const|GridName}
Allow_osTeleportAgent = PARCEL_OWNER
```

---

## 4.9 Debuggen von Skripten

### Kompilier-Fehler

Fehlermeldungen werden in die Konsole des Simulators und als `llOwnerSay`-Nachricht an den Skript-Besitzer gesendet.

### Laufzeit-Fehler

Stack-Traces erscheinen in der Server-Konsole. Mit `ScriptDebug = true` in der Konfiguration werden IL-Code und Quelldateien gespeichert.

### Konsolenbefehle

```bash
scripts list                  # Alle laufenden Skripte
scripts show <uuid>           # Details einer Instanz
scripts suspend <uuid>        # Skript pausieren
scripts resume <uuid>         # Skript fortsetzen
scripts stop <uuid>           # Skript stoppen
```

---

*Weiter: [Kapitel 5 – Services / Backend-Dienste](05_Services.md)*
