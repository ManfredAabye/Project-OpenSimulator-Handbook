# Kapitel 6: PhysicsModules – Physik-Engines

**Pfad:** `OpenSim/Region/PhysicsModules/`

---

## Übersicht

OpenSimulator unterstützt mehrere auswechselbare Physik-Engines. Die Engine wird über die `[Startup]`-Sektion in `OpenSim.ini` konfiguriert:

```ini
[Startup]
physics = BulletSim    ; oder ubODE
meshing = Meshmerizer  ; oder ubODEMeshmerizer
```

### Engines im Vergleich

| Engine | Modul-ID | Status | Beschreibung |
| --- | --- | --- | --- |
| **BulletSim** | `BulletSim` | Standard, empfohlen | Bullet Physics via C++-Wrapper |
| **ubODE** | `ubODE` | Alternativ | ODE-Fork mit Erweiterungen |

---

## 6.1 Gemeinsame Basisklassen

**Pfad:** `PhysicsModules/SharedBase/`

### `PhysicsScene` (abstrakt)

Abstrakte Basisklasse für alle Physik-Szenen. Definiert die API zwischen Simulator und Physik-Engine:

```csharp
public abstract class PhysicsScene
{
    // Einen Avatar hinzufügen
    public abstract PhysicsActor AddAvatar(
        string avName, Vector3 position, Vector3 velocity,
        Vector3 size, bool isFlying);

    // Ein Objekt hinzufügen
    public abstract PhysicsObject AddPrimShape(
        string primName, PrimitiveBaseShape pbs,
        Vector3 position, Vector3 size, Quaternion rotation,
        bool isPhysical, uint localID);

    // Simulation für ein Zeitintervall vorwärts laufen lassen
    public abstract float Simulate(float timeStep);

    // Kollisionsabfrage
    public abstract void GetResults();

    // Terrain-Daten übergeben
    public abstract void SetTerrain(float[] heightMap);
    public abstract void DeleteTerrain();

    // Raycast
    public abstract void RaycastWorld(Vector3 position, Vector3 direction,
        float length, RaycastCallback retMethod);
}
```

### `PhysicsActor` (abstrakt)

Repräsentiert ein physikalisches Objekt (Prim oder Avatar) in der Physik-Engine.

| Property | Typ | Beschreibung |
| --- | --- | --- |
| `Position` | `Vector3` | Aktuelle Position |
| `Velocity` | `Vector3` | Geschwindigkeit |
| `RotationalVelocity` | `Vector3` | Rotationsgeschwindigkeit |
| `Orientation` | `Quaternion` | Rotation |
| `IsPhysical` | `bool` | Physikalisch aktiv |
| `IsColliding` | `bool` | Aktuelle Kollision |
| `CollidingObj` | `bool` | Mit Objekt kollidierend |
| `Buoyancy` | `float` | Auftrieb (0=sinken, 1=schweben) |
| `Flying` | `bool` | Avatar fliegt |

### `RayFilterFlags`

Bitmaske für Raycast-Filter:

| Flag | Bedeutung |
| --- | --- |
| `water` | Wasseroberfläche |
| `land` | Gelände |
| `agent` | Avatare |
| `nonphysical` | Nicht-physikalische Prims |
| `physical` | Physikalische Prims |
| `phantom` | Phantom-Objekte |
| `volumedtc` | Volume-Detect-Objekte |
| `ClosestHit` | Nur nächsten Treffer zurückgeben |
| `BackFaceCull` | Rückseiten ignorieren |

---

## 6.2 BulletSim

**Pfad:** `PhysicsModules/BulletS/`  
**Assembly:** `OpenSim.Region.PhysicsModule.BulletS.dll`

BulletSim ist ein Wrapper um die **Bullet Physics Library** – eine weit verbreitete Open-Source-Physikengine (auch in OpenSim und vielen Spielen verwendet).

### Architektur

```text
BSScene               ← IRegionModule, PhysicsScene
├── BSPhysObject      ← Abstrakte Basis für alle Bullet-Objekte
│   ├── BSPrim        ← Prim / Mesh-Objekt
│   │   ├── BSPrimLinkable   ← Prim in Linkset
│   │   └── BSPrimDisplaced  ← Prim mit Versatzkorrektur
│   └── BSCharacter   ← Avatar
├── BSActors          ← Verhaltensmuster (Actor-Modell)
│   ├── BSActorAvatarMove    ← Avatar-Bewegungslogik
│   ├── BSActorHover         ← Schweben
│   ├── BSActorMoveToTarget  ← Fahrziel-Annäherung
│   ├── BSActorLockAxis      ← Achsensperren
│   ├── BSActorSetForce      ← Dauerkraft
│   └── BSActorSetTorque     ← Dauerdrehmoment
├── BSLinkset         ← Verwaltung verbundener Prims
│   ├── BSLinksetCompound    ← Effizientere Compound-Shape
│   └── BSLinksetConstraints ← Constraint-basiertes Linkset
├── BSConstraintCollection   ← Alle Constraints der Szene
├── BSShapeCollection ← Gecachte Kollisionsformen
├── BSTerrainManager  ← Terrain-Physik
└── BSDynamics        ← Fahrzeugphysik
```

### `BSScene` – Kernklasse

| Feld | Typ | Beschreibung |
| --- | --- | --- |
| `World` | `BulletWorld` | Handle zur Bullet-Welt |
| `PhysObjects` | `Dictionary<uint, BSPhysObject>` | Alle Physik-Objekte |
| `ObjectsWithCollisions` | `HashSet<BSPhysObject>` | Objekte mit aktiver Kollision |
| `ObjectsWithUpdates` | `HashSet<BSPhysObject>` | Objekte mit pending Updates |
| `m_maxSubSteps` | `int` | Max. Bullet-Substeps pro Frame |
| `m_fixedTimeStep` | `float` | Fester Physik-Zeitschritt (1/60s) |
| `m_simulationStep` | `long` | Aktueller Simulations-Schritt-Zähler |
| `Constraints` | `BSConstraintCollection` | Alle Joint-Constraints |

**Physik-Loop:**

```text
Scene.Update()
  └─ BSScene.Simulate(timeStep)
       1. BeforeStep Event
       2. Bullet-DLL: BulletSimAPI.PhysicsStep(world, timeStep, ...)
       3. Kollisionen einlesen → ObjectsWithCollisions
       4. Positions-Updates einlesen → ObjectsWithUpdates
       5. AfterStep Event
       6. Updates an SceneObjectPart/ScenePresence zurückschreiben
       7. Kollisions-Events an Scripts weitergeben
```

### `BSPrim` – Prim-Physik

Repräsentiert ein einzelnes Prim oder Mesh im Bullet-Universum.

Kollisionsformen (`BSShape`):

| Form | Verwendung |
| --- | --- |
| `BSShapeBox` | Einfaches Rechteck |
| `BSShapeSphere` | Kugel |
| `BSShapeConvexHull` | Konvexe Hülle (für Meshes) |
| `BSShapeMesh` | Trianguliertes Mesh (Static only) |
| `BSShapeCompound` | Zusammengesetztes Linkset |
| `BSShapeNative` | Bullet-native Primitiven (Zylinder etc.) |

### `BSDynamics` – Fahrzeugphysik

Implementiert das LSL-Fahrzeugsystem (22 Fahrzeugtypen, z. B. `VEHICLE_TYPE_CAR`, `VEHICLE_TYPE_BOAT`).

LSL-Fahrzeugparameter (aus `SOPVehicle.cs`):

| Parameter | Beschreibung |
| --- | --- |
| `LINEAR_MOTOR_DIRECTION` | Richtung + Geschwindigkeit |
| `ANGULAR_MOTOR_DIRECTION` | Rotationsrichtung |
| `LINEAR_FRICTION_TIMESCALE` | Brems-Zeitkonstante |
| `ANGULAR_FRICTION_TIMESCALE` | Rotations-Reibung |
| `HOVER_HEIGHT` | Schwebehöhe |
| `HOVER_EFFICIENCY` | Schwebepräzision |
| `BANKING_EFFICIENCY` | Kurvenneigung |

### `BSParam` – Konfigurierbare Parameter

Hunderte von Bullet-Parametern sind über `BSParam.cs` konfigurierbar und können per Konsolenbefehl zur Laufzeit geändert werden:

```bash
physics set avatarFriction 0.2
physics set vehicleRestitution 0.1
```

---

## 6.3 ubODE

**Pfad:** `PhysicsModules/ubOde/`  
**Assembly:** `OpenSim.Region.PhysicsModule.ubOde.dll`

ubODE ist ein Fork der **Open Dynamics Engine (ODE)** mit OpenSimulator-spezifischen Erweiterungen.

### Architektur ubODE

```text
ODEModule             ← INonSharedRegionModule
└── ODEScene          ← PhysicsScene
    ├── ODEPrim       ← Prim-Physik-Objekt
    ├── ODECharacter  ← Avatar-Physik
    ├── ODEDynamics   ← Fahrzeugphysik
    └── ODEMeshWorker ← Asynchrone Mesh-Vorbereitung
```

### Besonderheiten von ubODE

- Eigenes Meshing-System (`ubOdeMeshing`)
- Verbesserte Avatar-Kollisionsgeometrie
- Sitz-Avatar-Physik (`ODESitAvatar`)
- Async Mesh-Aufbereitung (`ODEMeshWorker`) für bessere Performance

### `ODEScene`

Verwaltet die ODE-Welt:

| Feld | Bedeutung |
| --- | --- |
| `world` | ODE-Welt-Handle |
| `space` | ODE-Collision-Space |
| `contactgroup` | ODE-Kontaktgruppe für Kollisionen |
| `m_rayCastManager` | `ODERayCastRequestManager` |

---

## 6.4 Meshing

**Pfad:** `PhysicsModules/Meshing/` und `PhysicsModules/ubOdeMeshing/`

### Aufgabe

Konvertiert Prim-Geometrie (aus `PrimitiveBaseShape`) in Dreiecksgitter für die Physik-Engine.

### `Meshmerizer` (Standard-Mesher)

Für BulletSim.

Unterstützte Geometrietypen:

| Typ | Beschreibung |
| --- | --- |
| Basic Prims | Box, Cylinder, Sphere, Torus, Tube, Ring |
| Sculpt Maps | Texturbild → 3D-Mesh |
| Mesh Assets | COLLADA-Mesh aus Asset |

### `ubODEMeshmerizer`

Spezialisierter Mesher für ubODE mit optimierter Kollisionsgeometrie.

---

## 6.5 ConvexDecompositionDotNet

**Pfad:** `PhysicsModules/ConvexDecompositionDotNet/`

Zerlegt ein konkaves Mesh in mehrere konvexe Teilformen (**HACD – Hierarchical Approximate Convex Decomposition**).

Wird verwendet für:

- Physikalische Mesh-Objekte (Type = Convex)
- Verbesserte Kollisionsgenauigkeit bei komplexen Meshes

---

## 6.6 Konfiguration

### BulletSim-Konfiguration (`OpenSim.ini`)

```ini
[BulletSim]
BulletEngine = BulletUnmanaged     ; oder BulletXNA
MaxSubSteps = 3
FixedTimeStep = 0.01667            ; 1/60s
MaxCollisionsPerFrame = 2048
MaxUpdatesPerFrame = 8192
DefaultFriction = 0.2
DefaultDensity = 10.0
DefaultRestitution = 0.0
AvatarCapsuleRadius = 0.37
AvatarCapsuleHeight = 1.5
```

### ubODE-Konfiguration

```ini
[ubODE]
Enabled = true
gravityx = 0
gravityy = 0
gravityz = -9.8
AvatarTerminalVelocity = 54.0
```

---

*Weiter: [Kapitel 7 – Data / Datenbankschicht](07_Data.md)*
