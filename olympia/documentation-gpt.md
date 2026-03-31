# Olympia

## Ucel dokumentace

Tato slozka slouzi jako sdilena technicka dokumentace pro projekt `olympia`.
Pri kazdem dalsim tasku je potreba nejdriv nacist tento soubor a navazat na aktualni zavery.

## Trvale pracovni zavery

- Komunikace probiha cesky.
- Projekt `olympia` je cilove postaveny na theme `CreativeHeroes`, ne na starem theme `contest`.
- Theme `contest` slouzi jako zdroj soutezniho frontendu a obsahovych vazeb, ne jako finalni aktivni theme.
- Soutezni business logika patri do pluginove vrstvy, primarne do `contest-manager-universal`, ne do sablon.
- `CH-cleanup` nema byt cilove pouzivan jako runtime plugin.
  - Cleanup a hardening ma byt standard v jadru projektu.
  - Primarni mista:
    - `wp-content/themes/CreativeHeroes/inc/cleanup.php`
    - `wp-content/mu-plugins/creativeheroes-security.php`

## Kriticke kompatibilitni body

- `contest-manager-universal` je pro soutezni logiku povinny.
  - Bez nej neni `olympia` funkcne ekvivalentni projektu `contest`.

- `creativeheroes-security` aktualne blokuje hostum `wp-json` a `rest_route`.
  - To je v konfliktu s `Contact Form 7`, ktery na frontendu pouziva REST endpointy.
  - Pred dalsi implementaci soutezniho formulare je potreba upravit allowlist nebo jinak bezpecne zpruchodnit nezbytne endpointy.

- `contest` theme je silne vazany na:
  - ACF options
  - prime SQL dotazy do `cm_*` tabulek
  - vlastni layout
  - vlastni SCSS a jQuery
  - vlastni age-gate flow

- V `olympia` zatim neni kodove verzovana ACF konfigurace.
  - Nebyl nalezen `acf-json`.
  - Pri dalsi fazi je potreba dostat field groups do repozitare.

## Pracovni pravidla pro dalsi implementaci

- `CreativeHeroes` zustava jediny cilovy theme.
- Novy soutezni frontend stavet jako:
  - page template
  - template parts
  - Stylus
  - Vite JS moduly bez jQuery

- Neprenasej stare theme `contest` mechanicky soubor po souboru.
  - Nejdriv oddel business logiku, texty, ACF a prezencni vrstvu.
  - Teprve pak portuj UI a layout.

- Prime SQL dotazy v sablonach postupne eliminovat.
  - Dotazy nad `cm_*` tabulkami presouvat do helper vrstvy nebo pluginu.

- U cleanup/hardening zmen vzdy vyhodnotit dopad na:
  - CF7 submit flow
  - route `/v/{cnp}`
  - upload uctenky
  - verejne endpointy potrebne pro soutez
  - Ecomail queue

- `CH-cleanup` pouzivat jen jako referencni zdroj feature parity.
  - Do jadra projektu prevzit pouze bezpecne a skutecne potrebne casti.

- Lokalni URL tohoto projektu je potvrzena jako `http://olympia.loc`.
  - `.htaccess` musi zustat root-based, ne pod `/wordpress/`.

- `contest-manager-universal` ted drzi i frontend integracni vrstvu:
  - `assets/css/frontend.css`
  - conditional enqueue pouze na contest strankach a contest shortcode route

- Pri neplatnem `/v/{cnp}` nikdy neincludovat `404.php` primo ze shortcode.
  - Korektni postup je:
    - nastavit `404` stav
    - vratit lehky contest-specific notice markup
    - nenechat theme vykreslit druhou zanozenou stranku uvnitr page obsahu

- `CreativeHeroes` snapshot v tomto projektu neobsahuje puvodni Stylus/Vite source tree.
  - Pri dalsi fazi portu je potreba rozhodnout, zda:
    - vratit zdrojove theme soubory
    - nebo frontend contest vrstvu udrzovat docasne v plugin assets

## Potvrzene technicke zavery z uvodni analyzy

- WordPress jadro je ve `wordpress`, `contest` i `olympia` ve verzi `6.9.4`.
- `olympia/wp-content/themes/CreativeHeroes` je shodny s `wordpress`.
- `olympia/wp-content/themes/contest` je shodny s `contest`.
- `olympia` ma proti `contest` temer stejny plugin set, ale chybi `contest-manager-universal`.
- `olympia/wp-config.php` vychazi z `wordpress`, ne z `contest`.
  - chybi contest-specific konstanty pro Ecomail a CF7

## Doporuceny cilovy smer

1. Doinstalovat nebo prenest `contest-manager-universal` do `olympia`.
2. Upravit security vrstvu pro kompatibilitu s CF7.
3. Vyexportovat nebo zaregistrovat ACF field groups v kodu.
4. Portovat contest frontend do `CreativeHeroes`.
5. Vybrane cleanup prvky z `CH-cleanup` presunout do theme/mu-plugin jadra.

## 2026-03-31 - Implementacni log po oddeleni do DB `olympia`

**Souvisejici commity:** `OL-0004 | Bootstrap contest manager integration`, `OL-0005 | Align Olympia runtime config and REST security`, `OL-0006 | Fix Olympia root WordPress rewrites`

### Co jsem udelal

- vytvoril jsem samostatnou DB `olympia`
- zkopiroval jsem do ni puvodni stav z `wordpress_2026`
- prepnul jsem `olympia/wp-config.php` na novou DB a pridal:
  - `WPCF7_AUTOP = false`
  - env-aware `ECOMAIL_*`
  - env-aware `WP_HOME/WP_SITEURL`
- aktivoval jsem v `olympia`:
  - ACF PRO
  - Contact Form 7
  - Contest Manager Universal

- v `contest-manager-universal` jsem dodelal runtime bootstrap:
  - provision upload page `contest-upload`
  - provision registracni page `contest-registration`
  - provision dva CF7 formulare pro registraci a vyherce
  - zapis jejich ID do contest options
  - zapis ACF local field group do kodu

- sjednotil jsem contest option pristup pres helpery:
  - `contest_manager_get_option_field()`
  - `contest_manager_get_option_defaults()`
  - `contest_manager_update_option_field()`

- upravil jsem `winner-page.php`, aby:
  - nebyla vazana na contest theme layout
  - pouzivala CreativeHeroes-compatible neutralni markup
  - renderovala CF7 formulare pres shortcode fallback

- upravil jsem `creativeheroes-security.php`, aby:
  - nerozbila verejne CF7 contest endpointy
  - stale skryvala ostatni systemove REST query requesty

### Proc jsem to udelal takto

- user chtel chirurgicke rezy, ne slepy prenos stareho theme
- nejvetsi problem nebyl vizual, ale chybici provozni bootstrap:
  - nova DB
  - neaktivni dependency
  - zadne ACF field groups
  - zadne CF7 formulare
  - contest page bez obsahu

- proto jsem contest zalezitosti presunul do mist, kam logicky patri:
  - settings a bootstrap do contest pluginu
  - security allowlist do mu-plugin hardening vrstvy
  - globalni config do `wp-config.php`

### Co je nyni potvrzene hotove

- `olympia` uz nepouziva `wordpress_2026`
- contest plugin je aktivni a vytvoril `cm_*` tabulky
- contest upload page existuje a ma shortcode obsah
- contest-specific CF7 formulare existuji
- winner shortcode nad validnim CNP renderuje formular
- registracni shortcode renderuje formular
- public CF7 REST route neni blokovana security vrstvou
- verejne contest routy na `olympia.loc` vraceji korektni statusy
- `.htaccess` je srovnana na root instalaci
- invalidni `/v/{cnp}` uz nema zanozenou 404 sablonu
- contest dashboard zobrazuje checklist chybejici konfigurace
- contest frontend ma vlastni nenarusivou styling vrstvu

### Co stale neni hotove

- DB stale nese zkopirovane `home/siteurl` z puvodniho `wordpress` projektu
  - runtime jsem pripravil pro env override
  - finalni lokalni/produkci URL je potreba potvrdit a srovnat

- stale neni portovan cely verejny contest frontend ze stareho theme `contest`
  - homepage sekce
  - vizualni vrstvy
  - obsahove ACF fieldy stare sablony

- v DB stale chybi realne soutezni provozni hodnoty
  - contest periods
  - prizes
  - pripadne Ecomail env konfigurace

- contest frontend vrstva je zatim provozni integrace, ne finalni designovy port stare souteze do `CreativeHeroes`

- `CH-cleanup` zatim zustava jen jako referencni zdroj
  - nic z nej jsem v teto fazi neaktivoval jako runtime plugin

## 2026-03-31 - CF7 frontend form refactor do Stylus pipeline

**Souvisejici commity:** `OL-0009 | Integrate contest frontend into CreativeHeroes`, `OL-0010 | Harden contest form processing and winner flow`

### Co jsem zmenil

- nasel jsem a potvrdil, ze projekt uz ma root source tree `src/assets/css`, `src/php` a Vite build do `wp-content/themes/CreativeHeroes/assets`
- vypnul jsem frontend CSS Contact Form 7 filtrem `wpcf7_load_css` v theme asset vrstve
- odstranil jsem docasne contest pluginove frontend CSS a jeho enqueue
- vytvoril jsem `src/assets/css/global/form.styl` a napojil ho do hlavniho `css.styl`
- prepsal jsem bootstrap definice registracniho a winner formulare na BEM markup s projektovymi tridami
- doplnil jsem helper pro render CF7 formulare s `html_class` a `html_id`
- winner page fallback uz renderuje standardizovany CF7 form s BEM tridami, pokud neni rucne vyplneny custom markup v options
- zvysil jsem `CONTEST_MANAGER_BOOTSTRAP_VERSION` na `3`
- doplnil jsem sync existujicich CF7 formularu v DB tak, aby se pri upgrade propsal novy `form` markup i `additional_settings`
- opravil jsem GDPR pole na validni `acceptance` syntax a zapnul `acceptance_as_validation: on`

### Proc jsem to udelal takto

- user chtel bezpodminecne vlastni frontend styly a zadny samostatny CF7 stylesheet z pluginu
- v projektu uz realne existuje proper Stylus/Vite source tree, takze docasne plugin assets prestaly davat smysl
- uplne prepsani vsech CF7 internich wrapperu by bylo rizikove a rozbilo by validaci nebo JS submit flow
- proto jsem zvolil bezpecny kompromis:
  - vsechna autorovana vrstva formularu je v BEM
  - nevyhnutelne `wpcf7-*` internaly zustavaji, ale jsou stylovane jen explicitne a lokalne pod `.form`

### Co je nyni potvrzene

- `contact-form-7/includes/css/styles.css` uz se na verejnem frontendu olympia nenasita
- `contest-manager-universal/assets/css/frontend.css` uz neni soucasti frontend stacku
- verejny frontend nacita contest formulare jen pres `CreativeHeroes` buildovany CSS bundle
- oba CF7 formulare jsou v DB aktualizovane na novou BEM strukturu
- runtime HTML opravdu obsahuje `form form--contest-registration` a `form form--contest-winner`
- acceptance checkbox uz se korektne parsuje a neni ponechan jako surovy text
- drivejsi poznamka, ze `CreativeHeroes` snapshot neobsahuje source tree, uz neplati; source tree je pritomen v rootu projektu

## 2026-03-31 - Contest page template a verejna prezencni vrstva

**Souvisejici commit:** `OL-0009 | Integrate contest frontend into CreativeHeroes`

### Co jsem zmenil

- pridal jsem do theme `src/php/inc/contest.php` novou helper vrstvu pro contest pages
- helper umi rozpoznat contest page variantu, sestavit stav souteze, contest metriky, prize groups, checklisty a support kontakt
- `src/templates/page.php` se nyni pro contest slugy nevykresluje jako bezna page, ale pres `template-parts/contest/page-shell.php`
- vytvoril jsem novy veřejny contest shell s hero sekci, status panelem, sidebar workflow kartami a prize section
- pridal jsem `src/assets/css/component/contest.styl`
- pridal jsem `src/assets/js/modules/contestCountdown.js` a napojil ho do `main.js`
- vse jsem propsal buildem i do runtime theme kopie `wp-content/themes/CreativeHeroes/*`

### Proc jsem to udelal takto

- samotne prehozeni formulare do Stylus pipeline nestacilo; contest pages byly porad jen kratky obsah uvnitr obecne page sablony
- stary `contest` frontend je obsahove silne zavisly na ACF, ktere v `olympia` zatim nemame portovane jako stabilni zdroj frontend dat
- proto jsem zvolil mezikrok, ktery je architektonicky cisty a provozne bezpecny:
  - verejna contest page vrstva patri do theme
  - soutezni business logika zustava v pluginu
  - frontend sekce se skladaji z realnych contest dat z `cm_*` tabulek a plugin helperu

### Co je nyni potvrzene

- `contest-registration` a `contest-upload` uz maji vlastni verejny layout v `CreativeHeroes`
- invalidni `v/{cnp}` route zustava `404`, ale uz v novem contest layoutu
- nove helpery nectou stary contest ACF frontend, ale vyhradne aktualni plugin data a WordPress nastaveni
- countdown je pripraveny pro stav `upcoming`, jakmile budou v DB nastavena contest periods
- prize section se aktivuje automaticky po naplneni `cm_prizes`

## 2026-03-31 - Presun favicon a robots tagu na konec head

**Souvisejici commit:** `OL-0009 | Integrate contest frontend into CreativeHeroes`

### Co jsem zmenil

- v `src/php/inc/cleanup.php` jsem zmenil prioritu `creativeheroes_output_favicon_fallback` z `1` na `9998`
- ve stejne vrstve jsem zmenil prioritu `creativeheroes_output_html5_robots_tag` z `1` na `9999`
- stejnou upravu jsem propsal i do runtime theme kopie `wp-content/themes/CreativeHeroes/inc/cleanup.php`

### Proc jsem to udelal takto

- user chtel, aby favicony, apple meta, manifest a `robots` byly az tesne pred `</head>`
- tyto tagy se uz negeneruji pres WordPress default poradi, ale pres nase theme hooky v `cleanup.php`
- nejcistsi reseni proto bylo jen upravit hook priority, ne zasahovat rucne do `header.php`

### Co je potvrzene

- na `http://olympia.loc/contest-registration/` jsou nyni uvedene favicon a `robots` tagy opravdu na konci `head`, za preloady, stylesheetem a canonicalem


## 2026-03-31 - Analýza CM Manualu proti olympia

**Souvisejici commit:** `OL-0007 | Add CM Manual source files`

### Co jsem načetl

V `src/docs/CM Manual/` jsem prošel:

- `CM Manual.fig`
- `CM Manual.pdf`
- `CM Manual.xml`

XML export nebyl čistě validní XML, takže jsem z něj text tahal robustně přes strip tagů a normalizaci entit. Z toho šlo vytáhnout celé relevantní workflow manuálu.

### Co manuál pokrývá

Manuál není jen jeden úzký návod, ale mix několika scénářů:

- univerzální CM pro single contest
- rozcestník / multi contest varianta
- české i slovenské varianty formulářů
- provozní workflow losování, schvalování účtenek a odesílání výher

Pro `olympia` jsem jako hlavní referenci použil single contest českou variantu.

### Co jsem ověřoval v projektu a databázi

Zkontroloval jsem zejména:

- `wp-config.php`
- root přítomnost `cron-runner.php`
- aktivní pluginy a theme
- permalink structure
- existenci stránek
- existenci a obsah CF7 formulářů
- existenci `cm_*` tabulek
- setup status contest pluginu
- ACF options a local field groups
- login route a security vrstvu
- public routy `contest-registration`, `contest-upload`, `v/{cnp}`
- home page render
- menu bootstrap
- role/capabilities

### Klíčová zjištění

- `contest-registration` a `contest-upload` existují a vrací `200`.
- invalidní `/v/SMOKE123/` vrací `404`.
- `WPCF7_AUTOP` je správně vypnuté.
- `DISABLE_WP_CRON` definované není.
- žádná z `ECOMAIL_*` konstant v instanci aktuálně definovaná není.
- `contest_manager_is_ecomail_configured()` tedy vrací false.
- root `cron-runner.php` existuje a používá bezpečný secret klíč.
- `contest-manager-universal` má `cm_ecomail` tabulku připravenou.
- `cm_periods`, `cm_prizes`, `cm_main_prize_periods`, `cm_contestants`, `cm_receipts`, `cm_winners` existují, ale jsou prázdné.
- role `view_report_page` a `view_shipping_page` jako WordPress role neexistují.
- capability stejného jména má pouze administrátor.
- aktivní není `WPS Hide Login`, ale login route je nahrazená přes `creativeheroes-security.php`.
- `/login/` dává `302` na `/creative/admin/` a samotná secure login route funguje.
- homepage `/` je stále defaultní `CreativeHeroes` page shell, nikoli contest homepage z manuálu.
- nejsou vytvořené GDPR ani thank-you stránky.
- option `gdpr_url` není vyplněná.
- current support email a sender fallbacky jsou pořád development hodnoty.

### Vyhodnocení odchylek

Některé odchylky jsou záměrné a správné:

- local ACF field groups místo importu JSON
- custom secure login route místo WPS Hide Login
- `CreativeHeroes` contest shell místo starých contest template souborů

Některé odchylky jsou ale stále otevřený gap:

- chybějící contest data
- chybějící Ecomail config
- chybějící role
- chybějící GDPR / rules / TP stránky
- chybějící contest homepage

### Praktický závěr

`olympia` je připravená jako stabilní technický základ pro contest projekt, ale ještě není připravená jako kompletně nakonfigurovaná a provozně dotažená contest instance podle CM manuálu.


### 2026-03-31 | Bezpečnostní hardening login aliasů a odmazání zbytečných pluginů

#### Co jsem měnil

- v `wp-content/mu-plugins/creativeheroes-security.php` jsem doplnil explicitní blok public aliasů `admin`, `dashboard`, `login`, `login.php`
- tím se zastaví WordPress core `wp_redirect_admin_locations()` scénář ještě před přesměrováním na `/wp-admin` nebo `/wp-login.php`
- z disku jsem odstranil neaktivní pluginy `wp-content/plugins/wps-hide-login` a `wp-content/plugins/ch-cleanup`
- v root `.htaccess` jsem doplnil přesné `404` rewrite rules pro systémové root cesty:
  - `wp-content`
  - `wp-content/plugins`
  - `wp-content/themes`
  - `wp-content/mu-plugins`
  - rooty jednotlivých plugin/theme adresářů
- v `wp-content/index.php`, `wp-content/plugins/index.php` a `wp-content/themes/index.php` jsem nahradil defaultní `Silence is golden` stub za explicitní `404 Not Found`

#### Proč

- samotná MU security vrstva řešila jen requesty, které šly přes WordPress bootstrap
- adresáře typu `/wp-content/plugins/` se ale obsluhovaly přímo přes Apache a lokální `index.php`, takže vracely `200` a obcházely WordPress hooky
- cílem bylo skutečně uzavřít veřejný vstup do admin/login aliasů a systémových directory rootů bez rozbití legitimních assetů pluginů

#### Co jsem ověřil

- `/login/` vrací `404`
- `/login.php` vrací `404`
- `/admin/` vrací `404`
- `/dashboard/` vrací `404`
- `/creative/admin/` zůstává funkční a vrací `200`
- `/wp-admin/` pro guest vrací `404`
- `/wp-content/`, `/wp-content/plugins/`, `/wp-content/plugins/contact-form-7/`, `/wp-content/themes/CreativeHeroes/` vrací `404`
- `CF7` asset `wp-content/plugins/contact-form-7/includes/js/index.js` zůstává dostupný
- theme asset `wp-content/themes/CreativeHeroes/assets/js/main-G0wMVNRl.js` zůstává dostupný
- homepage, `contest-registration` i `contest-upload` po hardeningu dál vrací `200`

#### Audit plugin stacku

Aktivní a dnes reálně potřebné:

- `advanced-custom-fields-pro/acf.php`
- `contact-form-7/wp-contact-form-7.php`
- `contest-manager-universal/index.php`
- `creativeheroes-media/creativeheroes-media.php`
- MU `creativeheroes-security.php`

Neaktivní a z pohledu minimálního stacku pravděpodobně zbytečné:

- `akismet`
- `autodescription`
- `classic-editor`
- `error-log-monitor`
- `user-role-editor`
- `wpcf7-redirect`
- `hello.php`

#### Důležité zjištění navíc

- veřejné contest formuláře dnes nemají samostatnou anti-bot vrstvu typu Turnstile / hCaptcha / honeypot
- to není chybějící plugin nutný pro funkci, ale je to relevantní bezpečnostní a provozní gap pro ostrý provoz
- pokud budeme chtít zachovat minimální plugin stack, je vhodnější řešit to custom vrstvou než dalším pluginem


## 2026-03-31 - Odstranění zbytných pluginů a přesun registrace na homepage

**Souvisejici commity:** `OL-0008 | Reduce plugin surface and harden public access`, `OL-0009 | Integrate contest frontend into CreativeHeroes`, `OL-0010 | Harden contest form processing and winner flow`

### Co jsem měnil

- fyzicky jsem odstranil tyto pluginy / soubory z `wp-content/plugins`:
  - `akismet`
  - `autodescription`
  - `classic-editor`
  - `wpcf7-redirect`
  - `hello.php`
- v DB jsem smazal legacy page `contest-registration`
- z menu jsem odstranil položky navázané na tuto stránku
- v `contest-manager-universal/includes/install.php` jsem změnil bootstrap tak, aby za registrační stránku považoval homepage a už nevytvářel samostatnou registrační page ani menu položky
- v `contest-manager-universal/includes/functions.php` jsem upravil setup status a render shortcodu registrace
  - pokud contest ještě není připravený, vrací se pouze povolená setup hláška
  - pokud contest ještě nezačal nebo už skončil, shortcode nevrací žádné pomocné bloky
- v `contest-manager-universal/includes/winner-page.php` jsem odstranil contest pomocné texty a vysvětlivky okolo upload flow
- v `src/templates/front-page.php` jsem vložil registrační shortcode přímo na homepage s guardem proti duplicitnímu renderu
- v `src/templates/template-parts/contest/page-shell.php` jsem contest shell zjednodušil na minimální wrapper bez hero bloků, metrik, kroků, support panelu a prize sekce

### Proč

- cílem bylo snížit plugin attack surface a odstranit z projektu komponenty, které contest runtime ani olympia základ nepotřebují
- současně bylo potřeba přesunout veřejnou registraci na homepage a odstranit dočasné pracovní copy, aby frontend nezobrazoval provizorní instruktážní obsah

### Co jsem ověřil

- `php -l` nad změněnými PHP soubory prošel bez chyby
- homepage na `http://olympia.loc/` vrací `200` a zobrazuje pouze ponechanou setup hlášku
- `http://olympia.loc/contest-registration/` vrací `404`
- `http://olympia.loc/contest-upload/` vrací `200`
- v menu už nezůstává položka `Soutěžní registrace`
- předchozí navazující roadmap návrh jsem zachoval v dokumentaci pro pozdější návrat


## 2026-03-31 - Oprava HTML validace theme a CSP script atributů

**Souvisejici commit:** `OL-0009 | Integrate contest frontend into CreativeHeroes`

### Co jsem měnil

- ve `front-page.php`, `page.php`, `single.php`, `index.php`, `404.php` a contest page shellu jsem nahradil layoutové `section` / `article` wrappery za `div`, pokud nenesly vlastní nadpis a byly jen kontejnery
- ve footer šabloně jsem změnil `footer__brand` z `h3` na `p`
- ve footer helperu `template-tags.php` jsem změnil `footer__column` wrappery ze `section` na `div` a `footer__title` z `h3` na `p`
- v MU pluginu `creativeheroes-security.php` jsem opravil injekci CSP/SRI atributů
  - externí script/style tagy dostávají atributy cíleně jen na tag s odpovídajícím `src` / `href`
  - přidal jsem `wp_inline_script_attributes` filter, který inline skriptům přidává pouze `nonce`
  - tím se odstranil nevalidní stav, kdy měl inline CF7 script atribut `integrity`

### Co jsem ověřil

- `php -l` prošel nad všemi změněnými PHP soubory
- homepage i winner upload page mají stále funkční frontend výstup
- inline CF7 script už nemá `integrity`, ale má `nonce`
- ve výstupu homepage ani upload page už nejsou `h3.footer__brand`, `h3.footer__title`, `article.article--page`, `section.section` ani `section.section--contest-page`
- ve výstupu homepage už nezůstává žádné `/>` u běžných HTML void elementů


## 2026-03-31 - Hluboký audit soutěžního flow a opravy pluginu

**Souvisejici commit:** `OL-0010 | Harden contest form processing and winner flow`

### Co jsem dělal

- prošel jsem contest stack kombinací statické analýzy a reálných integračních testů proti `http://olympia.loc`
- testoval jsem přímo:
  - CF7 REST submit registrace
  - admin AJAX losování a ukládání výherců
  - veřejnou winner route `/v/{cnp}`
  - CF7 REST submit výherní účtenky včetně uploadu souboru
  - admin AJAX schválení účtenky a odeslání výhry
- pro test jsem dočasně založil testovací periods, main periods, prizes a smoke soutěžící, po dokončení jsem vše odstranil

### Co jsem našel a opravil

#### 1. Rozbitý winner upload po našich úpravách

- soubor: `wp-content/plugins/contest-manager-universal/includes/functions.php`
- problém byl v helperu `contest_manager_add_cnp_to_cf7_posted_data()`
  - při REST submitu přepsal odeslané `cnp` na prázdný string, protože `get_query_var('cnp')` na `wp-json` requestu nevracel nic
- soubor: `wp-content/plugins/contest-manager-universal/includes/install.php`
- problém byl současně i v definici winner formuláře
  - formulář pořád obsahoval `[hidden cnp]`, i když plugin zároveň vkládá hidden `cnp` přes `wpcf7_form_hidden_fields`
  - výsledkem byly dva hidden inputy `cnp`, z nichž jeden byl prázdný
- oprava:
  - helper teď používá route `cnp` jen pokud skutečně existuje
  - jinak zachovává a sanitizuje odeslané `cnp`
  - z winner form definition jsem odstranil `[hidden cnp]`
  - v `index.php` jsem navýšil `CONTEST_MANAGER_BOOTSTRAP_VERSION` na `5`, aby se upravené definice a backfill propsaly runtime bootstrapem

#### 2. Závodní stav u duplicate flagu registrace

- soubor: `wp-content/plugins/contest-manager-universal/includes/cf7-handler.php`
- soubor: `wp-content/plugins/contest-manager-universal/includes/install.php`
- při dvou souběžných registracích se stejným datem a částkou mohly oba requesty projít s `duplicate = 0`
- původní logika kontrolovala duplicitu jen před insertem
- oprava:
  - doplnil jsem `contest_plugin_recalculate_contestant_duplicate_group()`
  - po každém insertu registrace se duplicate flag dopočítá nad celou skupinou `purchase_date + purchase_price`
  - doplnil jsem i `contest_plugin_maybe_recalculate_contestant_duplicates()` do bootstrapu jako backfill starších dat
- současně jsem srovnal mapu formátů u `wpdb->insert()`, kde bylo o jeden formát víc, než skutečně zapisovaných polí

### Co jsem ověřil po opravách

- `php -l` prošel pro:
  - `wp-content/plugins/contest-manager-universal/includes/functions.php`
  - `wp-content/plugins/contest-manager-universal/includes/install.php`
  - `wp-content/plugins/contest-manager-universal/includes/cf7-handler.php`
  - `wp-content/plugins/contest-manager-universal/index.php`
- registrace z homepage zapisuje soutěžícího a queue item `contest_entry`
- daily / weekly / main draw fungují při správně připravených testovacích datech a ukládají výherce do `cm_winners`
- winner route:
  - platný a čekající výherce dostane formulář
  - po odeslání účtenky dostane success stav
  - neplatný nebo nevýherní CNP vrací `404`
- upload účtenky po opravě korektně:
  - uloží soubor do `uploads/contest_uploads`
  - zapíše / aktualizuje `cm_receipts`
  - změní stav ve `cm_winners`
- schválení účtenky a odeslání výhry přes admin AJAX fungují a zapisují navazující queue eventy
- bootstrap backfill po navýšení verze opravil i už vzniklé špatně označené duplicity

### Cleanup po testu

- všechny dočasné smoke záznamy jsem z databáze odstranil
- smazal jsem i dočasně nahranou smoke účtenku a pomocné `/tmp` soubory
- po cleanupu jsou contest tabulky opět prázdné a homepage je zpět v setup stavu

### Důležitá poznámka

- jediné, co zatím nešlo ověřit end-to-end, je reálné odeslání transakčních e-mailů do Ecomail API
- fronta `cm_ecomail` a její zápisy byly ověřené, ale skutečný API call zůstává blokovaný chybějící `ECOMAIL_API_KEY`, `ECOMAIL_LIST_ID` a `ECOMAIL_TEMPLATE_ID`


## 2026-03-31 - Následný code review a cleanup po hlubokém auditu

**Souvisejici commity:** `OL-0009 | Integrate contest frontend into CreativeHeroes`, `OL-0010 | Harden contest form processing and winner flow`

### Co jsem dělal

- po předchozím contest auditu jsem prošel diff znovu jako code review se zaměřením na bugy, regresní rizika a mrtvý kód
- kontroloval jsem hlavně:
  - `contest-manager-universal/includes/cf7-handler.php`
  - `src/php/inc/contest.php`
  - `src/templates/page.php`
  - `src/assets/js/main.js`
  - `src/assets/css/component/contest.styl`
  - synchronizovanou runtime kopii v `wp-content/themes/CreativeHeroes`

### Co jsem našel

- po zjednodušení veřejného contest shellu zůstala v theme velká mrtvá contest vrstva
  - nevyužívaný countdown JS modul
  - nevyužitý PHP context builder s DB dotazy a přípravou metrik / prize groups
  - rozsáhlé contest CSS selektory pro hero, badge, panels, metrics, countdown a prize grid, které se už nikde nerenderovaly
- v registraci zůstávala implicitní normalizace `purchase_date` přes `date('Y-m-d', strtotime(...))`
  - fungovala pro běžný validní vstup, ale nebyla dost deterministická a zbytečně spoléhala na implicitní chování PHP

### Co jsem upravil

- odstranil jsem import i inicializaci `contestCountdown.js` z `src/assets/js/main.js`
- odstranil jsem samotný nepoužívaný modul `src/assets/js/modules/contestCountdown.js`
- `src/assets/css/component/contest.styl` jsem přepsal na minimální skutečně používanou vrstvu
- `src/php/inc/contest.php` jsem zredukoval na minimum potřebné pro detekci contest pages a body classes
- v `src/templates/page.php` jsem odstranil předávání nevyužitého `contest` contextu do shell šablony
- v `contest-manager-universal/includes/cf7-handler.php` jsem doplnil helper `contest_manager_normalize_purchase_date_value()`
  - pokud dostane `Y-m-d`, vrací jej beze změny
  - fallback parsuje přes `DateTimeImmutable`
  - invalidní datum vrací `null`
- po změnách jsem spustil `npm run build`, aby se změny propsaly i do runtime theme kopie v `wp-content/themes/CreativeHeroes`

### Co jsem ověřil

- `php -l` prošel nad upravenými PHP soubory
- `npm run lint:styles` prošel
- `npm run build` prošel
- homepage stále vrací setup hlášku bez contest regresu
- neplatná winner URL stále vrací `404`
- po buildu už v runtime theme nezůstávají reference na odstraněný countdown modul ani starý contest context builder

### Závěr review

- po provedeném cleanupu a refactoru jsem nenašel další nový funkční contest regres
- hlavní otevřený gap mimo kód zůstává stejný: reálné Ecomail API odeslání stále nejde ověřit bez `ECOMAIL_*` konfigurace

## 2026-03-31 - Uklid copy na custom login screenu

### Co jsem upravil

- v `wp-content/mu-plugins/creativeheroes-security.php` jsem zjednodusil textovy obsah custom login route `/creative/admin`
- vlevo zustava uz jen brand:
  - `Creative Heroes`
  - `Administrace`
- odstranil jsem doprovodnou copy o zabezpecenem vstupu a pristupu jen pro opravnena uzivatele
- v login karte jsem odstranil titul `Přihlášení` a pomocny text `Použijte své přihlašovací údaje.`
- tlacitko jsem prejmenoval na `Přihlásit`

### Co jsem nemenil

- login route zustava `/creative/admin`
- zustava stejna submit logika, nonce, redirect target i hardening kolem `wp-login.php`
- odkazy `Zapomenuté heslo` a `Zpět na web` zustaly beze zmeny

### Co jsem overil

- `php -l wp-content/mu-plugins/creativeheroes-security.php`
- HTML vystup `http://olympia.loc/creative/admin/` obsahuje uz jen pozadovanou zjednodusenou copy

## 2026-03-31 - Reset admin pristupu na zadost uzivatele

### Co jsem udelal

- pro admin ucet `creativeheroes_admin` jsem pres WordPress API zmenil e-mail na `o.vetiska@creativeheroes.cz`
- soucasne jsem provedl rotaci hesla podle hodnoty dodane uzivatelem
- heslo jsem zamerne nikam neukladal do dokumentace

### Jak jsem to delal

- nejprve jsem si vypsal administratory pres `get_users(["role" => "administrator"])`
- zmenu jsem provedl pres `wp_update_user()` po nacteni `wp-load.php`
- nasledne jsem stav overil pres `get_user_by()` a `wp_check_password()` proti nove ulozenemu hashi

## 2026-03-31 - Odstraneni widgetu `WordPress akce a novinky` z admin dashboardu

### Co jsem upravil

- do `src/php/inc/cleanup.php` i `wp-content/themes/CreativeHeroes/inc/cleanup.php` jsem doplnil funkci `creativeheroes_remove_core_dashboard_news_widgets()`
- ta na hooku `wp_dashboard_setup` s prioritou `999` vola `remove_meta_box('dashboard_primary', 'dashboard', 'side')`

### Proc takto

- `WordPress akce a novinky` je core dashboard widget `dashboard_primary`
- core ho registruje uvnitr `wp_dashboard_setup()` a az pote vyvola vnitrni `do_action('wp_dashboard_setup')`
- proto je spravne odstranit ho prave tady, ne drive
- po odstraneni se widget uz nevyrenderuje a nespousti se jeho dashboard output callback s feed obsahem

### Co jsem overil

- `php -l src/php/inc/cleanup.php`
- `php -l wp-content/themes/CreativeHeroes/inc/cleanup.php`
- `has_action('wp_dashboard_setup', 'creativeheroes_remove_core_dashboard_news_widgets')` vraci `999`
- po simulaci `wp_dashboard_setup()` je `dashboard_primary` v metaboxech uz jen jako `false`, ne jako aktivni widget k vykresleni

## 2026-03-31 - Font preload cleanup po zmene na Gilroy

### Co jsem upravil

- v `src/php/inc/assets.php` i `wp-content/themes/CreativeHeroes/inc/assets.php` jsem prepsal seznam fontovych manifest keys z puvodnich:
  - `Inter-Regular`
  - `PlayfairDisplay-Bold`
  - `PlayfairDisplay-Black`
  - `PlayfairDisplay-BoldItalic`
- na nove:
  - `Gilroy-Regular`
  - `Gilroy-Bold`
  - `Gilroy-Black`
  - `icon`
- pustil jsem `npm run build`, aby se propsal novy manifest a runtime CSS bundle

### Co jsem pri tom nasel

- `src/assets/css/global/font.styl` uz ukazoval na `Gilroy-*`, ale ve zdrojovem adresari chybel `src/assets/fonts/icon.woff2`
- build kvuli tomu predtim neumel `icon` dostat do manifestu a preloadu
- docasne jsem proto doplnil `src/assets/fonts/icon.woff2` z existujici runtime kopie, aby build zustal konzistentni
- nasledne jsem odstranil zbytecny PHP fallback pro `icon`, protoze po novem buildu uz je `icon` standardni soucasti manifestu a fallback by zpusoboval dvojity preload

### Co jsem overil

- `php -l src/php/inc/assets.php`
- `php -l wp-content/themes/CreativeHeroes/inc/assets.php`
- `npm run build`
- head `http://olympia.loc/` ted preloaduje jen ctyri fonty:
  - `Gilroy-Regular`
  - `Gilroy-Bold`
  - `Gilroy-Black`
  - `icon`

### Otevreny detail

- uzivatel uvedl, ze finalni `icon` font doda zvlast
- aktualni `src/assets/fonts/icon.woff2` je zatim pouze provizorni kopie pro konzistentni build a preload pipeline

## 2026-03-31 - Důsledné vyčištění `ping_sites`

### Co jsem upravil

- v `src/php/inc/cleanup.php` i `wp-content/themes/CreativeHeroes/inc/cleanup.php` jsem zvedl `CREATIVEHEROES_SITE_DEFAULTS_VERSION` na `3`
- do site defaults jsem doplnil `update_option('ping_sites', '')`
- doplnil jsem filtry:
  - `pre_option_ping_sites`
  - `pre_update_option_ping_sites`
- obě vrací / vynucují prázdný řetězec, takže se WordPress k žádné notifikační službě nebude připojovat a ani si ji z adminu znovu neuloží

### Co jsem musel dočistit runtime

- protože nový `pre_option_ping_sites` vracel už během změny prázdnou hodnotu, `update_option()` samo starý DB záznam nepřepsalo
- proto jsem finální vyčištění aktuální instance dokončil přímo přes `$wpdb->update()` nad option `ping_sites`

### Co jsem overil

- `get_option('ping_sites')` vraci `''`
- fyzicka hodnota `ping_sites` v DB je vycistena
- `default_ping_status` i `default_pingback_flag` zustavaji vypnute uz z drivejska

## 2026-03-31 - Seed testovaci souteze a docs vrstva pro provoz

**Souvisejici commit:** `OL-0012 | Add contest test seed and operational docs`

### Co jsem udelal

- vytvoril jsem `tools/seed-test-contest.php` jako opakovatelny provisioning testovaci souteze
- seeduje se:
  - homepage blok s testovacim obsahem
  - `/gdpr/`
  - `/pravidla-souteze/`
  - `cm_periods`
  - `cm_main_prize_periods`
  - `cm_prizes`
  - `cm_contestants`
- seed zaroven pred vlozenim novych testovacich dat uklizi stare testovaci zaznamy v:
  - `cm_contestants`
  - `cm_winners`
  - `cm_receipts`
  - `cm_ecomail`
- do seedu jsem dopsal i lidsky citelny summary vypis s CNP kody, ucelem testovacich osob a stavem Ecomail konfigurace
- do `src/docs` jsem pridal tri dokumenty:
  - `documentation.md`
  - `documentation-full.md`
  - `documentation-install.md`
- `README.md` jsem zmenil na rozcestnik dokumentace

### Proc jsem to udelal takto

- user chtel mit pripravenou testovaci soutez bez rucniho klikaní a bez rizika, ze se pri dalsim testu zapomene na nektery contest krok
- seed je lepsi nez rucni SQL, protoze je opakovatelny, citelny a drzi projektovy kontext v repozitari
- dokumentace v `src/docs` je potreba, protoze puvodni `CM Manual` uz neodpovida presne aktualnimu stavu `olympia`
  - registrace je na homepage
  - login je `/creative/admin`
  - ACF je v kodu
  - `wps-hide-login` se nepouziva

### Co je nyni potvrzene

- seed po opakovanem spusteni vraci stejny vychozi testovaci stav
- je pripraveny dataset pro:
  - registraci
  - daily / weekly / main draw
  - receipt upload
  - schvaleni a odeslani vyhry
  - duplicate scenar
- dokumentace v repozitari existuje a odpovida aktualnimu kodu
- `README.md` uz neobsahuje jen placeholder, ale vede na skutecne docs

### Co stale neni potvrzene end-to-end

- skutecne doruceni transakcnich emailu do Ecomail API
  - chybi `ECOMAIL_*` env konfigurace

## 2026-03-31 - Obchodnik a normalizace contest formularu

### Co jsem udelal

- potvrdil jsem, ze `first_name`, `last_name`, `gdpr` a `newsletter` uz ve formulari existuji a neni potreba je zavadet znovu
- do registracniho formulare jsem doplnil povinne pole `merchant_name`
- `purchase_date` jsem prevedl z nativniho `date` inputu na textovy vstup s maskou `DD.MM.RRRR`
- `purchase_price` jsem prevedl z nativniho `number` inputu na textovy numericky vstup bez spinneru
- `zip_code` u registrace i winner formulare jsem doplnil o frontend masku
- do `cf7-handler.php` jsem doplnil autoritativni normalizaci pro datum, cas, PSC a cenu
- do registracniho insertu jsem pridal `merchant_name`
- `merchant_name` jsem propsal do admin seznamu soutezicich, preview losovani a detailu vyherce
- seed testovacich contestant dat jsem rozsiryl o `merchant_name`
- build pipeline jsem doplnil o novy modul `src/assets/js/modules/contestFormFields.js`

### Dulezite technicke rozhodnuti

- masky na frontendu jsou pouze pomocne; rozhodujici je server-side normalizace a validace, aby neslo obejit pravidla pres upraveny request
- kvuli spolehlivosti jsem nedaval `merchant_name` jako slepou zmenu jen do `dbDelta`, ale doplnil jsem guardovanou migraci `contest_plugin_ensure_contestant_merchant_column()`
- pri kontrole jsem odhalil, ze sync bootstrap formulare prepina jen `form` a `additional_settings`; proto jsem ho rozsiryl i o `mail` properties a zvedl bootstrap verzi na `7`

### Co jsem overil

- `php -l` nad pluginovymi soubory a seed skriptem
- `npm run build`
- `contest_plugin_schema_version = 4`
- `contest_manager_bootstrap_version = 7`
- existence sloupce `merchant_name` v `cm_contestants`
- ulozena definice registracniho formulare obsahuje `merchant_name`, `js-form-date`, `js-form-time`, `js-form-zip`, `js-form-price`
- CF7 `mail.body` pro registracni formular uz obsahuje `[merchant_name]`
- winner formular obsahuje `js-form-zip`
- buildovany runtime JS bundle obsahuje nove maskovaci selektory

### Co zustava otevrene

- pokud se pozdeji ukaze potreba, muzeme `merchant_name` doplnit i do report/export vrstvy; dnes je pole pripraveno pro registraci, administraci, draw preview a seed/test data
- v `includes/pages/contestants.php` jsem oddelil admin sloupce `Jmeno` a `Prijmeni` a doplnil `last_name` i do `order_map`, aby slo tridit podle obou poli.
- do `includes/pages/contestants.php` jsem doplnil sloupec `Cas nakupu` vcetne `purchase_time` v `order_map` a zobrazeni ve formatu `H:i` s fallbackem `—` pro prazdny / nulovy cas.
- v `src/assets/js/modules/contestFormFields.js` jsem doplnil klientskou validaci pro contest registration i winner form; validují se required stavy, e-mail, telefon, PSČ, datum, čas, cena a receipt upload
- v `src/assets/css/global/form.styl` jsem doplnil `form__item--invalid` / `form__item--valid`, absolutně pozicované chybové hlášky a červené / zelené vizuální stavy inputů i checkbox labelů
- technicky jsou contest pole kromě newsletteru už v CF7 required; tento krok doplnil hlavně konzistentní UX vrstvu nad stávající server-side validací
- v `src/assets/css/global/icon.styl` jsem udelal minimalni syntaktickou opravu odsazeni, aby znovu prosla build pipeline bez zmeny obsahove definice ikon
- po teto oprave uz `npm run build` probehl a nova validacni vrstva je propsana i do runtime assetu theme
- do `includes/install.php` jsem doplnil centralni `contest_manager_get_cf7_messages_definition()` s ceskymi CF7 message texty pro oba contest formulare
- sync bootstrapu jsem rozsiryl i o `messages` properties a zvedl `CONTEST_MANAGER_BOOTSTRAP_VERSION` na `8`, aby se ceske texty propsaly i do uz ulozenych formulare
- overil jsem ulozene CF7 properties obou formularu: `invalid_required`, `invalid_email` i `upload_failed` jsou uz v cestine
- v `src/assets/css/global/form.styl` jsem prevedl contest checkboxy na plne custom `appearance none` variantu s animovanym checkem pres `::before` a `::after`, vcetne hover/focus/valid/invalid stavu a bez zasahu do CF7 HTML

## 2026-03-31 - Admin prepinac header search

### Co jsem menil

- v `src/php/inc/template-tags.php` a runtime kopii `wp-content/themes/CreativeHeroes/inc/template-tags.php` jsem rozsiryl `creativeheroes_navigation_settings` o `header_search_enabled`
- doplnil jsem helper `creativeheroes_is_header_search_enabled()` s defaultem `false`
- v metaboxu `CreativeHeroes Menu` na `nav-menus.php` jsem pridal checkbox `Zobrazit vyhledavani v hlavicce`
- v `src/templates/template-parts/layout/site-header.php` a runtime kopii jsem cely search blok obalil podminkou podle helperu

### Proc

- uzivatel chtel, aby slo header search zapinat a vypinat primo v administraci
- zaroven mel byt v defaultu vypnuty a pri vypnuti kompletne odstraneny z frontend HTML
- stavajici admin settings vrstva pro menu uz v theme existovala, takze bylo nejcistsi navazat na ni misto zavadeni dalsi options stranky

### Co jsem overil

- `php -l` nad obema upravenymi `template-tags.php` i `site-header.php`
- `curl http://olympia.loc/` potvrzuje, ze pri defaultnim stavu se search v hlavicce vubec nevyrenderuje
- `php -r` nad `wp-load.php` potvrzuje `header_search_enabled => false` a `creativeheroes_is_header_search_enabled() === false`

## 2026-03-31 - Commit mapa OL-0013 az OL-0015

- `OL-0013 | Add admin toggle for header search`
  - commit obsahuje jen `template-tags.php` a `site-header.php` v source i runtime kopii theme
- `OL-0014 | Extend contest registration fields and validation rules`
  - commit obsahuje contest plugin a seed upravy kolem `merchant_name`, normalizace vstupu, ACF hlasek a admin seznamu soutezicich
- `OL-0015 | Add contest form validation and refresh theme assets`
  - commit obsahuje frontend JS/CSS validaci, novy icon/font asset set, Vite manifest a runtime CSS/JS bundle
  - pred commitem jsem zkontroloval i regresni riziko noveho icon fontu a doplnil CSS fallbacky pro aktualne pouzivane `arrow`, `search` a `clock`

## 2026-03-31 - Checkbox UI tuning

- v `src/assets/css/global/form.styl` jsem upravil checkbox stavy tak, aby `newsletter` po zaskrtnuti svitil zelene i bez `form__item--valid`
- zaroven jsem vypnul obalovy label border u `.form__item--acceptance` pro `invalid` i `valid` stav; chyba se ted projevi jen na samotnem checkbox poli a pres hlasku pod nim
- po zmene jsem znovu spustil `npm run build`, takze se uprava propsala i do runtime theme CSS bundle

## 2026-03-31 - Screen reader response fix

- do `src/assets/css/global/form.styl` jsem doplnil `sr-only` pravidla pro `.wpcf7 .screen-reader-response`
- nevolil jsem `display: none`, aby se nerozbila pristupnost ani CF7 `aria-live` mechanika
- po uprave jsem znovu spustil `npm run build`, takze se zmena propsala do runtime theme CSS bundle

## 2026-03-31 - Winner preview route

- do `includes/functions.php` jsem doplnil helpery `contest_manager_get_winner_preview_token()`, `contest_manager_is_winner_preview_cnp()` a `contest_manager_get_winner_preview_url()`
- do `includes/winner-page.php` jsem pridal specialni preview vetveni, ktere nevyzaduje realneho vyherce v `cm_winners`
- renderovany CF7 markup v preview upravuju pres `contest_manager_prepare_winner_form_markup_for_preview()`
  - form dostane `onsubmit="return false;"`
  - submit se meni na `type="button"`
  - preview stranka zobrazi informaci, ze odeslani je vypnute
- v `includes/cf7-handler.php` a `cf7_save_receipt_file()` jsem doplnil server-side guard, aby preview token nesel zneuzit k zapisum ani pri rucnim POSTu
- overil jsem `php -l`, realny `200 OK` na preview route a to, ze HTML opravdu obsahuje `data-preview="winner"` a nepouziva submit flow
