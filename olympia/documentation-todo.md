# Olympia TODO

## Stav po uvodni analyze 2026-03-31

### Hotovo

- Nactena dokumentace `contest`.
- Projita struktura projektu `olympia`, `wordpress` a `contest`.
- Overena shoda WordPress jadra mezi vsemi tremi projekty.
- Overeno, ze:
  - `CreativeHeroes` v `olympia` odpovida `wordpress`
  - `contest` theme v `olympia` odpovida `contest`
  - `olympia/wp-config.php` odpovida `wordpress`, ne `contest`
- Identifikovan chybejici plugin `contest-manager-universal` v `olympia`.
- Identifikovana hlavni kolize:
  - `creativeheroes-security` blokuje `wp-json`
  - `Contact Form 7` potrebuje `wp-json` pro verejny submit
- Zpracovano doporuceni k `CH-cleanup`:
  - nepouzivat jako runtime plugin
  - cleanup drzet v jadru projektu

### Otevrene body

- Doplnit realna soutezni data do `olympia`.
  - contest periods
  - prizes
  - pripadne hlavni obdobi, pokud budou pouzita
  - bez toho zustane registrace spravne zablokovana jako nenakonfigurovana

- Vyresit zbyvajici ACF field groups pro verejny frontend stareho theme `contest`.
  - `contest-manager-universal` field groups jsou uz zapsane v kodu
  - otevrene zustavaji obsahove fieldy stare soutezni sablony

- Rozhodnout, ktere ACF option fieldy zustanou:
  - ve souteznim pluginu
  - v `CreativeHeroes` settings
  - nebo se prevedou na jinou konfiguraci

- Navrhnout novou strukturu soutezniho frontendu v `CreativeHeroes`.
  - page template
  - template parts
  - Stylus
  - JS bez jQuery

- Zabalit direct DB dotazy ze stareho `contest` theme do helper vrstvy nebo pluginu.

- Rozhodnout, ktere prvky z `CH-cleanup` se presunou do jadra.
  - kandidati:
    - `remove_jqmigrate`
    - `dequeue_dashicons`
    - `heartbeat_throttle`
    - `hide_theme_editors`

- Overit, zda bude potreba zachovat age-gate flow a jak presne zapadne do `CreativeHeroes`.

### Doporucene dalsi poradi

1. Potvrdit a srovnat finalni lokalni URL `olympia`, aby uz neodkazovala na zdedeny `wordpress` path.
2. Navrhnout a implementovat soutezni page template ve `CreativeHeroes`.
3. Prenest obsahove ACF fieldy stareho `contest` frontendu do kodu nebo do nove struktury settings.
4. Portovat verejny soutezni vizual a obsah ze stareho theme `contest` do `CreativeHeroes`.
5. Teprve potom resit dalsi cleanup standardizaci a jemne UI detaily.

## Stav po implementaci 2026-03-31

### Hotovo navic oproti uvodni analyze

- Vytvorena samostatna DB `olympia`.
- `olympia/wp-config.php` prepnut na DB `olympia`.
- Aktivovany pluginy:
  - ACF PRO
  - Contact Form 7
  - Contest Manager Universal
- `contest-manager-universal` dostal runtime bootstrap:
  - contest upload page
  - contest registration page
  - contest CF7 formulare
  - kodove ACF field groups pro plugin settings
  - centralni option helpery a fallbacky
- `creativeheroes-security` ma bezpecnou vyjimku pro verejne CF7 contest REST endpointy.
- Winner flow bylo provereno smoke testem nad docasnym testovacim CNP.
- Potvrzena finalni lokalni URL `http://olympia.loc`.
- Opravena `.htaccess` z puvodniho `/wordpress/` rewrite na root instalaci.
- `contest-registration` a `contest-upload` uz nevraceji `500`.
- Invalidni `/v/{cnp}` route vraci korektni `404` bez zanozene druhe sablony.
- Contest dashboard nove zobrazuje checklist konfigurace.
- Contest frontend dostal vlastni scoped stylesheet:
  - notice karty
  - shell pro registraci a winner flow
  - zakladni stylovani CF7 prvku

### Otevrene body po posledni integraci

- Vlozit realne contest periods a prizes, aby verejna registrace presla z onboarding warningu do aktivniho formulare.
- Doresit finalni Ecomail env hodnoty, pokud se ma testovat i e-mailova fronta end-to-end.
- Rozhodnout, zda vratit puvodni Stylus/Vite theme zdroje pro `CreativeHeroes`, nebo ponechat contest frontend vrstvu docasne v plugin assets.

## Aktualizace po CF7 frontend refactoru 2026-03-31

### Hotovo navic

- CF7 frontend CSS z pluginu je vypnute a verejny frontend jede jen na vlastnim theme bundle
- contest plugin uz nema vlastni frontend stylesheet ani jeho enqueue
- contest formularovy styling je presunut do `src/assets/css/global/form.styl`
- contest registracni i winner formular pouzivaji projektove BEM tridy v autorovanem HTML
- existujici CF7 formulare v DB byly synchronizovany na novou markup verzi
- GDPR acceptance je opravena na validni CF7 implementaci s realnou validaci
- otevreny bod o chybejicim Stylus/Vite source tree je uz uzavreny

### Otevrene body po refactoru

- Dodat realna contest periods a prizes, aby se verejna registrace odblokovala i na realne strance a nesla jen pres runtime testy.
- Projit finalni vizualni detaily `form.styl` po doplneni realneho obsahu a contest textu.
- Otestovat end-to-end submit registrace a winner flow na realnych datech po doplneni contest konfigurace.
- Doresit finalni Ecomail env hodnoty, pokud se ma overovat i mail queue a transakcni komunikace.

## Aktualizace po contest page template 2026-03-31

### Hotovo navic

- contest pages uz nejsou renderovane jako bezna `page.php`, ale pres vlastni verejny shell v `CreativeHeroes`
- existuje nova helper vrstva pro contest stav, metriky, prize grouping a support data
- contest registration a winner upload maji samostatny hero, workflow sidebar a verejnou prezencni strukturu
- countdown modul je pripraven pro budouci stav `upcoming`
- invalidni `v/{cnp}` stale vraci korektni `404`, ale v novem contest layoutu

### Otevrene body po tomto kroku

- Naplnit realna contest periods a prizes, aby se contest hero prepnul z `setup` stavu do realneho `upcoming` nebo `active` rezimu.
- Rozhodnout, zda a v jakem rozsahu vratit obsahovou paritu se starym frontendem `contest`:
  - sekce typu `jak vyhrat`
  - produktova sekce
  - pravidla a dalsi obsahove ACF bloky
- Otestovat countdown a prize section na realnych contest datech po naplneni DB.
- Doresit finalni copywriting a obsahove zdroje pro verejne contest sekce, pokud nemaji zustat plne systemove generovane.


## TODO po analýze CM Manualu 2026-03-31

### Kritické provozní body

- Doplnit `cm_periods`.
  - bez toho zůstává registrace v `setup` režimu
- Doplnit `cm_prizes`.
  - bez toho není připravená reálná prize banka ani winner prezentace
- Doplnit Ecomail konfiguraci.
  - `ECOMAIL_API_KEY`
  - `ECOMAIL_LIST_ID`
  - `ECOMAIL_TEMPLATE_ID`
- Doplnit do `wp-config.php` i `DISABLE_WP_CRON`, pokud chceme jet podle manuálu přes externí cron runner.

### Administrace a provozní workflow

- Vytvořit skutečné WordPress role pro provoz:
  - `view_report_page`
  - `view_shipping_page`
- Ověřit, zda chceme role zakládat automaticky pluginem, nebo je držet jako vědomý provisioning krok.
- Otestovat celý workflow:
  - registrace soutěžícího
  - zařazení do DB
  - losování
  - winner upload účtenky
  - schválení účtenky
  - odeslání výhry

### Obsahové a veřejné vrstvy

- Vytvořit GDPR stránku a uložit její URL do contest nastavení.
- Rozhodnout, zda chceme zachovat thank-you page model z manuálu.
  - aktuálně běží inline success flow
  - pokud PM trvá na TP stránkách, bude potřeba contest flow doplnit o redirect variantu
- Rozhodnout, zda projekt potřebuje age gate.
  - pokud ano, doplnit ho do `CreativeHeroes`
- Portovat contest homepage podle manuálu do `CreativeHeroes`.
  - registrace
  - výhry
  - jak soutěžit
  - pravidla
  - případně produktová sekce a další obsahové bloky
- Doplnit reálné rules / GDPR / contest texty od PM.

### Technické dotažení

- Nahradit development fallbacky:
  - `admin@localhost.test`
  - default sender values
- Rozhodnout, zda na nerelevantních stránkách selektivně vypnout i frontend JS Contact Form 7.
  - nyní je globálně načítaný i tam, kde není formulář
- Zvážit doplnění explicitní contest konfigurace pro support kontakty a právní odkazy, aby nezůstávaly jen na fallback hodnotách.


### Bezpečnost a redukce plugin stacku

- Rozhodnout, zda během další stabilizace odstranit i zbývající neaktivní pluginy:
  - `error-log-monitor`
  - `user-role-editor`
- Ověřit, zda chceme `creativeheroes-media` ponechat jako samostatný plugin, nebo jej časem absorbovat do základu projektu.
  - pro contest runtime není kritický
  - pro media workflow a optimalizované obrázky je ale hodnotný
- Navrhnout a implementovat anti-bot ochranu veřejných contest formulářů bez zbytečného plugin bloatu.
  - preferovaně custom honeypot / rate limiting / Turnstile integrace podle požadavků projektu
- Zvážit, zda chceme i selektivní server-level blokaci dalších system aliasů nebo direct PHP entrypointů, pokud se potvrdí potřeba dalšího hardeningu.


### Po přesunu registrace na homepage

- Rozhodnout finální obsah a strukturu homepage contest landing page.
  - nyní homepage renderuje pouze registrační shortcode a minimální setup stav
- Dodat finální veřejné texty místo odstraněných pomocných a onboarding bloků.
- Rozhodnout finální obsah winner upload stránky.
  - nyní je záměrně osekaná na minimum bez nápověd
- Doplnit contest periods a prizes, aby homepage registrace přešla ze setup stavu do ostrého provozu.
- Až budou role zakódované přímo v projektu, teprve potom zvážit odstranění `user-role-editor`.
- Rozhodnout, zda si během stabilizace chceme ponechat `error-log-monitor`, nebo jej také odstranit.


### HTML validace a výstup

- Po další větší obsahové úpravě znovu projet veřejné stránky přes HTML validator.
- Až se bude čistit starter copy, prověřit znovu i textový outline homepage a footeru.


### Po hlubokém contest auditu

- Po doplnění `ECOMAIL_*` konfigurace provést reálný end-to-end test doručení transakčních e-mailů přes Ecomail API.
  - samotné queue zápisy jsou ověřené
  - chybí jen potvrzení reálného API delivery
- Po založení reálných `cm_periods` a `cm_prizes` zopakovat plný integrační sweep na produkčně podobných datech.
  - registrace
  - losování daily / weekly / main
  - winner upload účtenky
  - schválení účtenky
  - odeslání výhry
- V administraci projít i vizuální GUI kontrolu stránek `draws`, `shipping`, `report` a `contestants` nad reálnými daty.
- Při dalším větším zásahu do contest pluginu zopakovat i souběžný test duplicity registrace.
  - závodní stav už je opravený v kódu
  - má smysl ho držet jako regresní scénář


### Po code review a cleanup refactoru

- Při příštím plném contest smoke testu znovu ověřit registraci i s reálným formulářovým submittem po založení `cm_periods`.
  - date normalization byla zpřesněná
  - má smysl ji držet v regresním checklistu
- Po doplnění reálného veřejného contest obsahu znovu zhodnotit, zda je potřeba vracet nějakou bohatší contest layout vrstvu.
  - aktuálně je shell záměrně minimalistický
  - odstraněná hero / metrics / prize prezentace byla mrtvý kód

## Aktualizace po seedu a dokumentaci 2026-03-31

Souvisejici commit: `OL-0012 | Add contest test seed and operational docs`

### Hotovo navic

- existuje opakovatelny seed testovaci souteze v `tools/seed-test-contest.php`
- seed pripravuje kompletni testovaci data pro contest workflow
- `src/docs` ted obsahuje:
  - `documentation.md`
  - `documentation-full.md`
  - `documentation-install.md`
- `README.md` funguje jako rozcestnik dokumentace
- projekt ma konecne zapsany standardni postup pro:
  - lokalni QA
  - zalozeni souteze
  - server instalaci
  - Ecomail a cron runner

### Otevrene body po tomto kroku

- Dopsat realne produkcni contest texty, GDPR a pravidla misto placeholderu ze seedu.
- Po dodani `ECOMAIL_*` konfigurace provest realny end-to-end test doruceni e-mailu do Ecomail API.
- Rozhodnout, zda chceme seed casem rozsirit i o volitelny `reset-only` mod nebo export test reportu.
- Po finalnim obsahovem doplneni znovu projet homepage a contest pages jako kompletni UX a HTML validačni sweep.
- Az se bude pripravovat ostry launch, oddelit testovaci seed data od produkcniho obsahu a zkontrolovat, ze seed nebude pouzit omylem v produkci.

## Aktualizace po rozsireni contest formularu 2026-03-31

### Otevrene body po tomto kroku

- Provest browser-level smoke test realneho odeslani registrace s formatovanymi vstupy `DD.MM.RRRR`, `1330`, `123 45` a decimalni cenou.
- Pri dalsim rozsirovani reportu rozhodnout, zda ma `merchant_name` vstoupit i do CSV / klientskych exportu.
- Pokud bude pozdeji pozadavek na dalsi dynamicka pole, navrhnout je jako rizene schema, ne jako obecny builder formulare.
- Provest rychly browser smoke test live validace contest formulare nad realnym homepage renderem po poslednich zmenach formulare.
- Pri dalsim plnem contest smoke testu znovu projit i realne AJAX odpovedi CF7, aby vsechny ceske systemove hlasky sedely i ve finale po submitu.
