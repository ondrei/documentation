# Contest Roadmap

## 2026-03-30 - Uvodni zapis po code review

### Kontext

Bylo provedeno uvodni code review pluginu v `wp-content/plugins` se zamerenim na:

- bezpecnost vsech pluginu
- zjevne chyby v custom pluginech
- specialni fokus na:
  - `wp-content/plugins/contest-manager-universal`
  - `wp-content/plugins/ch-cleanup`

V tomto kroku nebyly provedeny zadne zmeny v produkcnim kodu projektu `contest`.
Byl pripraven navrh fixu a priorit.

### Hlavni nalezy

#### 1. Admin mutace bez nonce a CSRF ochrany

Custom plugin `contest-manager-universal` ma vice admin akci, ktere meni data, ale nejsou chranene standardnim WordPress nonce flow.

Typicke dotcene soubory:

- `wp-content/plugins/contest-manager-universal/includes/pages/contestants.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/ecomail-queue.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/prizes.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/periods.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/main-periods.php`

Riziko:

- prihlasenemu adminovi lze podstrcit nechtenou mutaci pres odkaz nebo formular
- nejde o chybu losovani, ale o bezpecnostni slabinu administrace

Navrh fixu:

- vsechny mutace pres formular obalit `wp_nonce_field`
- serverove overovat `check_admin_referer`
- GET mutace prevadet na POST nebo admin AJAX
- zachovat stavajici business logiku a menit jen transport a ochranu

Dopad na soutez a losovani:

- nulovy nebo zanedbatelny
- jde o bezpecnostni hardening kolem admin akci

#### 2. Nekonzistentni model `cm_receipts`

Schema tabulky `cm_receipts` v `wp-content/plugins/contest-manager-universal/includes/install.php` negarantuje unikatnost na `contestant_id`, ale zapis v `wp-content/plugins/contest-manager-universal/includes/cf7-handler.php` pouziva `INSERT ... ON DUPLICATE KEY UPDATE`, jako by takovy unikni klic existoval.

Riziko:

- opakovane nahrani uctenky muze vytvorit vice radku pro jednoho souteziciho
- nasledne joiny muzou duplikovat radky ve:
  - vyhercich
  - odesilani
  - dashboardu
  - frontendu s vyherci

Navrh fixu:

- nejdriv audit existujicich dat
- navrhnout deduplikaci historickych radku
- pridat unikatni klic na `contestant_id`
- zachovat logiku upsertu, kterou autor pluginu evidentne zamyslel

Dopad na soutez a losovani:

- losovani samotne na `cm_receipts` nestoji
- fix je bezpecny, pokud se schema zmena provede az po deduplikaci dat

#### 3. `disable_xmlrpc` v `ch-cleanup` neni zapojeny

V `wp-content/plugins/ch-cleanup/ch-cleanup.php` existuje metoda pro vypnuti XML-RPC, ale neni pripojena na odpovidajici WordPress filter.

Riziko:

- UI tvrdi, ze funkce existuje
- ve skutecnosti nic nedela

Navrh fixu:

- dopojit metodu na standardni filter `xmlrpc_enabled`

Dopad na contest:

- zadny

#### 4. Capability `view_shipping_page` a `view_report_page` nejsou bootstrapovane

Plugin `contest-manager-universal` tyto capability pouziva v menu i ochrane AJAX endpointu, ale nikde je nevytvari ani nepridava rolím.

Riziko:

- na ciste instalaci budou tyto sekce bez dalsiho rucniho zasahu nepouzitelne
- chovani zavisi na externi konfiguraci mimo plugin

Navrh fixu:

- pri aktivaci pluginu capability vytvorit a priradit administratorum
- doplnit i bezpecny runtime backfill pro pripad, ze plugin uz aktivni je

Dopad na soutez:

- pozitivni stabilizace opravneni
- bez zmeny logiky losovani

#### 5. Cron secret je natvrdo v kodu i v admin rozhrani

`cron-runner.php` obsahuje natvrdo zapsany tajny klic a admin stranka na nej primo odkazuje.

Riziko:

- citlivy tajny token je vystaveny v kodu i HTML
- kdokoliv s pristupem do adminu vidi produkcni secret

Navrh fixu:

- odpojit admin tlacitko od primeho URL s tajnym parametrem
- admin spousteni resit nonce-protected akci
- secret presunout mimo repozitar nebo mimo renderovany vystup

Dopad na soutez:

- zadny primy
- tyka se e-mailove fronty a provozni bezpecnosti

### Dalsi poznamky

- Losovani v `contest-manager-universal` je uz dnes v kritickych AJAX endpointech chranene nonce a capability checkem.
- Nejcitlivejsi cast neni samotne losovani, ale:
  - stavove prechody vyherce
  - vazba na uctenky
  - integrity dat v `cm_receipts`

### Aktualni navrh realizace

1. Nejdriv low-risk fixy:
   - CSRF
   - capability bootstrap
   - `disable_xmlrpc`
   - cron secret
2. Pak audit a oprava `cm_receipts`.
3. Nakonec regresni testy nejdulezitejsich flow:
   - registrace souteziciho
   - losovani
   - potvrzeni vyherce
   - nahrani uctenky
   - schvaleni uctenky
   - odeslani vyhry
   - Ecomail queue

### Stav

- Dokumentace zalozena.
- Nalezy zapsany.
- Implementace zatim nezahajena.

## 2026-03-30 - Implementace bezpečnostních a schema fixů

### Kontext

Byla provedena implementace fixu v custom pluginech `contest-manager-universal` a `ch-cleanup` se zamerenim na:

- bezpecnost admin mutaci
- konzistenci `cm_receipts`
- opravneni pro shipping/report sekce
- odstraneni hardcoded cron secretu
- opravu dalsich nalezenych chyb bez zasahu do business logiky losovani

Pred implementaci probehl audit lokalni DB:

- `wp_cm_contestants = 0`
- `wp_cm_winners = 0`
- `wp_cm_receipts = 0`

To znamenalo, ze lokalni schema fix bylo mozne provest bez rizika kolize s historickymi daty.

### Co bylo upraveno

#### 1. Capability bootstrap a schema/runtime bootstrap v `contest-manager-universal`

Soubor:

- `wp-content/plugins/contest-manager-universal/index.php`

Zmena:

- doplneny konstanty `CONTEST_MANAGER_SCHEMA_VERSION` a `CONTEST_MANAGER_CAPS_VERSION`
- `includes/install.php` se nove nacita vzdy, ne jen pri aktivaci
- aktivace pluginu nove:
  - spousti `contest_plugin_install(true)`
  - registruje custom capabilities
  - flushuje rewrite rules pro route `/v/{cnp}`
- doplnen runtime backfill:
  - schema upgrade na `init`
  - capability backfill na `admin_init`
- doplnen nonce-protected admin action pro rucni spusteni Ecomail fronty

Duvod:

- fix chybejicich capabilities
- fix upgradu starsich instalaci bez deaktivace/reaktivace pluginu
- zachovani funkcnosti upload linku pro vyherce

Dopad:

- bez zmeny logiky losovani
- shipping/report menu je deterministicky dostupne administratorovi

#### 2. Oprava `cm_receipts` schema a kompatibilni migrace

Soubor:

- `wp-content/plugins/contest-manager-universal/includes/install.php`

Zmena:

- do SQL definice `cm_receipts` doplneno:
  - `UNIQUE KEY uniq_contestant_id (contestant_id)`
  - `KEY idx_cnp (cnp)`
- doplneny helpery:
  - nacitani/generovani cron secretu
  - deduplikace `cm_receipts` podle `contestant_id`
  - merge duplikatnich radku se zachovanim nejvyspelejsiho stavu
  - explicitni backfill indexu pres `ALTER TABLE`, ne jen pres `dbDelta`
- plugin si zapisuje `contest_plugin_schema_version`

Duvod:

- `cf7-handler.php` pouziva upsert semantiku, kterou stare schema negarantovalo
- samotne `dbDelta` se pri existujici tabulce nechovalo dostatecne deterministicky pro index backfill

Dopad:

- jedna uctenka na jednoho souteziciho
- odstraneni rizika duplicitnich joinu ve vyhercich, odesilani a frontendu
- pripraveno i pro starsi instalace s historickymi daty

#### 3. CSRF hardening admin mutaci

Soubory:

- `wp-content/plugins/contest-manager-universal/includes/pages/contestants.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/ecomail-queue.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/prizes.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/periods.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/main-periods.php`

Zmena:

- doplneno `check_admin_referer`
- doplneno `wp_nonce_field` u POST formularu
- GET mutace dostaly nonce-protected URL
- stranky dostaly explicitni capability guard

Duvod:

- zruseni admin akci, ktere slo podstrcit bez CSRF ochrany

Dopad:

- nulovy dopad na business logiku
- zmena pouze v ochrannem obalu admin akci

#### 4. Cron secret a rucni spousteni Ecomail fronty

Soubory:

- `wp-content/plugins/contest-manager-universal/includes/pages/ecomail-queue.php`
- `cron-runner.php`
- `wp-content/plugins/contest-manager-universal/includes/install.php`
- `wp-content/plugins/contest-manager-universal/index.php`

Zmena:

- admin uz neodkazuje na verejne URL s tajnym parametrem
- admin pouziva nonce-protected `admin-post` akci
- `cron-runner.php` uz neobsahuje hardcoded secret
- secret se generuje a uklada do WP option `contest_manager_cron_secret`

Duvod:

- odstraneni tajne hodnoty z repozitare a HTML vystupu

Dopad:

- zadny dopad na losovani
- provozne je potreba pri nasazeni potvrdit nebo upravit externi serverovy cron

#### 5. `disable_xmlrpc` v `ch-cleanup`

Soubor:

- `wp-content/plugins/ch-cleanup/ch-cleanup.php`

Zmena:

- dopojeni metody `maybe_disable_xmlrpc` na filter `xmlrpc_enabled`

Duvod:

- puvodni prepinac v UI byl mrtvy

Dopad:

- bez vazby na contest flow

#### 6. Opravy dalsich chyb nalezenych behem implementace

Soubory:

- `wp-content/plugins/contest-manager-universal/includes/ajax-handler.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/report.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/winners.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/sending-page.php`

Zmena:

- `ajax-handler.php` uz pri include neposila globalni JSON hlavicku a pri primem pristupu korektne konci `exit`
- doplnen `delete_main_period_ajax`
- `report.php` uz nema duplicitni deklaraci `chcr_week_bounds`
- odstraneny mrtve POST handlery v `winners.php` a `sending-page.php`

Duvod:

- slo o realne chyby nebo technicky dluh, ktery mohl pusobit nepredvidatelne chovani

Dopad:

- bez zmeny logiky losovani
- vyssi predvidatelnost adminu a reportu

### Jak bylo overeno, ze se nic nerozbilo

Byly provedeny tyto kontroly:

- `php -l` nad vsemi zmenenymi PHP soubory
- WordPress runtime bootstrap:
  - `contest_plugin_install()`
  - `contest_manager_register_capabilities()`
- overeni options:
  - `contest_plugin_schema_version = 2`
  - `contest_manager_caps_version = 1`
  - `contest_manager_cron_secret` existuje a ma delku 32
- overeni capabilities administratora:
  - `view_shipping_page = true`
  - `view_report_page = true`
- overeni XML-RPC:
  - `apply_filters('xmlrpc_enabled', true)` vraci vypnuto
- overeni DB schema:
  - `SHOW CREATE TABLE wp_cm_receipts`
  - `SHOW INDEX FROM wp_cm_receipts`
  - potvrzen `uniq_contestant_id`
  - potvrzen `idx_cnp`
- overeni `cron-runner.php`:
  - bez `key` vraci `Unauthorized`
  - se spravnym generovanym secret vraci uspesne spusteni

### Dotcene soubory

- `wp-content/plugins/contest-manager-universal/index.php`
- `wp-content/plugins/contest-manager-universal/includes/install.php`
- `wp-content/plugins/contest-manager-universal/includes/ajax-handler.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/contestants.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/ecomail-queue.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/prizes.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/periods.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/main-periods.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/winners.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/sending-page.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/report.php`
- `wp-content/plugins/ch-cleanup/ch-cleanup.php`
- `cron-runner.php`

### Aktualni stav

- Bezpecnostni a schema fixy implementovany.
- Lokalni testy prosly.
- Otevrene zustavaji uz jen navazne provozni a produktove body:
  - ACF PRO update
  - potvrzeni finalni produkcni cron konfigurace
  - rozhodnuti o `main-periods`
  - pripadne dokonceni report exportu


## 2026-03-30 - Dokonceni `main-periods`, hardening provozu bez zasahu na hostingu a cleanup kritickych toku

### Kontext

Na zaklade dalsi analyzy manualu a projektu bylo rozhodnuto:

- dokoncit `main-periods` jako realnou funkcni feature
- odstranit zavislost bezne funkcnosti na externim serverovem cronu
- dotahnout cleanup a refactor tak, aby nebylo potreba delat zadne dalsi serverove nebo hostingove zasahy mimo projekt
- zachovat business logiku losovani bez zmeny vyberoveho modelu soutezicich

Pri teto vlne implementace byla lokalni DB znovu potvrzena jako prazdna, takze bylo mozne bezpecne delat smoke testy se seed daty a naslednym cleanupem.

### Co bylo upraveno

#### 1. `main-periods` byly dodelany jako realna soucast pluginu

Soubory:

- `wp-content/plugins/contest-manager-universal/index.php`
- `wp-content/plugins/contest-manager-universal/includes/install.php`
- `wp-content/plugins/contest-manager-universal/includes/functions.php`
- `wp-content/plugins/contest-manager-universal/includes/ajax-handler.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/main-periods.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/draws.php`
- `wp-content/plugins/contest-manager-universal/assets/js/periods.js`
- `wp-content/plugins/contest-manager-universal/assets/js/draws.js`
- odstranena mrtva vetev `wp-content/plugins/contest-manager-universal/assets/js/main-periods.js`

Zmena:

- `cm_main_prize_periods` je nova standardni tabulka v instalaci/migraci pluginu
- admin stranka `Hlavni obdobi` je nova submenu polozka pluginu
- admin JS pro obdobi je sjednocen, main periods pouzivaji stejny overeny edit/delete flow jako bezna obdobi
- doplnen `update_main_period_ajax`
- helper `contest_manager_get_draw_scope()` centralizuje urceni rozsahu losovani pro:
  - daily
  - weekly
  - main
- hlavni losovani v `draws.php` i `ajax-handler.php` pouziva `cm_main_prize_periods`, pokud existuji zaznamy
- pokud `cm_main_prize_periods` nema zadna data, hlavni losovani fallbackuje na cele rozmezi beznych obdobi z `cm_periods`
- pri redraw preview i pri potvrzeni vyherce se nese `main_period_id`, aby se hlavni losovani drzelo spravneho rozsahu

Duvod:

- puvodni implementace `main-periods` byla technicky i produktove polovicata
- stranka existovala mimo menu, tabulka se tvorila az pri otevreni stranky a hlavni losovani ji ignorovalo
- to bylo neprijatelne pro ostry provoz a notarsky dohled nad losovanim

Dopad:

- logika filtrovani soutezicich pro losovani zustava stejna
- meni se pouze zdroj rozsahu pro hlavni losovani, pokud je feature `main-periods` realne nastavena
- fallback na `cm_periods` zachovava kompatibilitu pro stavajici projekty bez hlavnich obdobi

#### 2. Ecomail fronta byla predelana tak, aby projekt fungoval i bez externiho serveroveho cronu

Soubory:

- `wp-content/plugins/contest-manager-universal/includes/ecomail-handler.php`
- `wp-content/plugins/contest-manager-universal/includes/install.php`
- `wp-content/plugins/contest-manager-universal/includes/functions.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/ecomail-queue.php`
- `cron-runner.php`

Zmena:

- doplnen helper `contest_manager_is_ecomail_configured()`
- doplnen interni lock a throttling fronty pres transient + option timestamp
- plugin umi zpracovat Ecomail frontu automaticky pri beznych GET requestech pres:
  - `admin_init`
  - `template_redirect`
- automaticke zpracovani se nespousti pri AJAX, REST, cron ani POST requestech
- manualni admin trigger zustal zachovan
- `cron-runner.php` zustal jako volitelna externi cesta, ale uz neni nutny pro bezny provoz
- kvuli nulovemu dopadu na existujici nasazeni `cron-runner.php` akceptuje:
  - aktualni DB secret
  - puvodni legacy key
- fronta nove zpracovava nejstarsi pending emaily jako prvni a ignoruje radky s `ignored = 1`
- doplneny guardy proti fatalu pri chybejici Ecomail konfiguraci

Duvod:

- uzivatel explicitne pozadoval reseni bez potreby dalsich hostingu/server zasahu
- drivejsi fix s DB secretem sice zlepsil bezpecnost, ale stale pocital s externim cron jobem
- pred startem souteze bylo potreba mit bezpecnou a projektove sobestacnou variantu

Dopad:

- bezne fungovani queue uz neni zavisle na serverovem cronu
- zachovana je i zpetna kompatibilita pro pripad, ze externi cron uz existuje
- latence API volani se nepridava do POST a AJAX requestu

#### 3. Byly srovnany dalsi nekonzistence, ktere mohly v ostrém provozu zpusobovat chyby nebo matouci chovani

Soubory:

- `wp-content/plugins/contest-manager-universal/includes/cf7-handler.php`
- `wp-content/plugins/contest-manager-universal/includes/install.php`
- `wp-content/themes/contest/index.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/dashboard.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/contestants.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/winners.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/sending-page.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/ecomail-queue.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/periods.php`

Zmena:

- sjednocen adresar pro upload uctenek na `contest_uploads`
- odstranena duplicita CNP hidden field filtru mezi `functions.php` a `cf7-handler.php`
- validace data nakupu nyni nepada na prazdnem `cm_periods`
- frontend seznam vyhercu nově zahrnuje i stavy:
  - `Odesláno elektronicky`
  - `Předáno osobně`
- dashboard dotazy pouzivaji korektni `Daily/Weekly/Main`
- admin prehledy maji korektni mapovani sloupcu pro razeni a neopiraji se o neplatne aliasy
- formulare beznych obdobi uz nemaji neplatne hodnoty pro `input type=date`
- na Ecomail queue strance je warning pri chybejici konfiguraci

Duvod:

- tyto body samy o sobe nerozbily losovani, ale byly to realne provozni chyby nebo nekonzistence
- v ostrém provozu by zhorsovaly duveru v administraci a vysledky

Dopad:

- vyssi predvidatelnost adminu
- spravnejsi data ve frontendu a prehledech
- mensi prostor pro provozni chyby obsluhy

### Overeni

Bylo provedeno:

- `php -l` nad vsemi zmenenymi PHP soubory bez chyby
- bootstrap WordPressu po zmenach
- kontrola, ze:
  - tabulka `cm_main_prize_periods` existuje
  - administrator ma `view_shipping_page` i `view_report_page`
  - interni hooky pro Ecomail queue jsou registrovane na `admin_init` a `template_redirect`
  - `xmlrpc_enabled` vraci `false`
- kontrola `cron-runner.php`:
  - bez klice vraci `Unauthorized`
  - s legacy key funguje
  - s aktualnim DB secretem funguje
- smoke test nad lokalni prazdnou DB se seed daty a naslednym cleanupem:
  - `contest_manager_get_draw_scope()` vracel spravne rozsahy pro daily/weekly/main
  - admin stranka losovani renderovala hlavni obdobi i denni datumy
  - frontend dotaz na vypis vyhercu zahrnul stav `Odesláno elektronicky`
  - po testu zustaly vsechny contest tabulky znovu prazdne

### Otevrena rizika a dalsi kroky

- Nebyl proveden plny klikaci browser test administrace a frontend formulare.
- ACF PRO stale zustava otevreny bod pro update.
- Legacy kompatibilita cron key je zamerne ponechana kvuli nulovemu dopadu na stavajici nasazeni, ale po stabilizaci produkce je vhodne ji odstranit.
- Exporty v shipping/pending/report sekcich stale pouzivaji historicke UI a u casti z nich zustava otevrena otazka, zda je neprevest na interni export bez externich JS zavislosti.

## 2026-03-30 - Funkcni komentare v upravenych souborech

### Kontext

Uzivatel pozadoval doplnit komentar ke kazde upravene funkci ve formatu:

- `// -- HLAVNI POPIS | Detailnejsi strucny popis --`

Cilem bylo zvednout citelnost a auditovatelnost kodu pred startem souteze bez zasahu do business logiky.

### Dotcene soubory

- `wp-content/plugins/contest-manager-universal/index.php`
- `wp-content/plugins/contest-manager-universal/includes/install.php`
- `wp-content/plugins/contest-manager-universal/includes/functions.php`
- `wp-content/plugins/contest-manager-universal/includes/ecomail-handler.php`
- `wp-content/plugins/contest-manager-universal/includes/cf7-handler.php`
- `wp-content/plugins/contest-manager-universal/includes/ajax-handler.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/ecomail-queue.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/report.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/periods.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/main-periods.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/sending-page.php`
- `wp-content/plugins/contest-manager-universal/assets/js/draws.js`
- `wp-content/plugins/contest-manager-universal/assets/js/periods.js`
- `wp-content/plugins/ch-cleanup/ch-cleanup.php`
- `wp-content/themes/contest/index.php`

### Postup

1. Byl nacten aktualni seznam upravenych souboru a zmapovany vsechny dotcene deklarace funkci, metod a dulezitych callbacku.
2. Do PHP souboru byly doplneny komentare nad vlastni funkce, metody a relevantni anonymni callbacky v kritickych workflow castich.
3. Do upravenych JS souboru a inline skriptu byly doplneny komentare k funkcim a inicializacnim blokum, ktere ridi admin nebo frontend flow.
4. Probehla kontrola pokryti, aby v dotcenych upravenych souborech nezustaly neokomentovane deklarace s realnym provoznim vyznamem.
5. Nad vsemi dotcenymi PHP soubory byl znovu spusten `php -l` a nasledne byla synchronizovana sdilena dokumentace projektu.

### Zmena

- doplneny jednotne psane komentare nad vsechny dotcene pojmenovane PHP funkce a metody v upravenych souborech
- doplneny komentare i k dulezitym anonymnim callbackum tam, kde samy predstavuji samostatny workflow krok
- doplneny komentare k relevantnim frontend JS funkcim a inline callbackum v upravenych sablonach
- nebyla menena zadna business logika, SQL, stavove prechody ani losovaci pravidla

### Duvod

- pred startem souteze je potreba rychla orientace v kritickych castich pluginu pri kontrole, incidentu nebo notarni pritomnosti
- komentare maji zkracovat cas nutny pro audit bez rizika, ze se pri cteni spatne vylozi ucel funkci

### Dopad

- vyssi citelnost a rychlejsi orientace v kritickych workflow castich projektu
- nulovy funkcni dopad na soutez, losovani, Ecomail frontu i administraci
- lepsi pripravnost pro dalsi servisni zasahy a finalni predprodukci kontrolu

### Overeni

Bylo provedeno:

- `php -l` nad vsemi dotcenymi PHP soubory bez chyby
- rychla kontrola pokryti komentari nad pojmenovanymi funkcemi a relevantnimi callbacky v upravenych souborech
- kontrola, ze slo pouze o komentarovy refactor bez zasahu do chovani aplikace

## 2026-03-30 - Zavedeni commit konvence

### Kontext

Byla sjednocena commitovaci konvence pro projekt `contest`, aby dalsi historie byla konzistentni a dobre dohledatelna.

### Zavedene pravidlo

- Commity se cisluji od `CT-0001` vyse.
- Dalsi commit vzdy navazuje na nejvyssi jiz pouzite `CT-` cislo a zvysuje se o `+1`.
- Commit message se zapisuje ve formatu `CT-XXXX | Message in English`.
- Pri vice commitech v jednom tasku se pouzivaji po sobe jdoucí cisla bez preskakovani.

### Aktualni navazani

- V historii repozitare je aktualne nejvyssi potvrzene cislo `CT-0011`.
- Nasledujici commity tedy musi zacinat od `CT-0012`.

## 2026-03-30 - Rozdeleni zmen do commitu

### Kontext

Necommitnute zmeny byly po finalni kontrole rozdeleny do dvou logickych commitů, aby historie lepe oddelovala core soutezni logiku od provozni infrastruktury.

### Vytvorene commity

- `CT-0012 | Harden contest manager core flows and draw logic`
  - commit: `4778451`
  - obsahuje core zmeny v `contest-manager-universal`, schema/runtime bootstrap, `main-periods`, draw flow, receipt/winner navaznosti, admin hardening, frontend stavove opravy a souvisejici JS

- `CT-0013 | Stabilize ecomail queue cron runner and cleanup plugin`
  - commit: `626c229`
  - obsahuje zmeny kolem Ecomail fronty, manualniho/spolecneho cron flow, `cron-runner.php` a pluginu `ch-cleanup`

### Duvod rozdeleni

- lepsi orientace v historii
- snazsi budouci audit zmen s dopadem na losovani versus provozni infrastrukturu
- mensi riziko neprehlednych vseobjimajicich commitů v kritickem projektu
