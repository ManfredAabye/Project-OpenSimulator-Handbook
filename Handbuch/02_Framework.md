# Kapitel 2: Framework – Kernbibliothek

**Pfad:** `OpenSim/Framework/`  
**Assembly:** `OpenSim.Framework.dll`

---

## Übersicht

Das `Framework`-Projekt ist die gemeinsame Klassenbibliothek für alle anderen Teilprojekte (Region, Services, Data …).  
Es enthält **keine** Laufzeitlogik, sondern ausschließlich Interfaces, Datenmodelle und Hilfsfunktionen, auf die alle anderen Assemblies verweisen.

Abhängigkeiten des Frameworks:
- **OpenMetaverse.dll** – Datentypen (`UUID`, `Vector3`, `Quaternion`, Pakete)
- **Nini** – INI-Konfigurationsparser
- **log4net** – Logging

---

## 2.1 Zentrale Interfaces

### `IScene`
`Framework/IScene.cs`

Abstrahiert eine simulierte Region. Die konkrete Implementierung ist `Scene` in `Region/Framework/Scenes/`.

Wichtige Members:

| Member | Typ | Bedeutung |
| --- | --- | --- |
| `Name` | `string` | Regionsname |
| `ID` | `UUID` | Eindeutige Regions-UUID |
| `RegionInfo` | `RegionInfo` | Metadaten (Koordinaten, Port …) |
| `RegionStatus` | `enum` | `Down / Up / Crashed / Starting` |
| `LoginsEnabled` | `bool` | Logins aktiv? |
| `TimeDilation` | `float` | Simulation speed (1.0 = Echtzeit) |
| `AddNewAgent()` | Methode | Fügt Avatar (Root oder Child) hinzu |
| `CloseAgent()` | Methode | Trennt Verbindung eines Avatars |

`RegionStatus`-Enum:
```csharp
public enum RegionStatus : int
{
    Down    = 0,
    Up      = 1,
    Crashed = 2,
    Starting = 3,
}
```

---

### `IClientAPI`
`Framework/IClientAPI.cs`

Abstraktion für eine Viewer-Verbindung. Die Implementierung für das SL-Protokoll ist `LLClientView` in `Region/ClientStack/Linden/`.

Das Interface definiert mehrere hundert Delegates und Methoden – die wichtigsten Kategorien:

| Kategorie | Beispiel-Delegate / -Methode |
| --- | --- |
| Avatar-Bewegung | `OnAgentUpdate`, `OnSetAlwaysRun` |
| Chat / IM | `OnChatFromClient`, `OnInstantMessage` |
| Inventar | `OnFetchInventoryDescendents`, `OnMoveItemsAndLeaveCopy` |
| Objekte | `OnRezObject`, `OnObjectAttach`, `OnGrabUpdate` |
| Gelände | `OnModifyTerrain` |
| Appearance | `OnSetAppearance`, `OnAvatarNowWearing` |
| Teleport | `OnTeleportLocationRequest` |
| Senden | `SendAvatarDataImmediate()`, `SendPrimUpdate()`, `SendInventoryItem()` |

---

### `ISceneEntity` / `ISceneObject` / `ISceneAgent`
`Framework/ISceneEntity.cs` usw.

Hierarchie:
```
ISceneEntity
├── ISceneObject   → SceneObjectGroup (Linkset)
└── ISceneAgent    → ScenePresence    (Avatar)
```

---

## 2.2 Datenmodelle

### Assets

| Klasse | Datei | Beschreibung |
| --- | --- | --- |
| `AssetBase` | `AssetBase.cs` | Binärdaten + Metadaten (UUID, Name, Type, Data[]) |
| `AssetLandmark` | `AssetLandmark.cs` | Geparster Landmark-Asset |
| `AssetPermissions` | `AssetPermissions.cs` | Berechtigungs-Flags auf Asset-Ebene |

Asset-Typen (aus `OpenMetaverse.AssetType`):

| Wert | Typ |
| --- | --- |
| 0 | Texture (JPEG2000) |
| 1 | Sound |
| 6 | Object |
| 7 | Notecard |
| 10 | LSL Script |
| 17 | Bodypart |
| 18 | Sound WAV |
| 20 | Animation |
| 21 | Gesture |
| 49 | Mesh |

---

### Inventar

| Klasse | Datei | Beschreibung |
| --- | --- | --- |
| `InventoryItemBase` | `InventoryItemBase.cs` | Ein Inventar-Eintrag (UUID, Name, AssetID, Permissions) |
| `InventoryFolderBase` | `InventoryFolderBase.cs` | Ein Ordner (UUID, Name, ParentID, Type) |
| `InventoryFolderImpl` | `InventoryFolderImpl.cs` | Ordner mit Kinderliste (In-Memory-Baum) |
| `InventoryCollection` | `InventoryCollection.cs` | Menge von Ordnern + Items (für Bulk-Operationen) |
| `TaskInventoryItem` | `TaskInventoryItem.cs` | Inventory eines Prims (in-world Objekt) |

---

### Avatar & Appearance

| Klasse | Datei | Beschreibung |
| --- | --- | --- |
| `AvatarAppearance` | `AvatarAppearance.cs` | Komplettzustand des Avataraussehens |
| `AvatarWearable` | `AvatarWearable.cs` | Ein Kleidungsstück (ItemID + AssetID) |
| `AvatarAttachment` | `AvatarAttachment.cs` | Befestigtes Objekt (AttachPoint + ItemID) |
| `Animation` | `Animation.cs` | Eine laufende Animation (AnimID + SequenceNum) |
| `AnimationSet` | `AnimationSet.cs` | Alle aktiven Animationen eines Avatars |

---

### Regionen & Estates

| Klasse | Datei | Beschreibung |
| --- | --- | --- |
| `RegionInfo` | `RegionInfo.cs` | Name, UUID, Koordinaten, Port, INI-Werte |
| `RegionSettings` | `RegionSettings.cs` | Physik, Sun, Maturity, Teleport-Regeln |
| `EstateSettings` | `EstateSettings.cs` | Eigentümer, Altersverifikation, Estate-Bans |
| `EstateBan` | `EstateBan.cs` | Ein einzelner Ban-Eintrag |
| `LandData` | `LandData.cs` | Eine Parzelle (UUID, Flags, Preis, Owner …) |

---

### Prims & Geometrie

| Klasse | Datei | Beschreibung |
| --- | --- | --- |
| `PrimitiveBaseShape` | `PrimitiveBaseShape.cs` | Geometrie-Parameter (PathBegin, TwistBegin, SculptTexture …) |
| `SOPMaterial` | `Scenes/SOPMaterial.cs` | Material-Einstellungen eines Prim-Faces |
| `ExtraPhysicsData` | `ExtraPhysicsData.cs` | Physik-Override-Werte eines Prims |
| `PhysicsInertia` | `PhysicsInertia.cs` | Massenträgheit |
| `ColliderData` | `ColliderData.cs` | Kollisionsparameter |

---

### Gelände

| Klasse | Datei | Beschreibung |
| --- | --- | --- |
| `TerrainData` | `TerrainData.cs` | Abstrakte Geländedaten-Basis |
| `TerrainChannel` | `Scenes/TerrainChannel.cs` | Konkreter Daten-Buffer (256×256 double[]) |

---

## 2.3 Berechtigungssystem

`Framework/Util.cs` – `PermissionMask`-Enum:

```csharp
[Flags]
public enum PermissionMask : uint
{
    None        = 0,
    Transfer    = 1 << 13,   // Weitergabe erlaubt
    Modify      = 1 << 14,   // Verändern erlaubt
    Copy        = 1 << 15,   // Kopieren erlaubt
    Export      = 1 << 16,   // Export aus der virtuellen Welt erlaubt
    Move        = 1 << 19,   // Verschieben erlaubt
    Damage      = 1 << 20,   // Beschädigbar
    All         = 0x7FFFFFFF,
}
```

Jedes `InventoryItemBase` besitzt vier Permissions-Felder:
- `BasePermissions` – Maximum
- `CurrentPermissions` – Aktuelle
- `NextPermissions` – Beim nächsten Eigentümer
- `EveryOnePermissions` – Für alle
- `GroupPermissions` – Für Gruppe

---

## 2.4 Netzwerk & HTTP

### `WebUtil`
`Framework/WebUtil.cs`

Zentrales HTTP-Hilfsmodul für alle Service-Aufrufe. Kapselt `HttpClient` mit:
- JSON- und LLSD-Serialisierung
- Retry-Logik
- Timeouts
- Request-Logging

Häufig genutzte Methoden:

```csharp
// HTTP-POST mit JSON-Body
OSDMap result = WebUtil.PostToService(url, data, timeout);

// HTTP-GET
string result = WebUtil.MakeRequest("GET", url, timeout);
```

### `OutboundUrlFilter`
`Framework/OutboundUrlFilter.cs`

Whitelist/Blacklist für ausgehende HTTP-Anfragen aus Skripten (`llHTTPRequest`).  
Konfiguration in `OpenSim.ini` unter `[Network]`:
```ini
OutboundDisallowForUserScripts = "127.0.0.1, ::1, 10.0.0.0/8"
```

---

## 2.5 Konsolensystem

`Framework/Console/`

| Klasse | Beschreibung |
| --- | --- |
| `LocalConsole` | Interaktive Konsole mit Tab-Completion |
| `RemoteConsole` | Konsole über HTTP (für ConsoleClient) |
| `CommandConsole` | Basis-Konsole mit Command-Dispatcher |
| `MockConsole` | Test-Stub |

Konsolen-Befehle werden mit `MainConsole.Instance.Commands.AddCommand(...)` registriert.

---

## 2.6 Hilfsfunktionen (`Util.cs`)

`Framework/Util.cs` enthält ca. 2500 Zeilen Hilfsfunktionen:

| Methode | Beschreibung |
| --- | --- |
| `Util.GetSHA1Hash(data)` | SHA-1-Hash als Hex-String |
| `Util.GetMD5Hash(data)` | MD5-Hash |
| `Util.UUID2Dotty(uuid)` | UUID → "123.456"-Format |
| `Util.RegionHandleToWorldLoc(handle)` | Grid-Handle → Weltkoordinaten |
| `Util.GetDistanceTo(a, b)` | Euklidischer Abstand zweier `Vector3` |
| `Util.Clip(val, min, max)` | Wert begrenzen |
| `Util.TrySendMsg(...)` | Nicht blockierendes Nachrichtensenden |

---

## 2.7 Sonstiges

| Datei | Beschreibung |
| --- | --- |
| `Constants.cs` | Globale Konstanten (Teleport-Flags, Parcel-Flags …) |
| `SLUtil.cs` | SL-spezifische Konvertierungen (Asset-Type-Strings …) |
| `Culture.cs` | Stellt `en-US` als Laufzeitkultur sicher (Dezimalpunkte) |
| `GridInstantMessage.cs` | Grid-weite IM-Datenstruktur |
| `GridInfo.cs` | Metadaten des Grids (Login-URI, Name …) |
| `VersionInfo.cs` | Aktuelle Version (`0.9.3.1 Nessie Dev`) |

---

*Weiter: [Kapitel 3 – Region / Szenenmodell](03_Region_Scene.md)*
