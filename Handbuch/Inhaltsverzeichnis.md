# OpenSimulator – Handbuch zum Quellcode

> Verzeichnis: `opensim/OpenSim/`  
> Plattform: C# / .NET 8  
> Stand: Mai 2026

---

## Inhaltsverzeichnis

> **Detailkapitel** (separate Dateien):
>
> | Nr. | Kapitel | Datei |
> | --- | --- | --- |
> | 2 | Framework – Kernbibliothek | [02_Framework.md](02_Framework.md) |
> | 3 | Region – Simulationsserver, Scene, Module | [03_Region_Scene.md](03_Region_Scene.md) |
> | 4 | ScriptEngine – YEngine, LSL/OSSL | [04_ScriptEngine.md](04_ScriptEngine.md) |
> | 5 | Services – Backend-Dienste | [05_Services.md](05_Services.md) |
> | 6 | PhysicsModules – BulletSim / ubODE | [06_PhysicsModules.md](06_PhysicsModules.md) |
> | 7 | Data – Datenbankschicht | [07_Data.md](07_Data.md) |

---

- [OpenSimulator – Handbuch zum Quellcode](#opensimulator--handbuch-zum-quellcode)
  - [Inhaltsverzeichnis](#inhaltsverzeichnis)
  - [1. Projektübersicht](#1-projektübersicht)
  - [2. Framework – Kernbibliothek](#2-framework--kernbibliothek) → [02_Framework.md](02_Framework.md)
  - [3. Region – Simulationsserver](#3-region--simulationsserver) → [03_Region_Scene.md](03_Region_Scene.md)
    - [3.1 Application – Startlogik](#31-application--startlogik)
    - [3.2 Framework/Scenes – Szenenmodell](#32-frameworkscenes--szenenmodell)
    - [3.3 ClientStack – Linden-Protokoll (UDP/LLUDP)](#33-clientstack--linden-protokoll-udplludp)
    - [3.4 CoreModules – Eingebaute Module](#34-coremodules--eingebaute-module)
      - [3.4.1 Agent-Module](#341-agent-module)
      - [3.4.2 Avatar-Module](#342-avatar-module)
      - [3.4.3 World-Module](#343-world-module)
      - [3.4.4 Scripting-Module](#344-scripting-module)
      - [3.4.5 ServiceConnectorsIn / ServiceConnectorsOut](#345-serviceconnectorsin--serviceconnectorsout)
    - [3.5 OptionalModules – Optionale Module](#35-optionalmodules--optionale-module)
    - [3.6 ScriptEngine – Skriptlaufzeit](#36-scriptengine--skriptlaufzeit) → [04_ScriptEngine.md](04_ScriptEngine.md)
      - [3.6.1 YEngine (XMR-Engine)](#361-yengine-xmr-engine)
      - [3.6.2 Shared API (LSL / OSSL / MOD / AA)](#362-shared-api-lsl--ossl--mod--aa)
    - [3.7 PhysicsModules – Physik-Engines](#37-physicsmodules--physik-engines) → [06_PhysicsModules.md](06_PhysicsModules.md)
  - [4. Services – Backend-Dienste](#4-services--backend-dienste) → [05_Services.md](05_Services.md)
    - [4.1 AssetService](#41-assetservice)
    - [4.2 InventoryService](#42-inventoryservice)
    - [4.3 GridService](#43-gridservice)
    - [4.4 UserAccountService](#44-useraccountservice)
    - [4.5 PresenceService](#45-presenceservice)
    - [4.6 AuthenticationService](#46-authenticationservice)
    - [4.7 AvatarService](#47-avatarservice)
    - [4.8 FriendsService](#48-friendsservice)
    - [4.9 LLLoginService](#49-llloginservice)
    - [4.10 HypergridService](#410-hypergridservice)
    - [4.11 Weitere Services](#411-weitere-services)
  - [5. Data – Datenbankschicht](#5-data--datenbankschicht) → [07_Data.md](07_Data.md)
    - [5.1 Interfaces / abstrakte Basisklassen](#51-interfaces--abstrakte-basisklassen)
    - [5.2 MySQL](#52-mysql)
    - [5.3 SQLite](#53-sqlite)
    - [5.4 PostgreSQL (PGSQL)](#54-postgresql-pgsql)
  - [6. Capabilities – LLSD / CAPS-Schicht](#6-capabilities--llsd--caps-schicht)
  - [7. Addons – Erweiterungsmodule](#7-addons--erweiterungsmodule)
    - [7.1 Groups](#71-groups)
    - [7.2 OfflineIM](#72-offlineim)
    - [7.3 os-webrtc-janus](#73-os-webrtc-janus)
  - [8. ApplicationPlugins – Applikations-Plugins](#8-applicationplugins--applikations-plugins)
  - [9. Server – Robust-Server (Grid-Modus)](#9-server--robust-server-grid-modus)
  - [10. ConsoleClient](#10-consoleclient)
  - [11. Tools – Hilfsprogramme](#11-tools--hilfsprogramme)
    - [11.1 pCampBot](#111-pcampbot)
    - [11.2 Configger](#112-configger)
    - [11.3 LaunchSLClient](#113-launchslclient)
  - [12. Tests](#12-tests)
  - [13. Konfiguration & Deployment](#13-konfiguration--deployment)
  - [14. Glossar](#14-glossar)

---

## 1. Projektübersicht

OpenSimulator ist ein quelloffener virtueller Welten-Server, der das Second-Life-Protokoll (LLUDP) implementiert.  
Er kann als **Standalone** (ein einziger Prozess) oder als **Grid** (mehrere Regions-Server und ein Robust-Server) betrieben werden.

| Modus           | Prozess(e)    | Konfigurationsdatei          |
| --------------- | ------------- | ---------------------------- |
| Standalone      | `OpenSim.dll` | `OpenSim.ini`                |
| Grid – Region   | `OpenSim.dll` | `OpenSim.ini` + `Robust.ini` |
| Grid – Services | `Robust.dll`  | `Robust.ini`                 |

Laufzeitanforderung: **.NET 8 Runtime**

---

## 2. Framework – Kernbibliothek

**Pfad:** `OpenSim/Framework/`

Die gemeinsame Klassenbibliothek für alle anderen Teilprojekte. Enthält:

| Datei / Ordner                                   | Inhalt                                             |
| ------------------------------------------------ | -------------------------------------------------- |
| `IScene.cs`, `ISceneAgent.cs`, `ISceneEntity.cs` | Zentrale Scene-Interfaces                          |
| `IClientAPI.cs`                                  | Abstraktion für Client-Verbindungen                |
| `RegionInfo.cs`                                  | Metadaten einer Region (Name, UUID, Koordinaten …) |
| `AssetBase.cs`                                   | Asset-Datenmodell                                  |
| `InventoryItemBase.cs`, `InventoryFolderBase.cs` | Inventar-Datenmodell                               |
| `AvatarAppearance.cs`, `AvatarWearable.cs`       | Avatar-Aussehen                                    |
| `PrimitiveBaseShape.cs`                          | Geometrie-Datenmodell für Prims                    |
| `TerrainData.cs`                                 | Gelände-Datenmodell                                |
| `EstateSettings.cs`                              | Estate-Konfiguration                               |
| `Util.cs`                                        | Allgemeine Hilfsfunktionen                         |
| `WebUtil.cs`                                     | HTTP-Helfer (REST-Aufrufe)                         |
| `Console/`                                       | Konsolenimplementierungen                          |
| `Servers/`                                       | Basis-HTTP-Server                                  |
| `ServiceAuth/`                                   | Service-Authentifizierung                          |
| `Serialization/`                                 | XML-Serialisierung (OAR/IAR)                       |

---

## 3. Region – Simulationsserver

**Pfad:** `OpenSim/Region/`

### 3.1 Application – Startlogik

**Pfad:** `OpenSim/Region/Application/`

| Datei                      | Aufgabe                              |
| -------------------------- | ------------------------------------ |
| `OpenSim.cs`               | Einstiegspunkt, Konsolen-Loop        |
| `OpenSimBase.cs`           | Gemeinsame Basis (Standalone + Grid) |
| `ConfigurationLoader.cs`   | Lädt und fusioniert INI-Dateien      |
| `RegionApplicationBase.cs` | HTTP-Server, Modul-Startup           |

### 3.2 Framework/Scenes – Szenenmodell

**Pfad:** `OpenSim/Region/Framework/Scenes/`

| Datei                 | Aufgabe                                           |
| --------------------- | ------------------------------------------------- |
| `Scene.cs`            | Haupt-Szenenklasse; Tick-Loop, Entity-Verwaltung  |
| `SceneGraph.cs`       | Zugriff auf alle Objekte/Avatare der Szene        |
| `SceneObjectGroup.cs` | Ein Linkset (ein oder mehrere Prims)              |
| `SceneObjectPart.cs`  | Ein einzelnes Prim inkl. Physik, Skript, Material |
| `ScenePresence.cs`    | Ein eingeloggter Avatar                           |
| `EventManager.cs`     | Zentrales Event-Bus-System                        |
| `TerrainChannel.cs`   | Gelände-Datenpuffer                               |
| `KeyframeMotion.cs`   | Animierte Objektbewegung                          |
| `SOPVehicle.cs`       | Fahrzeugphysik-Parameter                          |
| `LinksetData.cs`      | Linkset-Schlüssel-Wert-Speicher                   |

### 3.3 ClientStack – Linden-Protokoll (UDP/LLUDP)

**Pfad:** `OpenSim/Region/ClientStack/Linden/`

Implementiert das LLUDP-Netzwerkprotokoll des Second-Life-Viewers. Verwaltet Pakete, Throttles und die `LLClientView`.

### 3.4 CoreModules – Eingebaute Module

**Pfad:** `OpenSim/Region/CoreModules/`

#### 3.4.1 Agent-Module

| Modul               | Aufgabe                           |
| ------------------- | --------------------------------- |
| `AssetTransaction/` | Upload-Transaktionen für Assets   |
| `IPBan/`            | IP-Sperrliste                     |
| `TextureSender/`    | Textur-Streaming zum Viewer       |
| `Xfer/`             | Xfer-Protokoll (Dateiübertragung) |

#### 3.4.2 Avatar-Module

| Modul             | Aufgabe                                         |
| ----------------- | ----------------------------------------------- |
| `Attachments/`    | An-/Ablegen von Objekten am Avatar              |
| `AvatarFactory/`  | Erstellung & Aktualisierung des Avataraussehens |
| `BakedTextures/`  | Baked-Textur-Verwaltung (Serverside Appearance) |
| `Chat/`           | Lokaler Chat, Shout, Whisper                    |
| `Combat/`         | Kampfsystem (Lebenspunkte)                      |
| `Friends/`        | Freundesliste                                   |
| `Gestures/`       | Gesten-Steuerung                                |
| `Gods/`           | Gott-Modus-Berechtigungen                       |
| `Groups/`         | Gruppen-Integration                             |
| `InstantMessage/` | Direktnachrichten                               |
| `Inventory/`      | Inventarverwaltung im Simulator                 |
| `Lure/`           | Teleport-Einladungen                            |
| `Profile/`        | Avatarprofil                                    |
| `UserProfiles/`   | Erweitertes Benutzerprofil                      |

#### 3.4.3 World-Module

| Modul          | Aufgabe                                        |
| -------------- | ---------------------------------------------- |
| `Access/`      | Zugangssteuerung (Banlist, Altersverifikation) |
| `Archiver/`    | OAR-Backup & -Restore                          |
| `Estate/`      | Estate-Verwaltung                              |
| `Land/`        | Parzellenrechte und -verwaltung                |
| `LegacyMap/`   | Map-Tiles (statisch)                           |
| `LightShare/`  | Umgebungsbeleuchtung                           |
| `Media/`       | Media on a Prim (MOAP)                         |
| `Objects/`     | Objektoperationen (Rez, Delete, Copy …)        |
| `Permissions/` | Berechtigungssystem                            |
| `Serialiser/`  | XML-Serialisierung von Szenenobjekten          |
| `Sound/`       | Sound-Abspielen & -Streaming                   |
| `Terrain/`     | Geländebearbeitung & -persistenz               |
| `Vegetation/`  | Bäume & Gras                                   |
| `Wind/`        | Windmodelle                                    |
| `WorldMap/`    | Dynamische Karte & MapImage                    |
| `Warp3DMap/`   | 3D-gerendertes Kartenbild                      |

#### 3.4.4 Scripting-Module

| Modul                | Aufgabe                                      |
| -------------------- | -------------------------------------------- |
| `DynamicTexture/`    | Laufzeit-Textur-Rendering                    |
| `EMailModules/`      | E-Mail-Versand aus Skripten                  |
| `HttpRequest/`       | HTTP-Anfragen aus Skripten (`llHTTPRequest`) |
| `LSLHttp/`           | HTTP-Server in Skripten (`llCreateURL`)      |
| `ScriptModuleComms/` | Kommunikation zwischen Modulen und Skripten  |
| `WorldComm/`         | `llListen` / Chat-Listener                   |
| `XMLRPC/`            | XMLRPC-Listener aus Skripten                 |

#### 3.4.5 ServiceConnectorsIn / ServiceConnectorsOut

Verbinden den Simulator mit den Backend-Diensten (Robust) über HTTP/REST.  

- **In**: empfangen Anfragen von außen (z. B. von anderen Regionen)  
- **Out**: senden Anfragen an den Robust-Server

### 3.5 OptionalModules – Optionale Module

**Pfad:** `OpenSim/Region/OptionalModules/`

| Modul                | Aufgabe                             |
| -------------------- | ----------------------------------- |
| `DataSnapshot/`      | Suchindex-Export (XMLRPC)           |
| `Materials/`         | PBR-Materialsystem                  |
| `PhysicsParameters/` | Laufzeit-Physikparameter per Skript |
| `PrimLimitsModule/`  | Prim-Limits pro Parzelle            |
| `UserStatistics/`    | Viewer-Statistiken                  |
| `ViewerSupport/`     | Viewer-Capabilities-Erweiterungen   |

### 3.6 ScriptEngine – Skriptlaufzeit

**Pfad:** `OpenSim/Region/ScriptEngine/`

#### 3.6.1 YEngine (XMR-Engine)

**Pfad:** `ScriptEngine/YEngine/`

Kompiliert LSL/OSSL zu .NET-IL und führt es in Micro-Threads aus.  
Zentrale Dateien:

| Datei                 | Aufgabe                                         |
| --------------------- | ----------------------------------------------- |
| `XMREngine.cs`        | Einstiegspunkt, Modul-Interface                 |
| `MMRScriptCompile.cs` | Kompilier-Pipeline                              |
| `MMRScriptCodeGen.cs` | IL-Codegenerator                                |
| `XMRInstMain.cs`      | Skript-Instanz Lebenszyklus                     |
| `XMRInstRun.cs`       | Ausführung, Zustandsspeicherung (Checkpointing) |
| `XMRScriptThread.cs`  | Thread-Verwaltung                               |

#### 3.6.2 Shared API (LSL / OSSL / MOD / AA)

**Pfad:** `ScriptEngine/Shared/Api/`

Enthält alle Skript-Funktionen als C#-Methoden:

| API      | Beschreibung                                  |
| -------- | --------------------------------------------- |
| LSL_Api  | Standard-Linden-Scripting-Language-Funktionen |
| OSSL_Api | OpenSimulator-Erweiterungen (`osXxx`)         |
| MOD_Api  | Modul-Kommunikation (`modXxx`)                |
| AA_Api   | Aurora/Advanced-Avatar-Erweiterungen          |

### 3.7 PhysicsModules – Physik-Engines

**Pfad:** `OpenSim/Region/PhysicsModules/`

| Modul                        | Beschreibung                             |
| ---------------------------- | ---------------------------------------- |
| `BulletS/`                   | Bullet-Physik (empfohlen, Standard)      |
| `ubOde/`                     | ODE-Physik (ubOde-Fork)                  |
| `Meshing/`                   | Mesh-Zerlegung für Kollisionsobjekte     |
| `ubOdeMeshing/`              | Meshing speziell für ubOde               |
| `ConvexDecompositionDotNet/` | Konvexe Zerlegung von Meshes             |
| `SharedBase/`                | Gemeinsame Basisklassen für alle Engines |

---

## 4. Services – Backend-Dienste

**Pfad:** `OpenSim/Services/`

Die Services laufen im **Robust**-Prozess (Grid-Modus) oder direkt im Simulator (Standalone-Modus).

### 4.1 AssetService

Speicherung und Abruf von Assets (Texturen, Sounds, Skripte, Meshes …).  
Unterstützt lokale DB und Remote-Proxy.

### 4.2 InventoryService

Verwaltung von Inventarordnern und -items pro Benutzer.

### 4.3 GridService

Registrierung von Regionen, Abfrage von Regionsinformationen und Karten-Tiles.

### 4.4 UserAccountService

Benutzerverwaltung (Name, UUID, Passwort-Hash, Level).

### 4.5 PresenceService

Verfolgt, in welcher Region ein Benutzer aktuell eingeloggt ist.

### 4.6 AuthenticationService

Token-basierte Authentifizierung beim Login und bei Service-Aufrufen.

### 4.7 AvatarService

Persistenz des Avataraussehens (Wearables, Attachments).

### 4.8 FriendsService

Freundeslisten und Rechte zwischen Benutzern.

### 4.9 LLLoginService

Implementiert den Second-Life-Login-Prozess (XMLRPC `login_to_simulator`).

### 4.10 HypergridService

**Pfad:** `Services/HypergridService/`

Ermöglicht Teleports zwischen unabhängigen Grid-Installationen:

| Datei                   | Aufgabe                            |
| ----------------------- | ---------------------------------- |
| `GatekeeperService.cs`  | Eingangs-Gatekeeper eines Grids    |
| `UserAgentService.cs`   | Verwaltung von Hypergrid-Besuchern |
| `HGInventoryService.cs` | Inventarzugriff über Grid-Grenzen  |
| `HGAssetService.cs`     | Asset-Zugriff über Grid-Grenzen    |
| `HGFriendsService.cs`   | Freunde über Grid-Grenzen          |

### 4.11 Weitere Services

| Service                | Aufgabe                                  |
| ---------------------- | ---------------------------------------- |
| `FSAssetService/`      | Filesystem-basierter Asset-Store         |
| `MapImageService/`     | Karten-Tile-Generierung und -Speicherung |
| `EstateService/`       | Estate-Datenpersistenz                   |
| `MuteListService/`     | Mute-Listen pro Benutzer                 |
| `UserProfilesService/` | Erweitertes Benutzerprofil               |
| `FreeswitchService/`   | VoIP-Integration (Freeswitch)            |

---

## 5. Data – Datenbankschicht

**Pfad:** `OpenSim/Data/`

### 5.1 Interfaces / abstrakte Basisklassen

Alle Datenbankzugriffe werden über Interfaces abstrahiert:

| Interface                            | Zweck                            |
| ------------------------------------ | -------------------------------- |
| `IAssetData`                         | Asset-Persistenz                 |
| `IInventoryData` / `IXInventoryData` | Inventar-Persistenz              |
| `IRegionData`                        | Regionsregistrierung             |
| `IUserAccountData`                   | Benutzerdaten                    |
| `IEstateDataStore`                   | Estate-Daten                     |
| `IAuthenticationData`                | Authentifizierungstokens         |
| `IPresenceData`                      | Präsenz-Tracking                 |
| `Migration.cs`                       | Schema-Migration (Versionierung) |

### 5.2 MySQL

**Pfad:** `Data/MySQL/`  
Produktivsystem für mittlere bis große Grids.

### 5.3 SQLite

**Pfad:** `Data/SQLite/`  
Standard für Standalone-Betrieb, keine separate Datenbankinstallation notwendig.

### 5.4 PostgreSQL (PGSQL)

**Pfad:** `Data/PGSQL/`  
Alternative für große Installationen.

---

## 6. Capabilities – LLSD / CAPS-Schicht

**Pfad:** `OpenSim/Capabilities/`

Implementiert das **Capabilities-System** des Linden-Protokolls.  
Viewer fordern Capabilities-URLs an; der Simulator stellt temporäre HTTPS-Endpunkte bereit.

| Datei             | Aufgabe                                           |
| ----------------- | ------------------------------------------------- |
| `Caps.cs`         | Capabilities-Verwaltung pro Agent                 |
| `CapsHandlers.cs` | Registrierung und Routing von Caps-URLs           |
| `LLSDHelpers.cs`  | LLSD-Serialisierung (XML/Binary)                  |
| `Handlers/`       | Konkrete Capability-Handler (z. B. Upload, Fetch) |

---

## 7. Addons – Erweiterungsmodule

**Pfad:** `OpenSim/Addons/`

### 7.1 Groups

Vollständige Gruppenimplementierung mit lokalem und Remote-Connector sowie Hypergrid-Unterstützung.

### 7.2 OfflineIM

Speichert Instant Messages für abwesende Benutzer und stellt sie beim nächsten Login zu.

### 7.3 os-webrtc-janus

WebRTC-Sprach-/Video-Integration über einen Janus-Gateway.

---

## 8. ApplicationPlugins – Applikations-Plugins

**Pfad:** `OpenSim/ApplicationPlugins/`

Werden beim Start von `OpenSim.dll` geladen.

| Plugin                     | Aufgabe                                                   |
| -------------------------- | --------------------------------------------------------- |
| `LoadRegions/`             | Liest `Regions/*.ini` und instanziiert Regionen           |
| `RegionModulesController/` | Lädt und initialisiert alle Region-Module per Mono.Addins |
| `RemoteController/`        | XMLRPC-Fernsteuerungsschnittstelle                        |

---

## 9. Server – Robust-Server (Grid-Modus)

**Pfad:** `OpenSim/Server/`

`Robust.dll` ist der zentrale Dienste-Server im Grid-Betrieb.  
Er hostet alle Services (Asset, Grid, Login …) als HTTP-Endpunkte.

| Datei / Ordner  | Aufgabe                                 |
| --------------- | --------------------------------------- |
| `ServerMain.cs` | Einstiegspunkt                          |
| `Base/`         | Service-Loader, Plugin-System           |
| `Handlers/`     | HTTP-Handler für die einzelnen Services |

---

## 10. ConsoleClient

**Pfad:** `OpenSim/ConsoleClient/`

Fernkonsole: verbindet sich per HTTP mit einem laufenden `OpenSim.dll`- oder `Robust.exe`-Prozess und ermöglicht Konsolenbefehle über das Netzwerk.

---

## 11. Tools – Hilfsprogramme

**Pfad:** `OpenSim/Tools/`

### 11.1 pCampBot

Lasttest-Bot: simuliert mehrere Avatare gleichzeitig, nützlich für Performance-Tests.

### 11.2 Configger

Liest und validiert INI-Konfigurationsdateien für Debugging und Dokumentation.

### 11.3 LaunchSLClient

Hilfsprogramm zum Starten eines kompatiblen Viewers direkt aus dem Build-Verzeichnis.

---

## 12. Tests

**Pfad:** `OpenSim/Tests/` sowie `Tests/`-Unterordner in den jeweiligen Modulen

Unit- und Integrationstests mit NUnit. Relevante Testklassen befinden sich nahe am zu testenden Code.

---

## 13. Konfiguration & Deployment

| Datei                                | Zweck                                           |
| ------------------------------------ | ----------------------------------------------- |
| `bin/OpenSim.ini.example`            | Hauptkonfiguration Simulator                    |
| `bin/Robust.ini.example`             | Hauptkonfiguration Robust-Server                |
| `bin/OpenSimDefaults.ini`            | Standardwerte (nicht editieren)                 |
| `bin/config-include/`                | Modulare Teilkonfigurationen                    |
| `bin/Regions/`                       | Regionsdefinitionen (je eine `.ini` pro Region) |
| `runprebuild.bat` / `runprebuild.sh` | Generiert Solution-/Projektdateien via Prebuild |

---

## 14. Glossar

| Begriff       | Bedeutung                                             |
| ------------- | ----------------------------------------------------- |
| **Prim**      | Primitives Objekt (Grundbaustein der virtuellen Welt) |
| **Linkset**   | Gruppe verbundener Prims (`SceneObjectGroup`)         |
| **SOP**       | `SceneObjectPart` – ein einzelnes Prim im Linkset     |
| **SP**        | `ScenePresence` – ein eingeloggter Avatar             |
| **OAR**       | OpenSim Archive – Backup einer Region                 |
| **IAR**       | Inventory Archive – Backup eines Inventars            |
| **LLUDP**     | Linden Lab UDP-Protokoll                              |
| **CAPS**      | Capabilities – dynamische HTTP-Endpunkte              |
| **LLSD**      | Linden Lab Structured Data – Serialisierungsformat    |
| **Robust**    | Grid-Services-Server-Prozess                          |
| **Hypergrid** | Protokoll für Grid-übergreifende Teleports            |
| **YEngine**   | Aktuelle Skript-Engine (früher XMREngine)             |
| **LSL**       | Linden Scripting Language                             |
| **OSSL**      | OpenSim Scripting Language (Erweiterungen)            |
| **Estate**    | Verwaltungsebene über Regionen (Eigentümer, Regeln)   |
