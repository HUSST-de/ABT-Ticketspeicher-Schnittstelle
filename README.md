## ABT/IDBT Ticketspeicher – Schnittstellen-Spezifikation

**Status:** Official Release

**Version:** 1.0.0

**Erstellt von:** **Arbeitsgruppe ABT / IDBT (by HUSST)** – weitere Informationen unter [husst.de](https://husst.de).

**Veröffentlichungsdatum:** **30.01.2026**

Dieses Repository enthält einen **Entwurf einer Spezifikation** für einen **ABT/IDBT-Ticketspeicher** inkl. **Use Cases**, **HTTP-API (OpenAPI)** sowie einem **Konzept für Authentifizierung & Autorisierung**.
Für **Fragen**, **Anmerkungen** und **Feedback** kann gerne die Issue-Funktion in diesem Github-Repository genutzt werden. Alternativ stehen wir für Nachfragen per E-Mail unter **ticketspeicher@husst.de** zur Verfügung.

- **Use Cases:** [Use Cases](UseCases.md)
- **AuthN/AuthZ-Konzept:** [Authentifizierung und Autorisierung](authorization_and_authentication.md)
- **OpenAPI-Spezifikation:** [Open API Spezifikation](openapi.yaml)
- **Statistiken & Reporting:** [Statistik und Reporting](statistics_and_reporting.md)
- **Weitere Technische Informationen zur Schnittstelle**: [Technische Details](technical_details.md)


---

## Einführung / Zusammenfassung

Ein **ABT-Ticketspeicher** (Account-Based Ticketing bzw. ID-Based Ticketing (IDBT)) verlagert die Fahrtberechtigung aus dem Medium (Karte/Barcode) in ein zentrales Backend („Cloud“). In heterogenen Systemlandschaften entstehen dabei Interoperabilitätsprobleme (z. B. **unterschiedliche Tokenisierung** und **fehlende gemeinsame Schnittstellen**).  
Der hier spezifizierte **Ticketspeicher** adressiert diese Lücke: Er ist eine **hochverfügbare, mandantenfähige** Speicherung von Tickets, die **übergreifend** von Systemen unterschiedlicher Hersteller genutzt werden kann – mit einem ersten Fokus auf eine **übergreifende Kontrolle von IDBT-Systemen**.

**Wichtig:** Der Ticketspeicher **berechnet keine Tariflogik** und hält **keine Zahlungs- oder Kundendaten**. Er speichert und liefert Ticketdaten (Container) plus minimal notwendige strukturierte Metadaten.

---

## Was ist der Ticketspeicher – und was soll er sein?

### Zielbild (hochverfügbare Datenbank + Domänenschicht)

Der Ticketspeicher ist im Kern eine **hochverfügbare Datenbank** mit einer schlanken fachlichen Schicht, die:

- **Tickets** mit einem **Transit-Token** verknüpft ablegt
- **mandanten-/rollenbasierten Zugriff** durchsetzt (z. B. nur „eigene Tickets“ schreiben/ändern)
- Tickets für **Kontrollzwecke** effizient auffindbar macht (Filterung über strukturierte Daten)

### Transit-Token (typisiert)

Tickets werden im Ticketspeicher **immer** zusammen mit einer Referenz auf einen **Transit-Token** gespeichert. Ein Transit-Token besitzt einen **Token-Typ** (`tokenType`), der das jeweilige Token-Schema beschreibt.

- **Aktueller Vorschlag (v1.0.0):** ausschließlich `iso24851`
- **Wichtig:** **ISO 24851** ist zum Zeitpunkt dieser Version **nur ein Entwurf (Draft)**. Aus urheber- und nutzungsrechtlichen Gründen werden in dieser Spezifikation **keine weiteren inhaltlichen Details** zum Token-Schema `iso24851` angeführt.
- **Ausblick:** In späteren Versionen können **weitere Transit-Token-Typen** ergänzt werden.

### Abgrenzung (bewusst nicht Teil des Systems)

- **Online-Anbindung erforderlich**; temporäre Netzausfälle sind durch **Caching** in Backend-Systemen abzufangen
- Die eigentliche **Kontrolle der Fahrtberechtigung** (räumlich/zeitlich/tariflich) erfolgt weiterhin lokal im Kontrollgerät bzw. Kontrollbackend und **nicht online** im Ticketspeicher.
- **Kein Payment** im Ticketspeicher: Es wird ausschließlich mit Transit-Tokens gearbeitet; Payment-Tokens und Zahlungsprozesse verbleiben vollständig in den jeweiligen Zahlungs- und Backend-Systemen.
- **Keine Tarif-/Preisbildung**
- **Keine Kunden-/Adress-/Kontodaten**
- **Keine Aktions-/Sperrlisten**: Sperr- und Freigabestände werden direkt am Ticket über den Status gepflegt.

### High-Level Use Cases

Die wichtigsten Use Cases (fachlich), Details in [UseCases.md](UseCases.md):

- **Ticket ablegen (Verkauf / Ausgabe einer Berechtigung)**: `POST /tickets`
- **Kontrolle** (Abruf relevanter Tickets für Prüf- und Kontrollzwecke): `GET /tickets`
- **Sperren/Entsperren**: `PATCH /tickets/{ticketRef}`
- **Ticket löschen**: `DELETE /tickets/{ticketRef}`
- **Ersatzmedium / Token-Wechsel**: `POST /tokens` (Neuzuordnung)

---

## OpenAPI / Artefakte

- **OpenAPI Spec:** [openapi.yaml](openapi.yaml)  
  - Enthält: Endpunkte, Modelle (Ticket/Entitlement/Token), Error-Model (RFC 7807), Security-Schemes (OAuth2 JWT + mTLS)
- **Use Cases:** [UseCases.md](UseCases.md)
- **AuthN/AuthZ:** [authorization_and_authentication.md](authorization_and_authentication.md)

---

## API-Überblick (konzeptionell)

### Transit-Token und Token-Typ

- Tickets werden über einen **Transit-Token** adressiert.
- Transit-Tokens sind **typisiert** (`tokenType` + `transitToken`).
- In **v1.0.0** ist als `tokenType` ausschließlich **`iso24851`** vorgesehen.

### Endpunkte (Kurzliste)

- **Tickets**
  - `POST /tickets` – Ticket anlegen
  - `GET /tickets` – Tickets zu einem Transit-Token abrufen (u. a. Kontrolle)
  - `PATCH /tickets` – Bulk-Statusänderungen (produktbezogen)
  - `DELETE /tickets` – Bulk-Delete (produktbezogen)
  - `PATCH /tickets/{entitlementRef}` – einzelnes Ticket aktualisieren
  - `DELETE /tickets/{entitlementRef}` – einzelnes Ticket löschen
- **Tokens**
  - `POST /tokens/{transitToken}` – Transit-Token ersetzen (Neuzuordnung auf neues Medium)

### Ticket-Modell (Kurzbeschreibung)

Ein Ticket besteht aus:

- **strukturierte Daten** (u. a. `status`, `effectiveTime`, `expirationTime`, `entitlementId`)
- **Container („Entitlement“)**: uninterpretiert im Ticketspeicher; erlaubt beliebige, abgestimmte Formate (z. B. VDV-KA/VDV-ETS, UIC, etiCore, …)

---

## Authentifizierung & Autorisierung (Konzept)

Details (normativ/lesbar): [authorization_and_authentication.md](authorization_and_authentication.md)

### Grundkonzept

**OAuth2 JWT + mTLS (zertifikatsgebunden)**, mit Wiederverwendung einer Branchen-PKI (z. B. VDV PKI).  
Wesentliche Idee: Zugriff erfolgt nur, wenn **JWT** und **mTLS-Zertifikat** zusammenpassen (Token-Bindung, z. B. via `cnf.x5t#S256`).

### Rollenbild (Beispiele aus dem Konzept)

_Referenz:_ Die Rollen-/Verantwortungsabgrenzung orientiert sich an **ISO 24014-1** (Interoperable fare management – Part 1: Architecture).

- **DL / SO (Dienstleister / Service Operator)**: Tickets lesen (Kontrolle)
- **KVP / CCP (Kundenvertragspartner / Customer Contract Partner)**: CRUD auf eigene Tickets
- **PV / PO (Produktverantwortlicher / Product Owner)**: Status eigener Tickets ändern (z. B. sperren/entsperren), Token aktualisieren

### Scopes (Beispiele)

Die Spezifikation nennt u. a.:

- `view:token`, `validate:token`, `view:ticket`, `replace:token`
- `create:ticket`, `update:ticket`, `delete:ticket`

### Architektur-Skizze (konzeptionell, inline)

```mermaid
flowchart LR
  Medium["Nutzermedium<br/>(Karte/Smartphone/Kreditkarte)"] -->|Token lesen| Validator[Validator / Kontrollgerät]
  Validator -->|Online-Request| KontrollBackend[Kontroll-/VU-Backend]
  KontrollBackend -->|OAuth2 JWT + mTLS| Tickethub[Ticketspeicher / Tickethub API]
  Tickethub -->|relevante Tickets| KontrollBackend
  KontrollBackend -->|Anzeige/Entscheidung| Validator
```

---

## Glossar (DE/EN) – Begriffe aus der (etiCORE-)Ticketing-Welt

| Deutsch | Englisch | Kurzbeschreibung |
|---|---|---|
| Ticketspeicher | Ticket store / Ticket hub | Hochverfügbare, mandantenfähige Speicherung von Tickets zu Transit-Tokens |
| Fahrtberechtigung | Entitlement | Berechtigung zur Nutzung einer Transportleistung (z. B. Ticket/Pass) |
| Ticket-ID | Ticket ID | Identifikator einer Berechtigung (z. B. `ccpOrgId` + `ticketRef`) |
| Ticketreferenz | Ticket reference | Referenz/Nummer einer Berechtigung innerhalb einer Organisation |
| Ticket-Container | Ticket container | Uninterpretiertes Payload-Feld `entitlement` für Ticketdaten (formatfrei/vereinbart) |
| Strukturierte Daten | Structured data | Metadaten für Filterung/Autorisierung (Status, Zeiten, IDs, …) |
| Transit-Token | Transit token | Typisierter Token zur Ticketzuordnung (`tokenType` + Tokenwert); aktuell nur `iso24851` |
| Payment-Token | Payment token | Zahlungsbezogener Token (z. B. Kreditkarte); **nicht** direkt im Ticketspeicher |
| Token-Typ | Token type | Schema/Regelwerk für Token-Matching (z. B. `iso24851`) |
| Mandant / OrgId | Tenant / OrgId | Organisationskennung (VDV Org-ID) zur Mandantentrennung |
| Produktverantwortlicher (PV) | Product Owner (PO) | Verantwortlich für Produkte/Statusänderungen (fachliche Hoheit) |
| Kundenvertragspartner (KVP) | Contract Customer Partner (CCP) | Verkehrsunternehmen/Vertragspartner, der Tickets ausgibt/verwaltet |
| Dienstleister (DL) | Service Operator (SO) | Partei für Kontrolle/Betrieb mit eingeschränkten Rechten |
| Kontrolle | Inspection / Validation | Abruf relevanter Tickets für Prüf-/Validierungsprozesse |
| Sperren | Block | Statusänderung zur Einschränkung der Nutzung (keine Sperrlisten) |
| Entsperren | Unblock | Rücknahme einer Sperre |
| Token ersetzen | Replace token | Übertrag von Tickets von altem auf neues Medium (Übergangszeit möglich) |
| Check-in / Check-out | Check-in / Check-out | Ereignisse zur Fahrt-Dokumentation (CiCo) |

---

## Wie wird der Ticketspeicher in lokale ABT/IDBT-Lösungen integriert?

### Grundprinzip

Der Ticketspeicher ist ein **übergreifender, zentraler** Speicher, der **lokale Systeme ergänzt** (nicht ersetzt). Lokale ABT/IDBT-Plattformen bleiben verantwortlich für:

- Tarif-/Produktlogik und Preisbildung
- Payment, Accounting, Kundenverwaltung
- Ausgabeprozesse (inkl. Medienverwaltung)
- Offline-Fähigkeit im Feld (über Cache/Backend-Strategien)

### Integrationsbausteine

- **Transit-Token-Erzeugung standardisieren**
  - Ein interoperabler Algorithmus (z. B. nach ISO 24851) muss herstellerübergreifend einheitlich sein
  - Bei Kreditkarten als Medium: **Mapping Payment-Token → Transit-Token** im VU-Backend / Validator (Ticketspeicher sieht nur Transit-Token)
- **Ticket-Deposit im Verkauf**
  - Nach Verkauf/Berechtigungsänderung: `POST /tickets`
  - Container: Ticketdaten in vereinbartem Format; `contentType` zur Kennzeichnung
- **Kontrollanbindung**
  - Kontrollgerät/Kontrollbackend liest Medium → Transit-Token → `GET /tickets`
  - Gerät/Backend entscheidet lokal über räumlich/zeitliche Gültigkeit (Ticketspeicher liefert „relevante Tickets“)


### Zertifikate, Rollen & Betrieb

- **mTLS** mit X.509 Zertifikaten (zertifikatsgebundenes OAuth2) zur starken Identitätsbindung
- Rollen/Scopes so schneiden, dass:
  - KVP/CCP nur „eigene“ Tickets schreibt/ändert
  - DL/SO für Kontrolle nur lesend/validierend zugreift

---


## License

This repository is licensed under the Creative Commons Attribution 4.0 International (CC BY 4.0). See [LICENSE](./LICENSE) for the full license text.

Empfohlene Attribution für Wiederverwendung:

"© 2026 Arbeitsgruppe ABT / IDBT (by HUSST) (Arbeitsgruppe des Fachausschuss HUSST von ITS Germany – Bundesverband der Wirtschaft und Wissenschaft für Verkehrstechnologien und intelligente Mobilität e.V.) — licensed under CC BY 4.0 (https://creativecommons.org/licenses/by/4.0/)"
