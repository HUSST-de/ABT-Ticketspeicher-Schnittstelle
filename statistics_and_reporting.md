# Konzeptspezifikation: Statistiken & Reporting im zentralen Ticketspeicher

*Dieses Dokument nutzt die englischen Abkürzungen für die verschiedenen Rollen*

| Deutsch | Englisch |
|-----------|--------|
| Dienstleister (DL) | Service Operator (SO) |
| Kundenvertragspartner (KVP) | Customer Contract Partner (CCP) |
| Produktverantwortlicher (PV) | Product Owner (PO) |


## 1. Management Summary und Zielsetzung
In ID-Based- / Account-Based-Ticketing-Systemen werden Fahrtberechtigungen in nur lokal erreichbaren Ticketspeichern der Verkehrsunternehmen gespeichert. Der zentrale Ticketspeicher fungiert als Datendrehscheibe für die interoperable und übergreifende Berechtigungsprüfung dieser Fahrtberechtigungen.

Damit dieses System betrieblich steuerbar ist und Interoperabilität nachgewiesen werden kann, ist ein umfassendes Reporting erforderlich. Ziel dieses Arbeitspakets ist die Definition der zu protokollierenden Ereignisse, der daraus abgeleiteten statistischen Kennzahlen sowie der Bereitstellungswege für die Akteure. Besonderer Fokus liegt dabei auf der Unterscheidung zwischen technischem Fehlverhalten (Latenzen) und fachlichen Ergebnissen (ungültige Tickets).

## 2. Akteure und Informationsbedarf
Um die richtigen Statistiken zu definieren, müssen die Perspektiven der beteiligten Akteure unterschieden werden:

* **Der Ticketspeicher-Betreiber (z. B. die eTS):** Benötigt technische Metriken zum Betriebsstatus des Ticketspeichers sowie zur Auslastung, Verteilung und Nutzung des Systems aggregiert zu den teilnehmenden Mandanten.
* **Der Berechtigungsausgeber (CCP - Customer Contract Partner):** Das VU, das das Ticket verkauft (z. B. Rheinbahn). Fragestellung: "Wie viele meiner Kunden mit übergreifenden Produkten werden auch übergreifend kontrolliert? Wo werden meine Kunden kontrolliert?"
* **Das kontrollierende Verkehrsunternehmen (SO - Service Operator):** Das VU, das die Kontrolle durchführt (z. B. KVB). Fragestellung: "Wie viele Fremdkunden habe ich heute geprüft? Wo erwerben Fremdkunden ihre Fahrtberechtigungen? Wie hoch ist die Quote ungültiger Token?"

## 3. Datenbasis: Protokollierung der Aktionen
Die Grundlage aller Auswertungen ist ein detailliertes Event-Logging der Interaktionen mit den fachlichen Endpunkten. Der Ticketspeicher muss u. a. folgende Kernaktionen der CCPs und SOs persistieren:

### 3.1 Event: TicketStored (Publishen von Berechtigungen)
Wird ausgelöst, wenn ein CCP ein Ticket in den Speicher schreibt.
* Zeitstempel der lokalen Ausgabe des Tickets.
* Zeitstempel des Empfangs im Hub.
* CCP Org-ID (Wer liefert?).
* Gültigkeitsbeginn und -ende des Tickets (lt. Fahrtberechtigung).

### 3.2 Event: TokenChecked (Abrufen von Berechtigungen)
Wird ausgelöst, wenn ein SO (Prüfgerät) eine Anfrage an den Hub stellt.
* Zeitstempel der Anfrage.
* SO Org-ID (Wer prüft?).
* Liste der zum Zeitpunkt der Anfrage gefundenen Fahrtberechtigungen inkl. Status (Gültig / Ungültig / Unbekannt).

### 3.3 Event: TokenExchanged (Transittoken ausgetauscht)
Wird ausgelöst, wenn ein CCP ein Transittoken durch ein anderes ausgetauscht hat.
* Zeitstempel der Anfrage.
* CCP Org-ID (Wer ändert?).
* Liste der zum Zeitpunkt der Anfrage mit dem alten Transittoken asoziierten Fahrtberechtigungen

## 4. Abzuleitende Metriken und Fachliche Statistiken

### 4.1 Volumen- und Last-Metriken
* **Total Stored Tokens:** Gesamtzahl der physisch im Speicher liegenden Datensätze.
* **Active Valid Tokens:** Anzahl der Token, deren Gültigkeitszeitraum den aktuellen Zeitpunkt einschließt.
* **Token types used:** Anzahl der Token aufgeschlüsselt nach Typen (inkl. Subtypen).
* **Transaction Volume:** Anzahl der Lese-Zugriffe und Schreib-Zugriffe pro Zeitintervall.
* **Peak Load:** Maximalwert der Transaktionen pro Minute innerhalb eines Berichtszeitraums.

### 4.2 Validierungsergebnisse
* **Success Rate (Gültig):** Verhältnis von Anfragen mit Ergebnis "Gültiges Tickets gefunden" zur Gesamtzahl.
* **Expiration Rate (Abgelaufen):** Das Token war bekannt, jedoch lag der Zeitpunkt der Kontrolle außerhalb des Gültigkeitszeitraums des Tickets.
* **Unknown Token Rate (Unbekannt):** Verhältnis von Anfragen, bei denen der Token im Hub nicht existiert.
* **Technical Error-Rate:** Verhältnis technisch fehlgeschlagener Abfragen (Timeouts, 5xx-Fehler, 4xx-Fehler).
* **Sleeping-Assets:** Verhältnis von nie kontrollierten Token an der Token-Gesamtanzahl.

### 4.3 Anomalie-Erkennung und Prozessqualität
Das System korreliert die Zeitstempel aus `TokenStored` und `TokenChecked`, um Latenzen von Nutzerfehlverhalten zu unterscheiden:

* **Nutzerseitiges Fehlverhalten:** Ticketspeicher meldet "Kein Ticket". Auch nach einer Karenzzeit (z. B. 24h) geht kein Ticket ein. Diese Fälle werden als "Fahren ohne gültigen Fahrausweis" (EBE) kategorisiert.
* **Systembedingte Latenz:** Kontrolle um 10:00 Uhr negativ, aber um 10:05 Uhr liefert der CCP ein Ticket ein, deren Gültigkeit bereits ab 09:00 Uhr bestand.
* **Metrik "System Latency":** Zeitdifferenz zwischen Ausgabezeitpunkt und Empfang im Ticketspeicher. Dient dem Qualitätsmanagement zu langsamer Hintergrundsysteme.

### 4.4 Interoperabilitäts-Matrix
Die Statistik bereitet den Traffic am Hub in einer Matrix auf:
* **Zeilen (Wer prüft?):** Der anfragende Service Operator (SO).
* **Spalten (Woher kommt der Kunde?):** Der ausgebende Customer Contract Partner (CCP).
* **Zelleninhalt:** Anzahl der positiven Validierungsanfragen.

## 5. Bereitstellung der Reports

### 5.1 Internes Betreiber-Cockpit (Operationelles Dashboard)
* **Format:** Web-GUI (Geschützter Admin-Bereich).
* **Intervall:** Echtzeit (Real-time) / On-Demand.
* **Funktionen:** Live-Monitoring, Mandanten-Drill-Down, Ad-hoc Exporte (CSV/Excel) und Audit-Log-Viewer.

### 5.2 Push-Reporting für Teilnehmer (Management Summary)
* **Format:** PDF (visuell) und CSV (Rohdaten als Anhang).
* **Intervall:** Periodisch und automatisch (z. B. monatlich).
* **Inhalt:** Mandantenspezifische Berichte über eigene Verkäufe, Kontrollen und Fremdnutzung.

### 5.3 Pull-API (Maschinenlesbare Schnittstelle)
* **Format:** REST-Endpunkt (JSON / CSV).
* **Zielgruppe:** IT-Abteilungen der VUs, Data Analysts.
* **Funktion:** Abfrage aggregierter Statistikdaten unter Berücksichtigung der Mandantentrennung.

## 6. Ausblick: Vorbereitung für Phase 2 (CiCo)
Die statistischen Auswertungen bilden das Fundament für zukünftige Check-in/Check-out (CiCo) Modelle. Die Interoperabilitäts-Matrizen dienen als Indikator für künftige Clearing-Ströme und sollen später nahtlos in externe Clearing-Systeme überführt werden.
