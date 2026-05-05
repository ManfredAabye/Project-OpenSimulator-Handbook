# Kapitel 5: Services – Backend-Dienste

**Pfad:** `OpenSim/Services/`

---

## Übersicht

Die Services bilden die Backend-Infrastruktur von OpenSimulator. Im **Grid-Modus** laufen sie im `Robust.exe`-Prozess und werden über HTTP/REST angeboten. Im **Standalone-Modus** werden sie direkt in `OpenSim.exe` instanziiert.

### Betriebsmodi

| Modus | Services-Prozess | Konfiguration |
| --- | --- | --- |
| Standalone | In `OpenSim.exe` integriert | `OpenSim.ini` → `[Startup]` |
| Grid – Robust | Separater `Robust.exe` | `Robust.ini` |
| Grid – Region | Connector-Klassen im Simulator | `OpenSim.ini` → `[GridService]` etc. |

### Service-Plugin-System

Services werden per Dependency-Injection über `ServerUtils.LoadPlugin<T>()` geladen:

```csharp
m_GridService = ServerUtils.LoadPlugin<IGridService>(
    "OpenSim.Services.GridService.dll:GridService",
    new object[] { config });
```

---

## 5.1 Interfaces

**Pfad:** `Services/Interfaces/`

Alle Services implementieren Interfaces aus diesem Verzeichnis. Der Simulator verwendet ausschließlich die Interfaces – nie die konkreten Klassen direkt.

| Interface | Implementierungen |
| --- | --- |
| `IAssetService` | `AssetService`, `XAssetService`, `FSAssetService`, `HGAssetService` |
| `IInventoryService` | `XInventoryService`, `HGInventoryService`, `HGSuitcaseInventoryService` |
| `IGridService` | `GridService` |
| `IUserAccountService` | `UserAccountService` |
| `IPresenceService` | `PresenceService` |
| `IAuthenticationService` | `AuthenticationService` |
| `IAvatarService` | `AvatarService` |
| `IFriendsService` | `FriendsService` |
| `IGridUserService` | `GridUserService` |
| `ILoginService` | `LLLoginService` |
| `IGatekeeperService` | `GatekeeperService` |
| `IUserAgentService` | `UserAgentService` |

---

## 5.2 AssetService

**Pfad:** `Services/AssetService/`

### Aufgabe AssetService

Persistente Speicherung und Abfrage von **Assets** (Binärdaten):

- Texturen (JPEG2000)
- Sounds (OGG/WAV)
- Skripte (LSL-Quelltext)
- Meshes (COLLADA)
- Animationen
- Inventar-Ordner-Thumbnails

### Varianten

| Klasse | Beschreibung |
| --- | --- |
| `AssetService` | Standard – speichert in DB-Tabelle `assets` |
| `XAssetService` | Erweiterter Asset-Service mit Hash-deduplizierung |
| `FSAssetService` | Filesystem-basiert – Dateien unter `AssetSets/` |
| `HGAssetService` | Proxy für Hypergrid-Asset-Zugriff |
| `HGRemoteAssetService` | Holt Assets von fremden Grids |

### Interface `IAssetService`

```csharp
AssetBase Get(string id);
AssetBase Get(string id, string ForeignAssetService, bool dummy);
AssetMetadata GetMetadata(string id);
byte[] GetData(string id);
bool Get(string id, object sender, AssetRetrieved handler);  // async
string Store(AssetBase asset);
bool UpdateContent(string id, byte[] data);
bool Delete(string id);
```

### Asset-Typen (wichtige Werte)

| Typ-ID | Name |
| --- | --- |
| 0 | Texture |
| 1 | Sound |
| 6 | Object |
| 7 | Notecard |
| 10 | LSL Script |
| 17 | Body Part |
| 20 | Animation |
| 21 | Gesture |
| 43 | Link |
| 49 | Mesh |

---

## 5.3 InventoryService

**Pfad:** `Services/InventoryService/`

### Aufgabe InventoryService

Verwaltung von Inventar-Ordnern und -Items für jeden Benutzer.

### Varianten InventoryService

| Klasse | Beschreibung |
| --- | --- |
| `XInventoryService` | Standard – Tabellen `inventoryfolders` + `inventoryitems` |
| `HGInventoryService` | Hypergrid-Proxy – leitet Anfragen weiter |
| `HGSuitcaseInventoryService` | "Koffer"-Konzept für Hypergrid-Besucher (eingeschränkter Zugriff) |

### Interface `IInventoryService`

```csharp
bool CreateUserInventory(UUID user);
List<InventoryFolderBase> GetInventorySkeleton(UUID userId);
InventoryFolderBase GetRootFolder(UUID userID);
InventoryFolderBase GetFolderForType(UUID userID, FolderType type);
InventoryCollection GetFolderContent(UUID userID, UUID folderID);
List<InventoryItemBase> GetFolderItems(UUID userID, UUID folderID);
bool AddFolder(InventoryFolderBase folder);
bool UpdateFolder(InventoryFolderBase folder);
bool DeleteFolders(UUID userID, List<UUID> folderIDs);
bool AddItem(InventoryItemBase item);
bool UpdateItem(InventoryItemBase item);
bool DeleteItems(UUID userID, List<UUID> itemIDs);
InventoryItemBase GetItem(UUID userID, UUID itemID);
```

---

## 5.4 GridService

**Pfad:** `Services/GridService/`

### Aufgabe

Registrierung und Abfrage von **Regionen** im Grid.

### `GridService`-Klasse

Wichtige Member:

| Feld | Beschreibung |
| --- | --- |
| `m_DeleteOnUnregister` | Regionen aus DB löschen beim sauberen Herunterfahren |
| `m_AllowDuplicateNames` | Gleiche Regionsnamen erlauben |
| `m_HypergridLinker` | Bindet fremde Grid-Regionen ein |

### Interface `IGridService`

```csharp
string RegisterRegion(UUID scopeID, GridRegion regionInfo);
bool DeregisterRegion(UUID regionID);
List<GridRegion> GetNeighbours(UUID scopeID, UUID regionID);
GridRegion GetRegionByUUID(UUID scopeID, UUID regionID);
GridRegion GetRegionByName(UUID scopeID, string regionName);
List<GridRegion> GetRegionsByName(UUID scopeID, string name, int maxNumber);
List<GridRegion> GetRegionRange(UUID scopeID, int xmin, int xmax, int ymin, int ymax);
List<GridRegion> GetDefaultRegions(UUID scopeID);
GridRegion GetRegionByPosition(UUID scopeID, int x, int y);
```

### `GridRegion`-Datenstruktur

| Property | Beschreibung |
| --- | --- |
| `RegionID` | UUID der Region |
| `RegionName` | Anzeigename |
| `RegionLocX` | X-Koordinate (256er-Raster) |
| `RegionLocY` | Y-Koordinate (256er-Raster) |
| `ExternalEndPoint` | `IPEndPoint` des Simulators |
| `ServerURI` | HTTP-URL des Simulators |
| `Access` | Altersfreigabe (PG / Mature / Adult) |
| `Flags` | `RegionFlags` (Default, Fallback, …) |

---

## 5.5 UserAccountService

**Pfad:** `Services/UserAccountService/`

### Aufgabe UserAccountService

Persistenz von Benutzerdaten (Konten).

### Interface `IUserAccountService`

```csharp
UserAccount GetUserAccount(UUID scopeID, UUID userID);
UserAccount GetUserAccount(UUID scopeID, string firstName, string lastName);
UserAccount GetUserAccount(UUID scopeID, string email);
List<UserAccount> GetUserAccounts(UUID scopeID, string query);
bool StoreUserAccount(UserAccount data);
```

### `UserAccount`-Datenstruktur

| Property | Beschreibung |
| --- | --- |
| `PrincipalID` | UUID des Benutzers |
| `FirstName` | Vorname |
| `LastName` | Nachname |
| `Email` | E-Mail-Adresse |
| `UserLevel` | 0 = Normal, -1 = Gesperrt, 200+ = Gott |
| `UserFlags` | Bitmaske (HGAuthorized etc.) |
| `ServiceURLs` | Dictionary mit Heimat-Service-URLs |
| `Created` | Unix-Timestamp der Erstellung |

---

## 5.6 PresenceService

**Pfad:** `Services/PresenceService/`

### Aufgabe PresenceService

Verfolgt, **wo** ein Benutzer aktuell eingeloggt ist (in welcher Region).

### Interface `IPresenceService`

```csharp
bool LoginAgent(string userID, UUID sessionID, UUID secureSessionID);
bool LogoutAgent(UUID sessionID);
bool LogoutRegionAgents(UUID regionID);
bool ReportAgent(UUID sessionID, UUID regionID);
PresenceInfo GetAgent(UUID sessionID);
PresenceInfo[] GetAgents(string[] userIDs);
```

---

## 5.7 AuthenticationService

**Pfad:** `Services/AuthenticationService/`

### Aufgabe AuthenticationService

Token-basierte Authentifizierung beim Login und bei internen Service-Aufrufen.

### Methoden

```csharp
string Authenticate(UUID principalID, string password, int lifetime);
bool Verify(UUID principalID, string token, int lifetime);
bool Release(UUID principalID, string token);
bool SetPassword(UUID principalID, string passwd);
AuthInfo GetAuthInfo(UUID principalID);
bool SaveAuthInfo(AuthInfo info);
```

Passwörter werden als `MD5(":" + password)` + Salt gespeichert.

---

## 5.8 LLLoginService

**Pfad:** `Services/LLLoginService/`

### Aufgabe LLLoginService

Implementiert den **Second-Life Login-Prozess** (`login_to_simulator` XMLRPC-Methode).

### Login-Ablauf

```text
Viewer → POST /  (XMLRPC login_to_simulator)
  └─ LLLoginService.Login()
       1. UserAccountService.GetUserAccount(name)
       2. AuthenticationService.Authenticate(uuid, password)
       3. PresenceService.LoginAgent(uuid, session)
       4. InventoryService.GetRootFolder(uuid)
       5. AvatarService.GetAppearance(uuid)
       6. GridService.GetDefaultRegions()
       7. SimulationService.CreateAgent() → Simulator antwortet
       8. Antwort an Viewer mit:
          - session_id, secure_session_id
          - agent_id
          - sim_ip, sim_port, region_x, region_y
          - seed_capability (CAPS-URL)
          - inventory-skeleton
          - buddy-list (Freunde)
```

### `LLLoginService`-Abhängigkeiten

Die Klasse hält Referenzen auf **alle** anderen Services:

```csharp
protected IUserAccountService m_UserAccountService;
protected IAuthenticationService m_AuthenticationService;
protected IInventoryService m_InventoryService;
protected IGridService m_GridService;
protected IPresenceService m_PresenceService;
protected IAvatarService m_AvatarService;
protected IFriendsService m_FriendsService;
protected IGatekeeperService m_GatekeeperConnector;
// ...
```

---

## 5.9 HypergridService

**Pfad:** `Services/HypergridService/`

### Aufgabe HypergridService

Ermöglicht **grid-übergreifende Teleports** – der sogenannte Hypergrid.

### Kernklassen

#### `GatekeeperService`

Der Eingangs-Gatekeeper eines Grids. Prüft eingehende Hypergrid-Besucher:

- Ist der Viewer erlaubt (Regex `AllowedClients`)?
- Ist die Herkunfts-URL auf der Blacklist?
- Sind Fremde überhaupt erlaubt (`ForeignAgentsAllowed`)?
- Doppelte Präsenzen erlaubt (`AllowDuplicatePresences`)?

Konfiguration in `Robust.ini`:

```ini
[GatekeeperService]
ExternalName = http://mein-grid.de:8002
AllowTeleportsToAnyRegion = true
ForeignAgentsAllowed = true
AllowedClients = ""
DeniedClients = ""
```

#### `UserAgentService`

Verwaltet Hypergrid-Besucher auf dem Heimat-Grid:

- Hält Session-Tokens für abwesende Benutzer
- Beantwortet Anfragen fremder Grids nach Benutzerprofilen
- Verwaltet `ServiceURLs` der Benutzer

#### `HGInventoryService`

Leitet Inventar-Anfragen für Hypergrid-Besucher an deren Heimat-Grid weiter.  
Implementiert das **Suitcase**-Konzept: Besucher sehen nur ihren „Koffer"-Ordner.

#### `HGAssetService`

Transparenter Asset-Proxy:

- Assets, die lokal nicht vorhanden sind, werden vom Heimat-Grid abgerufen
- Abgerufene Assets werden lokal gecacht

---

## 5.10 Weitere Services

| Service | Pfad | Aufgabe |
| --- | --- | --- |
| `FSAssetService` | `Services/FSAssetService/` | Speichert Assets als Dateien statt in DB |
| `MapImageService` | `Services/MapImageService/` | Karten-Tile-Speicherung und -Abfrage |
| `EstateService` | `Services/EstateService/` | Estate-Datenpersistenz (Eigentümer, Bans) |
| `MuteListService` | `Services/MuteListService/` | Mute-Listen pro Benutzer |
| `UserProfilesService` | `Services/UserProfilesService/` | Erweiterte Benutzerprofile |
| `FreeswitchService` | `Services/FreeswitchService/` | VoIP-Tokens für Freeswitch-Integration |
| `AvatarService` | `Services/AvatarService/` | Appearance-Persistenz |
| `FriendsService` | `Services/Friends/` | Freundeslisten und Rechte |

---

## 5.11 Connectors

**Pfad:** `Services/Connectors/`

Connector-Klassen werden im Simulator (nicht in Robust) verwendet, um Remote-Services über HTTP anzusprechen.

| Connector-Kategorie | Beschreibung |
| --- | --- |
| `Asset/` | HTTP-Aufrufe an Remote-AssetService |
| `Inventory/` | HTTP-Aufrufe an Remote-InventoryService |
| `Grid/` | HTTP-Aufrufe an Remote-GridService |
| `Presence/` | HTTP-Aufrufe an Remote-PresenceService |
| `Authentication/` | HTTP-Aufrufe an Remote-AuthService |
| `Hypergrid/` | Spezial-Connectors für Hypergrid-Protokolle |
| `InstantMessage/` | IM-Weiterleitung über Grids |

Jeder Connector serialisiert die Anfrage als JSON oder LLSD und deserializiert die Antwort.

---

*Weiter: [Kapitel 6 – PhysicsModules](06_PhysicsModules.md)*
