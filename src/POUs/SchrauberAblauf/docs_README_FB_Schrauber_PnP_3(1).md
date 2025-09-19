# FB_Schrauber_PnP_3 – Servoschrauber Pick-and-Place (Möbelindustrie)

Vollautomatischer Servoschrauber-Baustein für Schienen- und Einzelschrauben mit NC-Achsenanbindung, Tiefen-, Winkel- und Drehmomentüberwachung.

- Quelle: [src/POUs/SchrauberAblauf/FB_Schrauber_PnP_3.TcPOU](https://github.com/NKAutomations/Servo_Schienenschrauber_Entwickelung/blob/36b534d5bd3ecd58059b0af279687edaf51e5fbf/src/POUs/SchrauberAblauf/FB_Schrauber_PnP_3.TcPOU)
- TwinCAT: 3.1.4024.15
- Autor: N. Kersting

## Inhalt
- Zweck und Funktionsumfang
- Architektur und Ablauf
- Prüf- und Auswertelogik (Tiefe, Drehmoment, Winkel)
- Die 3 Schraubfälle
- Schnittstellen (Inputs/Outputs/InOut)
- Wichtige Methoden
- Parametrierung und Formeln
- Fehler- und Ereignismanagement
- Inbetriebnahme-Hinweise
- Troubleshooting
- Changelog

---

## Zweck und Funktionsumfang

Dieser Baustein steuert einen Servoschrauber im Pick-and-Place-Prozess:
- Anbindung an eine NC-Drehachse (Schraubmotor) inkl. Überwachung von Stillstand, Positionierfreigabe und NC-Fehlern.
- Prozessüberwachung:
  - Eindringtiefe der Schraube
  - Drehmoment (geglättet)
  - Nachlaufwinkel (zusätzliche Drehung nach Referenzpunkt)
  - Watchdog (Timeouts)
- Unterstützung verschiedener Betriebsarten (Voll-/Halbschraubung), Nachladen und Auswerfen.
- Fehlerbehandlung und detaillierte Protokollierung (Historie).

---

## Architektur und Ablauf

Der Baustein arbeitet als deterministische Zustandsmaschine mit nummerierten Schritten. Vereinfacht:

- 0: Initialisierung
- 10: Bereitschaft (Freigaben, Vorbereitungen, Auswahl Start: Schrauben/Nachladen/Auswerfen)
- 55–70: Schraubphase mit laufender Prüfung (Tiefe/Drehmoment/Winkel)
- 70: Anhalten/Stop (Motor Halt, Logging)
- 100: Fertig
- 200 ff.: Auswurfsequenz
- < 0: Fehlerzustände

Zur Veranschaulichung ein Flowchart des Kern-Schraubablaufs und der drei Schraubfälle:

```mermaid
flowchart TD
  A([Start / Initialisierung])
  B[Bereitschaft]
  A --> B

  %% Nebenabläufe
  B -->|Auswerfen angefordert| AW[Auswurf-Sequenz]
  AW --> B
  B -->|Nachladen erforderlich| NL[Nachlade-Sequenz]
  NL --> B

  %% Schraubstart
  B -->|Start Voll- / Halbschraubung| S[Schraubphase starten]
  S --> M[Messungen & Überwachung:
  - Eindringtiefe
  - Drehmoment (geglättet)
  - Zusatzdrehung (Winkel)
  - Zeitüberwachung (Watchdog)]

  M -->|Tiefe noch nicht OK, Drehmoment OK| F1
  M -->|Tiefe OK, Drehmoment noch nicht OK| F2
  M -->|Tiefe nicht OK, Drehmoment nicht OK| F3

  %% Schraubfall 1
  subgraph F1[Schraubfall 1: Drehmoment OK, Tiefe nicht OK]
    F1a{Zusatzdrehung < große Obergrenze?}
    F1a -- Nein --> E1F1[Fehler: Überdreht]
    F1a -- Ja --> F1b{Ist Tiefe jetzt OK?}
    F1b -- Ja --> STOP
    F1b -- Nein --> F1c{Watchdog abgelaufen?}
    F1c -- Ja --> T1[Fehler: Timeout (Tiefe nicht erreicht)]
    F1c -- Nein --> A1[Aktion: Drehmoment moderat erhöhen]
    A1 --> F1a
  end

  %% Schraubfall 2
  subgraph F2[Schraubfall 2: Tiefe OK, Drehmoment nicht OK]
    F2a{Zusatzdrehung < kleine Obergrenze?}
    F2a -- Nein --> E1F2[Fehler: Überdreht]
    F2a -- Ja --> F2b{Drehmoment OK UND Mindest-Nachdrehung erreicht?}
    F2b -- Ja --> STOP
    F2b -- Nein --> F2c{Watchdog abgelaufen?}
    F2c -- Ja --> E2F2[Fehler: Timeout (Drehmoment nicht erreicht)]
    F2c -- Nein --> F2a
  end

  %% Schraubfall 3
  subgraph F3[Schraubfall 3: Tiefe nicht OK, Drehmoment nicht OK]
    F3a{Zusatzdrehung < große Obergrenze?}
    F3a -- Nein --> E1F3[Fehler: Überdreht]
    F3a -- Ja --> F3b{Tiefe OK UND Drehmoment OK?}
    F3b -- Ja --> STOP
    F3b -- Nein --> F3c{Watchdog abgelaufen?}
    F3c -- Ja --> E2F3[Fehler: Timeout]
    F3c -- Nein --> F3d{Tiefe noch nicht OK?}
    F3d -- Ja --> A3[Aktion: Drehmoment moderat erhöhen]
    F3d -- Nein --> F3a
    A3 --> F3a
  end

  %% Abschluss / Logging
  STOP((Motor stoppen))
  STOP --> LOG[Protokollieren:
  - Tiefe, Drehmoment-Mittelwert, Winkel, Zeit
  - Fehler-/Statusmeldungen setzen]
  LOG --> DONE[Fertig / Zurück in Bereitschaft]
  DONE --> B
```

---

## Prüf- und Auswertelogik

- Tiefenprüfung:
  - Prüft, ob die erreichte Eindringtiefe im Sollbereich liegt (Mindesttiefe bis Solltiefe).
  - Separate Minimalgrenzen z. B. für Aufsetzen und Halbschraubung.

- Drehmomentprüfung:
  - Glättung über 10 Messwerte (gleitender Mittelwert), um kurze Peaks zu filtern.
  - Vergleich des Mittelwerts mit dem Sollmoment.

- Drehwinkelprüfung:
  - Ermittelt Zusatzdrehung ab einem gespeicherten Referenzpunkt.
  - Mindest-Nachdrehung erforderlich.
  - Obergrenzen verhindern Überdrehen (großzügig, wenn Tiefe fehlt; restriktiv, wenn Tiefe bereits passt).

- Watchdog:
  - Zeitüberwachung der Schraubphase, führt bei Überschreitung zu definierten Fehlern.

---

## Die 3 Schraubfälle

1) Drehmoment OK, Tiefe nicht OK  
- Ziel: Tiefe noch erreichen.  
- Aktion: Weiterschrauben innerhalb einer großen Winkelobergrenze. Bei Bedarf Drehmomentvorgabe moderat erhöhen.  
- Abbruch: Timeout oder Überdrehung.

2) Tiefe OK, Drehmoment nicht OK  
- Ziel: Drehmoment sauber aufbauen ohne Überdrehen.  
- Aktion: Weiterschrauben innerhalb kleiner Winkelobergrenze (nur vorspannen).  
- Abbruch: Timeout (Drehmoment nicht erreicht) oder Überdrehung.

3) Tiefe nicht OK und Drehmoment nicht OK  
- Ziel: Beide Kriterien erreichen.  
- Aktion: Weiterschrauben mit großer Winkelobergrenze, bei ausbleibender Tiefe Drehmomentvorgabe moderat erhöhen.  
- Abbruch: Timeout oder Überdrehung.

---

## Schnittstellen

### VAR_INPUT (Auszug, gruppiert)
- Steuerung:
  - I_bResetTeststation, I_bTaktFrg, I_bTaktStop, I_bAuto, I_bHand, I_bQuittFehler, I_bGrundstellung, I_bSchutzbereichOK
- Prozessstart:
  - I_bTaktStart (Vollschraubung), I_bTaktStartHalb (Halbschraubung)
- Prozesssensorik:
  - I_bGrdstlgSchrEinh (Grundstellung Schraubeinheit), I_bZustellZylAusgef (Zustellzylinder Endlage)
- Parameter:
  - I_fSchraubTiefeAuswurf, I_fTiefentoleranzPositiv/Negativ
- Achsen:
  - I_nAchsId, I_nEncId, I_stParam (Prozessparameterstruktur)
- Erweiterungen:
  - I_tAutoQuit, I_bSchraubeNachschiessen, I_bReferenzfahrtAktiv, I_bFreigabeAuswerfen
- Logging:
  - I_sDateipfad, I_bSchrauberLogOn, I_nMotorHersteller, I_bHuettenSchr

### VAR_OUTPUT (Auszug)
- Haupt:
  - Q_bBusy, Q_bDone, Q_bError
- Status:
  - Q_sPosZustand, Q_bGrundstellungAktiv, Q_bGrundstellungOk, Q_sStatus
- Aktorik:
  - Q_bSchraubHub, Q_bSchrAuswurf, Q_bSchrHalt, Q_nTorque, Q_bAxisReset
- Diagnose:
  - Q_nErrorId, Q_bDrehmomentFehler, Q_bTuerFreigabe, Q_nStep, Q_strStep, Q_bModuloBetrArt

### VAR_IN_OUT
- IQ_fbObjSchlauch: Schnittstelle zum Schlauch-/Vereinzelungssystem
- IQ_stMeldung: Meldungsstruktur des Schraubers

---

## Wichtige Methoden

- m_HandleReset: Behandelt Reset-Quellen (Nachrüttel-Taste, Test-Reset) und sperrt in kritischen Schritten.
- m_Hauptablauf: Zustandsmaschine inkl. Startlogik, Betriebsartenwahl, Auswurf-/Nachladeabläufe, Schraubphase.
- m_DrehmomentMittelwert: 10er-Gleitmittelwert des Drehmoments (nur aktiv in Prüfphase).
- m_DrehmomentPruefung: Vergleicht geglättetes Moment mit Sollwert.
- m_DrehwinkelPruefung: Ermittelt Zusatzdrehung und prüft Fenster (Mindest- und Maximalwinkel).
- m_DrehmomentRechnung: Rechnet Antriebs-Drehmomentistwert (in 1/1000 I_rated) in Nm um.
- m_DrehmomentVorgabe: Rechnet Sollmoment (Nm) in die Antriebseinheit (1/1000 I_rated) mit Skalierung.
- m_AchsKommunikation: Kapselt Ein-/Ausgänge zur NC-Achse (Signalworte, Freigaben, Status).
- m_DiagnoseHistorie: Speichert Historie (Tiefe, Fehlercode, Moment, Schraubzeit) bei Schraubende.
- m_Fehlerbehandlung: Setzt/cleart spezifische Fehlerbausteine und Aggregat-Fehlerflag.
- m_Ausgangszuweisung: Zentrale, sicherheitsverriegelte Ausgangsführung.

Hinweis: Im Code sind weitere Hilfsmethoden/Timer vorgesehen (Stillstandsüberwachung, Logging, Schraubtaktmessung).

---

## Parametrierung und Formeln

### Umrechnung Drehmoment (Ist)
Formel gemäß Beckhoff-Doku (Index 0x8010:54 = 0):
```
M[Nm] = ((TorqueActual/1000) * (I_rated/√2)) * torque_constant
```

### Umrechnung Drehmoment (Soll → Limitierung)
Inverse Umrechnung inkl. Skalierung:
```
TorqueLimit(1/1000 I_rated) = ((M_soll / torque_constant) / (I_rated / √2)) * 1000 * Skalierung
```

- Typische Skalierungsstrategie:
  - Start mit leicht abgesenkter oder erhöhter Vorgabe (Schutz vor Peaks/Überdrehen).
  - Bei ausbleibender Tiefe: begrenzte Erhöhung freigeben.

### Winkelgrenzen
- Große Obergrenze (wenn Tiefe noch fehlt): z. B. 720° (≈ 2 Umdrehungen).
- Kleine Obergrenze (wenn Tiefe passt): z. B. 180°.

### Drehzahlreduktion
- Reduktion nach „Tiefe OK“ (feineres Positionieren).
- Separate Reduktion für Halbschraubung.

### Mindesttiefen
- Mindesttiefe für Aufsetzen inkl. Toleranz.
- Mindest-/Maximalbereiche je Schraubstrategie (Voll/Halb).

---

## Fehler- und Ereignismanagement

- Spezifische Fehler (Beispiele, gemäß Code):
  - -40: Fehler Nachladen
  - -45: Langsame Geschwindigkeit nicht erreicht
  - -50: Tiefe für Solldrehzahl nicht erreicht
  - -60: Solltiefe nicht erreicht
  - -61: Solltiefe überschritten
  - -65: Solldrehmoment nicht erreicht
  - -80: Fehler Schrauberstange (Heben/Senken)
  - -210: Fehler Auswerfen
- Dynamische Fehler in Schraubfällen:
  - -1261, -1262, -1263: Winkelobergrenze überschritten in Fall 1/2/3
  - -61x / -62x / -63x: Timeout-/Kontextfehler je Fall
- Aggregierte Anzeige:
  - Q_bError aktiv bei Fehler und wenn Log-Modus nicht explizit aktiv ist.
- Historie:
  - Speicherung der letzten ~100 Schraubvorgänge (Tiefe, Fehlercode, Moment, Zeit).

---

## Inbetriebnahme-Hinweise

1) Antriebsdaten prüfen:
- Nennstrom (I_rated) und Motorkonstante korrekt parametrieren.
- Richtung/Kodierung des Winkel-/Positionsfeedbacks prüfen.

2) Sicherheitsfreigaben:
- Schutzbereich-Signale und Türfreigabe-Logik testen.
- NC-Fehler-/Freigabepfade (Positionierfreigabe, DC-Status) prüfen.

3) Prozessparameter:
- Sollmoment, Winkelgrenzen, Tiefenfenster und Toleranzen schrittweise einregeln.
- Reduktionsfaktoren für Drehzahl sinnvoll wählen (Feinpositionierung).

4) Tests:
- Halbschraubung separat testen (kleineres Fenster, andere Tiefe).
- Nachladen/Auswerfen-Funktionen unter Störung testen.

5) Logging:
- Dateipfade, Log-Modus und Historie prüfen (zur Qualitätssicherung).

---

## Troubleshooting (Kurz)

- „Drehmoment nicht erreicht“:
  - Mechanische Hemmnisse? Zu geringe Vorgabe? Skalierung/Drehzahl prüfen.
  - Prüfen, ob die kleine Winkelobergrenze zu restriktiv ist (Fall 2).

- „Tiefe nicht erreicht“:
  - Vorschub/Pneumatik und Aufsetzen prüfen.
  - Gegebenenfalls Drehmomentvorgabe moderat erhöhen (Begrenzung beachten).

- „Überdreht“:
  - Winkelobergrenzen reduzieren oder Mindest-Nachdrehung anpassen.
  - Früheres Umschalten auf „Tiefe OK“-Fenster sicherstellen.

- Häufige Timeouts:
  - Watchdog-Zeiten plausibilisieren.
  - Antriebsfreigabe/Stillstandsüberwachung prüfen.

---

## Changelog (aus Header)

- 17/09/2025 – v1.00 – First release
- 18/09/2025 – v1.01 – Erste erfolgreiche Verschraubungen mit Scope
- 19/09/2025 – v1.02 – Diverses

---

## Datei/Referenz

- Baustein: [FB_Schrauber_PnP_3.TcPOU](https://github.com/NKAutomations/Servo_Schienenschrauber_Entwickelung/blob/36b534d5bd3ecd58059b0af279687edaf51e5fbf/src/POUs/SchrauberAblauf/FB_Schrauber_PnP_3.TcPOU)

Bei Bedarf kann ein PR erstellt werden, der diese README im Repo ergänzt (z. B. im Ordner `docs/` oder direkt neben dem Baustein).