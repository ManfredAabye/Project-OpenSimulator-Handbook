# Kapitel 7: Data – Datenbankschicht

**Pfad:** `OpenSim/Data/`

---

## Übersicht

Die Datenbankschicht abstrahiert den Zugriff auf alle persistenten Daten hinter Interfaces. Konkrete Implementierungen existieren für **MySQL**, **SQLite** und **PostgreSQL**.

### Unterstützte Datenbanken

| DB | Pfad | Einsatzgebiet |
| --- | --- | --- |
| **SQLite** | `Data/SQLite/` | Standalone, Entwicklung, kleine Setups |
| **MySQL** | `Data/MySQL/` | Produktion, mittlere bis große Grids |
| **PostgreSQL** | `Data/PGSQL/` | Alternative für große Grids |

### Konfiguration

In `OpenSim.ini` (Standalone-Beispiel):

```ini
[DatabaseService]
StorageProvider = "OpenSim.Data.SQLite.dll"
ConnectionString = "URI=file:OpenSim.db,version=3"
```

Für MySQL:

```ini
[DatabaseService]
StorageProvider = "OpenSim.Data.MySQL.dll"
ConnectionString = "Data Source=localhost;Database=opensim;User ID=opensim;Password=geheim;"
```

---

## 7.1 Interfaces

**Pfad:** `OpenSim/Data/`

Alle Datenbankzugriffe laufen über wohldefinierte Interfaces. Kein Simulator-Code referenziert direkt MySQL- oder SQLite-Klassen.

### Vollständige Interface-Übersicht

| Interface | Datei | Zugehörige Tabellen |
| --- | --- | --- |
| `IAssetData` | `IAssetData.cs` | `assets` |
| `IXAssetData` | `IXAssetDataPlugin.cs` | `XAssets` |
| `IFSAssetData` | `IFSAssetData.cs` | Dateisystem |
| `IInventoryData` | `IInventoryData.cs` | `inventoryfolders`, `inventoryitems` |
| `IXInventoryData` | `IXInventoryData.cs` | `inventoryfolders`, `inventoryitems` |
| `IRegionData` | `IRegionData.cs` | `regions` |
| `ISimulationData` | (in Region) | `prims`, `primshapes`, `terrain`, `land`, `primitems` |
| `IUserAccountData` | `IUserAccountData.cs` | `UserAccounts` |
| `IAuthenticationData` | `IAuthenticationData.cs` | `auth`, `tokens` |
| `IPresenceData` | `IPresenceData.cs` | `Presence` |
| `IAvatarData` | `IAvatarData.cs` | `Avatars` |
| `IFriendsData` | `IFriendsData.cs` | `Friends` |
| `IEstateDataStore` | `IEstateDataStore.cs` | `estate_settings`, `estate_users` etc. |
| `IGridUserData` | `IGridUserData.cs` | `GridUser` |
| `IGroupsData` / `IXGroupData` | `IGroupsData.cs` etc. | `os_groups_*` |
| `IOfflineIMData` | `IOfflineIMData.cs` | `im_offline` |
| `IMuteListData` | `IMuteListData.cs` | `MuteList` |
| `IProfilesData` | `IProfilesData.cs` | `userprofile_*` |
| `IAgentPreferencesData` | `IAgentPreferencesData.cs` | `AgentPrefs` |

---

## 7.2 Migrations-System

**Datei:** `Data/Migration.cs`

Das Migrations-System automatisiert Datenbankschema-Updates nach dem **Rails-Migrations-Prinzip**:

### Funktionsweise

Jede Implementierung liefert SQL-Migrationsdateien als eingebettete Ressourcen:

```bash
MySQL/Resources/
├── RegionStore.migrations
├── UserAccount.migrations
├── InventoryStore.migrations
└── ...
```

Jede `.migrations`-Datei enthält nummerierte Versionsblöcke:

```sql
:VERSION 5   # -------------------------
BEGIN;
CREATE TABLE IF NOT EXISTS `UserAccounts` (
  `PrincipalID` char(36) NOT NULL,
  ...
);
COMMIT;

:VERSION 6   # -------------------------
BEGIN;
ALTER TABLE `UserAccounts` ADD `active` INT NOT NULL DEFAULT '1';
COMMIT;
```

Beim Start:

1. `Migration.Update()` liest aktuelle Schema-Version aus der DB
2. Alle noch nicht angewendeten Versionen werden sequenziell ausgeführt
3. Neue Version wird gespeichert

### Anwendung

```csharp
Migration um = new Migration(dbConnection, assembly, "UserAccount");
um.Update();   // Führt alle fehlenden Migrationen aus
```

---

## 7.3 MySQL – Datenbankschema

**Pfad:** `Data/MySQL/`

### Tabellen-Übersicht

#### Szene / Prims

| Tabelle | Beschreibung |
| --- | --- |
| `prims` | Alle Prims einer Region (Position, Rotation, Flags …) |
| `primshapes` | Geometrieparameter (`PrimitiveBaseShape`) |
| `primitems` | Inventar-Items von Prims (Skripte, Notecards …) |
| `terrain` | Terrain-Daten als komprimierter Blob |
| `landaccesslist` | Zugangs-ACL pro Parzelle |
| `land` | Parzellen (Grenzen, Flags, Eigentümer) |
| `regionban` | Gesperrte Agenten pro Region |
| `regionsettings` | Region-Einstellungen (Sun, Maturity …) |
| `regionextra` | Zusätzliche Regionsparameter |
| `linksetdata` | LinksetData-Schlüssel/Wert-Paare |

#### `prims`-Tabelle (wichtigste Spalten)

| Spalte | Typ | Beschreibung |
| --- | --- | --- |
| `UUID` | `char(36)` | Primärer Schlüssel (Prim-UUID) |
| `RegionUUID` | `char(36)` | Zugehörige Region |
| `SceneGroupID` | `char(36)` | UUID des Root-Prims (Linkset) |
| `CreatorID` | `varchar(255)` | Ersteller |
| `OwnerID` | `char(36)` | Eigentümer |
| `GroupID` | `char(36)` | Gruppe |
| `PositionX/Y/Z` | `double` | Weltkoordinaten |
| `RotationX/Y/Z/W` | `double` | Quaternion-Rotation |
| `VelocityX/Y/Z` | `double` | Geschwindigkeit |
| `OwnerMask` | `int` | Berechtigungs-Flags |
| `ObjectFlags` | `int` | Prim-Flags (physisch, phantomhaft …) |
| `Text` | `varchar(255)` | Floating Text |
| `Description` | `varchar(255)` | Prim-Beschreibung |
| `SitName` | `varchar(255)` | Sitz-Button-Text |
| `ScriptAccessPin` | `int` | Script-Zugangs-PIN |
| `ParticleSystem` | `blob` | Partikeleffekt-Daten |
| `TextureAnimation` | `blob` | Textur-Animations-Daten |

#### Inventar

| Tabelle | Beschreibung |
| --- | --- |
| `inventoryfolders` | Inventar-Ordner (UUID, ParentID, Name, Type) |
| `inventoryitems` | Inventar-Einträge (UUID, AssetID, Name, Permissions) |

#### Benutzer & Authentifizierung

| Tabelle | Beschreibung |
| --- | --- |
| `UserAccounts` | Benutzerkonten (Name, UUID, Level, Flags) |
| `auth` | Passwort-Hashes und Salt |
| `tokens` | Authentifizierungs-Session-Tokens |
| `GridUser` | Letzter Login-Ort und Online-Status |
| `Presence` | Aktive Sessions (SessionID → RegionID) |

#### Grid

| Tabelle | Beschreibung |
| --- | --- |
| `regions` | Registrierte Regionen (UUID, Name, Koordinaten, URI) |
| `Avatars` | Avatar-Appearance (UUID → XML-Blob) |
| `Friends` | Freundeslisten (PrincipalID, Friend, Flags) |
| `MuteList` | Mute-Einträge pro Benutzer |
| `im_offline` | Gespeicherte Offline-Nachrichten |

#### Estates

| Tabelle | Beschreibung |
| --- | --- |
| `estate_settings` | Estate-Konfiguration |
| `estate_managers` | Estate-Manager-Liste |
| `estate_users` | Zugelassene Benutzer |
| `estate_groups` | Zugelassene Gruppen |
| `estateban` | Gesperrte Benutzer |
| `estate_map` | Zuordnung Region → Estate |

---

## 7.4 SQLite

**Pfad:** `Data/SQLite/`

Verwendet `System.Data.SQLite` (und/oder `Mono.Data.Sqlite`).  
Speichert alle Daten in **einer einzigen Datei** (`OpenSim.db` im `bin/`-Verzeichnis).

Gleiche Tabellen wie MySQL, jedoch mit SQLite-spezifischen Datentypen.

Geeignet für:

- Standalone-Betrieb mit wenigen Benutzern
- Entwicklung und Tests
- Schnelle Ersteinrichtung ohne separaten DB-Server

---

## 7.5 PostgreSQL

**Pfad:** `Data/PGSQL/`

Vollständige Implementierung für PostgreSQL (via `Npgsql`).  
Schema und Daten identisch mit MySQL, jedoch mit PostgreSQL-Syntax.

Besonderheiten:

- Unterschiedliche Datentypen (z. B. `uuid` statt `char(36)`)
- Sequences statt AUTO_INCREMENT
- Eigene Migrationsdateien (`PGSQL/Resources/`)

---

## 7.6 Simulation Data – Persistenz der Szene

Die Szenen-Daten (Prims, Terrain, Land) werden über `ISimulationDataService` gespeichert:

```csharp
public interface ISimulationDataService
{
    void StorePrimGroup(SceneObjectGroup obj);
    void RemoveObject(UUID uuid, UUID regionUUID);
    List<SceneObjectGroup> LoadObjects(UUID regionID);

    void StoreTerrain(double[] ter, UUID regionID);
    double[] LoadTerrain(UUID regionID);

    void StoreLandObject(ILandObject Parcel);
    List<LandData> LoadLandObjects(UUID regionID);

    void RemoveRegion(UUID regionID);
}
```

Beim Regionstart lädt `Scene.LoadPrimsFromStorage()` alle Objekte aus der DB und recreiert die `SceneObjectGroup`-Instanzen.

Regelmäßige Backups erfolgen alle `m_update_backup`-Frames (Standard: 200 Frames ≈ alle 2 Sekunden für geänderte Objekte).

---

*Zurück: [Kapitel 6 – PhysicsModules](06_PhysicsModules.md)*  
*Zurück zum: [Inhaltsverzeichnis](Inhaltsverzeichnis.md)*
