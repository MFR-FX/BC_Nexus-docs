---
title: "BC Nexus — Beispielaufrufe für den veröffentlichten Webservice"
permalink: /help/examples/
---

# BC Nexus — Beispielaufrufe für den veröffentlichten Webservice

Diese Beispiele belegen von außen, dass der BC-Nexus-Webservice erreichbar ist und Daten
tatsächlich in Business Central ankommen. Sie sind bewusst gegen die mitgelieferten
Beispieldaten geschnitten (Schnittstelle `CURRENCYIMPORT` auf der Standardtabelle *Currency*),
damit für den Test kein eigenes Datenmodell aufgebaut werden muss.

Alle Beispiele arbeiten mit den ISO-4217-Platzhaltercodes `XTS`, `XTA`, `XTB` und `XTC`. Diese
Codes sind keiner realen Währung zugeordnet — ein Testlauf überschreibt damit keinen echten
Währungssatz.

---

## 1. Voraussetzungen

| # | Voraussetzung | Wo eingerichtet |
|---|---|---|
| 1 | BC Nexus ist installiert und **Allow HttpClient Requests** ist aktiv | Extension Management |
| 2 | Codeunit **73710475** ist als Webservice mit dem Service-Namen `FXNINexusWebservice` veröffentlicht | Seite **Web Services**, siehe [Setup Guide]({{ site.baseurl }}/help/setup-guide/) Abschnitt 2 |
| 3 | Beispieldaten sind geladen (Endpunkt `DEMO`, Schnittstelle `CURRENCYIMPORT` mit drei Feldzuordnungen) | Aktion **Beispieldaten laden** auf der Seite **BC Nexus Setup** |
| 4 | Nur bei Beispieldaten aus einer Version vor 27.2.0.0: die beiden korrigierten Feldnummern nachziehen — siehe Abschnitt 2 | Seite **Feldzuordnung** der Schnittstelle `CURRENCYIMPORT` |
| 5 | Der aufrufende technische Benutzer hat den Berechtigungssatz `FXNI Nexus Integr.` **und** einen Satz mit Lese-/Schreibrecht auf die Zieltabelle (z. B. `D365 BUS FULL ACCESS`) — `FXNI Nexus Integr.` deckt keine `tabledata`-Rechte auf Zieltabellen ab | Benutzerberechtigungen |
| 6 | Eine Entra-App-Registrierung mit Client-Credentials und erteilter Business-Central-Berechtigung existiert | siehe [Setup Guide]({{ site.baseurl }}/help/setup-guide/) Abschnitt 3 |
| 7 | Für Abschnitt 8 (Publish): eine Schnittstelle vom Typ `Publish` ist angelegt | Seite **BC Nexus Setup**, Unterliste Schnittstellendefinitionen |

Die Einrichtung der Entra-App (App-Registrierung, Client Secret, Anwendungsberechtigung,
Zuordnung des Anwendungsbenutzers in Business Central) ist **nicht** Teil dieses Dokuments; ein
kurzer Hinweis dazu steht im [Setup Guide]({{ site.baseurl }}/help/setup-guide/) (Abschnitt 3) — die Entra-Einrichtung
selbst bleibt eine allgemeine Business-Central-/Azure-Aufgabe.

---

## 2. Beispieldaten und die Feldnummern der Zuordnung

Ab Version 27.2.0.0 legt die Aktion **Beispieldaten laden** die drei Feldzuordnungen von
`CURRENCYIMPORT` korrekt an: `code` auf Feldnr. 1, `description` auf Feldnr. 15 und `isoCode`
auf Feldnr. 4 der Standardtabelle *Currency* — bis 27.1.2.0 zeigten die letzten beiden auf
Feldnr. 2 (*Last Date Modified*, Datum) und 9 (*Realized Losses Acc.*), was jeden Beispiellauf
mit einem Konvertierungsfehler beendete; wer die Beispieldaten unter einer älteren Version
geladen hat, korrigiert die beiden Feldnummern einmalig auf der Seite **Feldzuordnung**.

---

## 3. Aufrufform (OData V4)

Ein als Webservice veröffentlichtes Codeunit stellt seine öffentlichen Prozeduren als
**unbound actions** bereit. Der Aktionsname ist `{ServiceName}_{ProcedureName}`:

```
POST https://api.businesscentral.dynamics.com/v2.0/{tenantId}/{environmentName}/ODataV4/FXNINexusWebservice_{ProcedureName}?company={companyName}
Authorization: Bearer {token}
Content-Type: application/json
```

- `{companyName}` ist der Mandantenname, URL-kodiert — aus `Meine Firma` wird
  `Meine%20Firma`. Alternativ akzeptiert BC die Mandanten-GUID: `?company={companyId}`.
- Gleichwertig ist die Pfadform mit Mandantensegment, die auch
  [API Reference]({{ site.baseurl }}/help/api-reference/) als Basis-URL nennt:
  `.../ODataV4/Company('{companyName}')/FXNINexusWebservice_{ProcedureName}`.
- Der Body ist ein JSON-Objekt, dessen Schlüssel **exakt** den AL-Parameternamen entsprechen —
  inklusive Groß-/Kleinschreibung. Für `Receive` sind das `InterfaceCode` und `RequestData`.
- Die Antwort ist immer `{"@odata.context": "...", "value": <Rückgabewert>}`.

> **Vor dem ersten Aufruf prüfen.** Rufen Sie einmal
> `GET .../ODataV4/$metadata` ab und suchen Sie das Element
> `<Action Name="FXNINexusWebservice_Receive">`. Dort stehen die Parameternamen so, wie der
> Server sie erwartet. Das ist die einzige verlässliche Quelle, falls ein Aufruf mit
> *„The parameter … is not defined"* abgewiesen wird.

### Prozeduren und ihre Bodys

| Prozedur | AL-Signatur | Body | `value` in der Antwort |
|---|---|---|---|
| `Receive` | `Receive(InterfaceCode: Code[20]; RequestData: Text): Boolean` | `{"interfaceCode": "...", "requestData": "..."}` | `true` |
| `ReceiveAndGetTransactionEntryNo` | `ReceiveAndGetTransactionEntryNo(InterfaceCode: Code[20]; RequestData: Text; var TransactionEntryNo: Integer): Boolean` | wie `Receive`, liefert zusätzlich die Transaktions-Eintragsnummer, aber ohne die Metadatenfelder von `ReceiveWithMetadata` | `true`, Form mit `var`-Parameter ungeprüft — siehe Hinweis unten |
| `ReceiveWithMetadata` | `ReceiveWithMetadata(InterfaceCode: Code[20]; RequestData: Text; CorrelationId: Text[100]; IdempotencyKey: Text[100]; ExternalMessageId: Text[100]; ContractVersion: Code[20]; var TransactionEntryNo: Integer): Boolean` | alle sechs Eingabeparameter | `true`, Form mit `var`-Parameter ungeprüft — siehe Hinweis unten |
| `ReceiveAttachment` | `ReceiveAttachment(InterfaceCode: Code[20]; RecordKey: Text; AttachmentName: Text; ContentType: Text; FileExtension: Text; Base64Content: Text): Boolean` | sechs Textparameter | `true` |
| `Send` | `Send(InterfaceCode: Code[20]): Text` | `{"interfaceCode": "..."}` | **String** mit dem Antwortkörper des Endpunkts |
| `SendWithOptions` | `SendWithOptions(InterfaceCode: Code[20]; RequestData: Text): Text` | `{"interfaceCode": "...", "requestData": "{…Abrufoptionen…}"}` | **String** mit dem Antwortkörper des Endpunkts, genau wie `Send` |
| `Publish` | `Publish(InterfaceCode: Code[20]; RequestData: Text): Text` | `{"interfaceCode": "...", "requestData": ""}` | **String**, der das Antwort-Envelope enthält — siehe Abschnitt 8 |

**Zwei Fallen, die jeden Client betreffen:**

1. **`RequestData` ist ein JSON-*String*, kein JSON-Objekt.** Die Nutzlast muss escaped im
   String stehen. Deshalb liegt jedes Beispiel doppelt vor: `*.payload.json` ist die rohe
   Nutzlast, `*.json` derselbe Inhalt als fertiger OData-Body mit escapetem `RequestData`.
2. **`Send` und `Publish` liefern `Text`.** `value` ist damit eine Zeichenkette, die ihrerseits
   JSON enthält — sie muss ein zweites Mal geparst werden
   (`$antwort.value | ConvertFrom-Json`).

**`ReceiveWithMetadata` und der `var`-Parameter.** `TransactionEntryNo` ist in AL ein
`var`-Parameter. Ob OData V4 ihn im Rückgabewert mitgibt, ist gegen einen laufenden Mandanten
nicht geprüft. Prüfen Sie das `$metadata` der Aktion: taucht `TransactionEntryNo` dort nicht im
`ReturnType` auf, kommt die Nummer nicht über OData zurück. Die Transaktion finden Sie dann in
der Transaktionsliste über die mitgesendete `CorrelationId`. Für reine Erreichbarkeitstests ist
`Receive` die einfachere Wahl.

---

## 4. Die Beispieldateien

Die Beispieldateien lassen sich über die Links in der folgenden Tabelle herunterladen.

| Datei | Zweck | Erwartetes Ergebnis |
|---|---|---|
| [`receive-currency-single.json`](receive-currency-single.json) | OData-Body für `Receive`, ein Währungssatz | HTTP 200, `{"value": true}`. Nach der Verarbeitung existiert Währung `XTS` mit Beschreibung *Testwaehrung XTS* und ISO-Code `XTS`. |
| [`receive-currency-single.payload.json`](receive-currency-single.payload.json) | Dieselbe Nutzlast unescaped | Zum Einfügen in die Aktion **Verarbeitungstest** auf der Schnittstellendefinition |
| [`receive-currency-array.json`](receive-currency-array.json) | OData-Body für `Receive`, JSON-Array mit drei Sätzen | HTTP 200, `{"value": true}`. Nach der Verarbeitung existieren `XTA`, `XTB` und `XTC`. |
| [`receive-currency-array.payload.json`](receive-currency-array.payload.json) | Dasselbe Array unescaped | Verarbeitungstest |
| [`receive-currency-update.payload.json`](receive-currency-update.payload.json) | Nutzlast mit demselben Code `XTS`, aber neuer Beschreibung | Es entsteht **kein** zweiter Satz. Die Beschreibung von `XTS` wechselt auf *Testwaehrung XTS (Update)* — Beleg für Update über den Primärschlüssel. |
| [`receive-currency-missing-mandatory.payload.json`](receive-currency-missing-mandatory.payload.json) | Nutzlast ohne den Pflichtschlüssel `code` | Die Transaktion muss mit *„Mandatory field 'code' (Code) is missing or empty in the incoming data."* abgewiesen werden und auf Status **Fehler** landen. **Der HTTP-Aufruf selbst liefert trotzdem 200** — siehe Abschnitt 5. |
| [`receive-currency.csv`](receive-currency.csv) | CSV-Nutzlast: Semikolon, Kopfzeile, drei Zeilen, ein gequoteter Wert mit Semikolon und doppeltem Anführungszeichen | Nur mit einer CSV-Schnittstelle verwendbar — siehe Abschnitt 7 |
| [`receive-currency-csv.json`](receive-currency-csv.json) | Dieselbe CSV als OData-Body, Zeilenumbrüche als `\n` escaped | wie oben |
| [`publish-request.json`](publish-request.json) | Body für `Publish` ohne Abrufoptionen | `value` ist ein String mit dem Antwort-Envelope über allen Währungssätzen — siehe Abschnitt 8 |
| [`publish-currency-filter.json`](publish-currency-filter.json) | Body für `Publish` mit einer Filterbedingung und `includeCount` | Nur die vier Testwährungen, `count` = 4 |
| [`publish-currency-filter.payload.json`](publish-currency-filter.payload.json) | Dieselben Abrufoptionen unescaped | Zum Einfügen in die Aktion **Verarbeitungstest** |
| [`publish-currency-filter.response.json`](publish-currency-filter.response.json) | Das erwartete Antwort-Envelope zu diesem Aufruf | Vergleichswert, keine Eingabedatei |
| [`publish-currency-page.json`](publish-currency-page.json) | Derselbe Filter, aber Seitengröße 2 | Erste Seite mit zwei Sätzen, `hasMore` = true, `nextPageToken` gefüllt |
| [`publish-currency-page.payload.json`](publish-currency-page.payload.json) | Dieselben Abrufoptionen unescaped | Verarbeitungstest |
| [`publish-currency-page.response.json`](publish-currency-page.response.json) | Das erwartete Antwort-Envelope der ersten Seite | Vergleichswert. Das Token darin ist ein Platzhalter |
| [`publish-currency-page2.json`](publish-currency-page2.json) | Zweite Seite: derselbe Filter, dieselbe Größe, dazu `page.token` | `hasMore` = false, `nextPageToken` = null. **Token vorher aus der echten Antwort ersetzen** |
| [`publish-currency-unknown-key.payload.json`](publish-currency-unknown-key.payload.json) | Abrufoptionen mit einem JSON-Schlüssel, den die Feldzuordnung nicht kennt | Muss mit *„This interface cannot be filtered on 'symbol'."* fehlschlagen — kein leeres Ergebnis |
| [`send-currency-options.json`](send-currency-options.json) | Body für `SendWithOptions` | Verlangt eine eigene Schnittstelle vom Typ `Senden` — siehe Abschnitt 8 |
| [`send-currency-options.payload.json`](send-currency-options.payload.json) | Dieselben Abrufoptionen unescaped | Verarbeitungstest |
| [`send-currency-bare.json`](send-currency-bare.json) | Body für `SendWithOptions` mit `"envelope": false` | Payload/Antwort ist das nackte Array statt des Envelope-Objekts — siehe Abschnitt 8 |

Die Reihenfolge für einen vollständigen Durchlauf: erst `single`, dann `array`, dann `update`,
zuletzt `missing-mandatory`. Es genügt, die Aufrufe in dieser Reihenfolge abzusetzen — die Job
Queue arbeitet die offenen Transaktionen in Entry-No.-Reihenfolge ab, sodass `update` auch dann
auf den zuvor angelegten Satz `XTS` trifft, wenn beide Aufrufe unmittelbar nacheinander erfolgen.

---

## 5. Wichtig: `Receive` verarbeitet nicht, `Receive` reiht ein

`Receive` legt einen Transaktionssatz mit Status **Offen** an, speichert die Nutzlast und kehrt
sofort zurück. Die eigentliche Verarbeitung übernimmt die Job Queue (Codeunit 73710476), die einmal
pro Minute alle offenen Transaktionen abarbeitet.

Daraus folgt für jeden externen Aufrufer:

- **HTTP 200 heißt nur „angenommen", nicht „verarbeitet".** Auch die Nutzlast ohne Pflichtfeld
  wird mit `{"value": true}` quittiert. Der Fehler entsteht erst bei der Verarbeitung und ist
  ausschließlich in der Transaktionsliste sichtbar.
- **Synchron abgewiesen wird nur, was schon der Webservice prüft:** Schnittstelle existiert
  nicht (*„Interface definition '…' does not exist."*), Schnittstelle ist gesperrt
  (*„Interface '…' is blocked and cannot process requests."*), oder der Typ passt nicht
  (*„Interface '…' is not of type 'Receive'."*). Diese drei Fälle kommen als HTTP-Fehler zurück.
- **Ist *Transakt. automatisch verarbeiten* aus**, bleibt jede Transaktion auf **Offen** stehen, bis
  jemand in der Transaktionsliste **Manuell verarbeiten** wählt. Für einen Testlauf ist das die
  kontrollierbarere Variante.
- **Ist *Transakt. automatisch verarbeiten* an**, ist das Ergebnis nach spätestens einer Minute in der
  Transaktionsliste sichtbar. Ein Fehlschlag wird gemäß *Anzahl Wiederholungen pro Transaktion*
  (Standard 3) im Minutentakt wiederholt; erst danach steht der Status endgültig auf **Fehler**.
  Rechnen Sie beim Pflichtfeld-Beispiel also mit rund drei Minuten bis zum Endzustand.

`Send` und `Publish` verhalten sich umgekehrt: sie verarbeiten synchron im Vordergrund und geben
das Ergebnis direkt zurück. Ein Fehler dort ist sofort ein HTTP-Fehler.

Es gibt in dieser Version **keine API-Seite** für die Transaktionsliste. Der Status lässt sich
von außen nicht abfragen — die Prüfung nach Abschnitt 9 erfolgt im Client.

---

## 6. curl-Beispiele

Token holen (Client Credentials, Scope `https://api.businesscentral.dynamics.com/.default`):

```bash
curl -s -X POST \
  "https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "client_id={clientId}" \
  --data-urlencode "client_secret={clientSecret}" \
  --data-urlencode "scope=https://api.businesscentral.dynamics.com/.default" \
  --data-urlencode "grant_type=client_credentials"
```

Einzelnen Währungssatz senden:

```bash
curl -i -X POST \
  "https://api.businesscentral.dynamics.com/v2.0/{tenantId}/{environmentName}/ODataV4/FXNINexusWebservice_Receive?company={companyName}" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  --data-binary @docs/examples/receive-currency-single.json
```

Erwartete Antwort:

```json
{"@odata.context":"...","value":true}
```

Array senden:

```bash
curl -i -X POST \
  "https://api.businesscentral.dynamics.com/v2.0/{tenantId}/{environmentName}/ODataV4/FXNINexusWebservice_Receive?company={companyName}" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  --data-binary @docs/examples/receive-currency-array.json
```

Update und Pflichtfeld-Fall liegen nur als rohe Nutzlast vor. Der OData-Body wird daraus
gebaut, indem die Nutzlast als String in `RequestData` gesetzt wird — mit `jq`:

```bash
jq -Rs --arg ic CURRENCYIMPORT '{InterfaceCode: $ic, RequestData: .}' \
  docs/examples/receive-currency-update.payload.json
```

Mit Metadaten senden (Korrelation und Idempotenz — ein zweiter Aufruf mit demselben
`IdempotencyKey` und derselben Schnittstelle legt keine zweite Transaktion an):

```bash
curl -i -X POST \
  "https://api.businesscentral.dynamics.com/v2.0/{tenantId}/{environmentName}/ODataV4/FXNINexusWebservice_ReceiveWithMetadata?company={companyName}" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
        "interfaceCode": "CURRENCYIMPORT",
        "requestData": "{\"code\":\"XTS\",\"description\":\"Testwaehrung XTS\",\"isoCode\":\"XTS\"}",
        "CorrelationId": "smoke-2026-09-04-001",
        "IdempotencyKey": "smoke-2026-09-04-001",
        "ExternalMessageId": "",
        "ContractVersion": ""
      }'
```

Der siebte AL-Parameter `TransactionEntryNo` fehlt in diesem Body bewusst — er ist ein
`var`-Parameter und damit ein Rückgabekanal, kein Eingabewert. Weist BC den Aufruf mit einem
fehlenden Parameter zurück, ergänzen Sie `"TransactionEntryNo": 0`; welche der beiden Varianten
gilt, steht im `$metadata` der Aktion (siehe Hinweis in Abschnitt 3).

Publish abrufen:

```bash
curl -s -X POST \
  "https://api.businesscentral.dynamics.com/v2.0/{tenantId}/{environmentName}/ODataV4/FXNINexusWebservice_Publish?company={companyName}" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  --data-binary @docs/examples/publish-request.json | jq -r .value | jq .
```

Das doppelte `jq` ist kein Tippfehler: das erste holt den String aus `value`, das zweite parst
das darin enthaltene JSON-Array.

---

## 7. CSV-Beispiel

Die Beispielschnittstelle `CURRENCYIMPORT` steht auf **Feldzuordnungstyp** = `JSON`. Die
CSV-Dateien lassen sich damit nicht verwenden. Stellen Sie `CURRENCYIMPORT` **nicht** um — die
JSON-Beispiele wären danach unbrauchbar. Legen Sie stattdessen eine zweite Schnittstelle an:

| Feld | Wert |
|---|---|
| Code | `CURRENCYCSV` |
| Beschreibung | `Currency Import CSV (Beispiel)` |
| Tabellennr. | `4` (Currency) |
| Schnittstellentyp | `Receive` |
| Feldzuordnungstyp | `CSV` |
| CSV-Trennzeichen | `Semicolon` |
| CSV mit Kopfzeile | aktiv |
| Immer neue Einträge erstellen | inaktiv |

Feldzuordnung — bei CSV zählt **ausschließlich die Position**, der **JSON-Schlüssel** wird nicht
als Spaltenüberschrift ausgewertet. Die Kopfzeile der Datei dient nur der Lesbarkeit und wird
verworfen:

| Position | Feldnr. | Feldname | Pflichtfeld |
|---|---|---|---|
| 1 | 1 | Code | ja |
| 2 | 15 | Description | nein |
| 3 | 4 | ISO Code | nein |

Der Inhalt von [`receive-currency.csv`](receive-currency.csv):

```
code;description;isoCode
XTA;Testwaehrung Alpha;XTA
XTB;"Test B; Variante ""Nord""";XTB
XTC;Testwaehrung Charlie;XTC
```

Zeile 3 ist der eigentliche Prüfpunkt. Der Wert ist in Anführungszeichen eingeschlossen, das
Semikolon darin trennt deshalb keine Spalte, und `""` steht für ein einzelnes literales
Anführungszeichen. Nach dem Import muss die Beschreibung von `XTB` genau
`Test B; Variante "Nord"` lauten.

[`receive-currency-csv.json`](receive-currency-csv.json) ist derselbe Inhalt als OData-Body mit `\n` als Zeilenumbruch. Die
Datei zeigt auf `CURRENCYCSV` — passen Sie `InterfaceCode` an, falls Sie die Schnittstelle
anders benannt haben.

Zwei Eigenschaften des CSV-Pfads, die im Test auffallen:

- **Eine fehlerhafte Zeile bricht die gesamte Transaktion ab.** Es wird nichts geschrieben, und
  die Fehlermeldung nennt die 1-basierte Quellzeilennummer inklusive Kopf- und Leerzeilen.
- Zahlen und Datumswerte müssen im invarianten Format vorliegen (`1234.56`, `2026-08-19`). Ein
  UTF-8-BOM am Anfang der Nutzlast wird entfernt, bevor geparst wird, und landet damit nicht mehr
  in der ersten Spalte. Entfernt wird nur eine Markierung ganz am Anfang; an jeder anderen Stelle
  ist sie Nutzdaten.

---

## 8. Publish

`Publish` braucht eine eigene Schnittstelle vom Typ `Publish`; die Beispieldaten liefern keine.
Die Verarbeitung verlangt außerdem einen hinterlegten **Endpunktcode**, obwohl sie den Endpunkt
für Publish nie aufruft — der mitgelieferte Endpunkt `DEMO` genügt dafür.

| Feld | Wert |
|---|---|
| Code | `CURRENCYPUBLISH` |
| Beschreibung | `Currency Publish (Beispiel)` |
| Tabellennr. | `4` (Currency) |
| Schnittstellentyp | `Publish` |
| Feldzuordnungstyp | `JSON` |
| Endpunktcode | `DEMO` |

Feldzuordnung wie bei `CURRENCYIMPORT` (Position 1/2/3 → Feldnr. 1/15/4, JSON-Schlüssel `code`,
`description`, `isoCode`). Die Aktion **Aus Tabelle initialisieren** legt die Zeilen an, danach
löschen Sie alles außer den drei benötigten.

`"requestData": ""` in [`publish-request.json`](publish-request.json) bedeutet „keine Abrufoptionen" und liefert damit
denselben Datenbestand wie bisher: die Zieltabelle, eingeschränkt nur durch die an der
Schnittstelle hinterlegten Filterzeilen. Die Antwort enthält also **alle** Währungssätze des
Mandanten, nicht nur die vier Testcodes.

Antwortform — `value` ist ein String, dessen Inhalt so aussieht:

```json
{
  "value": [
    { "code": "EUR", "description": "Euro", "isoCode": "EUR" },
    { "code": "XTA", "description": "Testwaehrung Alpha", "isoCode": "XTA" },
    { "code": "XTB", "description": "Testwaehrung Bravo", "isoCode": "XTB" },
    { "code": "XTC", "description": "Testwaehrung Charlie", "isoCode": "XTC" },
    { "code": "XTS", "description": "Testwaehrung XTS (Update)", "isoCode": "XTS" }
  ],
  "pageSize": 1000,
  "hasMore": false,
  "nextPageToken": null,
  "count": null
}
```

> **Das ist neu und es ist eine Vertragsänderung.** Bis zu dieser Version war die Antwort das
> blanke JSON-Array. Es steht jetzt unter dem Schlüssel `value` eines Umschlags, der zusätzlich
> sagt, ob die Antwort vollständig ist. Ein bestehender Konsument liest `value` statt der Wurzel.
> Dasselbe gilt für die Nutzlast, die `Send` an den Endpunkt schickt.

Die Schlüssel innerhalb von `value` sind die konfigurierten **JSON-Schlüssel**; ist einer leer,
verwendet BC Nexus den Feldnamen. Zeilen mit **Überspringen wenn leer** entfallen im Objekt, wenn
der Wert leer ist. Damit ist `Publish` gleichzeitig die einzige Möglichkeit, das Ergebnis eines
`Receive`-Laufs von außen zu verifizieren, ohne sich am Client anzumelden.

### Abrufoptionen: filtern

`requestData` ist ab dieser Version ein Optionsobjekt. [`publish-currency-filter.payload.json`](publish-currency-filter.payload.json)
schränkt auf die Testwährungen ein und fordert die Gesamtzahl an:

```json
{
  "filter": [
    { "key": "code", "op": "startsWith", "value": "XT" }
  ],
  "includeCount": true
}
```

Erwartete Antwort: [`publish-currency-filter.response.json`](publish-currency-filter.response.json) — vier Sätze in `value`, `count` = 4,
`hasMore` = false, `nextPageToken` = null.

Drei Regeln, die den Unterschied zu einem gewöhnlichen OData-Filter ausmachen:

- **`key` ist der JSON-Schlüssel der Feldzuordnung, nicht der Feldname.** Gefiltert werden kann
  ausschließlich auf Felder, welche die Schnittstelle ohnehin ausliefert.
- **Ein unbekannter Schlüssel ist ein Fehler, kein leeres Ergebnis.**
  [`publish-currency-unknown-key.payload.json`](publish-currency-unknown-key.payload.json) zeigt das: `symbol` ist in der Feldzuordnung nicht
  vergeben, der Aufruf scheitert und liefert keine Datenzeile. Die Meldung nennt bewusst nicht,
  welche Schlüssel funktioniert hätten.
- **Die Filterzeilen der Schnittstelle lassen sich nicht aufweiten.** Eine Bedingung des
  Aufrufers, die außerhalb des konfigurierten Filters liegt, liefert ein leeres `value` und
  keinen Fehler.

Die zehn Operatoren heißen auf der Leitung `eq`, `ne`, `gt`, `ge`, `lt`, `le`, `between`,
`startsWith`, `contains` und `in`. Die vollständigen Regeln — welcher Operator zu welchem Feldtyp
passt und welche Zeichen `startsWith` und `contains` abweisen — stehen in
[API Reference]({{ site.baseurl }}/help/api-reference/#retrieval-options-send--publish).

### Abrufoptionen: blättern

[`publish-currency-page.payload.json`](publish-currency-page.payload.json) fordert dieselben vier Sätze in Seiten zu zwei an:

```json
{
  "filter": [
    { "key": "code", "op": "startsWith", "value": "XT" }
  ],
  "page": { "size": 2 },
  "includeCount": true
}
```

Die erste Antwort trägt `hasMore` = true und ein `nextPageToken`. **Dieses Token ist
undurchsichtig — kopieren Sie es unverändert**, konstruieren Sie es nicht.
[`publish-currency-page2.json`](publish-currency-page2.json) ist der Folgeaufruf; der Platzhaltertext darin muss vor dem
Absenden durch das echte Token ersetzt werden. Die zweite Seite liefert die restlichen zwei
Sätze, `hasMore` = false und `nextPageToken` = null.

Zwei Fehlerfälle, die absichtlich laut sind: ein Token, das sich nicht lesen lässt, und ein
Token, dessen Datensatz zwischen den beiden Aufrufen gelöscht wurde. Beide brechen den Aufruf ab,
statt still von vorn zu beginnen.

**Die Obergrenze gilt immer.** Jede Schnittstellendefinition trägt das Feld **Max. Seitengröße**
(Vorgabe 1000). Trifft ein Aufruf ohne jeden Blätterwunsch mehr Sätze als diese Grenze, wird er
**abgewiesen** — mit der Grenze und dem Fortsetzungstoken in der Meldung. Eine stillschweigend
gekürzte Antwort gibt es nicht, denn sie wäre von der vollständigen nicht zu unterscheiden.

### `SendWithOptions`

`Send` ist unverändert und schickt weiterhin den ganzen konfigurierten Bestand.
`SendWithOptions` ist derselbe Aufruf mit einem zweiten Parameter für dieselben Abrufoptionen.
[`send-currency-options.json`](send-currency-options.json) setzt das voraus:

| Feld | Wert |
|---|---|
| Code | `CURRENCYSEND` |
| Tabellennr. | `4` (Currency) |
| Schnittstellentyp | `Senden` |
| Feldzuordnungstyp | `JSON` |
| Endpunktcode | ein Endpunkt, der die Nutzlast tatsächlich annimmt |

Die Optionen werden vor der Verarbeitung auf der Transaktionszeile abgelegt. Die Zeile sagt
danach, mit welchem Filter, welcher Seitengröße und welchem Token der Aufruf gelaufen ist — es
gibt keinen zweiten Ort dafür und keinen Weg, einen Send-Aufruf ohne diesen Nachweis auszuführen.

Mit `"envelope": false` in den Abrufoptionen bleibt die Nutzlast das nackte Array der Sätze,
ohne `value`, `pageSize`, `hasMore` und die übrigen Envelope-Felder — für Partnersysteme, deren
eingehendes Format fest auf ein Array steht. [`send-currency-bare.json`](send-currency-bare.json) zeigt das. Blättern und
`includeCount` stehen dann nicht zur Verfügung: ein nacktes Array hat keinen Platz für
`hasMore`/`nextPageToken` oder `count`, deshalb wird die Kombination abgewiesen statt die Option
stillschweigend zu ignorieren. Das gilt für `SendWithOptions` genauso wie für `Publish`.

### Timeout-Fixture

Für den Timeout-Schritt des Smoke-Skripts (Abschnitt 10) braucht es eine eigene, per Hand
angelegte Schnittstelle — die Beispieldaten enthalten sie nicht:

| Objekt | Feld | Wert |
|---|---|---|
| Endpunkt `TIMEOUT` | URL | `https://httpbin.org/delay/10` |
| | HTTP Timeout (ms) | `3000` |
| | Authentifizierung | Keine |
| Schnittstelle `CURRENCYTIMEOUT` | Tabellennr. | `4` (Currency) |
| | Schnittstellentyp | `Senden` |
| | Feldzuordnungstyp | `JSON` |
| | Endpunktcode | `TIMEOUT` |
| | Feldzuordnung | wie `CURRENCYSEND` |

Der Schritt beweist, dass der konfigurierte Timeout vom SaaS-Laufzeitsystem tatsächlich
angewendet wird: `Send` schlägt nach rund 3 s mit `status 0` fehl, statt httpbins 10-Sekunden-
Verzögerung abzuwarten. Nicht beweisbar ist damit die Obergrenze von 300000 ms — die setzt der
Serverparameter NavHttpClientMaxTimeout, den ein SaaS-Mandant weder einsehen noch setzen kann.

### Feste Filter an der Schnittstelle

Unabhängig vom Aufrufer lässt sich eine Schnittstelle dauerhaft einschränken: Aktion **Filter**
auf der Schnittstellendefinition, eine Zeile je Bedingung aus Feldnr., Operator, Wert und —
bei `Zwischen` — Wert bis. Diese Zeilen gelten immer und lassen sich von außen nicht aufweiten.
Der Weg dorthin steht im [Setup Guide]({{ site.baseurl }}/help/setup-guide/#interface-filters-fixed-server-side-limits).

---

## 9. Prüfen im Client

### Transaktionsliste

Suchen Sie nach **BC Nexus Transaktionsliste** (oder öffnen Sie die Liste von der Seite **BC
Nexus Setup**). Jeder Aufruf erzeugt genau eine Zeile.

| Status | Bedeutung |
|---|---|
| Offen | Angenommen, wartet auf die Job Queue |
| Verarbeitet | Erfolgreich abgeschlossen |
| Fehler | Fehlgeschlagen; **Fehlermeldung** nennt die Ursache, **Anzahl verbleibender Versuche** zeigt, wie oft die Job Queue es noch probiert |
| Storniert | Manuell abgebrochen, wird nicht wiederholt |

Erwartung nach einem vollständigen Durchlauf:

| Aufruf | Status | Fehlermeldung |
|---|---|---|
| `receive-currency-single` | Verarbeitet | leer |
| `receive-currency-array` | Verarbeitet | leer |
| `receive-currency-update` | Verarbeitet | leer |
| `receive-currency-missing-mandatory` | Fehler | `Mandatory field 'code' (Code) is missing or empty in the incoming data.` |

Öffnen Sie die fehlerhafte Zeile mit **Bearbeiten**: die Transaktionskarte zeigt die gesendete
Nutzlast im Feld **Anfrage** (*Request*). Damit lässt sich belegen, dass genau der abgeschickte Text
angekommen ist. Über **Status zurücksetzen** wird die Zeile erneut zur Verarbeitung
freigegeben, über **Manuell verarbeiten** sofort im Vordergrund ausgeführt — Letzteres ist der
schnellste Weg, ohne auf die Job Queue zu warten.

Bei einem Array-Aufruf entsteht **eine** Transaktion für alle drei Sätze. Schlägt ein Satz fehl,
fällt der komplette Aufruf zurück, und die Meldung nennt die Position:
*„Record 2 of 3: …"*.

### Währungsliste

Suchen Sie nach **Währungen**. Erwartetes Ergebnis nach dem vollständigen Durchlauf:

| Code | Beschreibung | ISO-Code |
|---|---|---|
| `XTS` | `Testwaehrung XTS (Update)` | `XTS` |
| `XTA` | `Testwaehrung Alpha` | `XTA` |
| `XTB` | `Testwaehrung Bravo` | `XTB` |
| `XTC` | `Testwaehrung Charlie` | `XTC` |

Steht bei `XTS` noch `Testwaehrung XTS` ohne Zusatz, wurde die Update-Nutzlast nicht verarbeitet.
Schlägt die Update-Nutzlast stattdessen mit einem Primärschlüsselfehler fehl, ist auf der
Schnittstelle **Immer neue Einträge erstellen** aktiv: dann überspringt die Verarbeitung den
Abgleich über den Primärschlüssel und versucht in jedem Fall einzufügen. Für `CURRENCYIMPORT`
muss dieses Kennzeichen inaktiv bleiben.

### Aufräumen

Die vier Testwährungen lassen sich in der Währungsliste löschen, solange keine Buchungen darauf
verweisen. Sie sind bewusst so gewählt, dass sie keinen realen Währungssatz überschreiben.

