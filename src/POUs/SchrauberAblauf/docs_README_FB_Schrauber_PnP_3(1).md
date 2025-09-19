# FB_Schrauber_PnP_3 – Servoschrauber Pick-and-Place

Dieses Dokument erklärt den Ablauf des Servoschraubers in allgemein verständlichen Worten – mit Fokus auf die drei Schraubfälle (wichtigster Teil) und ASCII-Ablaufgrafiken, damit keine Bilddateien nötig sind.

Inhalt
- Überblick
- Kernprinzipien der Überwachung
- ASCII-Ablaufgrafik (Gesamt)
- Die 3 Schraubfälle – ausführlich
  - Fall 1: Drehmoment OK, Tiefe noch nicht OK
  - Fall 2: Tiefe OK, Drehmoment noch nicht OK
  - Fall 3: Weder Tiefe noch Drehmoment OK
- Typische Fehlerbilder und Reaktionen
- Parametrier- und Tuninghinweise
- Inbetriebnahme-Checkliste
- Referenz

---

## Überblick

Der Baustein steuert einen Servoschrauber mit:
- Überwachung von Eindringtiefe (wie tief die Schraube sitzt)
- Überwachung des Drehmoments (geglättet, um Ausreißer zu vermeiden)
- Überwachung der zusätzlichen Drehung (Nachlaufwinkel nach einem Referenzpunkt)
- Zeitüberwachung (Watchdog), um Hängenbleiben zu verhindern
- Nachladen/Auswerfen der Schraube, sichere Ausgänge, Fehler- und Ereignisprotokollierung

Der eigentliche Schraubprozess verzweigt – abhängig von dem, was bereits erreicht wurde – in drei klar definierte Schraubfälle.

---

## Kernprinzipien der Überwachung

- Tiefe:
  - Es gibt einen zulässigen Bereich vom Mindestniveau bis zur angestrebten Tiefe.
  - Ziel: „Genug drin“, aber nicht über das zulässige Maß hinaus.

- Drehmoment:
  - Es wird als gleitender Mittelwert über mehrere Messwerte bewertet (kein Rohwert).
  - Ziel: Saubere, belastbare Vorspannung ohne Fehlentscheidungen durch kurze Peaks.

- Zusatzdrehung (Winkel):
  - Gemessen als Differenz seit einem Merkmoment (z. B. seit „Tiefe erreicht“).
  - Zwei Obergrenzen:
    - Großzügig, wenn Tiefe noch fehlt (man „arbeitet“ noch).
    - Restriktiv, wenn Tiefe bereits passt (nur noch „sauber vorspannen“).
  - Mindest-Nachdrehung sorgt dafür, dass nicht „auf Kante“ gestoppt wird.

- Watchdog:
  - Setzt eine Obergrenze für die Schraubzeit in jedem Fall.
  - Bei Überschreitung: definierter Abbruch mit Fehlermeldung.

---

## ASCII-Ablaufgrafik (Gesamt)

```
+------------------+
| Start/Init       |
+--------+---------+
         |
         v
+------------------+       +---------------------+       +-------------------+
| Bereitschaft     |-----> | Auswurf-Sequenz     | ----> | zurück Bereitschaft|
+----+----+--------+       +---------------------+       +-------------------+
     |    |
     |    +-----------> Nachlade-Sequenz ----------+
     |                                            |
     |                                            v
     +-----------------> Schraubphase starten  --------+
                                                     |
                                                     v
                                         +-----------------------------+
                                         | Messen & Überwachen:        |
                                         | - Tiefe                     |
                                         | - Drehmoment (geglättet)    |
                                         | - Zusatzdrehung (Winkel)    |
                                         | - Watchdog (Zeit)           |
                                         +---------------+-------------+
                                                         |
             +-------------------------------------------+-------------------------------------------+
             |                                                                                       |
             v                                                                                       v
   [Fall 1] Drehmoment OK,                                                              [Fall 2] Tiefe OK,
         Tiefe noch nicht OK                                                           Drehmoment noch nicht OK
             |                                                                                       |
             v                                                                                       v
       (siehe unten)                                                                             (siehe unten)

             +-------------------------------------------------------------------------------------------+
             |                                                                                           |
             v                                                                                           v
   [Fall 3] Weder Tiefe noch Drehmoment OK                                                       (siehe unten)
             |
             v
        (siehe unten)


                       +--------------------------+
                       | Motor stoppen / Loggen  |
                       +-----------+--------------+
                                   |
                                   v
                         zurück zur Bereitschaft
```

---

## Die 3 Schraubfälle – ausführlich

Gemeinsamkeiten:
- Jeder Fall läuft unter Zeitüberwachung (Watchdog).
- Während des Weiterdrehens werden die aktuell relevanten Kriterien laufend geprüft.
- Bei Überdrehung oder abgelaufener Zeit wird sauber gestoppt und ein Fehler gemeldet.
- Wenn Tiefe noch fehlt, darf die Drehmomentvorgabe moderat angehoben werden (kontrolliert und begrenzt), um das „Durchkommen“ zu unterstützen.

Wichtige Begriffe hier:
- „Große Winkelobergrenze“: großzügig, z. B. bis ungefähr zwei Umdrehungen. Einsatz: Tiefe fehlt noch.
- „Kleine Winkelobergrenze“: restriktiv, z. B. bis ungefähr eine halbe Umdrehung. Einsatz: Tiefe ist bereits OK.

### Fall 1: Drehmoment OK, Tiefe noch nicht OK

Einstieg:
- Das System erkennt: Vorspannung ist grundsätzlich da (Drehmoment passt), aber die Schraube sitzt noch nicht tief genug.

Ziel:
- Die fehlende Tiefe erreichen, ohne zu überdrehen.

Arbeitsfenster:
- Zusatzdrehung ist erlaubt bis zur großen Obergrenze.
- Mindest-Nachdrehung ist implizit erfüllt (Drehmoment ist ja bereits OK), Fokus liegt auf Tiefe.

Ablauf (vereinfachtes ASCII-Diagramm):
```
[Start Fall 1]
      |
      v
+-------------------------------+
| Zusatzdrehung < große Grenze? |
+---------+---------------------+
          |Ja
          v
   +------------------------+
   | Tiefe jetzt erreicht?  |
   +-----+------------------+
         |Ja                      Nein
         v                        |
  [Motor stoppen]                 v
      |                      +-------------------+
      |                      | Watchdog abgelaufen? |
      |                      +----+--------------+
      |                           |Ja
      |                           v
      |                      [Fehler: Timeout]
      |                           ^
      |                           |
      |                      Nein |
      |                           v
      |                  [Drehmoment moderat erhöhen]
      |                           |
      +---------------------------+
```

Reaktionen:
- Wenn die Tiefe noch nicht kommt, darf die Vorgabe für das Drehmoment innerhalb sicherer Grenzen leicht erhöht werden.
- Überschreitung der großen Winkelobergrenze führt sofort zum Abbruch „Überdreht“.

Typische Ursachen/Tuning:
- Mechanische Hemmnisse, Reibung, Materialschwankungen.
- Tuning: leichte Drehmomentanhebung zulassen; evtl. Drehzahl moderat anpassen; große Obergrenze so wählen, dass noch „Arbeit“ möglich ist, aber kein Risiko für Bauteil/Schraube entsteht.

---

### Fall 2: Tiefe OK, Drehmoment noch nicht OK

Einstieg:
- Die Schraube sitzt in der gewünschten Tiefe; es fehlt nur noch die definierte Vorspannung.

Ziel:
- Das geforderte Drehmoment aufbauen, ohne die Schraube „weiter reinzudrehen“ und zu überdrehen.

Arbeitsfenster:
- Zusatzdrehung nur innerhalb der kleinen Obergrenze erlaubt.
- Mindest-Nachdrehung ist erwünscht, damit nicht „auf Kante“ gestoppt wird.

Ablauf (vereinfachtes ASCII-Diagramm):
```
[Start Fall 2]
      |
      v
+-------------------------------+
| Zusatzdrehung < kleine Grenze?|
+---------+---------------------+
          |Ja
          v
   +------------------------------+
   | Drehmoment OK UND Mindest-   |
   | Nachdrehung erreicht?        |
   +-----+------------------------+
         |Ja                         Nein
         v                           |
  [Motor stoppen]                    v
      |                        +-------------------+
      |                        | Watchdog abgelaufen? |
      |                        +----+--------------+
      |                             |Ja
      |                             v
      |                        [Fehler: Timeout]
      |                             ^
      +-----------------------------+
            Nein (weiter prüfen)
```

Reaktionen:
- Keine Drehmomentanhebung nötig, die Tiefe ist ja bereits erreicht; es geht nur noch um sauberes Anziehen.
- Überschreitung der kleinen Obergrenze führt sofort zum Abbruch „Überdreht“.

Typische Ursachen/Tuning:
- Zu restriktives Winkel-Fenster, zu niedrige Drehmomentvorgabe, zu starke Dämpfung.
- Tuning: kleine Obergrenze so wählen, dass das Drehmoment aufgebaut werden kann, ohne die Verbindung unnötig weiter zu drehen; Mindest-Nachdrehung sinnvoll wählen.

---

### Fall 3: Weder Tiefe noch Drehmoment OK

Einstieg:
- Weder Tiefe noch Drehmoment sind im grünen Bereich. Es muss beides erreicht werden.

Ziel:
- Erst die Tiefe sicherstellen und gleichzeitig das Drehmoment in den Sollbereich bringen – effizient, aber ohne Überdrehen.

Arbeitsfenster:
- Zusatzdrehung ist erlaubt bis zur großen Obergrenze (ähnlich Fall 1).

Ablauf (vereinfachtes ASCII-Diagramm):
```
[Start Fall 3]
      |
      v
+-------------------------------+
| Zusatzdrehung < große Grenze? |
+---------+---------------------+
          |Ja
          v
   +-----------------------------------+
   | Tiefe OK UND Drehmoment OK ?      |
   +-----+-----------------------------+
         |Ja                               Nein
         v                                 |
  [Motor stoppen]                          v
      |                              +-------------------+
      |                              | Watchdog abgelaufen? |
      |                              +----+--------------+
      |                                   |Ja
      |                                   v
      |                              [Fehler: Timeout]
      |                                   ^
      |                                   |
      |                              Nein |
      |                                   v
      |                        +--------------------------+
      |                        | Tiefe noch nicht OK ?    |
      |                        +----+---------------------+
      |                             |Ja
      |                             v
      |                  [Drehmoment moderat erhöhen]
      |                             |
      +-----------------------------+
                Nein (weiter prüfen)
```

Reaktionen:
- Wenn Tiefe ausbleibt, ist eine moderate, befristete Anhebung der Drehmomentvorgabe erlaubt.
- Überschreitung der großen Obergrenze führt sofort zum Abbruch „Überdreht“.

Typische Ursachen/Tuning:
- Kombination aus Reibung, Material, Geometrie; eventuell zu niedrige Startvorgaben.
- Tuning: begrenzte Drehmomentanhebung für „Durchkommen“, große Obergrenze korrekt dimensionieren.

---

## Typische Fehlerbilder und Reaktionen

- Timeout (Zeit überschritten):
  - Fallbezogener Abbruch mit entsprechendem Hinweis (z. B. „Tiefe/Drehmoment nicht erreicht“).
  - Prüfen: Freigaben, Mechanik, Parameter, Drehzahl.

- Überdrehung (Winkelobergrenze überschritten):
  - Sofortiger Abbruch.
  - Prüfen: Grenzen realitätsnah parametrieren (klein/groß), Mindest-Nachdrehung, Umschaltpunkte.

- Tiefe nicht erreicht:
  - Prüfen: Vorschub, Aufsetzen, Bauteiltoleranzen.
  - Maßvoll Drehmomentvorgabe anheben (nur solange Tiefe fehlt, niemals unkontrolliert).

- Drehmoment nicht erreicht:
  - Prüfen: Reibwerte, Material, ggf. zu restriktives Winkel-Fenster im Fall 2.

---

## Parametrier- und Tuninghinweise

- Winkelobergrenzen:
  - Großzügige Grenze (wenn Tiefe fehlt): erlaubt „Arbeit“ am Bauteil; Erfahrungswert z. B. bis ca. zwei Umdrehungen.
  - Kleine Grenze (wenn Tiefe passt): begrenzt Nachlauf; Erfahrungswert z. B. bis ca. eine halbe Umdrehung.

- Mindest-Nachdrehung:
  - Verhindert Stop genau am Schwellwert; sorgt für reproduzierbares Setzen der Verbindung.

- Drehmomentvorgabe:
  - Anfangs realistisch wählen; bei Tiefe-Problemen in Fall 1/3 moderat erhöhen (mit Obergrenzen).
  - Drehmomentbewertung immer als geglätteter Mittelwert denken (Peaks ignorieren).

- Drehzahlreduktion:
  - Nach „Tiefe OK“ sinnvoll, um feinfühlig das Moment aufzubauen (Vermeidung von Überschwingen).

- Watchdog:
  - Realistisch zur Applikation wählen, damit Abbrüche echte Fehler abbilden und keine „falschen Positiven“ erzeugen.

---

## Inbetriebnahme-Checkliste

- Sicherheitskette / Schutzbereich geprüft (Freigaben, Türlogik)?
- Antrieb bereit, Fehlerpfade getestet (Freigaben, Stillstandserkennung)?
- Sensorik plausibel (Tiefe, Position, Drehmomentanzeige)?
- Parameter grob eingestellt:
  - Sollmoment, Mindest-/Solltiefe, Mindest-Nachdrehung
  - Winkelobergrenzen (klein/groß)
  - Watchdog-Zeiten
- Test Vollschraubung und Halbschraubung getrennt durchführen.
- Fehlerfälle gezielt provozieren (Timeout/Überdrehung) und Reaktionen prüfen.
- Historie/Protokollierung verifizieren (Tiefe, Moment, Zeit, Fehler).

---

## Referenz

- Baustein-Datei: [src/POUs/SchrauberAblauf/FB_Schrauber_PnP_3.TcPOU](https://github.com/NKAutomations/Servo_Schienenschrauber_Entwickelung/blob/36b534d5bd3ecd58059b0af279687edaf51e5fbf/src/POUs/SchrauberAblauf/FB_Schrauber_PnP_3.TcPOU)

Hinweis: Diese README beschreibt den Funktionsablauf bewusst ohne interne Variablennamen. Sie eignet sich zur Weitergabe an Bediener, QS und Kollegen aus Mechanik/Elektrik.
