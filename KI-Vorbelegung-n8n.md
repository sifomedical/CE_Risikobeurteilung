# KI-Vorbelegung: n8n-Spezifikation

Diese Spezifikation beschreibt, wie aus dem Katalog-Datensatz per Claude eine vorbelegte Risikobeurteilung als Entwurf entsteht, die der Experte nur noch bewertet. Sie hält die zweigeteilte Feldlogik ein: Was aus dem Katalog ableitbar ist, wird gefüllt. Was Design-Daten braucht, wird ausgewiesen und nicht erfunden.

## Zwei Klassen von Feldern

**Klasse A, aus Katalog und Kontext ableitbar.** Diese Felder darf die KI füllen: Gefährdungssituation, auslösendes Ereignis, möglicher Schaden, betroffene Personen, Maßnahmenvorschläge über alle drei Stufen, Normbezüge, Prüfhinweise sowie ein erster Entwurf der Schwere (S) und der Exposition (E). Grundlage sind Verwendung, Fehlanwendung, Lebensphasen, Personengruppen und die Gefährdungsart.

**Klasse B, design-abhängig.** Diese Werte hängen an Geometrie, Energie und realer Steuerung, die der Katalog nicht kennt: die konkrete Abstands- und Restenergiebewertung, die Wirksamkeit einer Schutzeinrichtung, der erforderliche PL/SIL, und in vielen Fällen die Eintrittswahrscheinlichkeit (W) und die Vermeidbarkeit (V). Für diese Felder gilt: keine erfundenen Zahlen, keine erfundenen Spaltmaße, kein behaupteter PL-Wert. Die KI setzt `designReq: "ja"`, benennt in `designAspects`, welche Design-Daten fehlen, senkt die Konfidenz und hält S/E/W/V nur als klar gekennzeichnete Annahme oder lässt sie offen.

Diese Trennung ist der Grund, warum das Ergebnis verkaufbar und haftungssicher bleibt: Der Experte urteilt, und die design-kritischen Lücken sind sichtbar statt still gefüllt.

## Eingabe

Der Katalog liefert den Maschinen-Datensatz (Schema siehe README). Für die KI-Vorbelegung sind je Zeile bereits `family`, `hazard`, `persons` und ein Normhinweis gesetzt, dazu der gesamte `header` als Kontext. Die KI reichert jede Zeile an.

## Ausgabe je Zeile

Die KI gibt exakt das Zeilenschema des Tools zurück, ergänzt um die Freigabe-Felder:

```json
{
  "lifecycle": "", "family": "", "hazard": "", "situation": "",
  "event": "", "harm": "", "persons": "",
  "S": 3, "E": 3, "W": null, "V": null,
  "plrOverride": "",
  "m1": "", "m2": "", "m3": "",
  "S2": null, "E2": null, "W2": null, "V2": null,
  "residual": "", "residualWhy": "",
  "norms": "", "verify": "",
  "status": "ki",
  "confidence": 72,
  "derivation": "Warum diese Situation, dieser Schaden, diese S/E-Einstufung.",
  "designReq": "ja",
  "designAspects": "Spaltmaß und Schutztürhöhe für Abstandsbewertung nach EN ISO 13857; PLr der Türverriegelung."
}
```

`status` ist immer `"ki"`. `confidence` ist 0 bis 100. `residual` nimmt `akzeptabel`, `bedingt` oder `nicht`. Fehlen belastbare Grundlagen, bleibt ein Feld leer oder `null`, statt geraten zu werden.

## System-Prompt für den Anthropic-Node

Temperatur 0.1. Modell: aktuelles Claude-Modell mit ausreichend Kontext. Ausgabe strikt JSON, kein Fließtext, keine Markdown-Zäune.

```
Du bist Sicherheitsingenieur für Maschinen und erstellst den ENTWURF einer
Risikobeurteilung nach EN ISO 12100 im Rahmen der Verordnung (EU) 2023/1230.
Du erhältst den Maschinenkontext (header) und eine Liste identifizierter
Gefährdungen (rows) mit family, hazard und persons.

Für jede Zeile leitest du nachvollziehbar ab und füllst NUR, was aus dem
gegebenen Kontext seriös ableitbar ist:
- situation: die konkrete Gefährdungssituation als Zusammentreffen von Person,
  Gefährdung und Aufgabe in einer Lebensphase. Nutze header.intendedUse,
  header.misuse und plausible Lebensphasen.
- event: das auslösende Ereignis.
- harm: der mögliche Schaden, konsistent zur Gefährdungsart (Hazard ->
  Hazardous Situation -> Harm).
- S (Schwere) und E (Exposition): Entwurf mit Begründung.
- m1/m2/m3: Maßnahmen streng in der 3-Stufen-Hierarchie. m1 zuerst inhärent
  sichere Konstruktion, dann m2 technische Schutzmaßnahmen, dann m3
  Benutzerinformation. Schlage nur einschlägige, typische Maßnahmen vor.
- norms: verfeinere den vorgegebenen Normhinweis auf die einschlägige B-/C-Norm,
  wenn eindeutig. Sonst belasse den Hinweis und markiere ihn als zu prüfen.
- verify: Prüf-/Verifikationshinweis.

Design-abhängige Werte NICHT erfinden. Wenn ein Wert von Geometrie, Restenergie,
Schutzeinrichtungs-Wirksamkeit oder realer Steuerung abhängt (typisch W, V, der
konkrete Sicherheitsabstand, die Restenergie und der PL/SIL):
- setze designReq = "ja",
- benenne in designAspects genau die fehlenden Design-Daten,
- setze W und V nur als ausdrücklich gekennzeichnete Annahme oder auf null,
- nenne KEINE konkreten Millimeter-, Joule- oder PL-Werte als Tatsache.

Erfinde keine Fakten. Nenne keine Normnummer, deren Einschlägigkeit du nicht
begründen kannst. Wenn unsicher, senke confidence und schreibe die Unsicherheit
in derivation.

confidence-Bänder:
- 80 bis 100: Standardgefährdung, Situation und Maßnahmen eindeutig aus Kontext.
- 50 bis 79: plausibel, aber design- oder auslegungsabhängig.
- unter 50: stark annahmebehaftet, Experte muss neu bewerten.

status ist immer "ki". Gib ausschließlich ein JSON-Objekt der Form
{"rows":[ ... ]} zurück, mit denselben Zeilen in derselben Reihenfolge,
angereichert. Keine weiteren Felder, kein Text außerhalb des JSON.
```

Als User-Message übergibst du den Maschinen-Datensatz als JSON. Optional davor eine kurze Zeile: „Maschinenkontext und Gefährdungen folgen als JSON."

## n8n-Ablauf

1. **Trigger.** Webhook oder manueller Start. Eingang ist der Katalog-Datensatz (JSON).
2. **Prompt bauen.** Ein Set/Function-Node legt system und user zusammen. user enthält `{"header": ..., "rows": [...]}`.
3. **Anthropic-Node.** Modellaufruf mit Temperatur 0.1, System-Prompt oben, ausreichend max tokens für alle Zeilen. Falls viele Zeilen, in Blöcken von etwa 8 bis 12 Zeilen aufrufen und danach zusammenführen.
4. **Parsen und mergen.** Ein Function-Node parst die JSON-Antwort, mappt jede zurückgegebene Zeile auf die Ursprungszeile (Reihenfolge oder ein mitgeschicktes Index-Feld) und schreibt die angereicherten Felder in den Maschinen-Datensatz. `status` bleibt `"ki"`.
5. **Validieren.** Prüfe, dass `residual` nur erlaubte Werte hat, `S/E/W/V` Zahl oder null sind, `confidence` im Bereich 0 bis 100 liegt. Zeilen ohne diese Form werden markiert, nicht stillschweigend übernommen.
6. **Zurückgeben.** Ausgabe ist der fertige Maschinen-Datensatz als JSON. Übergabe an das Tool per JSON-Import (Import-Knopf) oder, bei Web-Einbettung, per Ablage unter dem localStorage-Schlüssel `sifo_ra_handoff` und Weiterleitung auf `risikobeurteilung.html`.

## Beispiel

Eingabe-Zeile (aus dem Katalog):

```json
{ "family": "Mechanisch", "hazard": "Quetschen", "persons": "Bediener, Einrichter",
  "norms": "EN ISO 13857 / EN ISO 13854", "verify": "Gefahrstellen frei zugänglich: ja" }
```

Angereicherte Ausgabe (Entwurf):

```json
{
  "lifecycle": "Rüsten", "family": "Mechanisch", "hazard": "Quetschen",
  "situation": "Beim Rüsten greift der Einrichter in den Bewegungsbereich des Schließwerkzeugs, während Restbewegung möglich ist.",
  "event": "Unerwarteter Anlauf oder Nachlauf bei geöffneter Schutzstellung.",
  "harm": "Quetschung der Hand oder Finger.",
  "persons": "Bediener, Einrichter",
  "S": 3, "E": 2, "W": null, "V": null, "plrOverride": "",
  "m1": "Bewegungsbereich konstruktiv reduzieren, Quetschstellen durch Abstände oder Kraftbegrenzung vermeiden.",
  "m2": "Trennende Schutzeinrichtung mit Verriegelung, sichere Stillstandsüberwachung im Rüstbetrieb.",
  "m3": "Rüstanweisung, Warnkennzeichnung, Zustimmtaster-Betrieb dokumentieren.",
  "S2": null, "E2": null, "W2": null, "V2": null,
  "residual": "", "residualWhy": "",
  "norms": "EN ISO 13857, EN ISO 13854; Steuerung nach EN ISO 13849-1 prüfen.",
  "verify": "Abstands- und Nachlaufmessung, Wirksamkeit der Verriegelung prüfen.",
  "status": "ki", "confidence": 68,
  "derivation": "Schwere aus Quetschung an Werkzeug abgeleitet, Exposition aus Rüsttätigkeit. W und V hängen von Nachlaufzeit, Spaltmaß und Steuerungsauslegung ab und sind ohne Design nicht seriös schätzbar.",
  "designReq": "ja",
  "designAspects": "Nachlaufweg und -zeit, Spaltmaß der Gefahrstelle, Höhe und Abstand der Schutzeinrichtung nach EN ISO 13857, erreichter PL der Verriegelung."
}
```

## Compliance-Vorbehalt

Die von der KI vorgeschlagenen Normzuordnungen sind die Stelle mit dem höchsten Halluzinationsrisiko. Sie gehören vom Maschinen-Fachmann gegen den Primärtext geprüft, bevor eine Zeile auf „geprüft" gesetzt wird. Kein Entwurf ersetzt die fachkundige Freigabe.
