# Contest TODO

## Stav po implementaci 2026-03-30

### Hotovo

- Opraveny admin mutace bez nonce a CSRF ochrany v `contest-manager-universal`.
  - `includes/pages/contestants.php`
  - `includes/pages/ecomail-queue.php`
  - `includes/pages/prizes.php`
  - `includes/pages/periods.php`
  - `includes/pages/main-periods.php`

- Opraven datovy model `cm_receipts`.
  - probahl audit lokalni DB
  - lokalni DB byla prazdna
  - schema doplneno o `UNIQUE KEY uniq_contestant_id (contestant_id)`
  - schema doplneno o `KEY idx_cnp (cnp)`
  - pripraven runtime dedup/backfill pro starsi instalace

- Zapojen prepinac `disable_xmlrpc` v `ch-cleanup`.

- Doresen capability bootstrap v `contest-manager-universal`.
  - `view_shipping_page`
  - `view_report_page`
  - aktivace + runtime backfill pro administratora

- Dokoncena feature `main-periods`.
  - tabulka `cm_main_prize_periods` je soucast schema instalace
  - stranka je v admin menu
  - editace/mazani funguje pres standardni admin AJAX flow
  - hlavni losovani pouziva `cm_main_prize_periods`, pokud existuji zaznamy
  - fallback bez hlavniho obdobi zustava na cele rozmezi `cm_periods`

- Ecomail fronta uz nepotrebuje externi serverovy cron jako podminku bezne funkcnosti.
  - plugin umi zpracovat frontu pri beznych GET requestech
  - zachovan rucni admin trigger
  - zachovana kompatibilita `cron-runner.php` s legacy key i aktualnim DB secretem

- Opraveny dalsi realne chyby nalezene pri implementaci.
  - sjednocena slozka pro ukladani uctenek na `contest_uploads`
  - frontend vypis vyhercu zobrazuje i elektronicky a osobne predane vyhry po schvaleni uctenky
  - dashboard pouziva korektni hodnoty `Daily/Weekly/Main`
  - srovnane razeni v admin prehledech `contestants`, `winners`, `sending-page`, `ecomail-queue`
  - odstranena mrtva JS vetev `assets/js/main-periods.js`
  - `includes/ajax-handler.php` uz neposila globalni JSON hlavicku na vsechny requesty
  - `includes/pages/report.php` uz nema duplicitni deklaraci helper funkce
  - odstraneny mrtve POST handlery ve `winners.php` a `sending-page.php`
  - doplnen `delete_main_period_ajax`
  - pri aktivaci pluginu se nove flushnou rewrite rules pro `/v/{cnp}`

- Doplneny funkcni komentare v upravenych souborech a synchronizovana dokumentace.
  - komentare doplneny do kritickych PHP, JS i inline callbacku
  - probehl znovu syntax check `php -l` nad dotcenymi PHP soubory
  - krok byl detailne zapsan do `documentation-roadmap.md` a pravidla doplnena do `documentation-gpt.md`



- Zavedena commit konvence `CT-XXXX | Message in English` a aktualni zmeny byly rozdeleny do:
  - `CT-0012 | Harden contest manager core flows and draw logic`
  - `CT-0013 | Stabilize ecomail queue cron runner and cleanup plugin`

### Otevrene body

- Provest klikaci browser test celeho admin flow na lokalni instanci.
  - registrace souteziciho
  - losovani daily/weekly/main
  - potvrzeni vyherce
  - nahrani a schvaleni uctenky
  - odeslani vyhry
  - exporty v shipping/pending/report sekcich

- Provest kontrolu a pripadny upgrade ACF PRO z verze `6.2.4`.

- Rozhodnout, zda po startu soutěze odstranit legacy podporu puvodniho cron key.
  - aktualne je ponechana kvuli nulovemu dopadu na provoz a zpetne kompatibilite

- Rozhodnout, zda ponechat aktualni JS XLSX exporty z CDN, nebo je prevest na interne generovane CSV/XLSX bez externi zavislosti.

- Doresit nebo potvrdit stav report exportu.
  - helper vrstva je srovnana
  - samotne export UI v `report.php` stale neni plne dodelane

### Doporucene dalsi poradi

1. Provest browserove end-to-end otestovani celeho soutezniho flow.
2. Udelat ACF PRO update a znovu projet smoke testy.
3. Rozhodnout o report exportech a exportech bez CDN.
4. Po stabilizaci produkce zvazit odstraneni legacy cron key kompatibility.

## Commit konvence

- Aktivni commitovaci standard pro projekt `contest` je `CT-XXXX | Message in English`.
- Nove commity musi navazovat na nejvyssi existujici `CT-` commit v historii repozitare.
- Po stavu `CT-0011` dalsi commit zacina na `CT-0012`.
