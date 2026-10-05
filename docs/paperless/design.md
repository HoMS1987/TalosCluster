# Paperless-ngx – Konzept

Stand: 2026-10-05

Dieses Dokument beschreibt den technischen Teil. Die persönliche Einteilung (Korrespondenten, Typen, Tags, Felder) liegt bewusst nicht in diesem öffentlichen Repository, sondern lokal unter `D:\Paperless-Import\Einteilung.md`.

## Ziel

Paperless-ngx wird das Archiv für wichtige Dokumente und Schriftverkehr: Bank, Versicherungen, Rechnungen, Steuer, Gehalt, Urkunden, Ausweise, Fahrzeuge, Gesundheit, Haustier. Arbeitsdateien und Sammlungen (Bücher, Quellcode, Foliensätze, Normen, Noten) bleiben in Nextcloud.

Erfolg heißt:

- Jedes Dokument ist in Sekunden über Volltextsuche, Korrespondent, Typ oder Tag auffindbar.
- Neue Dokumente werden weitgehend automatisch zugeordnet; der Mensch prüft nur noch über den Posteingang.
- Ein verschlüsseltes Backup außer Haus lässt sich nachweislich wiederherstellen.
- Ein Ausstieg aus Paperless ist ohne Datenverlust möglich.

## Entscheidungen

| Thema | Entscheidung |
|---|---|
| Inhalt | Nur wichtige Dokumente und Schriftverkehr |
| Chart | TrueCharts `paperless-ngx` (15.11.0, Paperless 3.1.3) |
| Erreichbarkeit | Nur intern (Ingress `internal`), von außen per VPN |
| Office-Dateien | Tika und Gotenberg aktiviert |
| Nutzer | Zunächst ein Nutzer; Workflow setzt den Eigentümer auf jedes Dokument, damit spätere Konten das Archiv nicht sehen |
| Einteilung | Schlank: Korrespondent + 16 feste Typen + wenige Tags + 2 Zusatzfelder |
| Eingang | Scanner am PC → SMB-Freigabe `consume`; Browser-Upload; später Outlook-Ordner-Abruf |
| Archiv | NFS auf TrueNAS |
| Backup | Nächtlicher `document_exporter` nach TrueNAS, von dort TrueNAS Cloud Sync verschlüsselt nach Cloudflare R2 |
| Parallelbetrieb | Originale bleiben vorerst in Nextcloud; Entfernen erst nach Parallelphase und erfolgreichem Wiederherstellungstest |

## Betrieb im Cluster

Ablage im Repository nach dem üblichen App-Muster:

```text
clusters/main/kubernetes/apps/paperless-ngx/
├── ks.yaml
└── app/
    ├── kustomization.yaml
    ├── namespace.yaml
    └── helm-release.yaml
```

Wichtige Werte:

- Ingress `internal`, Host `paperless.${DOMAIN_0}`, Zertifikat über `domain-0-le-prod`.
- CNPG-Datenbank mit täglichem Backup nach R2 (Muster wie bei Immich) und `retentionPolicy: "7d"`; Valkey aus dem Chart.
- `tika.enabled: true`.
- Admin-Zugang: Paperless startet mit dem Chart-Standard `admin`/`admin`. Das Passwort wird bei der ersten Anmeldung geändert und in KeePassXC abgelegt.
  - `PAPERLESS_ADMIN_USER` legt den Benutzer nur an, wenn er fehlt, und ändert nie ein bestehendes Passwort.
  - Darum den Benutzer `admin` nie umbenennen oder löschen, sonst entsteht er beim nächsten Pod-Start neu mit Passwort `admin`.
  - Eine Instanz für eine Wiederherstellung bekommt einen anderen Admin-Namen, damit sie nicht mit dem `admin` aus dem Export kollidiert.
- `PAPERLESS_SECRET_KEY` erzeugt das Chart selbst (Secret `…-secrets`).

Paperless-Einstellungen:

| Variable | Wert | Grund |
|---|---|---|
| `PAPERLESS_URL` | `{{ .Values.chartContext.appUrl }}` | CSRF, Links; aus dem Ingress abgeleitet |
| `PAPERLESS_OCR_LANGUAGE` | `deu+eng` | Texterkennung Deutsch und Englisch |
| `PAPERLESS_OCR_LANGUAGES` | `deu eng` | Sprachpakete |
| `PAPERLESS_CONSUMER_POLLING_INTERVAL` | `60` | NFS meldet neue Dateien nicht zuverlässig (seit 3.0 umbenannt, früher `PAPERLESS_CONSUMER_POLLING`) |
| `PAPERLESS_CONSUMER_RECURSIVE` | `true` | Unterordner im Eingang |
| `PAPERLESS_CONSUMER_SUBDIRS_AS_TAGS` | `true` | Unterordner werden zu Tags (erster Import) |
| `PAPERLESS_CONSUMER_DELETE_DUPLICATES` | `true` | Seit 3.0 werden Duplikate sonst eingelesen |
| `PAPERLESS_FILENAME_DATE_ORDER` | `YMD` | Datum aus Dateinamen wie `2024-03-21 …` |
| `PAPERLESS_FILENAME_FORMAT` | `{{ correspondent }}/{{ created_year }}/{{ created }} {{ title }}` | Lesbare Ablage und lesbarer Export |

## Speicher und Backup

| Pfad im Pod | Speicher | Sicherung |
|---|---|---|
| `/media` (Archiv) | NFS `/mnt/MAIN/paperless/media` | ZFS-Snapshots auf TrueNAS |
| `/consume` | NFS `/mnt/MAIN/paperless/consume`, zusätzlich SMB-Freigabe für den PC | keine (Durchlaufware) |
| `/export` | NFS `/mnt/MAIN/paperless/export` | TrueNAS Cloud Sync, verschlüsselt nach R2 |
| `/data` (Index, Klassifikator) | Longhorn-Volume | keine; wird beim Import neu aufgebaut |
| Datenbank | CNPG | tägliches Backup nach R2 |

Export:

- CronJob `paperless-export` jede Nacht um 02:45 UTC (nach dem Plex-Backup, alle Cluster-Zeitpläne laufen in UTC). Er ruft per `kubectl exec` im Paperless-Pod `document_exporter /export -d -f --no-progress-bar` auf und nutzt so die Datenbank- und Speicher-Einstellungen der App. Dafür hat er einen eigenen ServiceAccount, der nur `pods/exec` im Namespace `paperless-ngx` darf.
- Der Export ist inkrementell (nur geänderte und neue Dateien) und enthält Originale, PDF/A-Fassungen und `manifest.json` mit allen Metadaten. Datenbank und Dateien sind darin zeitgleich gesichert.
- `-d` entfernt gelöschte Dokumente aus dem Export; `-f` nutzt das Dateinamen-Schema.

Cloud Sync:

- Eigener R2-Bucket `paperless-export` mit eigenem API-Token, der nur auf diesen Bucket darf.
- Remote Encryption an; Passwort und Salt im KeePassXC-Container ablegen.
- Modus „Copy“ statt „Sync“, damit Löschungen oder Schadsoftware nicht in die Cloud durchschlagen.
- Täglich 07:00 Uhr TrueNAS-Ortszeit, also nach Export und allen Cluster-Backups (Übersicht in `Notes.md`).

Schlüssel außer Haus: `age.agekey` und Cloud-Sync-Passwort und -Salt liegen im KeePassXC-Container, der auf mehrere Geräte synchronisiert wird.

Warum kein VolSync für `media`: Die TrueCharts-Bibliothek legt VolSync-Objekte nur für Persistenz vom Typ `pvc` an. Bei `type: nfs` wird ein `volsync`-Block ohne Fehlermeldung ignoriert (geprüft per `helm template` mit Chart 15.11.0).

Wiederherstellung: neue Paperless-Instanz aufsetzen, Export aus R2 zurückholen, `document_importer` ausführen.

- Getestet am 2026-10-05: R2 → `rclone` (crypt mit Passwort und Salt der Cloud-Sync-Aufgabe, `filename_encryption=standard`) → vorübergehende Instanz → `document_importer`. Dokumente, Korrespondenten, Typen, Tags, Felder und Workflows stimmten mit dem Manifest überein.
- In PowerShell die rclone-Werte in einfache Anführungszeichen setzen, sonst verfälschen Sonderzeichen im Passwort den Schlüssel.
- Der Importer ersetzt den vorübergehenden Admin der neuen Instanz durch `admin` aus dem Export. Danach gilt das gewohnte `admin`-Passwort.
- Der CronJob hat ein Zeitlimit von einer Stunde (`activeDeadlineSeconds`), damit ein hängender Export nicht alle folgenden blockiert.

## Vorbereitung auf TrueNAS

- Dataset `paperless` mit den Kind-Datasets `media`, `consume`, `export`; Besitzer `apps` (UID/GID 568), unter dieser Kennung arbeitet Paperless im Pod.
- `consume` als Multiprotocol-Dataset mit NFSv4-ACL: `owner@` Vollzugriff, benannte Gruppe `apps` und der SMB-Benutzer je *Modify*, alle vererbt. So darf Paperless Dateien lesen und löschen, die per SMB abgelegt wurden.
- NFS-Freigaben für die drei Ordner:
  - *Hosts* nur der Talos-Node, damit kein anderes Gerät im Netz das Archiv einbinden kann.
  - *Mapall User/Group* `apps`. Ohne diese Einstellung macht TrueNAS aus `root` im Container den Benutzer `nobody`; die vom Container beim Start angelegten Ordner (`media/documents/originals` usw.) waren dann für Paperless nicht beschreibbar.
- SMB-Freigabe für `consume`; der SMB-Benutzer braucht zusätzlich einen Eintrag in der Freigabe-ACL der SMB-Freigabe.
- Periodische Snapshot-Aufgabe für `paperless/media`, täglich, Aufbewahrung 4 Wochen.
- Cloud-Zugangsprofil (S3-kompatibel, R2-Endpunkt) und Cloud-Sync-Aufgabe für `paperless/export`.

## Erster Import

Quelle ist der Nextcloud-Sync-Ordner. Er wird nur gelesen und nie als Eingangsordner eingebunden.

1. **Zuordnungsliste:** Quelldatei → Zielordner → Tags. Wird vor dem Kopieren freigegeben.
2. **Kopieren** in den Zwischenordner `D:\Paperless-Import` (außerhalb des Nextcloud-Sync).
3. **Aufbereiten:**
   - Inhaltsgleiche Dateien nur einmal, mit der Vereinigung aller Tags.
   - Mehrseitige JPG-Scans zu einer PDF zusammenfügen.
   - Dateinamen mit Datum vereinheitlichen (`YYYY-MM-DD Titel`).
   - Verschlüsselte PDFs markieren und vorab entsperren.
   - Dateien ohne Dokumentcharakter auslassen.
4. **Ordnerstruktur = Tags**, verschachtelte Ordner ergeben mehrere Tags.
5. **Etappen:**
   - Etappe 1 mit etwa 50 vielfältigen Dokumenten, von Hand zugeordnet, damit der Klassifikator lernt.
   - Danach Bereich für Bereich mit Prüfung über den Posteingang.

Spätere Sammel-Downloads (Bank, Versicherungsportal) erzeugen oft neue PDF-Dateien für bereits vorhandene Dokumente. Paperless erkennt nur identische Dateien als Duplikat. Darum nur fehlende Zeiträume laden oder vorher gegen den Bestand abgleichen.

## Reihenfolge der Umsetzung

1. TrueNAS vorbereiten.
2. Paperless per Branch und PR bereitstellen; Merge nach grünen Checks.
3. Grundeinrichtung:
   - Anmeldung und 2FA.
   - Einteilung, Workflows und Dateiablage per Skript über die Paperless-API anlegen.
4. Backup:
   - Export-CronJob und Cloud Sync einrichten.
   - Wiederherstellungstest mit Testdokumenten in eine vorübergehende zweite Instanz.
5. Erster Import in Etappen.
6. Später:
   - Outlook-Ordner-Abruf (Entra-App-Registrierung).
   - Sammel-Downloads.
   - Ggf. Freigabelinks (nur Pfad `/share/` extern).
7. Nach der Parallelphase:
   - Wiederherstellungstest mit echtem Bestand.
   - Danach Entscheidung über das Entfernen der Originale aus Nextcloud.

## Sicherheit

- Nur interner Zugriff; 2FA für jedes Konto.
- Standard-Passwort `admin` direkt nach dem ersten Deploy ändern; Zugangsdaten in KeePassXC.
- Workflow setzt den Eigentümer, damit Dokumente nicht ohne Eigentümer für alle sichtbar sind.
- Persönliche Einteilung nicht im öffentlichen Repository.

## Ausstieg

- `document_exporter -f` liefert alle Originale in lesbarer Ordnerstruktur plus `manifest.json` mit allen Metadaten.
- Der Export ist auch ohne Paperless nutzbar.

## Offene Punkte (in der Umsetzung zu prüfen)

- **TrueNAS Cloud Sync:** ob die installierte TrueNAS-Version R2 als eigenen Anbieter kennt; sonst S3 mit eigenem Endpunkt.
- **Copy-Modus:** ob `document_importer` mit übrig gebliebenen alten Dateien im Export klarkommt. Beim ersten Wiederherstellungstest gab es noch keine; im zweiten Test nach der Parallelphase prüfen.
- **Mailversand:** ob SMTP-Versand über das Outlook.com-Konto noch mit Passwort möglich ist (Testversand).

Geklärt:

- **Export-CronJob:** eigener CronJob mit `kubectl exec` statt Zusatz-Workload im Chart, weil er so keine Umgebungsvariablen der App duplizieren muss.
- **Chart-Annotation:** `trueforge.org/max_kubernetes_version: 1.35.0` ist nur informativ; Paperless läuft seit dem 2026-10-05 auf Kubernetes 1.36.5.
- **JPG zu PDF:** Pillow (Python) fügt mehrseitige Scans zu einer PDF zusammen, kein zusätzliches Werkzeug nötig.
