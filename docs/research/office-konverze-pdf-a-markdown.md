# Převod Office dokumentů do PDF a Markdownu: průzkum enginů pro docshift

Stav k 2026-11-25. Každé tvrzení má odkaz na primární zdroj v závorce (číslo `[n]` odkazuje do sekce Zdroje na konci). Co se nepodařilo ověřit z primárního zdroje, je výslovně označeno **NEOVĚŘENO**. Nic nebylo instalováno ani spouštěno, vše je z dokumentace a zdrojového kódu.

Pozn. k metodě: stránky byly čteny přes nástroj WebFetch, který vrací shrnutí stránky. Citace názvů voleb a řádků kódu tedy pocházejí z těchto shrnutí. Před implementací je vhodné klíčové volby ověřit proti nainstalované verzi (`--help`).

---

## A) Převod do PDF

### A1. LibreOffice headless (`soffice --headless --convert-to pdf`)

- **Příkaz a volby.** Syntaxe `--convert-to OutputFileExtension[:OutputFilterName[:OutputFilterParams]]`, výstupní složka `--outdir`, bez ní se použije pracovní adresář. `--headless` je určen pro řízení aplikace externími klienty. Dále existují `--norestore`, `--nolockcheck`, `--infilter`. [1]
- **Profil a souběh.** `-env:UserInstallation=file:///...` nastaví jiný uživatelský profil, LibreOffice potřebuje do profilu zapisovat. [1] Hodnota musí být URL, ne holá cesta (zkušenost z fóra, sekundární, ale shodná s příkladem v oficiální nápovědě [1]; viz také Ask LibreOffice [2]). V bug trackeru The Document Foundation jsou hlášení, že druhá headless instance se stejným profilem skončí tiše bez výstupu (bug 37531 [3]) nebo se druhý převod na Windows zasekne, dokud běží první (bug 113259 [4]); obejitím uváděným v diskusích je unikátní profil na každé volání. Jde o uživatelská hlášení v trackeru, ne o oficiální garanci. Platnost v nejnovější verzi 26.8 **NEOVĚŘENO**. Doporučení pro docshift: samostatný adresář profilu na každý souběžný proces (nebo pool pevného počtu pracovníků, každý s vlastním profilem), ne jeden sdílený profil.
- **Časové limity a zasekávání.** Z oficiální dokumentace příkazové řádky `soffice` nevyplývá žádný přepínač pro timeout převodu [1]. Timeout a zabíjení procesu (včetně potomků, na Windows `taskkill /T`) musí řešit volající (docshift). Dialogy a obnovení po pádu potlačuje `--norestore` [1]. Chování při pádu/zaseknutí na konkrétních souborech **NEOVĚŘENO** z primárního zdroje.
- **Heslem chráněné soubory.** Z dokumentace `--convert-to` [1] nevyplývá volba pro zadání hesla. Hesla umí předat Gotenberg přes pole `password` (viz A2) [10]; samotné CLI **NEOVĚŘENO**. Prakticky: docshift by měl takový soubor rozpoznat (neúspěšný převod, nulový/žádný výstupní PDF) a vrátit definovanou chybu.
- **Poškozené soubory.** Specifikace chování (kód výstupu, zpráva) v oficiální dokumentaci není. **NEOVĚŘENO**. Doporučení: kontrolovat existenci a nenulovou velikost PDF, ne jen kód návratu.
- **Licence.** Zdrojový kód je pod Mozilla Public License v2.0; obsahuje i komponenty pod řadou dalších open-source licencí, které se liší podle verze; přesné znění je v souboru LICENSE instalace nebo v Nápověda > Licenční informace. [5] Použití jako samostatného podprocesu (bez linkování) je z hlediska MPL nejjednodušší. Právní posouzení redistribuce v Docker imagi je mimo rozsah (nutná kontrola licencí přibalených fontů, **NEOVĚŘENO**, stránka fonty neuvádí [5]).
- **Instalace.** Oficiální stahování nabízí Windows MSI (x86-64, ARM64), Linux balíky deb/rpm (tar.gz, x86-64 a aarch64) a macOS DMG; aktuální stabilní vydání je 26.8 (26.8.3). Velikost instalace stránka neuvádí. [6] Oficiální Docker image od The Document Foundation: v použitých zdrojích **NEOVĚŘENO**; běžná praxe je `apt install libreoffice-impress libreoffice-writer` v Debian/Ubuntu imagi (názvy balíků nutno ověřit v dokumentaci distribuce, zde **NEOVĚŘENO**).
- **Věrnost.** Primární zdroj, který by kvantifikoval věrnost PPTX/DOCX ve srovnání s MS Office, neexistuje. Obecně známé, ale zde **NEOVĚŘENO**: rozdíly v písmech (chybějící fonty se nahradí), SmartArt, některé animace/přechody, složité grafy. Doporučení: otestovat na vlastních vzorcích a do Docker image přidat metricky kompatibilní fonty (Liberation/Carlito/Caladea), jejich licence ověřit.

### A2. Gotenberg (HTTP API nad LibreOffice a Chromium)

- Endpoint `POST /forms/libreoffice/convert` (multipart), vstupy `.docx .xlsx .pptx .odt` a další; kontroluje jen příponu, formát pozná LibreOffice. [10]
- Pole: `password` (špatné/chybějící heslo vrací 400; u starých `.doc/.xls/.ppt` jen obecnou chybu), `exportNotes`, `exportNotesInMargin`, `exportNotesPages` (poznámky prezentace), `exportOnlyNotesPages`, `nativeQuality` (JPEG 1 až 100, výchozí 90), `nativePdfa`, `nativePdfua`, `nativeTaggedPdf`, `singlePage`, `showTrackedChanges`. [10] (Dvě stránky dokumentace popisují `exportNotes` odlišně; ověřit proti verzi. [10][11])
- Konfigurace: `--libreoffice-restart-after` (výchozí 10 převodů), `--libreoffice-start-timeout` (20 s), `--api-timeout` (30 s), `--libreoffice-auto-start` (false), `--libreoffice-max-queue-size` (0 = bez limitu). Dokumentace nepopisuje, zda LibreOffice běží jako jeden proces se sériovým zpracováním; existence fronty to nasvědčuje, **NEOVĚŘENO**. [12]
- Licence MIT, image `gotenberg/gotenberg:9` (Docker Hub), balí Chromium i LibreOffice. [13] Licence balených komponent (LibreOffice, Chromium) se řídí jejich vlastními licencemi.
- Výhoda: hotové řešení pro Docker/Linux a sjednocené chyby (400 u hesla). Nevýhoda: vyžaduje běžící Docker a HTTP službu, což je těžké pro CLI „soubor na vstupu, složka na výstupu“ na Windows bez Dockeru. Výchozí `--api-timeout` 30 s je třeba pro velké soubory zvýšit. [12]

### A3. unoserver / unoconv

- unoserver drží LibreOffice jako listener, aby se pro každý soubor nespouštěl nový proces; README uvádí 2 až 4× více dokumentů za stejnou dobu. Klient `unoconvert`. Licence MIT. [14]
- Souběh: více instancí, každá s vlastním `--port` a `--uno-port`; vyvažování zátěže musí zajistit klient. [14]
- Timeouty: `--conversion-timeout` zastaví LibreOffice i server při dlouhém převodu; `--stop-after` ukončí server po N požadavcích. LibreOffice se po pádu nerestartuje, server se má zastavit a restartovat správcem procesů. [14]
- Windows a macOS: README výslovně uvádí, že podpora je „zatím netestovaná“. Porty nemají žádné zabezpečení, nesmí být dostupné zvenčí. Nutnost instalace do Pythonu, který LibreOffice používá. [14]
- Hodnocení: pro docshift (jednorázový podproces, Windows i Linux) je to zbytečná komplikace; dává smysl jen u dávkového/serverového režimu na Linuxu. unoconv (předchůdce) nebyl zkoumán.

### A4. MS Office přes COM (docx2pdf apod.)

- `docx2pdf`: licence MIT, jen Windows/macOS, vyžaduje nainstalovaný Microsoft Word, na Windows přes `win32com`; v README se uvádí jen DOCX, o PPTX nic. Linux se nezmiňuje. [15]
- Microsoft výslovně nedoporučuje a nepodporuje automatizaci Office z bezobslužných/serverových komponent (může viset, zobrazit dialog) a licence Office neumožňuje obsluhovat klienty bez vlastní licence. [16]
- Závěr: nehodí se jako výchozí engine knihovny/CLI s cílem Linux i Windows. Jako volitelný „vysokověrný“ backend jen na licencovaném Windows stroji s interaktivním uživatelem.

### A5. Aspose (a obdobné komerční knihovny)

- Aspose.Slides: komerční licence; evaluační verze přidává na každý slide vodoznak a zkracuje čtený text; licence se nastavuje souborem `.lic` (`set_license`), 30denní dočasná licence je k dispozici. [17] Cena a podmínky redistribuce: **NEOVĚŘENO**.
- PyMuPDF Office (placený doplněk PyMuPDF 2.x): čte DOC, DOCX, XLS, XLSX, PPT, PPTX a umí PDF i Markdown, ale vyžaduje licenční klíč (bez něj jen první 3 strany a časově omezená evaluace); základní PyMuPDF je AGPL-3.0 nebo komerční. [18] Pro docshift tedy nevhodné bez komerční licence.

### A6. Shrnutí k otázce A

Jediný volně licencovaný engine použitelný na Windows i Linuxu, bez licence třetí strany a bez serveru, je LibreOffice (MPL 2.0) spouštěný jako podproces. Gotenberg a unoserver jsou jen obaly nad ním, takže věrnost je shodná a liší se provozem. Komerční a COM varianty mají licenční a provozní omezení.

---

## B) Převod do Markdownu

### B1. Microsoft MarkItDown

- **Licence a runtime.** MIT (Copyright Microsoft). [19] Python 3.10 až 3.14. [20] Jádro závisí na beautifulsoup4, requests, markdownify, magika, charset-normalizer, defusedxml, certifi; extra `pptx` = python-pptx, `docx` = mammoth + lxml, `xlsx` = pandas + openpyxl, `pdf` = pdfminer.six + pdfplumber. [21] Licence jednotlivých závislostí stránka neuvádí; python-pptx je MIT [22], mammoth BSD-2-Clause [23]; ostatní **NEOVĚŘENO**. Žádné modely ke stažení pro Office formáty (OCR a popisy obrázků jsou volitelně přes plugin `markitdown-ocr`, LLM klienta nebo Azure; vše vypnuto ve výchozím stavu). [20]
- **Formáty.** README uvádí DOCX, PPTX, XLSX (starší XLS přes extra `[xls]`). ODT/ODP README neuvádí a ve výpisu složky konvertorů žádný odpovídající konvertor není (výpis může být neúplný). [20][24] Pro ODT/ODP tedy nutná předkonverze LibreOffice na DOCX/PPTX.
- **CLI.** `markitdown vstup -o výstup.md`; volby `--use-plugins`, `--list-plugins`, volby Azure Document Intelligence/Content Understanding. CLI volba pro ukládání obrázků do souborů neexistuje. [20]
- **PPTX.** Číslo slidu jako HTML komentář `<!-- Slide number: N -->`; titulek jako `# `; tvary řazeny dle pozice; alt text obrázku z atributu `descr` (jinak název tvaru); tabulky přes HTML na Markdown; poznámky mluvčího pod `### Notes:`; grafy jako tabulky; skupiny rekurzivně. [25] Obrázky: bez `keep_data_uris` se vypíše odkaz na soubor odvozený z názvu tvaru s příponou `.jpg` (např. `Picture3.jpg`), **soubor se nezapisuje** a přípona je bez ohledu na skutečný formát; s `keep_data_uris` je obrázek vložen jako base64 data URI. [25] Zdrojový kód obsahuje i háček `_image_to_html` pro vlastní zpracování obrázků (podtřída) [25]; zda je v poslední vydané verzi na PyPI, **NEOVĚŘENO** (čteno z větve main).
- **DOCX.** Cesta mammoth → HTML → HtmlConverter; styly (nadpisy) a tabulky se zachovávají; konvertor má obdobný háček `_image_to_html`, výchozí je nativní reprezentace. [26] mammoth vkládá obrázky jako base64 data URI. [23] MarkItDown při převodu HTML na Markdown data URI ve výchozím stavu zkrátí na `data:image/png;base64...` (jen typ média) a celé je zachová jen s `keep_data_uris`. [27] Poznámky pod čarou a komentáře: **NEOVĚŘENO**.
- **XLSX.** pandas + openpyxl, listy jako tabulky (podrobnosti konvertoru nečteny, **NEOVĚŘENO**). [21]
- **Číslování obrázků.** Nativně nelze; docshift by musel obrázky vytáhnout sám (z OOXML balíčku nebo přes `keep_data_uris` a post-processing data URI → soubory `001.png` v pořadí výskytu v Markdownu).

### B2. Docling (IBM / LF AI & Data)

- **Licence a runtime.** Kód MIT, jednotlivé modely mají vlastní licence v původních balících. [28] Python ≥ 3.10. [28][29] Modely Docling závisejí na PyTorch; CPU varianta `pip install docling --extra-index-url https://download.pytorch.org/whl/cpu`. [30] Váhy modelů jsou potřeba jen pro PDF pipeline; DOCX, PPTX atd. takový požadavek nemají. [31] Velikost instalace s torch: **NEOVĚŘENO** (výrazně větší než MarkItDown, torch je velký). Licence konkrétních modelů (layout, TableFormer, Granite-Docling) **NEOVĚŘENO** (stránka modelu na HF se nepodařilo načíst); pro Office vstupy se nepoužívají [31].
- **Formáty.** DOCX, PPTX, XLSX; ODT, ODS, ODP (CLI `--from odf`); staré DOC/XLS/PPT vyžadují LibreOffice. [32][33]
- **CLI.** `docling <zdroj>` (výchozí výstup Markdown do `--output`, výchozí `.`); `--image-export-mode` s hodnotami `placeholder`, `embedded` (výchozí), `referenced` (PNG soubory odkazované z dokumentu); `--document-timeout`, `--abort-on-error`, `--ocr/--no-ocr`, `--device`, `--artifacts-path`. Volba pro názvy souborů obrázků v referenčním CLI není. [33]
- **Názvy obrázků.** Dokumentace názvy nepopisuje; vzor (v docling-core) se z přečtené části kódu nepodařilo potvrdit. **NEOVĚŘENO**. Pozn.: výstup bývá obsahově/hash-orientovaný, takže docshift stejně musí přejmenovat podle pořadí výskytu.
- **Obrázky z PPTX/DOCX.** WMF obrázky jen na Windows, jinak se ignorují. [31] EMF/WMF v PPTX potřebují k rasterizaci LibreOffice; bez něj zůstane pozice, ale bez dat obrázku. [34] Diskuze s botem (neautoritativní) uvádí, že obrázky se extrahují přes `doc.pictures` a `get_image`. [35]
- **PPTX.** Každý slide = jedna stránka, provenance `page_no = slide_ind + 1`. [34] Poznámky mluvčího jdou do vrstvy `ContentLayer.NOTES` [36]; zda je výchozí export do Markdownu zahrne, z primárního zdroje **NEOVĚŘENO** (sekundární shrnutí z portu docling.rs tvrdí, že ne). Označení čísla slidu v samotném Markdownu: **NEOVĚŘENO**. Alt text obrázků: **NEOVĚŘENO**. Tabulky: DocTags/tabulkový model, Markdown tabulky, konkrétní chování pro PPTX/DOCX **NEOVĚŘENO**.
- **ODP poznámky.** Byly původně ignorovány; oprava v PR #4869 (stav sloučení nepotvrzen). [37]
- Offline: Docling nepoužívá vzdálené služby, stačí ukázat runtime na umístění modelů. [31]

### B3. pandoc

- **Licence.** GPL v2 (soubor COPYING.md). [38] Spuštění jako samostatného programu je běžně v pořádku, ale distribuce binárky s aplikací zakládá GPL povinnosti (nutná právní kontrola, **NEOVĚŘENO** zde).
- **Vstupy.** Manuál uvádí `docx`, `odt`, `xlsx` i `pptx` jako vstupní formáty. [39] Čtečka PPTX je mladá: changelog verze 3.14 (2026-11-10) uvádí čtení poznámek mluvčího, grafů jako tabulek, tvarů ve skupinách a rovnic; zda čtečka vznikla ještě před 3.14, **NEOVĚŘENO**. [40][41] Starší verze pandoc (před zavedením čtečky) PPTX číst nemusí. Minimální verze pro docshift: ověřit `pandoc --list-input-formats`.
- **Obrázky.** `--extract-media=DIR`: pokud jsou původní cesty relativní bez `..`, použijí se; jinak se názvy tvoří ze SHA1 obsahu. [39] Pro DOCX/PPTX tedy názvy nejsou číslované v pořadí výskytu; docshift by je musel přejmenovat a přepsat odkazy.
- **Další volby.** `-t gfm`, `--wrap=none`, `--markdown-headings=atx`, `--track-changes=accept|reject|all` (jen čtečka docx). [39]
- **Čísla slidů, alt text, tabulky.** U PPTX **NEOVĚŘENO**. Poznámky: ano od 3.14 (blok `notes`). [40]
- **Runtime.** Jediný binární soubor (Haskell), bez Pythonu a bez modelů. Velikost binárky v použitých zdrojích **NEOVĚŘENO**.

### B4. Cesta přes PDF

- **pymupdf4llm / PyMuPDF.** `Document.to_markdown` má `write_images`, `image_path`, `image_format` (výchozí png), `dpi` (150), `embed_images`, `page_chunks`; vzor jmen u `to_markdown` není v dokumentaci, u `to_json` je `{image_path}/{filename}-pagenumber-image_number.{image_format}`. [42] PyMuPDF je AGPL-3.0 nebo komerční licence Artifex; Office dokumenty umí jen placený doplněk PyMuPDF Office s klíčem. [18] AGPL je u produktového kódu většinou překážka. Jde i o PDF vstup do PDF→MD, takže by se použila jen pro PDF vyrobené z LibreOffice.
- **marker.** README: vstupy PDF, obrázky, PPTX, DOCX, XLSX, HTML, EPUB; mimo PDF je nutný `marker-pdf[full]`; obrázky se extrahují výchozím nastavením (`--disable_image_extraction` vypíná); Python 3.10+, PyTorch. Kód Apache 2.0, váhy modelů pod modifikovanou Open RAIL-M (zdarma pro výzkum, osobní užití a startupy do 5 M USD financování/obratu, jinak placená licence). [43] (Starší zdroje uváděly GPL-3.0 kódu; aktuální README říká Apache 2.0.) Jak marker čte DOCX/PPTX (nepřímo přes PDF?), **NEOVĚŘENO**. Pro docshift je licence modelů rizikovější než u MarkItDown/Docling.

### B5. Srovnávací tabulka

| Kritérium | MarkItDown | Docling | pandoc | PyMuPDF4LLM / marker (přes PDF) |
|---|---|---|---|---|
| DOCX, PPTX, XLSX | ano [20] | ano [32] | ano (PPTX čtečka mladá) [39][40] | přes PDF (PyMuPDF Office je placený) [18], marker ano [43] |
| ODT/ODP | ne (nutná předkonverze) [20][24] | ano [32][33] | ODT ano, ODP **NEOVĚŘENO** [39] | přes PDF z LibreOffice |
| Obrázky do souborů | ne (jen data URI nebo falešné `.jpg` jméno) [25][27] | ano, `--image-export-mode referenced` [33] | ano, `--extract-media` [39] | ano [42][43] |
| Číslování 001.png v pořadí | ne | ne (**NEOVĚŘENO** jména) | ne (původní jména nebo SHA1) [39] | ne |
| Poznámky mluvčího | ano, `### Notes:` [25] | v notes vrstvě, export **NEOVĚŘENO** [36] | ano od 3.14 [40] | n/a |
| Číslo slidu v MD | ano, komentář [25] | **NEOVĚŘENO** | **NEOVĚŘENO** | n/a |
| Alt text | ano (`descr`) [25] | **NEOVĚŘENO** | **NEOVĚŘENO** | n/a |
| Runtime | Python, malé | Python + torch pro PDF, pro Office bez modelů [30][31] | binárka | Python, modely u marker |
| Licence | MIT [19] | MIT, modely vlastní [28] | GPL v2 [38] | AGPL / komerční [18]; Apache + RAIL-M [43] |

---

## C) Návrh kombinací (bez rozhodnutí za uživatele)

### Kombinace 1: LibreOffice → PDF + MarkItDown → MD (doporučená k prototypu)

- Výhody: obě součásti MIT/MPL, žádné modely, malá instalace MarkItDown, PPTX výstup už obsahuje čísla slidů, alt text, poznámky a tabulky [25], tedy přesně to, co docshift potřebuje v MD. Jeden Python balíček se spouští jako podproces.
- Nevýhody: obrázky MarkItDown nezapisuje do souborů (PPTX jen odkaz na neexistující `.jpg`, DOCX data URI) [25][27]; ODT/ODP vyžadují předkonverzi LibreOffice na DOCX/PPTX; číslování obrázků musí dodat docshift. Obsahuje i mírné riziko u věrnosti DOCX (cesta mammoth → HTML).
- Co by docshift dopsal: vlastní vytažení obrázků (z `ppt/media` v pořadí výskytu v XML slidu, případně přes `keep_data_uris` + post-processing), přejmenování na `001.png…`, přepis odkazů v MD, manifest (vstup, výstupy, počet slidů/obrázků, varování). Případně PNG konverze EMF/WMF/SVG přes LibreOffice.

### Kombinace 2: Docling pro Markdown (obrázky) + LibreOffice pro PDF

- Výhody: nativní export obrázků do souborů (`referenced`) [33], podpora ODT/ODP [32], poznámky v samostatné vrstvě [36], timeout `--document-timeout` [33]. Pro Office bez modelů [31].
- Nevýhody: těžší instalace (PyTorch jako závislost [30]), neověřené detaily (alt text, čísla slidů v MD, názvy souborů, export poznámek), jména souborů stejně nutno přejmenovat; mladší formáty ODF mají otevřené mezery (např. poznámky ODP [37]).
- Co by docshift dopsal: přejmenování/číslování obrázků, doplnění čísel slidů a poznámek přes API (`DoclingDocument`) místo CLI, manifest.

### Kombinace 3: Vlastní skládání (LibreOffice → PDF; vlastní extrakce OOXML/ODF + python-pptx/python-docx pro MD)

- Výhody: plná kontrola nad číslováním obrázků, alt textem, slidy, poznámkami, formátem pro LLM; minimum závislostí (python-pptx MIT [22]).
- Nevýhody: největší objem vlastního kódu a údržby (tabulky, vnořené skupiny, grafy, SmartArt, ODF). MarkItDown PPTX konvertor je v zásadě tenká vrstva nad python-pptx [25], takže realistická varianta je „MarkItDown jako základ + vlastní obrázkový modul" (= kombinace 1).
- Alternativa pro DOCX/ODT: pandoc (GPL v2, nutné posoudit licenci distribuce [38]) s `--extract-media`.

### Doporučení

Zdůvodněné pořadí k vyzkoušení: 1 (nejmenší závislosti, nejlepší shoda PPTX výstupu s požadavky), pak 2 pokud se ukáže, že ODT/ODP a nativní obrázky stojí za cenu PyTorche. Rozhodnutí je na uživateli po malém experimentu na reálných vzorcích. Společné pro všechny: PDF vždy z LibreOffice, číslování obrázků a manifest dopisuje docshift.

### Co musí dopsat docshift v každém případě

- Číslování obrázků `001.png…` v pořadí výskytu v MD a přepis odkazů; žádný zkoumaný engine to nedělá nativně.
- Správa procesů LibreOffice: unikátní profil na souběh, timeout, zabíjení celého stromu procesů, úklid profilu (viz CLAUDE.md projektu o stálé složce místo mazání/zakládání profilů; profil LibreOffice je také velký adresář).
- Detekce hesla/poškození a jednotné chybové kódy; manifest (JSON) s výsledky a varováními.
- Rasterizace EMF/WMF/SVG na PNG (u Docling vyžaduje LibreOffice [34], u MarkItDown nic neřeší).

---

## Otevřené otázky

1. Přesná věrnost LibreOffice PDF u reálných PPTX/DOCX uživatele (fonty, SmartArt, grafy): bez primárního zdroje, nutný experiment.
2. Chování `soffice` při heslem chráněném a poškozeném souboru (kód návratu, výstup): bez primárního zdroje, nutný experiment.
3. Platnost chyb se souběžným/sdíleným profilem (bugy 37531, 113259 [3][4]) ve verzi 26.8.
4. Docling: názvy referencovaných obrázků, alt text, čísla slidů a export poznámek do MD, velikost instalace s torchem, licence modelů.
5. MarkItDown: zda háček `_image_to_html` je v released verzi; chování XLSX a poznámek pod čarou v DOCX; licence tranzitivních závislostí.
6. pandoc: verze, od které existuje čtečka PPTX; chování u čísel slidů, alt textu a tabulek; velikost binárky; důsledky GPL v2 při distribuci.
7. Existuje oficiální Docker image LibreOffice a jaké balíky/fonty jsou minimální pro headless převod Office (Linux)?
8. Jak se má chovat docshift při chybějícím alt textu (název tvaru z MarkItDown není smysluplný pro LLM)?
9. Má se do MD přidávat i obsah poznámek mluvčího, a v jaké podobě (hlavní tok vs. samostatný blok)?

## Zdroje

[1] LibreOffice Help, Start Parameters: https://help.libreoffice.org/latest/en-US/text/shared/guide/start_parameters.html
[2] Ask LibreOffice, `-env:UserInstallation` jako URL (fórum, použito jen jako doplněk k [1]): https://ask.libreoffice.org/t/how-can-i-disable-popups-when-running-convert-command/65769
[3] Bug 37531, The Document Foundation: https://bugs.documentfoundation.org/show_bug.cgi?id=37531
[4] Bug 113259, The Document Foundation: https://bugs.documentfoundation.org/show_bug.cgi?id=113259
[5] LibreOffice, licence: https://www.libreoffice.org/about-us/licenses/
[6] LibreOffice, stažení: https://www.libreoffice.org/download/download-libreoffice/
[10] Gotenberg, LibreOffice convert: https://gotenberg.dev/docs/libreoffice/convert
[11] Gotenberg, Convert to PDF: https://gotenberg.dev/docs/convert-with-libreoffice/convert-to-pdf
[12] Gotenberg, konfigurace: https://gotenberg.dev/docs/configuration
[13] Gotenberg, repozitář (MIT, Docker image): https://github.com/gotenberg/gotenberg
[14] unoserver README (MIT): https://github.com/unoconv/unoserver
[15] docx2pdf README (MIT): https://github.com/AlJohri/docx2pdf
[16] Microsoft Support, Considerations for server-side Automation of Office: https://support.microsoft.com/en-us/topic/considerations-for-server-side-automation-of-office-48bcfe93-8a89-47f1-0bce-017433ad79e2
[17] Aspose.Slides for Python, licencování: https://docs.aspose.com/slides/python-net/licensing/
[18] PyMuPDF na PyPI (AGPL/komerční, PyMuPDF Office): https://pypi.org/project/pymupdf/
[19] MarkItDown LICENSE (MIT): https://raw.githubusercontent.com/microsoft/markitdown/main/LICENSE
[20] MarkItDown README: https://github.com/microsoft/markitdown
[21] MarkItDown pyproject.toml: https://github.com/microsoft/markitdown/blob/main/packages/markitdown/pyproject.toml
[22] python-pptx na PyPI (MIT): https://pypi.org/project/python-pptx/
[23] mammoth na PyPI (BSD-2-Clause, data URI): https://pypi.org/project/mammoth/
[24] MarkItDown, složka konvertorů: https://github.com/microsoft/markitdown/tree/main/packages/markitdown/src/markitdown/converters
[25] MarkItDown, `_pptx_converter.py`: https://github.com/microsoft/markitdown/blob/main/packages/markitdown/src/markitdown/converters/_pptx_converter.py
[26] MarkItDown, `_docx_converter.py`: https://github.com/microsoft/markitdown/blob/main/packages/markitdown/src/markitdown/converters/_docx_converter.py
[27] MarkItDown, `_markdownify.py`: https://github.com/microsoft/markitdown/blob/main/packages/markitdown/src/markitdown/converters/_markdownify.py
[28] Docling README (MIT, formáty, Python): https://github.com/docling-project/docling
[29] Docling na PyPI: https://pypi.org/project/docling/
[30] Docling, instalace (PyTorch, CPU): https://docling-project.github.io/docling/getting_started/installation/
[31] Docling FAQ (modely jen pro PDF, WMF, offline): https://docling-project.github.io/docling/faq/
[32] Docling, podporované formáty: https://docling-project.github.io/docling/usage/supported_formats/
[33] Docling, CLI reference: https://docling-project.github.io/docling/reference/cli/
[34] Docling, `mspowerpoint_backend.py`: https://github.com/docling-project/docling/blob/main/docling/backend/mspowerpoint_backend.py
[35] Docling, issue 2379 (obrázky z ne-PDF; odpovědi bota, neautoritativní): https://github.com/docling-project/docling/issues/2379
[36] Docling, issue 4737 (PPTX backend dává poznámky do `ContentLayer.NOTES`): https://github.com/docling-project/docling/issues/4737
[37] Docling, PR 4869 (poznámky ODP): https://github.com/docling-project/docling/pull/4869
[38] pandoc COPYING.md (GPL v2): https://raw.githubusercontent.com/jgm/pandoc/main/COPYING.md
[39] pandoc manuál: https://pandoc.org/MANUAL.html
[40] pandoc changelog: https://raw.githubusercontent.com/jgm/pandoc/main/changelog.md
[41] pandoc, vydání 3.14: https://github.com/jgm/pandoc/releases/tag/3.14
[42] PyMuPDF, Document.to_markdown: https://pymupdf.readthedocs.io/en/latest/document.html
[43] marker README (Apache 2.0 kód, váhy Open RAIL-M): https://github.com/datalab-to/marker
