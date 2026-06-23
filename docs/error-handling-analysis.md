# Tasker-Fehlerbehandlung für Musik Client

## Analyse der vorhandenen Exporte

Die Exporte enthalten 18 eigenständige Task-Dateien (`*.tsk.xml`), 3 Szenen-Dateien (`*.scn.xml`) und einen Projektexport (`Musik_Client.prj.xml`). Der Projektexport bündelt dieselben Tasks und Szenen. Für die Fehlerbehandlung ist deshalb die Bearbeitung im Projekt beziehungsweise in den einzelnen Task-/Szenen-Exporten identisch zu planen.

### Fehlerkritische Muster

Folgende Aktionsarten kommen besonders häufig vor und sollten priorisiert abgesichert werden:

| Task/Szene | Kritische Stellen | Empfehlung |
|---|---:|---|
| HTTP-/OAuth-Flows (`HBC Send Post Request`, `HBC Update ID & Secret`, `HBC Connect Spotify`) | HTTP Auth, HTTP Request, Task-Aufrufe, Variablen-Setzen | Immer Soft-Fehler direkt nach der Aktion prüfen; bei fehlendem Token/fehlender Response kritisch abbrechen. |
| Spotify-Steuerung (`HBC Next`, `HBC Previous`, `HBC Set PlayPause`, `HBC Set Repeat State`) | Task-Aufrufe auf `HBC Send Post Request`, JSON-/Response-Auswertung | Fehler im Request kritisch behandeln, UI-Aktualisierung nur soft. |
| Daten laden (`HBC Get Playing Song`, `HBC Get My Playlists`, `HBC List PL Songs`, `HBC Get Queue`) | Arrays, JSON Read, For-Schleifen, Scene Element setzen | Parser-/Array-Fehler soft loggen, wenn Fallback möglich; ansonsten kritisch. |
| Szenen (`HBC Startseite`, `HBC Playlist`, `HBC Einstellungen`) | Element-Click-Tasks, Scene Element Text/Image, Spinner/List-Auswahl | Element-Kontext als `<Szene>|<Element-Art>|<Element-ID>` loggen. |
| Tests (`HBC Tests`, `HBC PL Tests`) | Testdaten, UI-Listen, Schleifen | Optional absichern, aber nicht als produktionskritisch priorisieren. |

## Grundprinzip

Tasker hat bereits eingebaute Error-Variablen. Direkt nach jeder potentiell fehlerwerfenden Aktion sollte geprüft werden, ob ein Fehler entstanden ist. Wichtig ist dabei die Unterscheidung:

- **Kritischer Error:** Die fehlerwerfende Aktion beendet den Task beziehungsweise die Element-Aktionsliste. Dafür muss die Aktion selbst so konfiguriert werden, dass sie bei Fehlern nicht einfach weiterläuft. Danach wird ein Error-Logger-Block ausgeführt und anschließend `Stop` genutzt.
- **Soft-Error:** Die fehlerwerfende Aktion ist auf `Continue Task After Error` gestellt. Direkt danach prüft ein Logger-Block `%err`/`%errmsg`; falls gesetzt, wird geloggt und anschließend läuft der eigentliche Ablauf weiter.

Für beide Varianten wird derselbe zentrale Task `HBC Error-Handling` aufgerufen. Vorher werden drei parallele Arrays befüllt:

- `%HBC_Critical_Err()` / `%HBC_Soft_Err()` enthält die Error-Nachricht.
- `%HBC_Critical_Err_Task()` / `%HBC_Soft_Err_Task()` enthält Taskname oder Szenen-Kontext.
- `%HBC_Critical_Action()` / `%HBC_Soft_Action()` enthält `Aktionsname|Aktionsnummer`.

Alle Einträge werden per **Array Push** an Position `999999` geschrieben, mit aktivierter Option zum Auffüllen leerer Positionen.

## Logger-Blöcke zum Einfügen

### Soft-Error-Block direkt nach einer Aktion mit `Continue Task After Error`

```text
If %err Set OR %errmsg Set
  Array Push %HBC_Soft_Err() Position 999999 Value %errmsg Fill Spaces On
  Array Push %HBC_Soft_Err_Task() Position 999999 Value <Taskname oder Szene|Element-Art|Element-ID> Fill Spaces On
  Array Push %HBC_Soft_Action() Position 999999 Value <Aktionsname>|<Aktionsnummer> Fill Spaces On
  Perform Task HBC Error-Handling Parameter 1 soft
End If
```

### Kritischer Error-Block

Für kritische Aktionen sollte die fehlerwerfende Aktion nicht auf `Continue Task After Error` stehen. Damit trotzdem sauber geloggt wird, empfiehlt sich eine Wrapper-Struktur:

```text
<Action, die fehlschlagen kann>  # Continue Task After Error: On, damit der Logger sicher ausgeführt wird
If %err Set OR %errmsg Set
  Array Push %HBC_Critical_Err() Position 999999 Value %errmsg Fill Spaces On
  Array Push %HBC_Critical_Err_Task() Position 999999 Value <Taskname oder Szene|Element-Art|Element-ID> Fill Spaces On
  Array Push %HBC_Critical_Action() Position 999999 Value <Aktionsname>|<Aktionsnummer> Fill Spaces On
  Perform Task HBC Error-Handling Parameter 1 critical
  Stop
End If
```

Praktisch ist diese Variante robuster als ein echter automatischer Task-Abbruch, weil Tasker sonst nach dem Abbruch keinen nachfolgenden Logging-Code mehr ausführen kann.

## Kontextwerte

### Tasks

Bei normalen Tasks wird als Kontext ausschließlich der Taskname gespeichert:

```text
HBC Send Post Request
HBC Get Playing Song
HBC Update ID & Secret
```

### Szenen

Bei Szenen muss der Kontext immer aus drei Teilen bestehen:

```text
<Szenenname>|<Element-Art>|<Element-ID>
```

Beispiele:

```text
HBC Startseite|Button|mw_av_skip_next
HBC Playlist|List|playlist_list
HBC Einstellungen|TextEdit|Client ID
```

## Neuer Task `HBC Error-Handling`

### Ablauf

1. Prüfen, ob `%par1` `critical` oder `soft` ist.
2. Passende Arrays auswählen.
3. Letzte Position ermitteln, zum Beispiel über `%HBC_Critical_Err(#)` oder `%HBC_Soft_Err(#)`.
4. Werte per `Array Pop` auslesen und unmittelbar per `Array Push` an derselben Position wieder einfügen.
5. `%action_name_num` am Zeichen `|` splitten.
6. Prüfen, ob `%err_tsk_name` ein `|` enthält.
7. Log-Verzeichnisse erstellen.
8. Logeintrag an Datei anhängen.
9. Geräte-/Appdaten ermitteln.
10. Kontakttext und Betreff bauen.
11. Text und Betreff URL-codieren.
12. Dialog anzeigen mit Buttons `WhatsApp`, `E-Mail`, `Error ignorieren`.
13. Je nach Button den WhatsApp- oder Mail-Link öffnen.

### Pseudocode für Tasker-Aktionen

```text
If %par1 ~ critical
  Variable Set %array_pos To %HBC_Critical_Err(#)
  Array Pop %HBC_Critical_Err() Position %array_pos To %err_msg
  Array Push %HBC_Critical_Err() Position %array_pos Value %err_msg Fill Spaces On
  Array Pop %HBC_Critical_Err_Task() Position %array_pos To %err_tsk_name
  Array Push %HBC_Critical_Err_Task() Position %array_pos Value %err_tsk_name Fill Spaces On
  Array Pop %HBC_Critical_Action() Position %array_pos To %action_name_num
  Array Push %HBC_Critical_Action() Position %array_pos Value %action_name_num Fill Spaces On
  Variable Set %err_type_de To Kritischer Error
  Variable Set %err_log_file To Tasker/HBC/log/error/critical/critical_err.log
Else If %par1 ~ soft
  Variable Set %array_pos To %HBC_Soft_Err(#)
  Array Pop %HBC_Soft_Err() Position %array_pos To %err_msg
  Array Push %HBC_Soft_Err() Position %array_pos Value %err_msg Fill Spaces On
  Array Pop %HBC_Soft_Err_Task() Position %array_pos To %err_tsk_name
  Array Push %HBC_Soft_Err_Task() Position %array_pos Value %err_tsk_name Fill Spaces On
  Array Pop %HBC_Soft_Action() Position %array_pos To %action_name_num
  Array Push %HBC_Soft_Action() Position %array_pos Value %action_name_num Fill Spaces On
  Variable Set %err_type_de To Soft-Error
  Variable Set %err_log_file To Tasker/HBC/log/error/soft/soft_err.log
End If

Variable Split %action_name_num Splitter |
Variable Set %action_name To %action_name_num1
Variable Set %action_num To %action_name_num2

Create Directory Tasker/HBC/log/error/critical
Create Directory Tasker/HBC/log/error/soft

If %err_tsk_name !~R \|
  Write File %err_log_file Append On Text:
----------------------------------------
Datum: %DATE
Time: %TIME
Task: %err_tsk_name
Aktion: %action_name
Nummer: %action_num
Error: %err_msg
Else
  Variable Split %err_tsk_name Splitter |
  Variable Set %err_szn_name To %err_tsk_name1
  Variable Set %err_elm_art To %err_tsk_name2
  Variable Set %err_elm_id To %err_tsk_name3
  Write File %err_log_file Append On Text:
----------------------------------------
Datum: %DATE
Time: %TIME
Szene: %err_szn_name
Element: %err_elm_art
Element-ID: %err_elm_id
Aktion: %action_name
Nummer: %action_num
Error: %err_msg
End If
```

## Geräte- und Appinformationen

Empfohlene Variablen:

```text
Variable Set %device To %DEVID
Variable Set %manufacturer To %MANUFACTURER
Variable Set %model To %MODEL
Variable Set %android_version To %SDK oder Shell getprop ro.build.version.release
App Info / Package Info für com.plsreloadtasker.hbcclient -> %app_version
```

Falls Tasker für Hersteller/Modell keine direkte eingebaute Variable liefert, kann `Run Shell` ohne Root verwendet werden:

```text
getprop ro.product.manufacturer
getprop ro.product.model
getprop ro.build.version.release
```

## URL-Codierung

Baue zuerst den vollständig ausgefüllten Textkörper in `%contact_body` und den Betreff in `%mail_subject`. Danach ersetze mindestens:

```text
% -> %25  # zuerst, damit spätere Codes nicht erneut verändert werden
Leerzeichen -> %20
Neue Zeile -> %0A
? -> %3F
& -> %26
= -> %3D
! -> %21
, -> %2C
. -> %2E
: -> %3A
; -> %3B
```

Danach:

```text
Variable Set %wa_link To https://wa.me/491774766543?text=%contact_body_encoded
Variable Set %mail_link To mailto:felix.h.kaiser@outlook.de?subject=%mail_subject_encoded&body=%contact_body_encoded
```

## Dialog

Tasker-Dialog:

```text
Title: A critical error occurred while running %dialog_value
Text:
Es ist ein %err_type_de aufgetaucht:

Möchtest du den Entwickler kontaktieren und ihm den Fehler zusenden?

Bei Kritischen Fehlern wird das Kontaktieren des Entwicklers dringend empfohlen, da auch andere Nutzer betroffen sein können, und der Fehler dann so schnell wie möglich behoben werden kann.

Bitte wähle unten aus, wie du den Entwickler kontaktieren möchtest:

Error:
"%err_msg"

Button 1: WhatsApp
Button 2: E-Mail
Button 3: Error ignorieren
```

Nach dem Dialog:

```text
If %button ~ WhatsApp
  Browse URL %wa_link
Else If %button ~ E-Mail
  Browse URL %mail_link
End If
```

## Priorisierte Umsetzung

1. `HBC Error-Handling` als zentralen Task anlegen.
2. `HBC Send Post Request` absichern, weil fast alle API-Tasks davon abhängen.
3. Danach `HBC Update ID & Secret` und `HBC Connect Spotify` absichern.
4. Danach UI-/Datenlade-Tasks absichern: `HBC Get Playing Song`, `HBC Get My Playlists`, `HBC List PL Songs`.
5. Danach die Szenen-Element-Tasks absichern, insbesondere Click-Handler und List-Handler in `HBC Startseite` und `HBC Playlist`.
6. Test-Tasks zuletzt oder separat behandeln.

## Konkrete Regel für Kritikalität

- **Critical:** Authentifizierung fehlgeschlagen, HTTP-Request nicht ausführbar, API meldet ungültige Credentials, benötigte Response fehlt komplett, Pflicht-Array leer, Task-Aufruf eines Kern-Tasks schlägt fehl.
- **Soft:** Einzelnes UI-Element kann nicht gesetzt werden, Cover/Icon fehlt, optionale Playlist-/Queue-Zeile fehlt, optionale JSON-Eigenschaft fehlt, Test-/Debug-Ausgabe schlägt fehl.
