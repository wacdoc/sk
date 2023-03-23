# Vytvorte si vlastný server na odosielanie pošty SMTP

## preambula

SMTP môže priamo nakupovať služby od dodávateľov cloudu, ako napríklad:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali cloud email push](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Môžete si tiež vybudovať vlastný poštový server - neobmedzené odosielanie, nízke celkové náklady.

Nižšie uvádzame krok za krokom, ako vytvoriť vlastný poštový server.

## Výber servera

Server SMTP s vlastným hosťovaním vyžaduje verejnú IP s otvorenými portami 25, 456 a 587.

Bežne používané verejné cloudy tieto porty predvolene zablokovali a možno ich bude možné otvoriť zadaním pracovného príkazu, ale je to napokon veľmi problematické.

Odporúčam nákup od hostiteľa, ktorý má tieto porty otvorené a podporuje nastavenie reverzných doménových mien.

Tu odporúčam [Contabo](https://contabo.com) .

Contabo je poskytovateľ hostingu so sídlom v Mníchove v Nemecku, založený v roku 2003 s veľmi konkurenčnými cenami.

Ak si ako menu nákupu zvolíte Euro, cena bude lacnejšia (server s 8GB pamäťou a 4 CPU stojí približne 529 juanov ročne a počiatočný inštalačný poplatok je na jeden rok zadarmo).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Pri zadávaní objednávky uveďte, `prefer AMD` a server s procesorom AMD bude mať lepší výkon.

V nasledujúcom texte uvediem VPS od Contabo ako príklad, ktorý demonštruje, ako si vytvoriť svoj vlastný poštový server.

## Konfigurácia systému Ubuntu

Operačný systém je tu Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Ak server na ssh zobrazí `Welcome to TinyCore 13!` (ako je znázornené na obrázku nižšie), znamená to, že systém ešte nebol nainštalovaný. Odpojte ssh a počkajte niekoľko minút na opätovné prihlásenie.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Keď sa zobrazí `Welcome to Ubuntu 22.04.1 LTS` , inicializácia je dokončená a môžete pokračovať podľa nasledujúcich krokov.

### [Voliteľné] Inicializujte vývojové prostredie

Tento krok je voliteľný.

Pre pohodlie som inštaláciu a konfiguráciu systému softvéru ubuntu vložil na [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Spustite nasledujúci príkaz a nainštalujte ho jedným kliknutím.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Čínski používatelia, použite namiesto toho nasledujúci príkaz a jazyk, časové pásmo atď. sa automaticky nastaví.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo umožňuje IPV6

Povoľte IPV6, aby SMTP mohol odosielať e-maily aj s adresami IPV6.

upraviť súbor `/etc/sysctl.conf`

Upravte alebo pridajte nasledujúce riadky

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Pokračujte s [príručkou contabo: Pridanie pripojenia IPv6 na váš server](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Upravte `/etc/netplan/01-netcfg.yaml` , pridajte niekoľko riadkov, ako je znázornené na obrázku nižšie (predvolený konfiguračný súbor Contabo VPS už tieto riadky obsahuje, stačí ich odkomentovať).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Potom `netplan apply` aby sa upravená konfigurácia prejavila.

Po úspešnej konfigurácii môžete použiť `curl 6.ipw.cn` na zobrazenie adresy ipv6 vašej externej siete.

## Klonujte operácie úložiska konfigurácie

```
git clone https://github.com/wactax/ops.soft.git
```

## Vygenerujte si bezplatný certifikát SSL pre názov svojej domény

Odosielanie pošty vyžaduje certifikát SSL na šifrovanie a podpisovanie.

Na generovanie certifikátov používame [acme.sh.](https://github.com/acmesh-official/acme.sh)

acme.sh je open source automatický nástroj na podpisovanie certifikátov,

Vstúpte do konfiguračného skladu ops.soft, spustite `./ssl.sh` a v **hornom adresári** sa vytvorí priečinok `conf` .

Nájdite svojho poskytovateľa DNS z [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , upravte `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Potom spustite `./ssl.sh 123.com` a vygenerujte certifikáty `123.com` a `*.123.com` pre názov vašej domény.

Pri prvom spustení sa automaticky nainštaluje [acme.sh](https://github.com/acmesh-official/acme.sh) a pridá sa plánovaná úloha na automatické obnovenie. Môžete vidieť `crontab -l` , existuje takýto riadok nasledovne.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Cesta k vygenerovanému certifikátu je niečo ako `/mnt/www/.acme.sh/123.com_ecc。`

Obnova certifikátu zavolá skript `conf/reload/123.com.sh` , upravte tento skript, môžete pridať príkazy ako `nginx -s reload` na obnovenie vyrovnávacej pamäte certifikátov súvisiacich aplikácií.

## Zostavte SMTP server s chasquidom

[chasquid](https://github.com/albertito/chasquid) je open source SMTP server napísaný v jazyku Go.

Ako náhrada za staré programy poštových serverov, ako sú Postfix a Sendmail, je chasquid jednoduchší a ľahšie použiteľný a je tiež jednoduchší na sekundárny vývoj.

Spustiť `./chasquid/init.sh 123.com` sa nainštaluje automaticky jedným kliknutím (nahraďte 123.com názvom vašej odosielacej domény).

## Nakonfigurujte DKIM pre podpis e-mailu

DKIM sa používa na odosielanie e-mailových podpisov, aby sa zabránilo tomu, že listy budú považované za spam.

Po úspešnom spustení príkazu sa zobrazí výzva na nastavenie záznamu DKIM (ako je uvedené nižšie).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Stačí pridať záznam TXT do vášho DNS (ako je uvedené nižšie).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Zobrazenie stavu služby a denníkov

 `systemctl status chasquid` Zobrazenie stavu služby.

Stav normálnej prevádzky je znázornený na obrázku nižšie

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` alebo `journalctl -xeu chasquid` môže zobraziť protokol chýb.

## Reverzná konfigurácia názvu domény

Reverzný názov domény má umožniť preloženie IP adresy na zodpovedajúci názov domény.

Nastavenie reverzného názvu domény môže zabrániť tomu, aby boli e-maily identifikované ako spam.

Keď je pošta prijatá, prijímajúci server vykoná reverznú analýzu názvu domény na IP adrese odosielajúceho servera, aby potvrdil, či má odosielajúci server platný reverzný názov domény.

Ak odosielajúci server nemá reverzný názov domény alebo ak sa reverzný názov domény nezhoduje s IP adresou odosielajúceho servera, prijímajúci server môže rozpoznať e-mail ako spam alebo ho odmietnuť.

Navštívte [https://my.contabo.com/rdns](https://my.contabo.com/rdns) a nakonfigurujte, ako je uvedené nižšie

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Po nastavení reverzného názvu domény nezabudnite nakonfigurovať dopredné rozlíšenie názvu domény ipv4 a ipv6 na server.

## Upravte názov hostiteľa chasquid.conf

Upravte `conf/chasquid/chasquid.conf` na hodnotu reverzného názvu domény.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Potom spustite `systemctl restart chasquid` , aby ste reštartovali službu.

## Zálohujte conf do úložiska git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Napríklad zálohujem priečinok conf do vlastného procesu github nasledovne

Najprv vytvorte súkromný sklad

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Zadajte adresár conf a odošlite do skladu

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Pridať odosielateľa

behať

```
chasquid-util user-add i@wac.tax
```

Môže pridať odosielateľa

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Skontrolujte, či je heslo nastavené správne

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Po pridaní používateľa sa aktualizuje `chasquid/domains/wac.tax/users` , nezabudnite ho odoslať do skladu.

## DNS pridať SPF záznam

SPF (Sender Policy Framework) je technológia overovania e-mailov, ktorá sa používa na zabránenie podvodom s e-mailom.

Overuje identitu odosielateľa pošty kontrolou, či sa IP adresa odosielateľa zhoduje s DNS záznamami názvu domény, za ktorú sa vydáva, čím bráni podvodníkom v odosielaní falošných e-mailov.

Pridanie SPF záznamov môže čo najviac zabrániť tomu, aby boli e-maily identifikované ako spam.

Ak váš doménový server nepodporuje typ SPF, stačí pridať záznam typu TXT.

Napríklad SPF `wac.tax` je nasledovný

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF pre `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Všimnite si, že tu mám `include:_spf.google.com` , pretože neskôr nakonfigurujem `i@wac.tax` ako adresu odosielania v poštovej schránke Google.

## Konfigurácia DNS DMARC

DMARC je skratka pre (Domain-based Message Authentication, Reporting & Conformance).

Používa sa na zachytávanie nedostatkov SPF (možno spôsobené chybami v konfigurácii alebo sa niekto iný vydáva za vás, aby posielal spam).

Pridať záznam TXT `_dmarc` ,

Obsah je nasledovný

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Význam každého parametra je nasledujúci

### p (Zásady)

Označuje, ako spracovať e-maily, ktoré neprejdú overením SPF (Sender Policy Framework) alebo DKIM (DomainKeys Identified Mail). Parameter p možno nastaviť na jednu z troch hodnôt:

* žiadne: Nevykoná sa žiadna akcia, odosielateľovi sa odošle späť iba výsledok overenia prostredníctvom mechanizmu nahlasovania e-mailov.
* Karanténa: Umiestnite poštu, ktorá neprešla overením, do priečinka nevyžiadanej pošty, ale správu priamo neodmietne.
* odmietnuť: Priamo odmietnuť e-maily, ktoré zlyhali pri overení.

### fo (možnosti zlyhania)

Určuje množstvo informácií vrátených mechanizmom podávania správ. Dá sa nastaviť na jednu z nasledujúcich hodnôt:

* 0: Hlásiť výsledky overenia pre všetky správy
* 1: Hlásiť iba správy, ktoré zlyhali pri overení
* d: Hláste len zlyhania overenia názvu domény
* s: hlásiť iba zlyhania overenia SPF
* l: Hlásiť iba zlyhania overenia DKIM

### rua & ruf

* rua (URI prehľadov pre súhrnné prehľady): E-mailová adresa na prijímanie súhrnných prehľadov
* ruf (URI hlásenia pre forenzné správy): e-mailová adresa na prijímanie podrobných správ

## Pridajte záznamy MX na preposielanie e-mailov do služby Google Mail

Pretože som nenašiel bezplatnú firemnú schránku, ktorá by podporovala univerzálne adresy (Catch-All, môže prijímať akékoľvek e-maily odoslané na toto doménové meno, bez obmedzení na predpony), použil som chasquid na preposielanie všetkých e-mailov do mojej poštovej schránky Gmail.

**Ak máte vlastnú platenú firemnú poštovú schránku, neupravujte MX a preskočte tento krok.**

Upraviť `conf/chasquid/domains/wac.tax/aliases` , nastaviť poštovú schránku na preposielanie

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` označuje všetky e-maily, `i` je predpona e-mailovej adresy odosielajúceho používateľa vytvorená vyššie. Na preposielanie pošty musí každý používateľ pridať riadok.

Potom pridajte záznam MX (tu ukazujem priamo na adresu reverzného názvu domény, ako je znázornené v prvom riadku na obrázku nižšie).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Po dokončení konfigurácie môžete použiť iné e-mailové adresy na odosielanie e-mailov na adresy `i@wac.tax` a `any123@wac.tax` , aby ste zistili, či môžete prijímať e-maily v službe Gmail.

Ak nie, skontrolujte protokol chasquid ( `grep chasquid /var/log/syslog` ).

## Pošlite e-mail na adresu i@wac.tax pomocou služby Google Mail

Keď Google Mail dostal e-mail, prirodzene som dúfal, že odpoviem s `i@wac.tax` namiesto i.wac.tax@gmail.com.

Navštívte [stránku https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) a kliknite na položku Pridať ďalšiu e-mailovú adresu.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Potom zadajte overovací kód prijatý e-mailom, ktorý bol preposlaný.

Nakoniec ju možno nastaviť ako predvolenú adresu odosielateľa (spolu s možnosťou odpovedať s rovnakou adresou).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

Týmto sme dokončili zriadenie poštového servera SMTP a zároveň využívame Google Mail na odosielanie a prijímanie emailov.

## Odošlite testovací e-mail a skontrolujte, či je konfigurácia úspešná

Zadajte `ops/chasquid`

Spustiť `direnv allow` inštaláciu závislostí (direnv bol nainštalovaný v predchádzajúcom inicializačnom procese jedným kľúčom a do shellu bol pridaný hák)

potom utekaj

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Význam parametrov je nasledujúci

* užívateľ: SMTP užívateľské meno
* pass: heslo SMTP
* komu: príjemcovi

Môžete poslať skúšobný e-mail.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Odporúča sa používať Gmail na prijímanie testovacích e-mailov, aby ste skontrolovali, či sú konfigurácie úspešné.

### Štandardné šifrovanie TLS

Ako je znázornené na obrázku nižšie, je tu tento malý zámok, čo znamená, že certifikát SSL bol úspešne povolený.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Potom kliknite na „Zobraziť pôvodný e-mail“

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Ako je znázornené na obrázku nižšie, na pôvodnej poštovej stránke Gmailu sa zobrazuje DKIM, čo znamená, že konfigurácia DKIM je úspešná.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Začiarknite políčko Received v hlavičke pôvodného e-mailu a uvidíte, že adresa odosielateľa je IPV6, čo znamená, že IPV6 je tiež úspešne nakonfigurovaný.
