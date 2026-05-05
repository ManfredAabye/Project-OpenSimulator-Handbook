# Kapitel 3: Region – Simulationsserver

**Pfad:** `OpenSim/Region/`

---

## Übersicht

Das `Region`-Verzeichnis enthält die eigentliche Simulationslogik. Es ist in mehrere Unterassemblies aufgeteilt:

| Assembly | Pfad | Inhalt |
| --- | --- | --- |
| `OpenSim.Region.Framework` | `Region/Framework/` | Szenenmodell, Interfaces |
| `OpenSim.Region.ClientStack.LindenUDP` | `Region/ClientStack/Linden/` | LLUDP-Protokoll |
| `OpenSim.Region.CoreModules` | `Region/CoreModules/` | Standard-Module |
| `OpenSim.Region.OptionalModules` | `Region/OptionalModules/` | Optionale Module |
| `OpenSim.Region.ScriptEngine.Yengine` | `Region/ScriptEngine/YEngine/` | Skript-Engine |
| `OpenSim.Region.PhysicsModule.BulletS` | `Region/PhysicsModules/BulletS/` | Bullet-Physik |
| `OpenSim.Region.PhysicsModule.ubOde` | `Region/PhysicsModules/ubOde/` | ODE-Physik |

---

## 3.1 Application – Startlogik

**Pfad:** `Region/Application/`

### Startsequenz

```text
OpenSim.exe
  └─ OpenSim.cs              ← Einstiegspunkt, Konsolen-Loop
       └─ OpenSimBase.cs     ← Gemeinsame Basis
            └─ ConfigurationLoader.cs   ← Liest alle .ini-Dateien
            └─ RegionApplicationBase.cs ← Startet HTTP-Server, lädt Plugins
                 └─ ApplicationPlugins/LoadRegions  ← Lädt Regions/*.ini
                 └─ ApplicationPlugins/RegionModulesController ← lädt Module
```

### `ConfigurationLoader`

Fusioniert mehrere INI-Quellen in dieser Reihenfolge:

1. `OpenSimDefaults.ini` (nie editieren)
2. `OpenSim.ini`
3. Includes aus `[Includes]`-Sektion
4. Kommandozeilenparameter (`-inifile`, `-inimaster` …)

Alle Werte aus späteren Quellen überschreiben frühere.

---

## 3.2 Framework/Scenes – Szenenmodell

**Pfad:** `Region/Framework/Scenes/`

### Klassen-Hierarchie

```text
EntityBase             (abstrakt – alle Szeneobjekte)
├── SceneObjectGroup   (Linkset: ein oder mehrere Prims)
│    └── SceneObjectPart  (ein einzelnes Prim)
└── ScenePresence      (eingeloggter Avatar)
```

### `Scene` (Hauptklasse)

`Scene.cs` ist eine `partial class` – verteilt auf mehrere Dateien:

| Datei | Inhalt |
| --- | --- |
| `Scene.cs` | Tick-Loop, Startup/Shutdown, Physik-Integration |
| `Scene.Inventory.cs` | Inventaroperationen (Rez, Take, Drop …) |
| `Scene.PacketHandlers.cs` | Eingehende LLUDP-Paket-Callbacks |
| `Scene.Permissions.cs` | Berechtigungsprüfungen über `IPermissionsModule` |

**Tick-Loop** (`Scene.Update()`):

Der Simulator läuft mit einer konfigurierbaren Rate (Standard: 11250 µs = ~89 Frames/s).  
Pro Frame werden nacheinander verarbeitet:

1. Eingehende Pakete (ClientStack)
2. Agent-Updates (Bewegung, Animation)
3. Physik-Schritt (`PhysicsScene.Simulate()`)
4. Skript-Events abfeuern
5. Objekt-Updates an Clients senden
6. Periodisches Backup in die Datenbank

Relevante Konfigurationsparameter in `OpenSim.ini`:

```ini
[Startup]
physics = BulletSim          ; oder ubODE
meshing = Meshmerizer

[SceneUpdatePeriod]
; 1000ms / frames_per_second
```

---

### `SceneGraph`

`Scenes/SceneGraph.cs` – früher `InnerScene`

Verwaltet alle Objekte und Avatare einer Region in mehreren Lookup-Strukturen:

| Feld | Typ | Beschreibung |
| --- | --- | --- |
| `m_scenePartsByID` | `Dictionary<UUID, SceneObjectPart>` | Alle Prims per UUID |
| `m_scenePartsByLocalID` | `Dictionary<uint, SceneObjectPart>` | Alle Prims per LocalID |
| `m_scenePresenceMap` | `Dictionary<UUID, ScenePresence>` | Alle Avatare per UUID |
| `m_updateList` | `Dictionary<UUID, SceneObjectGroup>` | Objekte mit pending Updates |

Statistiken (thread-safe Zähler):

| Feld | Bedeutung |
| --- | --- |
| `m_numRootAgents` | Eingeloggte Avatare (Root) |
| `m_numChildAgents` | Nachbar-Avatare (Child) |
| `m_numPrim` | Anzahl Prims |
| `m_numMesh` | Anzahl Mesh-Objekte |
| `m_physicalPrim` | Physikalische Prims |
| `m_activeScripts` | Laufende Skripte |

---

### `SceneObjectGroup` (Linkset)

`Scenes/SceneObjectGroup.cs`

Ein `SceneObjectGroup` besteht aus einem **Root-Prim** und beliebig vielen **Child-Prims**, die als `SceneObjectPart`-Liste verwaltet werden.

Wichtige Eigenschaften:

| Property | Typ | Beschreibung |
| --- | --- | --- |
| `UUID` | `UUID` | Objekt-UUID |
| `LocalId` | `uint` | Lokale numerische ID (innerhalb Region) |
| `Name` | `string` | Objektname |
| `AbsolutePosition` | `Vector3` | Weltkoordinaten |
| `GroupRotation` | `Quaternion` | Rotation des Linksets |
| `RootPart` | `SceneObjectPart` | Das Root-Prim |
| `Parts` | `SceneObjectPart[]` | Alle Prims (Root + Children) |
| `IsSelected` | `bool` | Aktuell vom Editor selektiert |
| `IsPhysical` | `bool` | Physikalisches Objekt |
| `IsAttachment` | `bool` | Am Avatar befestigt |
| `KeyframeMotion` | `KeyframeMotion` | Animierter Pfad |

**LSL Script-Events** (definiert in `SceneObjectGroup.cs`):

```csharp
public enum ScriptEventCode : int
{
    attach = 0, state_exit = 1, timer = 2, touch = 3,
    collision = 4, collision_end = 5, collision_start = 6,
    control = 7, dataserver = 8, email = 9,
    http_response = 10, listen = 15, money = 16,
    on_rez = 36, sensor = 37, http_request = 38,
    linkset_data = 41,
    // ... insgesamt 42 Event-Codes
}
```

---

### `SceneObjectPart` (Prim)

`Scenes/SceneObjectPart.cs` – ca. 7000 Zeilen

Das zentrale Datenobjekt für jedes einzelne Prim in der Welt.

Wichtige Kategorien von Properties:

**Geometrie:**

| Property | Beschreibung |
| --- | --- |
| `Shape` | `PrimitiveBaseShape` – Geometrieparameter |
| `Scale` | `Vector3` – Größe |
| `OffsetPosition` | Position relativ zum Root-Prim |
| `RotationOffset` | Rotation relativ zum Root-Prim |
| `SitTargetPosition` | Sitz-Offset |

**Metadaten:**

| Property | Beschreibung |
| --- | --- |
| `Name` | Prim-Name |
| `Description` | Prim-Beschreibung |
| `CreatorID` | UUID des Erstellers |
| `OwnerID` | UUID des Eigentümers |
| `GroupID` | UUID der Gruppe |
| `LastOwnerID` | Vorheriger Eigentümer |
| `CreationDate` | Unix-Timestamp |

**Physik:**

| Property | Beschreibung |
| --- | --- |
| `PhysActor` | `PhysicsActor` – Physik-Objekt |
| `PhysicsShapeType` | Prim / Convex / None |
| `Buoyancy` | Schwimm-/Sinkverhalten |
| `VehicleType` | Fahrzeugtyp |

**Skript:**

| Property | Beschreibung |
| --- | --- |
| `Inventory` | `SceneObjectPartInventory` – Skripte, Items |
| `scriptEvents` | Bitmaske aktiver Script-Events |
| `PassTouches` | Weitergabe an Root |

---

### `ScenePresence` (Avatar)

`Scenes/ScenePresence.cs` – ca. 5500 Zeilen

Repräsentiert einen eingeloggten Avatar.

**Root vs. Child Agent:**

| Typ | Bedeutung |
| --- | --- |
| Root Agent | Avatar ist in *dieser* Region eingeloggt |
| Child Agent | Avatar ist in einer Nachbarregion; wird für Sichtweite und Crossing vorgehalten |

Wichtige Eigenschaften:

| Property | Beschreibung |
| --- | --- |
| `UUID` | Avatar-UUID |
| `Name` | Anzeigename |
| `IsChildAgent` | `true` = Child, `false` = Root |
| `AbsolutePosition` | Weltkoordinaten |
| `Velocity` | Bewegungsvektor |
| `Rotation` | Blickrichtung |
| `Appearance` | `AvatarAppearance` |
| `ControlFlags` | Tastatureingaben (WASD, Maus …) |
| `SitOnObjectLocalPartID` | Lokal-ID des Prims, auf dem gesessen wird |
| `Flying` | Fliegen aktiv? |
| `GodController` | Gott-Modus-Status |
| `ScriptControlled` | Steuerung durch Skript übernommen |
| `Environment` | Viewer-Environment (EEP) |

---

### `EventManager`

`Scenes/EventManager.cs`

Der zentrale Event-Bus der Szene. Module abonnieren Events, die bei bestimmten Aktionen gefeuert werden.

Auswahl wichtiger Events:

| Event | Wann | Abonnenten (typisch) |
| --- | --- | --- |
| `OnFrame` | Jeden Tick | Sun, Wind, Clouds, Scripting |
| `OnNewClient` | Avatar verbindet sich | Fast alle Module |
| `OnClientClosed` | Avatar trennt Verbindung | Cleanup-Module |
| `OnAddToScene` | Objekt zur Szene hinzugefügt | Physik, Skript |
| `OnSceneObjectLoaded` | Objekt aus DB geladen | Physik |
| `OnObjectBeingRemovedFromScene` | Objekt wird entfernt | Physik, Skript |
| `OnChatFromClient` | Chat vom Viewer | Chat-Modul, Scripting |
| `OnChatBroadcast` | Chat-Aussendung | Alle Clients |
| `OnNewInventoryItemUploadComplete` | Asset hochgeladen | Inventar |
| `OnTeleportStart` | Teleport beginnt | Diverse Module |
| `OnTerrainTainted` | Gelände verändert | Terrain-Persistenz |
| `OnParcelPropertiesUpdateRequest` | Parzelle geändert | Land-Modul |

Abonnement-Muster:

```csharp
scene.EventManager.OnFrame += MyFrameHandler;
scene.EventManager.OnNewClient += MyNewClientHandler;
```

---

## 3.3 ClientStack – LLUDP-Protokoll

**Pfad:** `Region/ClientStack/Linden/`

Implementiert das **Linden Lab UDP Protocol** (LLUDP) – das proprietäre Netzwerkprotokoll des Second-Life-Viewers.

### Architektur

```text
UDP Socket (Port 9000)
  └─ LLUDPServer           ← Empfangs-/Sendeloop
       └─ LLClientView     ← Pro Verbindung eine Instanz
            ├─ PacketHandler  ← Eingehende Pakete dispatchen
            └─ OutgoingQueue  ← Ausgehende Pakete priorisieren
```

### `LLClientView`

Implementiert `IClientAPI`. Kernaufgaben:

- Pakete deserialisieren und in OpenSim-Events übersetzen
- Updates (Positionen, Texturen, Inventar …) an den Viewer senden
- Throttle-Management (Bandbreite pro Kanal begrenzen)

**Throttle-Kanäle:**

| Kanal | Inhalt |
| --- | --- |
| Resend | Verlorene Pakete |
| Land | Parzellinformationen |
| Wind | Winddaten |
| Cloud | Wolkendaten |
| Task | Allgemeine Szenen-Updates |
| Texture | Texturen |
| Asset | Asset-Downloads |

---

## 3.4 CoreModules – Eingebaute Module

**Pfad:** `Region/CoreModules/`

Module werden über das **Mono.Addins**-System geladen. Jedes Modul implementiert `IRegionModule` oder `INonSharedRegionModule`.

### Modul-Lebenszyklus

```csharp
interface INonSharedRegionModule
{
    void Initialise(IConfigSource source);   // 1. Konfiguration laden
    void AddRegion(Scene scene);              // 2. In Szene einhängen
    void RegionLoaded(Scene scene);           // 3. Nach Laden aller Module
    void RemoveRegion(Scene scene);           // 4. Beim Herunterfahren
    void Close();                             // 5. Aufräumen
}
```

### Wichtige CoreModules im Detail

#### Terrain-Modul

`CoreModules/World/Terrain/`

Verwaltet Geländehöhen (256×256 Float-Werte).  
Unterstützte Werkzeuge:

- Raise / Lower / Smooth / Noise / Flatten / Revert
- Import/Export als PNG, RAW, Terragen

#### Archiver (OAR)

`CoreModules/World/Archiver/`

Erstellt und lädt OpenSim-Archive (`.oar`):

- **Speichern:** Alle Objekte, Assets, Terraindaten, Parzelldaten → ZIP
- **Laden:** Stellt eine Region komplett wieder her

Konsolenbefehl:

```bash
save oar /path/to/backup.oar
load oar /path/to/backup.oar
```

#### Permissions

`CoreModules/World/Permissions/`

Prüft für jede Operation (Move, Copy, Delete, Script, …):

1. Ist der Avatar der Eigentümer?
2. Ist die Gruppe autorisiert?
3. Gilt God-Mode?
4. Gelten Parzel-Flags?

#### Land / Parcel

`CoreModules/World/Land/`

Verwaltet Parzellen innerhalb einer Region:

- Aufteilung, Zusammenlegung, Verkauf
- Access Control Lists (ACL)
- Prim-Limits und Statistiken

---

## 3.5 OptionalModules

**Pfad:** `Region/OptionalModules/`

Aktivierung per Konfiguration. Wichtige Module:

### Materials (`OptionalModules/Materials/`)

PBR-Materialsystem (seit OpenSim 0.9.x).  
Datei: `MaterialsModule.cs`

Jedes Prim-Face kann ein Material mit folgenden Kanälen haben:

- Normale Map (Bumpmapping)
- Specular Map (Glanz)
- GLTF-Material (PBR: Albedo, Roughness, Metalness, Emissive)

Materials werden als `AssetType.Material` (UUID-referenziert) gespeichert.

### PhysicsParameters (`OptionalModules/PhysicsParameters/`)

Erlaubt das Lesen und Setzen von Physik-Parametern zur Laufzeit über Konsolenbefehle oder OSSL.

### PrimLimitsModule (`OptionalModules/PrimLimitsModule/`)

Blockiert das Rezzen von Objekten, wenn das Parcel-Limit erreicht ist.

### DataSnapshot (`OptionalModules/DataSnapshot/`)

Exportiert Regionsdaten als XML für externe Suchmaschinen (z. B. `search.secondlife.com`-Protokoll).

---

## 3.6 ScriptEngine – YEngine

Siehe [Kapitel 4 – ScriptEngine / YEngine](04_ScriptEngine.md)

---

## 3.7 PhysicsModules

Siehe [Kapitel 6 – PhysicsModules](06_PhysicsModules.md)

---

*Weiter: [Kapitel 4 – ScriptEngine / YEngine](04_ScriptEngine.md)*
