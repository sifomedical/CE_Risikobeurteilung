# CE Risikobeurteilung

Zwei-Schritt-Werkzeug für die dokumentierte Risikobeurteilung nach **EN ISO 12100** im Rahmen der **Verordnung (EU) 2023/1230**. Statische Webseiten, keine Server, keine externen Aufrufe. Speicherung läuft im Browser.

## Ablauf

**Schritt 1 — Katalog (`index.html`).** Der strukturierte Fragenkatalog ist der Eingang. Er erfasst die Grenzen der Maschine (Verwendung, Fehlanwendung, Bediener, Lebensphasen, Personengruppen), die Einstufung nach 2023/1230 und die vorhandenen Gefährdungsfamilien inklusive der neuen Themen Cybersecurity/Korrumpierung und autonomes Verhalten. Der Button **„Zur Risikobeurteilung →"** erzeugt aus den Antworten einen Übergabe-Datensatz und öffnet Schritt 2.

**Schritt 2 — Risikobeurteilung (`risikobeurteilung.html`).** Editierbare Tabelle mit einer Zeile je Gefährdung. Aus dem Katalog werden Maschinensteckbrief und die Gefährdungszeilen als **Entwurf** vorbelegt: Gefährdungsfamilie, konkrete Gefährdung, betroffene Personen und ein Normhinweis stehen bereits, S/E/W/V und Maßnahmen werden hier ergänzt. Der Risiko-Index rechnet transparent nach `Index = S × (E + W + V)` und schlägt einen PLr nach EN ISO 13849-1 vor. Beides ist je Zelle überschreibbar. Export als CSV und JSON, plus druckfertige Deliverable-Ansicht (A4 quer, PDF).

## Übergabe zwischen den Seiten

Der Katalog schreibt den Datensatz unter dem Schlüssel `sifo_ra_handoff` in den `localStorage` und leitet weiter. Die Risikobeurteilung liest den Schlüssel beim Laden aus, legt eine neue Maschine an und löscht den Schlüssel wieder. Ist `localStorage` nicht verfügbar (z. B. in einer Sandbox-Vorschau), lädt der Katalog den Datensatz stattdessen als JSON herunter, das sich über **Import** manuell einspielen lässt.

## Datenschema (Maschine)

Der Übergabe- und Import-Datensatz ist ein JSON-Objekt. Das ist zugleich der Vertrag für den n8n-Workflow, falls dieser Zeilen automatisch anreichern soll.

```json
{
  "name": "Maschinenname",
  "header": {
    "company": "", "machineName": "", "machineType": "", "drawingNo": "",
    "intendedUse": "", "misuse": "", "operators": "",
    "classification": "", "annexI": "", "assessor": "", "date": "", "revision": ""
  },
  "rows": [
    {
      "lifecycle": "", "family": "", "hazard": "", "situation": "",
      "event": "", "harm": "", "persons": "",
      "S": null, "E": null, "W": null, "V": null,
      "idxBeforeOverride": "", "plrOverride": "",
      "m1": "", "m2": "", "m3": "",
      "S2": null, "E2": null, "W2": null, "V2": null,
      "idxAfterOverride": "",
      "residual": "", "residualWhy": "", "norms": "", "verify": ""
    }
  ],
  "summary": { "residualSummary": "", "openPoints": "" }
}
```

Fehlende Felder je Zeile werden beim Import mit Standardwerten aufgefüllt. `S/E/W/V` sind ganze Zahlen oder `null`. `residual` nimmt `akzeptabel`, `bedingt` oder `nicht`.

## Bewertungsschlüssel

- **S** Schwere: 1 leicht, 2 mittel, 3 schwer, 4 tödlich
- **E** Exposition/Häufigkeit: 1 selten … 4 ständig
- **W** Eintrittswahrscheinlichkeit: 1 unwahrscheinlich … 4 sehr wahrscheinlich
- **V** Vermeidbarkeit: 1 möglich, 2 bedingt, 3 kaum
- **Index** = `S × (E + W + V)`, Klassen: gering ≤6, mittel 7–14, hoch 15–26, sehr hoch ≥27
- **PLr-Vorschlag** über den Risikograph nach EN ISO 13849-1 Anhang A (S1/S2, F1/F2, P1/P2 → a…e)

Die Schwellen und die PLr-Zuordnung liegen in `risikobeurteilung.html` und lassen sich an die validierte Bewertungsmatrix anpassen.

## Deployment

Die Dateien sind statisch. Für GitHub Pages im Repo unter *Settings → Pages* den Branch wählen, die Startseite ist `index.html`. Ebenso lauffähig auf dem Pimcore-Host oder lokal per Doppelklick.

## Scope

Das Werkzeug erzeugt die dokumentierte Risikobeurteilung als **Entwurf** und erfüllt damit den Risikobeurteilungs-Baustein der Verordnung (EU) 2023/1230. Es ist expertengeprüft freizugeben und ersetzt weder die fachkundige Bewertung noch das Konformitätsbewertungsverfahren, die technische Dokumentation, die EU-Konformitätserklärung, die Betriebsanleitung oder die CE-Kennzeichnung. Der Index ist ein Priorisierungsvorschlag, nicht das Urteil über ausreichende Risikominderung. Anhang-Zuordnungen und Normstellen sind vor Verwendung am amtlichen Verordnungstext zu prüfen.
