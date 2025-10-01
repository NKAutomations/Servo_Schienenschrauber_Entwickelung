# Dokumentation Baustein fb_SKSSchraubNob

## 1. Überblick / Zweck
Der Baustein `fb_SKSSchraubNob` kapselt den **reinen servo‑motorischen Schraubprozess** (Drehzahl-, Drehmoment-, Winkel- und Tiefenführung) und lässt alle peripheren Funktionen (z. B. Pneumatik, Schraubenzuführung) im bestehenden Standard-Schrauberbaustein. Dadurch kann der Motor-Schraubkern in bestehende Anlagen integriert werden, ohne tiefgreifende Umbauten in der Peripherie vornehmen zu müssen.  
Unterstützt werden:
- Drehmomentüberwachung  
- Drehwinkelberechnung / -begrenzung  
- Tiefenerfassung (Position entspricht Einschraubtiefe)  
- Klassifizierung des Verlaufs in definierte Schraubfälle  
- Zeit- und Qualitätsauswertung  

## 2. Integration in einen vorhandenen Schrauberablauf
Einbettung erfolgt typischerweise so:
1. Bestehender Ablauf (PnP / Handlingslogik) positioniert Werkzeug, versorgt Schraube, setzt ggf. Pneumatik.
2. Achsstruktur (IN/OUT) wird an den Baustein übergeben (standardisierte Servoantriebs-Schnittstelle).
3. Parametrierstruktur (Schraubparameter) wird 1:1 durchgereicht (keine Aufsplittung nötig).
4. Start-/Freigabe-/Reset-/Quittiersignale kommen aus übergeordneter Ablaufsteuerung.
5. Prozessdaten (Tiefe, Drehmoment, Winkel, Dauer, Klassifikation) werden nach Abschluss zurückgemeldet und können für Trace, Qualität oder MES erfasst werden.
6. Fehler- oder Schraubfall-Ergebnis kann direkt in nachfolgende Entscheider (OK/NOK/Sortierung) einfließen.

Vorteile:
- Plug-in für bestehenden PnP-Ablauf statt monolithischem Schrauber-FB.
- Leichte Austauschbarkeit oder parallele Variantenentwicklung.
- Geringeres Risiko bei Änderungen (nur Kernlogik betroffen).
- Einheitliche Achsschnittstelle fördert Wiederverwendung.

## 3. Architektur & Zustandslogik (technischer Überblick)
Der Baustein verwendet eine deterministische Zustandsmaschine (nummerierte Schritte). Kernphasen:
1. Initialisierung / Bereitschaft
2. Anlauf mit reduzierter Drehzahl
3. Übergang auf Hauptdrehzahl
4. Drehzahlreduktion vor Auswertung
5. Schraubfall-Erkennung (Entscheidung: welcher Verlauf liegt vor?)
6. Feinschraubphase je nach Schraubfall (1, 2 oder 3)
7. Bremsen / Stillsetzen
8. Ergebnis- und Fehlerauswertung
9. Rückkehr in Bereitschaft

Ergänzend: eigene Sequenz zur Leerlauf-/Schwergangprüfung mit separater Unter-Zustandsmaschine.

## 4. Schraubfälle (Klassifikation des Verlaufs)
Die Klassifikation beruht auf der Relation zwischen erreichten Zielgrößen (Tiefe, Drehmoment) zum Zeitpunkt der Auswertung:

| Schraubfall | Auslöser (Situation) | Zielstrategie | Typische Maßnahmen | Abbruchkriterien |
|-------------|----------------------|---------------|--------------------|------------------|
| 1 | Drehmoment erreicht, Tiefe noch nicht | Weiter eindringen ohne Überdrehen | Winkelreserve aus fehlender Tiefe + Nachziehreserve, ggf. Drehmoment moderat erhöhen | Timeout, Winkelgrenze überschritten |
| 2 | Tiefe erreicht, Drehmoment noch nicht | Vorspannung sauber aufbauen | Begrenztes Nachdrehen in kleinem Winkel-Fenster | Timeout (Moment fehlt), Fenster überschritten |
| 3 | Weder Tiefe noch Moment erreicht | Beides sicher erreichen | Zunächst größere Winkel-/Tiefenreserve, dann Übergang wie Fall 1 bzw. 2 | Timeout, Winkelobergrenze |
| Leerlaufprüfung | Vor dem eigentlichen Prozess: Antrieb läuft „frei“ | Erkennen von Schwergang / Blockade | Prüfen ob Geschwindigkeit mit reduziertem Moment erreicht wird | Timeout oder zu niedrige Drehzahl |

Zusammenfassung:
- Fall 1: Verbindung zieht früh (Moment da), Schraube steht noch sichtbar über Oberfläche.
- Fall 2: Schraube liegt bündig (Tiefe da), aber Vorspannung fehlt noch.
- Fall 3: „Träger Verlauf“ – weder Tiefe noch Moment früh erreicht, daher weiter beobachten und adaptiv reagieren.
- Leerlaufprüfung: Schutzmechanismus / Frühdiagnose.

## 5. Ablaufbeschreibung (für Nicht-Programmierer – ohne Variablennamen)
1. Der Baustein wartet, bis die übergeordnete Steuerung signalisiert, dass alles bereit ist (Werkzeug positioniert, Schraube vorhanden, Sicherheitsfreigaben aktiv).
2. Beim Start setzt er interne Messgrößen (z. B. Zähler und Qualitätsmerker) zurück.
3. Der Schraubvorgang beginnt langsam, um ein ruckfreies Ansetzen sicherzustellen und frühe Auffälligkeiten (Blockade, verkantete Schraube) zu erkennen.
4. Erhöht sich die Einschraubtiefe wie erwartet, wird beschleunigt, um Zeit zu sparen.
5. Kurz vor einer vorgesehenen Übergangszone wird wieder auf eine niedrigere Geschwindigkeit reduziert, um kontrolliert in den sensiblen Bereich (Finalisierung) einzufahren.
6. Nun vergleicht das System: Ist bereits die benötigte Tiefe erreicht? Ist das gewünschte Drehmoment erzielt? Aus dieser Kombination erkennt es, welcher Verlauf vorliegt.
7. Je nach Verlauf wendet es ein passendes Feinkonzept an:
   - Nur Tiefe fehlt → etwas weiter eindrehen, ohne zu fest anzuziehen.
   - Nur Drehmoment fehlt → gezielte Nachdrehung, begrenzt im Winkel.
   - Beides fehlt → weitere kontrollierte Bewegung, mit eventueller Anpassung.
8. Währenddessen werden laufend Messwerte gemittelt (z. B. Drehmoment), um Ausreißer zu filtern.
9. Erreichen alle Zielgrößen gültige Werte (Tiefe im Fenster, Drehmoment ausreichend, Winkel nicht zu groß), wird der Antrieb gestoppt.
10. Der Prozess wird protokolliert: Zeitbedarf, erreichte Tiefe, maximales Drehmoment, gedrehter Winkel und Klassifikation.
11. Bei Unregelmäßigkeiten (z. B. Zeit überschritten, Winkel zu groß) markiert das System den Vorgang als fehlerhaft und wartet auf eine Quittierung.
12. Nach Freigabe für den nächsten Zyklus kehrt der Baustein in die Ausgangsbereitschaft zurück.

## 6. Prozessdaten (Ergebnis- / Qualitätsdaten)
| Name (Ausgabe) | Bedeutung | Einheit | Hinweise |
|----------------|----------|---------|----------|
| nPD_Tiefe | Erreichte minimale Einschraubtiefe (negativer Weg oder mm je nach Bezug) | mm | Wird während der aktiven Phase aktualisiert (kleinster Wert, falls Tiefe Richtung Werkstück negativ zählt) |
| nPD_Drehmoment | Höchstes gemitteltes Drehmoment im Prozess | % (0.1%-Skalierung) oder intern skaliert | Basierend auf Mittelwertbildung zur Glättung |
| nPD_Drehwinkel | Größter errechneter/erfasster Drehwinkel (Differenz seit Auswertungspunkt) | Grad / Umdrehungsäquivalent | Berechnet aus Achsposition und Steigung |
| tPD_Schraubtakt_Komplett | Gesamtzeit von Start bis Abschluss/Fehler | Zeit | Für Taktoptimierung |
| tPD_Schraubtakt_Schrauben | Reine Schraubzeit (ohne z. B. Zylinderwege) | Zeit | Differenziert Ursachen von Verzögerungen |
| iPD_Schraubfall | Klassifizierter Schraubfall (1–3) | Zahl | 0 vor Auswertung |
| bPD_SchraubVorgOk | Gesamt-OK-Flag des Vorgangs | BOOL | FALSE bei Fehler oder Abbruch |
| bPD_DrehmomOk | Soll-Drehmoment erreicht | BOOL | Teilkriterium Qualität |
| bPD_DrehwinkelOk | Nachdreh-/Winkelkriterium erfüllt | BOOL | Verhindert Überdrehen |
| bPD_TiefeOk | Tiefe innerhalb Toleranzfenster | BOOL | Oberflächenbündigkeit / Setzpunkt |

## 7. Parameterbeschreibung (Eingabestruktur – soweit aus Code ersichtlich)
(Hinweis: Nicht alle Parameter der Struktur waren vollständig sichtbar. Liste basiert auf den im Code referenzierten Feldern.)

| Parameter | Bedeutung / Funktion | Einheit / Bereich | Wirkung / Bemerkungen |
|-----------|----------------------|-------------------|-----------------------|
| nDrehzahlSollAnfang | Anfangsdrehzahl für Schon-Anlauf | U/min | Reduziert Risiko von Verkanten |
| nDrehzahlSoll | Hauptdrehzahl (Schnellphase) | U/min | Produktivgeschwindigkeit |
| nDrehzahlSollEnde | Reduzierte Drehzahl in Feinschraubphase | U/min | Feinfühliges Erreichen der Ziele |
| nTiefeSollDrehzahl | Einschraubtiefe, ab der auf Hauptdrehzahl hochgeschaltet wird | mm | Umschaltkriterium Beschleunigung |
| nTiefeLangsamDrehzahl | Einschraubtiefe, ab der wieder verlangsamt wird | mm | Vorbereitung Feinauswertung |
| nMinimalSchraubtiefe | Untere Toleranzgrenze für Tiefe | mm | Qualitätsfenster |
| nSchraubtiefeSoll | Zieltiefe | mm | Nennwert Tiefe |
| nDrehmomentSoll | Ziel-Drehmoment | Skaliert / % / Nm-Äquivalent | Referenz für Freigabekriterium |
| nDrehmomentSkalierVorgabe | Globaler Skalierungsfaktor | Faktor | Multipliziert Ziel-Moment (z. B. Softstart) |
| nDrehmomentSkalVerringern | Temporäre Reduktion für Brems-/Übergangsphase | Faktor | Vor Auswertung wird Moment gesenkt |
| nDrehmomentErhoehung | Zusatzmoment bei Fall 1 / 3 wenn Tiefe „hängt“ | Skaliert | Adaptive Nachregelung |
| nSollmomentLeerlaufPruefung | Momentbegrenzung in Leerlaufprüfung | % / skaliert | Erkennung Schwergang ohne Schäden |
| tLeerlaufPruefzeit | Prüfzeitfenster Leerlauf | Zeit | Timeout-Kriterium |
| tZeitdSchraubTakt | Allgemeines Zeitlimit für Teilphasen / Nachziehfenster | Zeit | Wiederverwendetes Watchdog-Intervall |
| tSchraubfallAuswertung | Spezifisches Zeitfenster für Klassifikation | Zeit | Verhindert endloses Abwarten |
| nHoeheSchraubenDrehung | Steigung (Tiefe pro Umdrehung oder pro Segment) | mm/U | Grundlage für Winkelberechnung |
| nSollDrehwinkelNachTiefeOK | Mindestnachdrehwinkel nach Tiefeerfüllung | Grad | Sicherstellung Setzvorgang |
| nDrehwinkelNachziehen_F1 | Zusatz-Winkelreserve bei Fall 1 | Grad | Verhindert Untervorspannung |
| nMaxDrehwinkelNachTiefeOk_F1 | Sicherheitslimit Fall 1 | Grad | Abbruch bei Überdrehung |
| nMaxDrehwinkelNachTiefeOk_F2 | Sicherheitslimit Fall 2 | Grad | Engeres Fenster |
| (vermutet) nMaxDrehwinkel_F3 | Limit für Fall 3 (im Code angelegt) | Grad | Analog Fall 1/2 (nicht komplett sichtbar) |
| nNennleistungMotor | Referenz Motorstrom / Bemessung | A / % | Für Berechnung Drehmomentgrenze |
| nDrehmomentKonstante | Umrechnungsfaktor (Strom→Moment) | Nm/A | Erforderlich für physikalische Skalierung |

Skalierungsprinzip (vereinfacht):  
Roh-Parameter → (Skalierung / Umrechnungsfaktoren) → Internes Moment-Limit → Ausgabe an Achse.

## 8. Fehlermanagement
Der Baustein nutzt:
- Nummerncodierte Fehlerzustände (negative Schritt-/Fehlerkennungen).
- Watchdog-Zeiten pro kritischer Phase (Anlauf, Übergang, Auswertung, Nachziehfenster, Leerlaufprüfung).
- Separaten Merker für „Fehler-Schritt“ zur späteren Auswertung.
- Quittierlogik: Nach Fehler bleibt Zustand stehen bis quittiert.
- Differenzierte Fehlerarten (Geschwindigkeit nicht erreicht, Tiefe für Umschaltung fehlte, Timeout Auswertung, Überdrehwinkel je Schraubfall, Drehmoment nicht erreicht, Leerlaufprüfung fehlgeschlagen).

Vorteile:
- Schnelle Diagnose: Schritttext + Fehlercode.
- Zielgerichtete Service-Aktion (z. B. Mechanik prüfen vs. Parametrierung).

## 9. Vorteile & Erweiterbarkeit
| Bereich | Vorteil | Mögliche Erweiterung |
|---------|---------|----------------------|
| Modularität | Reiner Motor-Kern austauschbar | Alternative Schraubstrategien (z. B. adaptiv KI-basiert) |
| Qualität | Separate Klar-Ergebnisflags (Tiefe, Moment, Winkel) | Trendanalyse / SPC |
| Sicherheit | Winkel- und Zeitlimits vermeiden Überdrehen | Dynamische Limits abhängig Material |
| Performance | Phasenweises Speed-Management | Automatisches Taktzeit-Tuning |
| Diagnose | Eindeutige Schritt- & Fehlertexte | Web-/HMI-Debug Dashboard |

## 10. Einsatzempfehlungen / Best Practices
- Vor Erstinbetriebnahme: Leerlaufprüfung aktiv auswerten → Mechanische Leichtgängigkeit sicherstellen.
- Parametrierung iterativ: Zuerst konservative Drehmomentskalierung, danach hochtrimmen.
- Winkel-Limits enger setzen, sobald reale Prozess-Streuung bekannt ist.
- Mittelwertbildung des Drehmoments ausreichend dimensionieren (Glättung vs. Reaktionszeit).
- Prozessdaten historisieren → Früherkennung schleichender Verschlechterung (Werkzeugverschleiß, Materialänderungen).
- Unterschiedliche Schraubentypen über Satz gespeicherter Parameter laden (Rezeptverwaltung).

## 11. Hinweise zu Sichtbarkeit / Vollständigkeit
Die hier dokumentierten Parameter und Abläufe basieren auf dem vorliegenden Codeauszug. Einzelne Felder der Parametrierstruktur, insbesondere für Schraubfall 3 (weitere Winkel-/Limit-Parameter), könnten außerhalb des sichtbaren Bereiches definiert sein. Bei Ergänzungen bitte diese Tabelle fortschreiben.

## 12. Kurzglossar
- Setzphase: Bereich, in dem Schraube auflegt und Vorspannung entsteht.
- Nachziehwinkel: Definierter kleiner zusätzlicher Winkel nach Erreichen der Tiefe oder vor endgültigem Stop zur Sicherstellung der Vorspannung.
- Reservewinkel: Berechnete maximal zulässige Drehwinkelspanne bis potenzieller Überdrehung.

## 13. Zusammenfassung
`fb_SKSSchraubNob` liefert eine klar strukturierte, modular eingebettete und qualitätsorientierte Schraubprozess-Steuerung. Er trennt Motorlogik von Peripherie, klassifiziert real auftretende Schraubverläufe und stellt aussagekräftige Prozessdaten für Qualitätssicherung und Optimierung bereit. Dies reduziert Integrationsaufwand, erhöht Transparenz und verbessert Wartbarkeit.

---

Bei Bedarf kann eine ergänzende Parametrierungsrichtlinie oder ein Fehlersuch-Flowchart erstellt werden. Einfach melden.