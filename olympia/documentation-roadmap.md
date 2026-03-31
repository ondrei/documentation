# Olympia Roadmap

## 2026-03-31 - Uvodni analyza propojeni wordpress + contest

### Kontext

Byla provedena uvodni technicka analyza projektu `olympia` s cilem overit:

- zda je zaklad z `www/wordpress` kompatibilni se souteznim projektem `www/contest`
- jak bezpecne propojit theme `CreativeHeroes` a `contest`
- zda ma smysl ponechat `CH-cleanup` jako plugin, nebo cleanup presunout do jadra projektu

Analyza vychazela z:

- `www/wordpress`
- `www/contest`
- `www/documentation/contest/*`
- aktualniho stavu `www/olympia`

V tomto kroku nebyly provedeny zmeny produkcni logiky projektu `olympia`.
Byla provedena pouze analyza a zapsani zavěru do dokumentace.

### Overeny aktualni stav projektu

#### 1. WordPress jadro je ve vsech trech projektech shodne

- `olympia`, `wordpress` i `contest` bezi na `WordPress 6.9.4`
- zaklad `olympia` odpovida startovacimu projektu `wordpress`

#### 2. `olympia` je skutecne slozenina obou projektu, ale ne kompletni

Potvrzeno:

- `wp-content/themes/CreativeHeroes` v `olympia` je shodny s `wordpress`
- `wp-content/themes/contest` v `olympia` je shodny s `contest`
- `wp-content/mu-plugins/creativeheroes-security.php` v `olympia` je shodny s `wordpress`
- plugin set v `olympia` je skoro shodny s `contest`

Zasadni rozdil:

- v `olympia` chybi `wp-content/plugins/contest-manager-universal`
- v `olympia` je navic `wp-content/plugins/creativeheroes-media`

To znamena, ze aktualni `olympia` zatim nema prenesenou hlavni soutezni business logiku.

#### 3. `wp-config.php` v `olympia` odpovida `wordpress`, ne `contest`

Potvrzeno:

- `olympia/wp-config.php` je shodny se `wordpress/wp-config.php`
- oproti `contest/wp-config.php` chybi minimalne:
  - `WPCF7_AUTOP`
  - `ECOMAIL_API_KEY`
  - `ECOMAIL_LIST_ID`
  - `ECOMAIL_TEMPLATE_ID`
  - `DISABLE_WP_CRON`

Dopad:

- soutezni e-mailovy flow nebude po samotnem zkopirovani pluginu plne funkcni bez dalsi konfigurace
- `contest-manager-universal` umi Ecomail vypnout fallbackem, ale nepujde o funkcne ekvivalentni nasazeni vuci projektu `contest`

### Hlavni kompatibilitni nalezy

#### 1. Nejvetsi blocker je kolize `creativeheroes-security` vs. Contact Form 7 frontend

`CreativeHeroes` bezpecnostni mu-plugin blokuje pro neprihlasene uzivatele:

- cestu `wp-json`
- dotazy pres `rest_route`

Soucasne `Contact Form 7 6.1.5` na frontendu pouziva REST endpointy typu:

- `/wp-json/contact-form-7/v1/contact-forms/{id}/feedback`

Dopad:

- verejne soutezni formulare v aktualnim security setupu nebudou fungovat
- to se tyka jak registrace souteziciho, tak pravdepodobne i upload flow vyherce

Zaver:

- bez upravy `creativeheroes-security` neni mozne soutezni frontend na `CreativeHeroes` zakladu bezpecne zprovoznit

#### 2. Theme `contest` neni architektonicky kompatibilni jako cilovy aktivni theme

Theme `contest` je silne vazany na:

- ACF fieldy a options
- prime SQL dotazy do `cm_*` tabulek primo v sablone
- vlastni header/footer/layout
- vlastni SCSS assety
- vlastni jQuery-based frontend skript

Theme `CreativeHeroes` naopak stoji na:

- vlastnim Vite buildu
- Stylus asset pipeline
- modularnich template parts
- theme options pres vlastni settings API
- agresivnejsim cleanup/hardening bootstrapu

Zaver:

- theme `contest` neni vhodne dlouhodobe "sloucit" s `CreativeHeroes` na urovni dvou rovnocennych theme
- spravna cesta je ponechat `CreativeHeroes` jako jediny cilovy theme a soutezni frontend do nej portovat

#### 3. Soutezni vrstva je rozdelena mezi theme a plugin a musi se oddelit

V puvodnim `contest` projektu je soutezni funkcnost rozdelena mezi:

- `contest-manager-universal`
- theme `contest`
- ACF options zapsane v DB

Primo zavislosti:

- shortcode `[winner_upload_receipt]`
- ACF option keys pro texty, formulare, maily a menu
- theme sablona `index.php` obsahuje prime DB dotazy nad `cm_periods`, `cm_winners`, `cm_receipts`, `cm_prizes`

Zaver:

- pred dalsi implementaci je potreba rozdelit:
  - co patri do pluginu jako business logika
  - co patri do `CreativeHeroes` jako prezencni vrstva
  - co patri do kodove spravovane konfigurace ACF

#### 4. ACF konfigurace neni ve verzi v kodu

Nebyly nalezeny:

- `acf-json`
- exporty field groups v repozitari

To znamena, ze aktualni field groups jsou zavisle na DB stavu puvodnich projektu.

Dopad:

- bez exportu nebo kodove registrace neni projekt plne prenositelny
- dalsi faze musi obsahovat migraci ACF struktury do repozitare

### Analyza `CH-cleanup` vs. cleanup v zakladu `wordpress`

#### Shrnuty zaver

Plugin `CH-cleanup` nedoporucuji ponechat jako runtime plugin pro `olympia`.

Doporuceny smer:

- cleanup a hardening drzet jako standard v jadru projektu
- primarne v:
  - `CreativeHeroes/inc/cleanup.php`
  - `wp-content/mu-plugins/creativeheroes-security.php`
- pouze vybrane a bezpecne funkce z `CH-cleanup` doplnit primo do jadra

#### Proc nedavat `CH-cleanup` jako aktivni plugin

1. `CreativeHeroes` uz dnes pokryva velkou cast cleanup logiky:
   - emoji
   - comments off
   - feeds/head cleanup
   - oEmbed/head cleanup
   - attachment redirect
   - XML-RPC disable
   - pingback header cleanup
   - block/global styles cleanup

2. `creativeheroes-security` pokryva dalsi bezpecnostni vrstvu:
   - response headers
   - CSP
   - hidden login route
   - XML-RPC hard block
   - application passwords off
   - REST restrikce pro comments/users/oEmbed

3. Paralelni pouzivani `CH-cleanup` by vedlo k:
   - duplicitnim hookum
   - hur predvidatelnemu chovani
   - obtiznejsi diagnostice
   - zbytecne pluginove konfiguraci pro funkcnost, kterou chceme mit jako standard

#### Co ma `CH-cleanup` navic a dava smysl zvazit k prevzeti

Z pluginu `CH-cleanup` dava smysl zvazit presunuti jen vybranych prvku:

- `remove_jqmigrate`
- `dequeue_dashicons`
- `heartbeat_throttle`
- `hide_theme_editors`
- pripadne `generic_login_errors`

Naopak jako nedoporucene nebo podminene hodnotim:

- `preload_styles`
  - `CreativeHeroes` ma vlastni asset pipeline a rucni prepis vsech stylu na preload neni vhodne brat bez testu
- `remove_wp_version_from_assets`
  - nizka realna bezpecnostni hodnota, muze komplikovat diagnostiku
- `enable_svg_uploads`
  - jen pokud bude skutecna potreba a bude jasne urceny bezpecny workflow
- `disable_rest_public`
  - pro soutezni frontend spis riziko nez benefit
- `reorder_admin_menu`
  - neni to jadro projektu, spis volitelny UX doplnek

#### Cílovy cleanup pristup

Doporuceny cil:

- nepouzivat `CH-cleanup` jako plugin
- z `CH-cleanup` vyzobat jen overene prvky, ktere:
  - nebudou kolidovat s contest flow
  - nebudou duplikovat mu-plugin
  - budou davat smysl jako nemenny standard

### Doporuceny technicky postup

#### Faze 1 - stabilizace zakladu

1. Ponechat `CreativeHeroes` jako jediny cilovy aktivni theme.
2. Dointegrovat `contest-manager-universal` do `olympia`.
3. Upravit `creativeheroes-security`, aby pustil nezbytne verejne endpointy pro CF7 contest flow.
4. Rozhodnout, ktere soutezni konstanty a provozni nastaveni musi byt preneseny z `contest/wp-config.php`.

#### Faze 2 - oddeleni business logiky od stareho theme

1. Sepsat vsechny zavislosti stareho `contest` theme:
   - ACF fieldy
   - menu
   - options
   - shortcode flow
   - stranky a slugs
2. Presunout vsechny soutezni texty a konfigurace pod jasne spravovanou konfiguraci:
   - idealne ACF export do kodu nebo PHP registrace
3. Zachovat business logiku v pluginu, ne v sablonach.

#### Faze 3 - port soutezniho frontendu do `CreativeHeroes`

1. Vytvorit v `CreativeHeroes` specialni page template nebo soutezni komponentovou vrstvu.
2. Portovat frontend z `contest` do:
   - template parts
   - Stylus
   - Vite JS modulu bez jQuery
3. Nenechavat v sablone prime SQL dotazy bez zapouzdreni.

#### Faze 4 - cleanup standardizace

1. `CH-cleanup` nepouzivat jako soucast vysledneho stacku.
2. Vybrane funkce presunout do:
   - `CreativeHeroes/inc/cleanup.php`
   - `creativeheroes-security.php`
3. Po kazdem presunu otestovat soutezni flow:
   - registrace
   - redirect po submitu
   - upload uctenky
   - route `/v/{cnp}`
   - Ecomail queue

### Stav po tomto kroku

- Dokumentace `olympia` zalozena a naplnena uvodni analyzou.
- Kompatibilita byla proverena na urovni kodu a struktury projektu.
- Byly identifikovany hlavni blockery a doporuceny cilovy smer.
- Implementace zatim nezahajena.

## 2026-03-31 - Oddeleni do vlastni DB a contest bootstrap na CreativeHeroes zakladu

**Souvisejici commity:** `OL-0004 | Bootstrap contest manager integration`, `OL-0005 | Align Olympia runtime config and REST security`

### Kontext

Po uvodni analyze byl projekt `olympia` oddelen do vlastni databaze a nasledne byla provedena prvni realna integrační vlna pro `contest-manager-universal`, `ACF PRO` a `Contact Form 7` na aktivnim theme `CreativeHeroes`.

### Co bylo upraveno

#### 1. Projekt byl oddelen do vlastni databaze `olympia`

Soubory:

- `wp-config.php`

Zmena:

- vytvorena nova MySQL databaze `olympia`
- vytvoren a otestovan DB uzivatel `olympia`
- aktualni stav puvodni DB `wordpress_2026` byl zkopirovan do `olympia`
- `wp-config.php` byl prepnut na:
  - `DB_NAME = olympia`
  - `DB_USER = olympia`
- doplnen contest-oriented config bootstrap:
  - `WPCF7_AUTOP = false`
  - env-aware nacitani `ECOMAIL_*`
  - env-aware override `WP_HOME` a `WP_SITEURL`

Duvod:

- `olympia` nesmi dal sdilet databazi s jinym starter projektem
- contest a dalsi projektove zmeny musi byt izolovane
- bylo potreba pripravit cisty zaklad pro dalsi implementaci bez rizika prepisu jineho projektu

Dopad:

- vsechny dalsi zmeny v `olympia` se ukladaji do DB `olympia`
- runtime uz nepise do `wordpress_2026`

#### 2. Contest plugin dostal vlastni runtime bootstrap a prestal byt zavisly na rucnim nastavovani v DB

Soubory:

- `wp-content/plugins/contest-manager-universal/index.php`
- `wp-content/plugins/contest-manager-universal/includes/functions.php`
- `wp-content/plugins/contest-manager-universal/includes/install.php`
- `wp-content/plugins/contest-manager-universal/includes/acf-settings.php`

Zmena:

- doplnen runtime bootstrap verzovany pres `contest_manager_bootstrap_version`
- contest plugin si nove umi sam zajistit:
  - upload page `contest-upload`
  - verejnou registracni page `contest-registration`
  - contest-specific CF7 formulare
  - vazbu ID formularu do contest options
  - default shortcode pro vyherni formular
- contest settings byly zapsany do kodu pres `acf_add_local_field_group()`
- doplnen centralni helper pro contest options s fallback defaulty
- doplnen shortcode `[contest_registration_form]`

Duvod:

- puvodni contest stack byl zavisly na nezdokumentovanych ACF field groups v DB
- v nove DB nebyly zadne CF7 formulare ani ACF option fieldy
- bez teto vrstvy by plugin byl aktivni, ale prakticky nepouzitelny

Dopad:

- projekt je prenositelnejsi
- ACF field groups pro plugin uz nejsou zavisle na manualnim importu
- registracni i vyherni contest flow maji vlastni provisioned CF7 formulare

#### 3. Public winner flow byl adaptovan na CreativeHeroes a zbaven zavislosti na stare contest sablone

Soubory:

- `wp-content/plugins/contest-manager-universal/includes/winner-page.php`
- `wp-content/plugins/contest-manager-universal/includes/cf7-handler.php`
- `wp-content/plugins/contest-manager-universal/includes/ecomail-handler.php`
- `wp-content/plugins/contest-manager-universal/includes/email-texts.php`
- `wp-content/plugins/contest-manager-universal/includes/ajax-handler.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/contestants.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/prizes.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/pending-winners.php`

Zmena:

- winner shortcode nově:
  - pouziva centralni option helpery
  - renderuje neutralni markup vhodny pro `CreativeHeroes`
  - umi automaticky vyrenderovat CF7 shortcode podle ID formulare
- contest option cteni bylo sjednoceno mimo prime `get_field()` volani
- mena, sender identity, validation texty a email texty uz maji centralni fallbacky

Duvod:

- puvodni implementation byla navazana na starou ACF/contest theme vrstvu
- `winner_form` byl v praxi zavisly na raw ACF obsahu a nemel robustni fallback

Dopad:

- vyherni upload flow je pouzitelne i na `CreativeHeroes`
- contest plugin ma mene skrytych zavislosti na starem theme

#### 4. Security vrstva byla zpruhodnena pro nezbytne contest/CF7 REST flow

Soubor:

- `wp-content/mu-plugins/creativeheroes-security.php`

Zmena:

- doplneno rozpoznani pozadovaneho REST route
- doplnen allowlist pro verejne CF7 endpointy pod:
  - `/contact-form-7/v1/contact-forms/`
- hostum nadale zustavaji skryte ostatni systemove REST query requesty

Duvod:

- contest registrace i dalsi CF7 frontend submit flow musi jit pres verejny REST endpoint
- hardening nesmel zustat blanketni tak, aby rozbil legitimni soutezni submit

Dopad:

- verejny contest submit neni blokovan security vrstvou
- zachovava se princip minimalni verejne plochy

### Aktivovane pluginy v `olympia`

- `advanced-custom-fields-pro/acf.php`
- `contact-form-7/wp-contact-form-7.php`
- `contest-manager-universal/index.php`
- `creativeheroes-media/creativeheroes-media.php`

### Overeni

Bylo provedeno:

- `php -l` nad vsemi zmenenymi PHP soubory bez chyby
- potvrzeni aktivni DB:
  - `DB_NAME = olympia`
  - `DB_USER = olympia`
- potvrzeni bootstrap stavu:
  - `contest_manager_bootstrap_version = 1`
  - `contest_plugin_schema_version = 3`
  - `contest_manager_caps_version = 1`
- kontrola vytvorenych contest tabulek:
  - `wp_cm_contestants`
  - `wp_cm_periods`
  - `wp_cm_main_prize_periods`
  - `wp_cm_receipts`
  - `wp_cm_winners`
  - `wp_cm_prizes`
  - `wp_cm_ecomail`
  - `wp_cm_disqualified`
- kontrola upload page:
  - existuje `contest-upload`
  - obsahuje `[winner_upload_receipt]`
- kontrola registracni page:
  - existuje `contest-registration`
  - obsahuje `[contest_registration_form]`
- kontrola provisioned CF7 formularu:
  - `Soutěžní registrace`
  - `Nahrání výherní účtenky`
- smoke test registracniho shortcode:
  - `[contest_registration_form]` renderuje validni CF7 markup
- smoke test winner shortcode:
  - nad docasnym testovacim CNP renderuje vyherni formular
- REST smoke test:
  - contest CF7 endpoint neni blokovan `creativeheroes-security`

### Stav po tomto kroku

- `olympia` uz bezi na vlastni DB
- core contest stack je v `olympia` realne aktivni a bootstrapnuty
- pluginove a settings zavislosti pro prvni contest flow jsou srovnane
- vizualni port celeho soutezniho frontendu ze stareho theme do `CreativeHeroes` zatim jeste neni hotovy

## 2026-03-31 - Dokonceni request vrstvy a provozni contest integrace na `olympia.loc`

**Souvisejici commit:** `OL-0006 | Fix Olympia root WordPress rewrites`

### Kontext

Po prvni bootstrap fazi se ukazalo, ze contest page sice existuji, ale verejne routy nebyly provozne stabilni:

- `contest-registration` vracela `500`
- `contest-upload` vracela `500`
- route `/v/{cnp}` s neplatnym kodem vykreslovala zanorenou `404.php` uvnitr bezne page sablony

Soucasne porad chybela jemnejsi integrace contest shortcodu do vizualni vrstvy `CreativeHeroes` a contest dashboard neumel jasne rict, ktera konfigurace jeste chybi.

### Co bylo upraveno

#### 1. Oprava root rewrite vrstvy pro novy projekt

Soubor:

- `.htaccess`

Zmena:

- prepsan zdedeny WordPress rewrite z puvodniho `wordpress` projektu:
  - `RewriteBase /wordpress/` -> `RewriteBase /`
  - `RewriteRule . /wordpress/index.php [L]` -> `RewriteRule . /index.php [L]`

Duvod:

- projekt uz neni instalace pod `/wordpress/`
- zdedena `.htaccess` zpusobovala Apache rewrite loop a `AH00124`

Dopad:

- `contest-registration` a `contest-upload` uz vraceji korektni `200`
- request vrstva odpovida realne lokalni URL `http://olympia.loc`

#### 2. Contest plugin dostal frontend fallbacky a diagnostiku konfigurace

Soubory:

- `wp-content/plugins/contest-manager-universal/includes/functions.php`
- `wp-content/plugins/contest-manager-universal/includes/pages/dashboard.php`
- `wp-content/plugins/contest-manager-universal/assets/css/main.css`

Zmena:

- doplneny helpery:
  - `contest_manager_build_front_button()`
  - `contest_manager_build_admin_button()`
  - `contest_manager_render_front_notice()`
  - `contest_manager_get_table_row_count()`
  - `contest_manager_get_setup_status()`
- registracni shortcode nove:
  - vraci strukturovany notice stav misto holeho odstavce
  - rozlisuje chybejici CF7 formular a chybejici contest periods
  - umi zobrazit admin CTA jen prihlasenemu adminovi
  - obaluje aktivni formular do konzistentniho shell markup
- contest dashboard nove zobrazuje integracni checklist:
  - stav obou CF7 formularu
  - pocet beznych a hlavnicich obdobi
  - pocet vyher
  - stav Ecomail env konfigurace
  - rychle odkazy do konfiguracnich obrazovek

Duvod:

- po bootstrapu bylo treba videt, zda je contest jen technicky nainstalovany, nebo i skutecne nakonfigurovany
- bez tohoto kroku bylo verejne chovani funkcni, ale ne dostatecne citelne pro dalsi provozni praci

Dopad:

- chyba konfigurace je ted citelna jak na frontendu, tak v administraci
- chybejici soutezni data uz nejsou zamaskovana jako technicka chyba

#### 3. Winner route uz nepouziva zanozenou `404.php` uvnitr shortcode

Soubor:

- `wp-content/plugins/contest-manager-universal/includes/winner-page.php`

Zmena:

- pri chybejicim `cnp` se nove vraci standardni contest notice
- pri neplatnem `cnp` se stale nastavi HTTP `404`, ale misto include stare `404.php` se vrati:
  - contest-specific error notice
  - CTA odkaz na registracni stranku
- stav po jiz odeslane uctence vraci success notice
- aktivni vyherni formular je zasazen do stejneho shell layoutu jako registrace

Duvod:

- puvodni `include get_template_directory() . '/404.php'; exit;` rozbijel page layout a duplikoval HTML strukturu uvnitr `CreativeHeroes`

Dopad:

- `/v/{neplatny-kod}` vraci korektni `404`
- frontend uz nema zanozenou druhou stranku uvnitr editor obsahu

#### 4. Contest frontend dostal vlastni prezencni vrstvu kompatibilni s `CreativeHeroes`

Soubory:

- `wp-content/plugins/contest-manager-universal/index.php`
- `wp-content/plugins/contest-manager-universal/assets/css/frontend.css`

Zmena:

- doplnen conditional enqueue `contest_manager_enqueue_frontend_assets()`
- novy stylesheet se nacte jen na:
  - `contest-registration`
  - `contest-upload`
  - route s `cnp`
  - nebo stranky s contest shortcode
- novy frontend stylesheet resi:
  - contest notice karty
  - winner/registration shell layout
  - CF7 fieldy, checkboxy, file inputy, submit tlacitko
  - validacni a response vrstvy

Duvod:

- `CreativeHeroes` snapshot v projektu obsahuje jen zbuildene assety bez Stylus zdroju
- bylo potreba contest shortcody integrovat vizualne, ale bez rozbijeni existujiciho theme bundle

Dopad:

- contest verejne stranky uz nevypadaji jako syrovy CF7 dump
- frontend asset se nacta jen tam, kde je opravdu potreba

### Overeni

Bylo provedeno:

- `php -l` bez chyby nad:
  - `includes/functions.php`
  - `includes/winner-page.php`
  - `includes/pages/dashboard.php`
  - `index.php`
- `curl -si http://olympia.loc/contest-registration/`
  - vraci `200`
  - nacte `contest-manager-universal/assets/css/frontend.css`
  - zobrazuje strukturovany warning o chybejicich contest periods
- `curl -si http://olympia.loc/contest-upload/`
  - vraci `200`
  - zobrazuje korektni warning pri primem vstupu bez `cnp`
- `curl -si http://olympia.loc/v/SMOKE123/`
  - vraci `404`
  - zobrazuje jednotny contest error notice bez zanozene druhe `404` sablony
- `curl -si -X POST -F ... http://olympia.loc/wp-json/contact-form-7/v1/contact-forms/17/feedback`
  - vraci `200`
  - endpoint neni blokovan security vrstvou
  - odpovida standardnim `validation_failed`, tedy CF7 flow je verejne dostupne
- `curl -si http://olympia.loc/`
  - nepotvrzuje nacteni `contest-manager-frontend.css`
  - frontend contest asset se tedy nenacita globalne

### Stav po tomto kroku

- finalni lokalni URL je potvrzena jako `http://olympia.loc`
- zdedeny rewrite loop z `wordpress` zakladu je odstraneny
- verejne contest routy jsou funkcni a bez `500`
- invalidni winner route ma korektni `404` chovani
- contest plugin ma operacni onboarding vrstvu v dashboardu
- stale chybi realna soutezni data:
  - contest periods
  - prizes
  - pripadne navazujici provozni texty a Ecomail env

## 2026-03-31 - Presun CF7 frontend stylu do CreativeHeroes Stylus pipeline

**Souvisejici commity:** `OL-0009 | Integrate contest frontend into CreativeHeroes`, `OL-0010 | Harden contest form processing and winner flow`

### Kontext

Bylo rozhodnuto, ze frontend Contact Form 7 stylu nebudeme nacitat z pluginu ani z docasne contest plugin vrstvy.
Cilem bylo:

- odstranit `contact-form-7/includes/css/styles.css` z verejne hlavy
- odstranit docasny `contest-manager-universal/assets/css/frontend.css`
- prenest soutezni formularovy styling do hlavniho `CreativeHeroes` asset pipeline
- sjednotit markup contest formularu na projektove BEM tridy, kde to CF7 bezpecne dovoluje

### Provedene zmeny

- v `src/php/inc/assets.php` byl doplnen filtr `wpcf7_load_css`, ktery vypina frontend CSS Contact Form 7 mimo admin
- contest plugin uz nenačita vlastni `frontend.css`; soubor byl odstraneny
- vznikl novy zdrojovy soubor `src/assets/css/global/form.styl`
- `src/assets/css/css.styl` nove importuje `global/form`
- contest registracni a winner CF7 formularove sablony byly prepsany na BEM markup vrstvu:
  - `.form`
  - `.form__grid`
  - `.form__item`
  - `.form__label`
  - `.form__control`
  - `.form__input`
  - `.form__actions`
  - `.form__submit`
- render CF7 formulare je centralizovan tak, aby do `<form>` pridaval projektove `html_class` a `html_id`
- bootstrap verze contest pluginu byla zvysena na `3`, aby se nove formularove sablony propsaly i do existujicich CF7 formularu v DB
- sync formularu nyni neaktualizuje jen `form` markup, ale i `additional_settings`
- GDPR acceptance field byl prepsan na validni CF7 syntax a formulare maji zapnute `acceptance_as_validation: on`

### Overeni

Bylo overeno:

- v hlave `http://olympia.loc/contest-registration/` se uz nenasita:
  - `contact-form-7/includes/css/styles.css`
  - `contest-manager-universal/assets/css/frontend.css`
- verejny frontend nacita pouze buildovany stylesheet `wp-content/themes/CreativeHeroes/assets/css/style-*.css`
- ulozene CF7 formulare v DB uz obsahuji novy BEM markup
- vyrenderovany HTML vystup CF7 formularu obsahuje:
  - `class="form form--contest-registration"`
  - `class="form form--contest-winner"`
  - BEM field wrapptery `form__item`, `form__label`, `form__control`, `form__input`
- `npm run build` probehl uspesne
- `npm run lint:styles` po autofixu probehl uspesne

### Dulezite implementacni omezeni

- Contact Form 7 stale vnitrne generuje nektere vlastni wrappery `wpcf7-*`
- tyto internaly zustavaji zachovany kvuli kompatibilite validace, spinneru, response output a JS submit flow
- BEM vrstva je proto aplikovana na vsech autorovanych obalech a kontrolach, zatimco nevyhnutelne CF7 internaly jsou jen podchycene explicitnimi selektory uvnitr `form.styl`

## 2026-03-31 - Verejny contest page template v CreativeHeroes

**Souvisejici commit:** `OL-0009 | Integrate contest frontend into CreativeHeroes`

### Kontext

Po presunu CF7 formulare a jejich stylu do hlavni theme pipeline zustavaly `contest-registration` a `contest-upload` stale jen bezne WordPress stranky se shortcodem uvnitr standardni page sablony.
V tomto kroku byla vytvorena samostatna verejna prezencni vrstva pro contest slugy primo v theme `CreativeHeroes`.

### Provedene zmeny

- do theme byl pridan novy helper `inc/contest.php`
- `page.php` v `CreativeHeroes` se nyni umi vetvit podle contest slug variant:
  - `contest-registration`
  - `contest-upload`
- pro contest pages vznikl novy template part `template-parts/contest/page-shell.php`
- nova verejna contest sablona renderuje:
  - samostatny hero blok se stavem souteze
  - metriky z `contest-manager-universal`
  - hlavni obsahovou oblast s registraci nebo winner flow
  - sidebar s procesnimi kroky a checklistem
  - kontaktni support panel
  - u registrace i sekci s vyhrami, pokud jsou vyplnene v `cm_prizes`
- do frontend JS byl doplnen modul `contestCountdown.js` pro countdown do startu souteze
- do Stylus pipeline byl doplnen novy component `component/contest.styl`

### Datove zdroje nove sablony

Verejna contest sablona nyni cte provozni data primo z aktualni integrace, bez zavislosti na starem ACF frontendu z theme `contest`:

- stav registrace z `contest_manager_get_overall_contest_range()`
- setup readiness z `contest_manager_get_setup_status()`
- pocty registraci, vyhercu, prize banky a obdobi z `cm_*` tabulek
- prize grouping z `cm_prizes`
- kontaktní e-mail z contest options nebo fallback `admin_email`

### Overeni

Bylo overeno:

- `contest-registration` se renderuje pres novou contest page sablonu v `CreativeHeroes`
- `contest-upload` se renderuje pres novou contest page sablonu v `CreativeHeroes`
- invalidni `v/{cnp}` stale vraci `404 Not Found`, ale uz v novem contest layoutu
- frontend nacita novy build `style-*.css` a `main-*.js`
- `npm run lint:styles` probehl uspesne
- `npm run build` probehl uspesne

### Dulezite omezeni

- stary verejny frontend z theme `contest` jeste neni portovan 1:1 obsahove
- nova sablona je zamerne postavena nad dostupnou provozni logikou a databazovymi daty, nikoli nad puvodnimi ACF sekcemi `how to win`, `products`, `rules` apod.
- pro plnou obsahovou paritu bude potreba rozhodnout, ktere casti puvodniho ACF frontendu ma smysl do `CreativeHeroes` vratit a ktere uz ne


## 2026-03-31 - Důsledná analýza CM Manualu proti aktuálnímu stavu olympia

**Souvisejici commit:** `OL-0007 | Add CM Manual source files`

### Kontext

Byla provedena důsledná kontrola lokálně nahraného manuálu `src/docs/CM Manual/`:

- `CM Manual.fig`
- `CM Manual.pdf`
- `CM Manual.xml`

Prakticky použitelným zdrojem pro analýzu byl hlavně XML export, ze kterého šlo vytáhnout celý textový obsah manuálu. Manuál obsahuje několik vrstev:

- single contest workflow pro `Contest Manager Universal`
- rozcestník / multi contest workflow
- české a slovenské varianty formulářů

Pro `olympia` je v této fázi relevantní především single contest český workflow. Multi contest a slovenské varianty zatím nejsou pro aktuální implementaci povinné.

### Co je vůči manuálu připravené nebo funkčně ekvivalentní

- WordPress běží na vlastní DB `olympia` a contest plugin je aktivní.
- Existují a fungují contest stránky:
  - `contest-registration`
  - `contest-upload`
  - winner route `v/{cnp}`
- Pretty permalinky jsou nastavené na `/%postname%/`, což manuál pro contest flow vyžaduje.
- `cron-runner.php` je v rootu projektu přítomen.
- Contact Form 7 formuláře pro registraci a pro winner upload existují a jsou bootstrapované pluginem.
- ACF field groups pro contest plugin už nejsou závislé na ručním importu `acf-export.json`, protože jsou registrované v kódu přes `acf_add_local_field_group()`.
- Veřejný CF7 REST submit není blokovaný security vrstvou.
- Contest login hardening je vyřešen kvalitativně lépe než v manuálu:
  - místo `WPS Hide Login` běží custom secure route `/creative/admin/`
  - `/login/` pouze přesměrovává na tuto zabezpečenou route
- Upload page není vázaná na starou page template z theme `contest`, ale je renderovaná přes novou contest shell vrstvu v `CreativeHeroes`.

### Co je proti manuálu stále nedokončené nebo chybí

- V DB nejsou vyplněna soutěžní období.
  - bez nich zůstává registrace správně v `setup` stavu
- V DB nejsou vyplněny výhry.
  - bez nich nefunguje reálný contest bank a prize prezentace
- V `wp-config.php` stále chybí `DISABLE_WP_CRON`.
  - manuál ho výslovně požaduje
- Ecomail není nakonfigurovaný.
  - nejsou definované `ECOMAIL_API_KEY`, `ECOMAIL_LIST_ID`, `ECOMAIL_TEMPLATE_ID`
  - transakční fronta sice existuje, ale reálně se neodešle
- Neexistují dedikované role pro obsluhu provozu:
  - `view_report_page`
  - `view_shipping_page`
  - plugin zatím přidává capability jen administrátorovi
- Neexistuje GDPR stránka ani uložená GDPR URL.
  - manuál s touto vazbou počítá explicitně
- Neexistují thank-you stránky.
  - aktuální olympia implementace používá inline success flow místo redirect/TP modelu z manuálu
- Domovská stránka zatím není contest landing podle manuálu.
  - homepage je stále obecný `CreativeHeroes` starter
  - chybí contest sekce typu registrace, výhry, jak soutěžit, pravidla, případně produkty
- Není připraven age gate flow.
  - to je v pořádku pouze tehdy, pokud ho projekt reálně nepotřebuje
- Support / admin kontaktní vrstva stále používá lokální fallbacky typu `admin@localhost.test`
  - to je v pořádku jen pro development, ne pro provozní připravenost

### Důležité vědomé odchylky od manuálu

Tyto body nejsou samy o sobě chyba, ale je potřeba je brát jako vědomou změnu architektury oproti původnímu manuálu:

- ACF import z JSON byl nahrazen field groups v kódu.
- WPS Hide Login byl nahrazen vlastní bezpečnostní login vrstvou.
- Upload a registration stránky neběží přes staré contest page templates, ale přes novou theme integraci v `CreativeHeroes`.
- Thank-you pages nejsou použity; místo nich je nyní inline response flow přes CF7 a plugin handler.

### Verdikt

Aktuální `olympia` základ není ve stavu "vše připraveno podle manuálu", ale je ve stavu:

- technický základ contest integrace je hotový
- architektura je stabilní a ve více bodech čistší než původní manuál
- stále ale chybí několik provozně zásadních vrstev, bez kterých nelze tvrdit plnou připravenost

Největší otevřené blokery jsou nyní:

1. contest data v DB
2. Ecomail konfigurace a cron režim
3. role a provozní workflow
4. GDPR / rules / obsahové stránky
5. contest homepage a obsahová parita veřejného frontendu


### 2026-03-31 | Bezpečnostní dotažení login aliasů a redukce plugin stacku

Byla dotažená bezpečnostní vrstva kolem přístupu do administrace a zároveň zúžen plugin stack:

- veřejné aliasy `/login`, `/login.php`, `/admin`, `/dashboard` už nevedou na WordPress interní routy, ale vrací `404`
- jediný veřejný login vstup zůstává `/creative/admin`
- neaktivní pluginy `wps-hide-login` a `ch-cleanup` byly fyzicky odstraněné z `wp-content/plugins`
- přímý přístup na systémové directory rooty typu `/wp-content/`, `/wp-content/plugins/`, `/wp-content/themes/` a rooty konkrétních plugin/theme adresářů je nově blokovaný na webserverové vrstvě přes `.htaccess`
- současně zůstávají funkční legitimní veřejné assety pluginů a theme, např. CF7 JS a theme bundle

#### Dopad na architekturu

Tímto se potvrzuje, že:

- `WPS Hide Login` už projekt nepotřebuje ani jako fallback
- `CH-cleanup` už projekt nepotřebuje, protože cleanup/hardening je řešený v jádru olympia základu a MU security vrstvě
- veřejná attack surface se zmenšila bez zavádění dalšího pluginu

#### Aktuální doporučení ke pluginům

Nutné aktivní pluginy nyní jsou:

- `advanced-custom-fields-pro`
- `contact-form-7`
- `contest-manager-universal`
- `creativeheroes-media`
- MU plugin `creativeheroes-security.php`

Nenutné / dnes neaktivní a kandidáti na další redukci jsou:

- `akismet`
- `autodescription`
- `classic-editor`
- `error-log-monitor`
- `user-role-editor`
- `wpcf7-redirect`
- `hello.php`

#### Co stále není otázka pluginu, ale konfigurace nebo custom vrstvy

- contest data v DB
- Ecomail env konfigurace
- role a provozní workflow
- GDPR / rules / obsahové stránky
- antispam / anti-bot ochrana veřejných contest formulářů


## 2026-03-31 - Redukce plugin stacku a přesun registrace na homepage

**Souvisejici commity:** `OL-0008 | Reduce plugin surface and harden public access`, `OL-0009 | Integrate contest frontend into CreativeHeroes`, `OL-0010 | Harden contest form processing and winner flow`

### Co bylo provedeno

- z projektu byly fyzicky odstraněny neaktivní pluginy a soubory:
  - `akismet`
  - `autodescription`
  - `classic-editor`
  - `wpcf7-redirect`
  - `hello.php`
- soutěžní registrace už nově neběží přes samostatnou stránku `/contest-registration/`
- registrační shortcode `[contest_registration_form]` se nyní renderuje přímo na homepage v rámci `CreativeHeroes`
- legacy registrační page byla odstraněna i z databáze a z navigací
- contest shell a winner flow byly zjednodušené tak, aby na frontendu nezůstávaly pracovní nápovědy, onboarding texty ani pomocné informační bloky
- zachovaná zůstala pouze setup hláška:
  - `Soutěž zatím není připravená`
  - `Nejprve nastavte soutěžní období v administraci, aby bylo možné registraci zpřístupnit.`

### Jak je to nyní řešeno

- plugin `contest-manager-universal` už nepředpokládá existenci stránky `contest-registration`
- za registrační URL se nyní považuje homepage, pokud je WordPress nastavený na statickou úvodní stránku
- automatické zakládání registrační stránky a jejích menu položek je vypnuté
- homepage šablona sama vloží `[contest_registration_form]`, pokud už shortcode není vepsaný přímo v obsahu stránky
- winner upload stránka zůstává zachovaná jako samostatné flow, ale bez zbytečné instruktážní copy

### Ověření

- homepage vrací `200`
- `/contest-registration/` vrací `404`
- `/contest-upload/` vrací `200`
- v navigacích už není položka `Soutěžní registrace`
- veřejně zůstává pouze povolená setup hláška, ostatní contest helper copy byla odstraněna

### Poznámka k dalšímu postupu

- dříve navržený navazující implementační postup zůstává platný a je tímto v dokumentaci zachovaný pro pozdější návrat


## 2026-03-31 - HTML validace a semantický cleanup theme výstupu

**Souvisejici commit:** `OL-0009 | Integrate contest frontend into CreativeHeroes`

### Co bylo upraveno

- byly odstraněné problematické wrappery `section` a `article` tam, kde nenesly vlastní nadpis a sloužily jen jako layout kontejnery
- footer brand a footer column titles už nepoužívají heading elementy, ale neutrální textové tagy, aby nevznikaly skoky v heading outline
- contest page shell byl převeden z `section` na `div`, aby nevznikalo varování na stránkách bez vlastního nadpisu v sekci
- security vrstva pro CSP a SRI byla opravená tak, aby atributy `integrity` a `crossorigin` dostávaly jen externí asset skripty a styly
- inline skripty nově dostávají pouze `nonce`, což je správně pro CSP a současně validní HTML

### Ověřený dopad

- homepage už negeneruje warningy kvůli `section` bez nadpisu nebo `article` bez nadpisu
- footer už negeneruje warning kvůli skoku z `h1` na `h3`
- inline script `contact-form-7-js-before` už nemá neplatný atribut `integrity`
- ve frontend HTML už nezůstávají XHTML-style self-closing void elementy typu `/>`


## 2026-03-31 - Hluboký audit contest flow a oprava nalezených regresí

**Souvisejici commit:** `OL-0010 | Harden contest form processing and winner flow`

### Co bylo provedeno

- proběhl hluboký runtime audit contest stacku nad reálnými HTTP a admin AJAX testy, ne jen statická kontrola zdrojáků
- byly end-to-end ověřené klíčové flow:
  - registrace soutěžícího z homepage
  - vytvoření soutěžícího v `cm_contestants`
  - zařazení e-mailu do `cm_ecomail`
  - losování daily / weekly / main kandidáta
  - uložení výherce do `cm_winners`
  - veřejná winner URL `/v/{cnp}`
  - nahrání výherní účtenky
  - schválení účtenky
  - odeslání výhry
- během auditu byly nalezené a opravené dvě reálné chyby v pluginové logice

### Opravené chyby

#### 1. Winner upload ztrácel `cnp` při REST submitu CF7

- helper vrstva přepisovala `cnp` z route i při REST requestu, kde query var neexistoval
- výsledkem bylo, že se účtenka fyzicky nahrála do `uploads`, ale nepropsala se do `cm_receipts` ani do stavu výherce
- oprava:
  - zachovává se odeslané `cnp`, pokud není k dispozici route parametr
  - z CF7 winner formuláře byl odstraněný duplicitní hidden field `cnp`
  - bootstrap verze pluginu byla navýšená, aby se definice formuláře propsala do DB

#### 2. Duplicitní registrace měly závodní stav při souběžném submitu

- dvě souběžné registrace se stejným `purchase_date` a `purchase_price` mohly projít s `duplicate = 0`
- příčina byla v tom, že oba requesty stihly pre-insert kontrolu duplicity před insertem druhého requestu
- oprava:
  - po každém insertu registrace se nyní duplicate flag dopočítá deterministicky nad celou kolizní skupinou
  - do bootstrapu byl doplněný backfill, který opraví i dříve vzniklé chybné duplicate flagy

### Ověřený výsledek

- registrace z homepage funguje korektně a zapisuje data do DB
- draw AJAX pro daily / weekly / main vrací korektní kandidáty při správně připravených datech
- winner route s platným CNP zobrazuje formulář nebo success stav podle aktuálního stavu výherce
- neplatné nebo neaktivní CNP vrací `404`
- upload účtenky po opravě správně zakládá / aktualizuje `cm_receipts` a mění stav v `cm_winners`
- schválení účtenky i odeslání výhry správně mění navazující statusy a zapisují queue eventy
- bootstrap verze `5` zpětně opravuje již vzniklé špatné duplicate flagy

### Úklid po testu

- všechna dočasná smoke data byla po auditu odstraněna
- vyčištěny byly testovací záznamy v:
  - `cm_periods`
  - `cm_main_prize_periods`
  - `cm_prizes`
  - `cm_contestants`
  - `cm_winners`
  - `cm_receipts`
  - `cm_ecomail`
- smazané byly i dočasné uploady účtenek a pomocné soubory v `/tmp`
- homepage je po cleanupu zpět v setup stavu s ponechanou hláškou `Soutěž zatím není připravená`

### Co stále nelze potvrdit bez další konfigurace

- skutečné doručení transakčních e-mailů do Ecomail API zatím nešlo ověřit, protože stále chybí produkční / testovací `ECOMAIL_*` konfigurace


## 2026-03-31 - Code review, cleanup a refactor po contest auditu

**Souvisejici commity:** `OL-0009 | Integrate contest frontend into CreativeHeroes`, `OL-0010 | Harden contest form processing and winner flow`

### Co bylo zrevidováno

- proběhlo následné code review nad contest integrací v pluginu, theme vrstvě i runtime synchronizaci `src -> CreativeHeroes`
- revize byla zaměřená na:
  - mrtvý kód po zjednodušení veřejného contest shellu
  - zbytečné JS/CSS zatížení bundlu
  - zbytečné DB dotazy a nevyužitý PHP kontext
  - robustnost normalizace registračního data nákupu

### Provedený cleanup a refactor

- z theme byl odstraněný nevyužívaný contest countdown JS modul a návazné volání v hlavním bundle bootstrapu
- `contest.styl` byl zredukovaný pouze na skutečně používané wrapper a layout selektory pro contest pages
- `inc/contest.php` v `CreativeHeroes` byl zjednodušený na minimální funkční vrstvu:
  - detekce contest page varianty
  - helper `creativeheroes_is_contest_page()`
  - body classes pro contest page
- z `page.php` bylo odstraněné předávání nevyužitého contest contextu do template partu
- v `contest-manager-universal/includes/cf7-handler.php` byla zpřesněná normalizace `purchase_date`
  - už se nespoléhá na implicitní chování `strtotime` + `date`
  - date-only vstup `Y-m-d` se zachová deterministicky bez rizika posunu
  - duplicate pre-check se opírá o normalizované datum jen pokud je skutečně validní

### Ověření po refactoru

- `php -l` prošel pro upravené pluginové i theme PHP soubory
- `npm run lint:styles` prošel
- `npm run build` prošel a synchronizoval runtime theme kopii
- homepage dál korektně zobrazuje setup hlášku
- neplatná winner URL dál vrací `404`
- po cleanupu se ve frontend výstupu už neobjevují nevyužité contest countdown / hero stopy

## 2026-03-31 - Zjednoduseni copy na custom login route `/creative/admin`

**Souvisejici commit:** neni zatim commitnuto v tomto repozitari

- custom login screen v MU pluginu byl textove zjednoduseny podle aktualniho zadani
- levy sloupec ponechava uz jen:
  - `Creative Heroes`
  - `Administrace`
- z praveho sloupce login formulare byly odstranene pomocne texty:
  - `Přihlášení`
  - `Použijte své přihlašovací údaje.`
- submit button byl zkraceny z `Vstoupit do administrace` na `Přihlásit`
- logika custom route, nonce, redirect target ani security flow se nemenily

### Overeni

- `php -l wp-content/mu-plugins/creativeheroes-security.php` proslo bez chyby
- `curl http://olympia.loc/creative/admin/` potvrzuje, ze se ve vystupu renderuje pouze:
  - `Creative Heroes`
  - `Administrace`
  - button `Přihlásit`

## 2026-03-31 - Reset administracniho pristupu pro olympia

**Souvisejici commit:** neni commitnuto, jde o runtime zmenu v databazi

- na zadost uzivatele byl pro hlavniho administratora instance `olympia` aktualizovany prihlasovaci e-mail na `o.vetiska@creativeheroes.cz`
- soucasne probehla rotace hesla administracniho uctu
- tajny udaj nebyl zapsan do dokumentace ani do repozitare

### Overeni

- zmena probehla pres WordPress API, ne primym zasahem do hash sloupce v DB
- overeno bylo, ze aktualizovany administrator `creativeheroes_admin` ma novy e-mail a heslo odpovida nove ulozenemu hashi

## 2026-03-31 - Odstraneni dashboard widgetu `WordPress akce a novinky`

**Souvisejici commit:** neni zatim commitnuto v tomto repozitari

- do cleanup vrstvy `CreativeHeroes` byl doplnen admin dashboard cleanup hook pro odstraneni core widgetu `dashboard_primary`
- z dashboardu tak mizi box `WordPress akce a novinky`
- zmena je propsana do zdrojove vrstvy `src/php/inc/cleanup.php` i do runtime kopie `wp-content/themes/CreativeHeroes/inc/cleanup.php`

### Poznamka k chovani

- WordPress si widget nejprve zaregistruje v core `wp_dashboard_setup()` a nasledne je nasim hookem odstraneny pres `remove_meta_box()` v internim `do_action('wp_dashboard_setup')`
- v interni strukture metaboxu pak zustava jako `false`, tedy nevyrenderuje se
- tim padem se nespousti ani jeho vystup a nedochazi k nacitani feed obsahu pres dashboard widget render

### Overeni

- `php -l` proslo pro obe cleanup kopie
- hook `creativeheroes_remove_core_dashboard_news_widgets` je zaregistrovany na `wp_dashboard_setup` s prioritou `999`
- simulace dashboard setupu ukazuje, ze `dashboard_primary` je po aplikaci hooku nastaveny na `false`

## 2026-03-31 - Srovnani font preloadu na nove Gilroy fonty

**Souvisejici commit:** neni zatim commitnuto v tomto repozitari

- preload whitelist ve `src/php/inc/assets.php` a `wp-content/themes/CreativeHeroes/inc/assets.php` byl upraven na nove fontove vstupy:
  - `Gilroy-Regular.woff2`
  - `Gilroy-Bold.woff2`
  - `Gilroy-Black.woff2`
  - `icon.woff2`
- ze seznamu preloadu byl odstranen stary `PlayfairDisplay-BoldItalic`
- po buildu se frontend head procistil na ctyri realne preloadovane fonty bez duplicity

### Dulezita poznamka

- pri kontrole se ukazalo, ze zdrojove `src/assets/fonts/` neobsahovalo `icon.woff2`, ackoli na nej Stylus odkazoval
- proto byl do `src/assets/fonts/icon.woff2` docasne doplnen existujici runtime icon font, aby build zustal funkcni
- uzivatel avizoval, ze doda finalni `icon` soubor pozdeji; stavajici `icon.woff2` je tedy docasny placeholder

### Overeni

- `npm run build` uspesne vygeneroval nove runtime assety a manifest
- homepage head nyni preloaduje presne:
  - `Gilroy-Regular`
  - `Gilroy-Bold`
  - `Gilroy-Black`
  - `icon`

## 2026-03-31 - Vypnuti WordPress notifikacnich sluzeb `ping_sites`

**Souvisejici commit:** neni zatim commitnuto v tomto repozitari

- z projektu bylo odstraneno vychozi WordPress nastaveni `https://rpc.pingomatic.com/` v option `ping_sites`
- nejde o soucast contest stacku ani naseho custom setupu, slo o cisty WordPress default
- v cleanup vrstve `CreativeHeroes` je nyni `ping_sites` natvrdo vypnute:
  - pri site defaults se zapisuje prazdna hodnota
  - `pre_option_ping_sites` vraci vzdy prazdny retezec
  - `pre_update_option_ping_sites` blokuje znovuzapsani jakekoli adresy z administrace
- soucasne byla ulozena hodnota v aktualni DB instance fyzicky procistena

### Overeni

- `php -l` proslo pro obe cleanup kopie
- runtime `get_option('ping_sites')` vraci prazdny retezec
- admin Writing settings uz tak nebude nabizet `rpc.pingomatic.com` jako aktivni cil notifikace

## 2026-03-31 - Seed testovaci souteze a provozni dokumentace

**Souvisejici commit:** `OL-0012 | Add contest test seed and operational docs`

### Kontext

Byl pripraven opakovatelny testovaci setup, aby slo v `olympia` bez dalsiho rucniho skladani proverit:

- registraci
- losovani daily / weekly / main
- upload uctenky
- schvaleni uctenky
- expedici vyhry
- Ecomail frontu

Soucasne byla do repozitare dopsana trojice markdown dokumentu pro bezny provoz, plnou contest dokumentaci a instalaci projektu.

### Provedene zmeny

#### 1. Byl pridan idempotentni seed testovaci souteze

Soubor:

- `tools/seed-test-contest.php`

Seed zajistuje:

- placeholder stranky `/gdpr/` a `/pravidla-souteze/`
- testovaci blok na homepage
- tri bezna obdobi v `cm_periods`
- dve etapy v `cm_main_prize_periods`
- tri testovaci vyhry v `cm_prizes`
- devet testovacich soutezicich v `cm_contestants`
- cisty start bez `cm_winners`, `cm_receipts` a `cm_ecomail`

Zamerne jsou pokryte i specialni scenare:

- daily flow
- weekly flow
- main flow
- courier / electronic / personal delivery vetve
- duplicitni registrace
- aktualni registrace na hrane aktivniho obdobi

#### 2. Seed vypisuje provozni summary po spusteni

Skript po dobehu vypisuje:

- homepage URL
- upload page URL
- pocty obdobi a vyher
- seznam seedovanych osob vcetne CNP a ucelu
- doporuceny admin postup
- informaci, zda je Ecomail skutecne nakonfigurovany

#### 3. Byly dopsany projektove markdown dokumenty do `src/docs`

Pridane soubory:

- `src/docs/documentation.md`
- `src/docs/documentation-full.md`
- `src/docs/documentation-install.md`

`README.md` byl zmenen na rozcestnik techto tri dokumentu.

Obsah dokumentace pokryva:

- rychly contest checklist
- kompleti provozni navod od zalozeni souteze po expedici vyhry
- instalaci na macOS, Windows a server
- odchylky proti puvodnimu contest manualu
- Ecomail, cron runner a bezpecnostni specifika `olympia`

### Overeni

Bylo overeno:

- `php -l tools/seed-test-contest.php`
- opakovane spusteni seedu vraci deterministicky stav:
  - `contestants = 9`
  - `duplicates = 1`
  - `winners = 0`
  - `receipts = 0`
  - `ecomail = 0`
  - `periods = 3`
  - `main_periods = 2`
  - `prizes = 3`
- homepage po seedu vraci `200` a obsahuje testovaci blok i registracni formular
- upload route bez predchoziho vylosovani vyherce zustava korektne neaktivni

### Dulezita poznamka

Seed pripravi kompletni testovaci data a workflow, ale bez `ECOMAIL_*` env promennych nelze overit realne doruceni e-mailu do Ecomail API.
Testuje se pouze vnitrni tvorba a zpracovani fronty.

## 2026-03-31 - Rozsireni contest registrace o obchodnika a normalizaci vstupu

### Co bylo upraveno

- Registračni formular v `contest-manager-universal` byl rozšířen o povinne pole `Obchodnik`.
- Rozdelene `Jmeno` a `Prijmeni`, `Souhlas se zpracovanim osobnich udaju` a `Newsletter` zustaly zachovane, jen byly potvrzene jako soucast standardniho formulare.
- `Datum nakupu` bylo zmenene z nativniho `date` inputu na textovy vstup s projektovym formatem `DD.MM.RRRR`.
- `Cena nakupu` byla zmenena z nativniho `number` inputu na textovy numericky vstup bez prohlizecovych spinneru.
- Pro `PSC`, `datum`, `cas` a `cenu` byla doplnena frontend maska i server-side normalizace.
- Nove pole `merchant_name` bylo doplneno i do DB schematu `cm_contestants`, administracniho seznamu soutezicich, preview pri losovani, detailu vyherce a seed testovacich dat.

### Normalizace a validace

- Datum prijima `DD.MM.RRRR`, `D.M.RRRR`, `YYYY-MM-DD` i osmimistny zapis a uklada se kanonicky jako `Y-m-d`.
- Cas prijima bezny zapis `HH:MM` i zkraceny zapis typu `1330` nebo `930`; pred ulozenim se normalizuje na `HH:MM:SS`.
- PSC se na frontendu zobrazuje jako `123 45`, ale do DB se uklada bez mezery jako pet cifer.
- Cena prijima pouze numericky zapis s jednim desetinnym oddelovacem a do DB se uklada jako `DECIMAL(10,2)`.
- Do ACF options pribyly nove hlasky `msg_date_format` a `msg_price_invalid`.

### Technicky detail migrace

- Schema verze contest pluginu byla zvysena na `4` a bootstrap verze na `7`.
- Kvuli spolehlivosti migrace je `merchant_name` doplnovan pres guardovanou `ALTER TABLE` migraci jen pokud sloupec skutecne chybi.
- Sync Contact Form 7 byl rozsireny i o `mail` properties, aby se nova pole propsala nejen do markupu, ale i do interni konfigurace formulare.

### Overeni

- `php -l` prosel nad vsemi zmenenymi PHP soubory.
- `npm run build` probehl bez chyby a novy JS bundle obsahuje masky `js-form-date`, `js-form-time`, `js-form-zip` a `js-form-price`.
- Runtime bootstrap formulare se zvedl na verzi `7`.
- Render registracniho formulare potvrzuje pole `Obchodnik`, textove datum, textovou cenu a vsechny pozadovane CSS/JS classy pro maskovani.
- Server-side normalizace byla proverena na vzorech `31.03.2026`, `31032026`, `1330`, `930`, `123 45`, `1299,90` a `1 299.9`.
- Admin stranka `Seznam soutezicich` ted oddeluje `Jmeno` a `Prijmeni` do samostatnych sloupcu a umi podle obou poli i tridit.
- Admin stranka `Seznam soutezicich` zobrazuje i `Cas nakupu` a umi podle nej tridit.
- Registrační contest formulář má teď živé validační stavy: chybné nebo prázdné povinné pole se zvýrazní červeně a pod polem se absolutně vypíše chybová hláška, validní vyplněné pole dostane zelený stav.
- Newsletter zůstává jako jediné volitelné pole bez povinné validace.
- Vite build byl po minimalni syntakticke oprave odsazeni v `src/assets/css/global/icon.styl` znovu uspesne dokoncen a nove validacni assety jsou propsane i do runtime theme bundle.
- Contest formulare maji od bootstrap verze `8` lokalizovane CF7 systemove a validacni zpravy v cestine (`invalid_required`, `invalid_email`, upload chyby i submit statusy).
- Checkboxy contest formulare jsou ted kompletne custom: nativni vzhled je vypnuty, box ma vlastni border a animovany check vykresleny pres `::before` a `::after`; validni a chybove stavy se propisuji i do checkboxove varianty.

## 2026-03-31 - Admin toggle pro vyhledavani v hlavicce

### Co bylo upraveno

- Do existujici theme settings vrstvy `creativeheroes_navigation_settings` pribyl novy boolean prepinac `header_search_enabled`.
- Prepinac je dostupny v administraci v sekci `Vzhled -> Menu` v metaboxu `CreativeHeroes Menu`.
- Vychozi stav je `vypnuto`, takze na ciste instalaci se header search vubec nevyrenderuje do frontend HTML.
- Samotny render vyhledavani v `site-header.php` je ted podminen helperem `creativeheroes_is_header_search_enabled()`.

### Overeni

- `creativeheroes_get_navigation_settings()` vraci pri vychozim stavu `header_search_enabled => false`.
- `creativeheroes_is_header_search_enabled()` vraci pri vychozim stavu `false`.
- Homepage HTML uz ve vychozim stavu neobsahuje `.header__search`, `.js--search-button` ani `get_search_form()` markup.
- `siteHeader.js` zustava kompatibilni, protoze uz predtim pocital s tim, ze search markup muze chybet.

## 2026-03-31 - Commit mapa OL-0013 az OL-0015

- `OL-0013 | Add admin toggle for header search`
  - admin prepinac pro zobrazeni hledani v hlavicce v `Vzhled -> Menu`
  - vychozi stav vypnuto a search se bez zapnuti nevyrenderuje do frontend HTML
- `OL-0014 | Extend contest registration fields and validation rules`
  - rozsireni contest registrace o `Obchodnik`
  - server-side normalizace a validace data, casu, PSC a ceny
  - propsani noveho pole do DB schematu, seedu, losovaciho preview a admin seznamu soutezicich
- `OL-0015 | Add contest form validation and refresh theme assets`
  - frontend masky a ziva validace contest formularu
  - custom checkboxy, lokalizovane chybove stavy a nova `contestFormFields.js` vrstva
  - refresh font stacku a icon assetu vcetne noveho Vite bundle

## 2026-03-31 - Doladeni checkboxu contest formulare

### Co bylo upraveno

- Newsletter checkbox ma ted pri zaskrtnuti zeleny checked stav i bez validacni tridy.
- U GDPR acceptance radku byl odstraneny obalovy border pro `valid` i `invalid` stav; pri chybe zustava cerveny jen samotny checkbox a textova hlaska.
- Theme CSS bundle byl po teto uprave znovu vygenerovan.

## 2026-03-31 - Vizuální skrytí CF7 screen reader bloku

### Co bylo upraveno

- `Contact Form 7` blok `.screen-reader-response` byl vrácen do správného `sr-only` režimu v projektových stylech.
- Blok tak zůstává funkční pro přístupnost a `aria-live`, ale vizuálně už nerozbíjí layout formuláře.
- Veřejně zůstávají viditelné jen naše validační hlášky u jednotlivých polí a standardní response output formuláře.

## 2026-03-31 - Preview winner link pro styling upload stranky

### Co bylo upraveno

- Contest plugin nově umí speciální hash preview link pro winner upload stránku bez vazby na reálného výherce.
- Preview link renderuje skutečný winner formulář včetně polí a frontend stylů, ale v bezpečném náhledovém režimu.
- V preview režimu je submit na klientu vypnutý a server-side guard zároveň blokuje ukládání účtenek i zápisy do DB při případném ručním POSTu.
- Reálný winner flow přes `/v/{cnp}/` zůstává beze změny.
