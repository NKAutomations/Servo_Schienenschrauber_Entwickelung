# Servo_Schienenschrauber_Entwickelung
Servo Schrauberablauf neu entwickeln und Fehler im bestehenden Projekt beseitigen


# Ablaufbeschreibung des Funktionsblocks `FB_Schrauber_PnP`

Diese Ablaufbeschreibung erläutert die Funktionsweise des Schrauber-Funktionsblocks, wie er in der Datei `FB_Schrauber_PnP.TcPOU` implementiert ist. Der Ablauf basiert auf einer zyklischen Schrittkette (Step-Logik), die die Steuerung und Fehlerbehandlung eines automatisierten Schraubprozesses regelt.

## Übersicht

- Der Funktionsblock steuert einen servo-basierten Schrauber in einer Fertigungsanlage.
- Die Steuerung erfolgt über eine Schrittkette, die verschiedene Zustände und Aktionen abbildet.
- Die wichtigsten Ein- und Ausgänge sind: Taktanforderungen, Hand/Auto-Modus, Fehlerquittierung, Achs-/Encoder-IDs, Parameterstrukturen, Zustandsrückmeldungen und Aktionsbefehle.
- Es gibt spezielle Abschnitte für das Setzen der Schraube, das Nachladen, das Auswerfen, die Fehlerbehandlung und das Zurücksetzen.

## Ablauf der Schrittkette im Detail

### Initialisierung (`Step 0: Setzen_Init`)
- Schraubhub und Auswurf werden deaktiviert.
- Autoquit (automatisches Quittieren eines Fehlers) ist inaktiv.
- Der Modus wird auf Modulo-Betrieb gesetzt.
- Warten auf Taktanforderung (`I_bTaktFrg`). Wenn keine Grundstellung anliegt, Wechsel zu Step 10.

### Start (`Step 10: Start`)
- Fehlermerker und Testvariablen werden zurückgesetzt.
- Fehlerüberwachung und Reset werden konfiguriert.
- Nachlade- und Auswurflogik:
  - **Auswerfen:** Wenn Auswurfstart oder Vereinzelungstaste, Wechsel zu Step 200.
  - **Nachladen:** Wenn Schrauber nicht belegt, aber Schlauch belegt, Wechsel zu Step 11/12.
  - **Setzen:** Wenn Schrauber belegt und Startsignal (voll/halb) anliegt, Wechsel zu Step 20.

### Nachladen (`Step 11, 12`)
- Nachladen wird aktiviert, auf Schrauberstange oben gewartet.
- Nach Abschluss zurück zu Step 10.

### Setzvorgang (`Steps 20–100`)
- **Step 20:** Schrauber bereit, Startsignal erkannt.
- **Step 30:** Schraubhub wird aktiviert, Zustellhub ausgeführt.
- **Step 40:** Auswerfhub wird aktiviert, Tiefe beim Aufsetzen geprüft.
  - Wenn die Tiefe im Sollbereich liegt → Flag setzen.
  - Gelingt das Setzen nicht → Fehler (-40).
- **Step 45:** Schraubmotor langsam einschalten, Geschwindigkeit überwachen.
- **Step 50:** Motor läuft schnell, Ziel-Tiefe für Drehzahlwechsel geprüft.
- **Step 60:** Tiefenabfrage, Mittelwertbildung, ggf. Reduzierung der Drehzahl für Drehmomentgenauigkeit.
- **Step 65:** Drehmomentabfrage, Mittelwertbildung, Entscheidung ob Moment erreicht.
- **Step 70:** Schraubmotor Halt, Achse stoppen, Istwert auf Sollwert setzen.
- **Step 75:** Warten bis Motor stillsteht.
- **Step 80/85:** Spindelhub anheben, um Seitenhochziehen zu vermeiden.
- **Step 90/95:** Zylinder fährt in Grundstellung, antriebsgeführtes Referenzieren.
- **Step 100:** Setzvorgang Ende, Rücksetzen der Variablen, Rücksprung zu Step 10.

### Auswerfen (`Steps 200–250`)
- **Step 200:** Auswerfen gestartet, Wartezeit.
- **Step 205–230:** Schraub- und Auswerfhub, Tiefe wird kontrolliert.
- **Step 240:** Auswerfhub vor, Rücksprung bei Fehler.
- **Step 250:** Auswerfen beendet, Rücksprung zu Step 10.

### Fehlerbehandlung (negative Steps)
- Für jeden Fehlerzustand (z.B. -40 Schraube fehlt, -45 Geschwindigkeit, -50 Tiefe für Drehzahl, usw.) gibt es eigenen Step.
- Fehler werden quittiert, Autoquit aktiviert, Auswurfstart ausgelöst.
- Nach Quittierung Rücksprung zu Step 100 (Ende Setzen).

### Sonderfunktionen
- **Autoquit:** Automatische Fehlerquittierung nach Ablauf einer Zeit, um Endlosschleifen zu vermeiden.
- **Mittelwertbildung:** Für Drehmomentmessung, um Ausreißer zu glätten.
- **Diagnosearrays:** Speicherung der letzten Ist-Tiefen, Fehlernummern, Momente und Schraubzeiten für Diagnosezwecke.
- **Timer und Verzögerungen:** Diverse TON-Strukturen zur Ablaufsteuerung und Zeitüberwachung.

## Ablaufdiagramm (Textuell)

```
[Start] --> [Setzen_Init] --> [Start]
          |         |            |
          |         +--> [Nachladen]
          |         +--> [Auswerfen]
          |         +--> [Setzen]
          |
          +--> [Setzvorgang]
                |-- [Schraubhub_vor]
                |-- [Auswerfhub_vor]
                |-- [Schraubmotor langsam]
                |-- [Schraubmotor schnell]
                |-- [Tiefenabfrage]
                |-- [Drehmomentabfrage]
                |-- [Schraubmotor Halt]
                |-- [Spindelhub heben]
                |-- [Zylinder Grundstellung]
                +-- [Setzen Ende]
          |
          +--> [Fehlerbehandlung]
          |
          +--> [Auswurfablauf]
                |-- [AuswerfenStart]
                |-- [Schraubhub_vor]
                |-- [Auswerfhub_zur]
                |-- [AuswerfenEnde]
                +-- [Fehler Auswerfen]
```

## Zusammenfassung

Der Funktionsblock `FB_Schrauber_PnP` implementiert eine komplexe Ablaufsteuerung für einen automatischen Schrauber. Die Schrittkette deckt alle relevanten Prozessschritte ab: Initialisierung, Nachladen, Setzen der Schraube, Auswerfen, Fehlerbehandlung und Rücksetzen. Jeder Schritt ist klar strukturiert und mit Zustands- und Fehlerüberwachung versehen. Die Prozesssicherheit wird durch umfangreiche Timer, Zustandsabfragen, Mittelwertbildungen und Diagnosemechanismen erhöht.

---

**Hinweis:**  
Die genaue Parametrisierung (z.B. Zeiten, Toleranzen, Sollwerte) erfolgt über die diversen Eingangsparameter und Strukturvariablen und ist flexibel anpassbar. Die Fehlerbehandlung und Diagnose ermöglichen eine robuste und nachvollziehbare Steuerung des Schraubprozesses.
