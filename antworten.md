# Antworten – Phase 4: Den Workflow verstehen

## Teil A: Aufbau der Workflow-Datei

**1. Events:** Der Workflow startet bei drei Ereignissen:
- `push` – bei jedem Push, egal auf welchen Branch
- `pull_request` – bei jedem Pull Request mit Ziel `main`
- `workflow_dispatch` – von Hand über den Knopf „Run workflow“ im Tab Actions

**2. Runner:** `ubuntu-latest`. Das ist eine virtuelle Ubuntu-Maschine, die GitHub für jeden Lauf neu bereitstellt und danach wieder löscht (GitHub-hosted Runner). Docker und Docker Compose sind schon installiert.

**3. Job und Steps:** Der Job heißt `stack-test` (Anzeigename „Compose-Stack testen (Port 8085)“) und hat **10 Steps**.

**4. Action oder eigene Befehle:**
- Nur Step 1 „Code auschecken“ nutzt eine fertige Action: `uses: actions/checkout@v7`. Sie lädt den Inhalt des Repos auf den Runner.
- Die Steps 2 bis 10 führen eigene Shell-Befehle mit `run:` aus.

**5. Rechte:** `permissions: contents: read`. Der Workflow darf den Code nur lesen, aber nichts ändern, pushen oder veröffentlichen. Das ist das **Prinzip der minimalen Rechte**: Wird ein Schritt manipuliert (z. B. über eine fremde Action), kann er kaum Schaden anrichten.

**6. `if: ${{ !github.event.repository.is_template }}`:**
- In der **Vorlage** (Template-Repository) liegt noch kein Stack. Der Job wird dort übersprungen, damit die Vorlage nicht ständig rot ist.
- In **meinem Repo** ist das kein Template, die Bedingung ist also wahr und der Test läuft immer.

## Teil B: Was wird geprüft?

**7. Welcher Step schlägt zuerst fehl?**

| Fall | Step |
|---|---|
| a) Tabulator statt Leerzeichen in der `compose.yml` | **Step 3** – YAML-Syntax prüfen |
| b) Ordner `html/` nie committet | **Step 7** – Inhalt prüfen (`html/index.html` fehlt) |
| c) `ports: - "8085:8080"` | **Step 6** – Erreichbarkeit: Docker leitet auf Container-Port 8080 weiter, der Webserver lauscht aber auf 80 |
| d) `compose.yml` liegt in einem Unterordner | **Step 2** – Compose-Datei suchen (sucht nur im Hauptordner) |

**8. Warum vergleicht Step 7 den Inhalt, statt nur den Port zu testen?**
Ein Webserver antwortet auch dann auf Port 8085, wenn er die **falsche Seite** ausliefert. Genau das passiert in Phase 5: Nach dem Wechsel auf Apache hängt das Volume noch am nginx-Ordner `/usr/share/nginx/html`. Apache läuft und antwortet, zeigt aber seine Standardseite „It works!“ statt meiner Statusseite. Ein reiner Port-Test wäre grün, obwohl der Dienst für den Kunden falsch ist. Ein guter Test prüft das **Ergebnis, das der Kunde erwartet**, nicht nur, ob „irgendetwas läuft“.

**9. Wozu Step 10 „Aufräumen“?**
`docker compose down -v` entfernt Container, Netzwerke und Volumes, damit der Lauf sauber endet. Bei GitHub-Runnern wird die Maschine zwar sowieso gelöscht, aber auf **selbst gehosteten Runnern**, die mehrfach benutzt werden, blieben sonst Container liegen und blockierten z. B. Port 8085 für den nächsten Lauf. `if: always()` sorgt dafür, dass auch nach einem Fehler aufgeräumt wird.

## Teil C: Einordnen

**10. CI, Continuous Delivery oder Continuous Deployment?**
Es ist **Continuous Integration**: Jede Änderung wird automatisch getestet. Es wird aber kein fertiges Paket (z. B. ein Image in einer Registry) bereitgestellt (das wäre Delivery), und nichts wird automatisch auf den Server ausgerollt (das wäre Deployment). Den LXC aktualisiere ich selbst mit `git pull` und `docker compose up -d`.

**11. Zwei Vorteile gegenüber „mal kurz mit curl testen“:**
- Der Test läuft **bei jeder Änderung automatisch** und wird nie vergessen.
- Er läuft **immer gleich in einer sauberen Umgebung**, „bei mir geht's“ zählt nicht.
- Fehler fallen **vor** dem Betrieb auf, und das Ergebnis ist **für alle sichtbar dokumentiert** (Haken am Commit, Abzeichen im README).

## Teil D (Experten): Die DoD-Abzeichen

**12.** Ein Abzeichen zeigt immer den letzten Lauf auf **einem** Branch. DoD 3 soll zeigen, ob es den Branch `dev` mit eigener Änderung gibt. Darum läuft die Prüfung bei einem Push auf `dev`, und das Abzeichen ist mit `?branch=dev` eingebunden. Apache (DoD 4) und Pull Request (DoD 5) betreffen den Betriebsstand, also `main`.

**13.** `dod-5-pull-request.sh` braucht von der GitHub-API:
- den zuletzt gemergten Pull Request von `dev` nach `main` mit seinem Merge-Zeitpunkt (`merged_at`) und seinem letzten Commit (`head.sha`)
- das Ergebnis des Checks „Compose-Stack testen (Port 8085)“ für genau diesen Commit und wann er fertig war (`completed_at`)

DoD 5 ist nur grün, wenn dieser Check **erfolgreich** war **und vor dem Merge** fertig war.
