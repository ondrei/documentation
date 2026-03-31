# Contest

## Ucel dokumentace

Tato slozka slouzi jako sdilena technicka dokumentace pro projekt `contest`.
Pri kazdem dalsim tasku je nutne tento soubor nejdriv nacist a aplikovat jeho pokyny.

## Pracovni pravidla pro dalsi tasky

- Komunikace probiha cesky.
- Priorita je dukladnost, bezpecnost, stabilita a predvidatelnost chovani.
- Kazda zmena musi byt posouzena i z pohledu dopadu na soutez, losovani, evidenci vyhercu, uctenek a odesilani vyher.
- U zmen v `contest-manager-universal` je nutne povazovat za kriticke hlavne:
  - losovani vyherce
  - potvrzeni vyherce
  - stavy vyherce a uctenky
  - vazby mezi `cm_contestants`, `cm_winners`, `cm_receipts`, `cm_prizes`, `cm_ecomail`
- Pred kazdou zmenou, ktera saha na DB logiku nebo stavove prechody, je potreba:
  - zkontrolovat vsechny usage dane tabulky napric pluginem i tematem
  - vyhodnotit, zda jde o aditivni fix bez zmeny business logiky
  - pokud je nutna migrace dat, navrhnout ji oddelene a bezpecne
- Pred kazdou implementaci je potreba urcit:
  - co je ciste security hardening
  - co je functional bugfix
  - co je schema nebo data migrace
- Po kazde zmene zapisovat do `documentation-roadmap.md`:
  - datum a kontext zmeny
  - co bylo upraveno
  - proc byla zmena potreba
  - jake soubory byly dotceny
  - jaky je dopad na funkci projektu
  - jak bylo overeno, ze se nic nerozbilo
- Do `documentation-todo.md` zapisovat otevrene body, priority, zavislosti a dalsi kroky.
- Po kazdem dokoncene tasku synchronizovat dokumentaci podle relevance:
  - `documentation-roadmap.md` pro detailni changelog a overeni
  - `documentation-todo.md` pro hotovo a otevrene body
  - `documentation-gpt.md` pro nove trvale pracovni pokyny

## Aktualni technicke zavery z review

- Klicovy custom plugin je `wp-content/plugins/contest-manager-universal`.
- Druhy custom plugin je `wp-content/plugins/ch-cleanup`.
- Nejvyssi prioritu ma bezpecne opravovat pouze aditivne a bez zmeny business logiky losovani.
- Kriticke body projektu jsou:
  - rozsahy losovani
  - stavy vyherce a uctenky
  - Ecomail fronta a odesilani notifikaci
  - frontend vypis potvrzenych vyhercu

## Aktualni technicky stav po implementaci

- `cm_receipts` je navrzena jako tabulka s jednim radkem na jednoho souteziciho.
  - Schema pouziva `UNIQUE KEY uniq_contestant_id (contestant_id)`.
  - Soucasne je pridany index `idx_cnp (cnp)`.
  - Pro starsi instalace je pripraven runtime upgrade:
    - audit existence tabulky
    - deduplikace podle `contestant_id`
    - doplneni indexu explicitnim `ALTER TABLE`, ne pouze pres `dbDelta`

- Plugin `contest-manager-universal` drzi technicky stav v options:
  - `contest_plugin_schema_version`
  - `contest_manager_caps_version`
  - `contest_manager_cron_secret`
  - `contest_manager_ecomail_queue_last_run`

- Capability `view_shipping_page` a `view_report_page` se:
  - prideluji administratorovi pri aktivaci
  - bezpecne backfilluji i za behu u starsich instalaci

- `main-periods` jsou od 2026-03-30 realna funkcni cast pluginu.
  - Data jsou v tabulce `cm_main_prize_periods`.
  - Admin stranka je v menu pod slugem `contest-main-periods`.
  - Hlavni losovani pouziva `cm_main_prize_periods`, pokud v tabulce existuji zaznamy.
  - Pokud hlavni obdobi neexistuji, plugin fallbackuje na cele rozmezi beznych obdobi z `cm_periods`.
  - Pri budoucich zmenach logiky hlavniho losovani je nutne zacit ve helperu `contest_manager_get_draw_scope()`.

- Manualni spusteni Ecomail fronty se nedeje pres renderovane verejne URL s tajnym parametrem.
  - Admin pouziva nonce-protected `admin-post` akci.
  - `cron-runner.php` overuje DB secret.
  - Kvuli zpetne kompatibilite pred soutezi akceptuje i legacy key z puvodni implementace.
  - Pro bezny provoz uz plugin neni zavisly na externim serverovem cronu.
  - Fronta se umi bezpecne zpracovat pri beznych GET requestech pres `admin_init` a `template_redirect`, s lockem a throttlingem.

- Uctenky se ukladaji do jednotne slozky `wp-content/uploads/contest_uploads`.
  - Pri dalsich zmenach nepouzivat zadnou jinou variantu nazvu adresare.

- `ch-cleanup` ma aktivni wiring pro `disable_xmlrpc` pres filter `xmlrpc_enabled`.

- Frontend vypis vyhercu v tematu `contest` zobrazuje i vyhry se stavy:
  - `Připraveno na odeslání`
  - `Odesláno elektronicky`
  - `Předáno osobně`
  - `Odesláno`

## Overene technicke kontroly

- `php -l` probehl nad vsemi upravenymi PHP soubory bez chyby.
- WordPress bootstrap po zmenach overen.
- Potvrzeno:
  - existence tabulky `cm_main_prize_periods`
  - capability backfill pro shipping/report
  - hooky pro interni zpracovani Ecomail fronty
  - `cron-runner.php` vraci `403` bez klice
  - `cron-runner.php` funguje s legacy klicem i s aktualnim DB secretem
- Nad prazdnou lokalni DB probehl smoke test se seed daty a naslednym cleanupem.
  - helper `contest_manager_get_draw_scope()` vracel spravne rozsahy pro daily/weekly/main
  - admin stranka losovani renderovala hlavni obdobi i denni datumy
  - frontend dotaz na vyherce zahrnul i stav `Odesláno elektronicky`
  - po testu byly vsechny contest tabulky znovu vycisteny

- V upravenych souborech komentovat vlastni funkce a dulezite callbacky jednotnym formatem:
  - `// -- HLAVNI POPIS | Strucny vecny detail --`
  - komentar ma vysvetlit ucel funkce, ne jen opsat jeji nazev
  - pri cistem komentovacim refactoru se nesmi menit business logika ani stavove prechody

## Commit konvence

- Commity v projektu `contest` cislovat ve formatu `CT-0001` a vyse.
- Vzdy navazovat na nejvyssi dosud pouzite `CT-` cislo a pokracovat `+1`.
- Commit message zapisovat vzdy ve formatu:
  - `CT-0012 | Message in English`
- Pokud vznikne vice navazujicich commitů v jednom tasku, cisla musi byt bez mezer a bez preskakovani.
- Pred vytvorenim noveho commitu nejdriv overit posledni pouzite `CT-` cislo v historii repozitare.
