# docshift

Převodník, který z kancelářského dokumentu vyrobí PDF a Markdown s odděleně uloženými obrázky, aby ho mohl přečíst LLM.

## Language

**Zdrojový dokument**:
Soubor kancelářského formátu (např. DOCX, PPTX, XLSX, ODT, ODP), který se převádí.
_Avoid_: Vstup, originál, dokument (bez přívlastku)

**Převod**:
Jedno zpracování jednoho Zdrojového dokumentu do Výstupní složky.
_Avoid_: Konverze, job, úloha

**Výstupní složka**:
Složka, do které Převod uloží všechny své výsledky.
_Avoid_: Cílová složka, output

**PDF**:
Výstup Převodu, který věrně zachovává vzhled Zdrojového dokumentu po stranách nebo slidech.
_Avoid_: Tisková verze

**Markdown**:
Výstup Převodu, který zachovává text, strukturu a tabulky Zdrojového dokumentu a na Obrázky odkazuje jejich čísly.
_Avoid_: MD text, textová verze

**Obrázek**:
Rastrový nebo vektorový obraz vložený do Zdrojového dokumentu, uložený jako samostatný soubor.
_Avoid_: Grafika, asset, příloha

**Číslo obrázku**:
Pořadové číslo Obrázku v pořadí jeho prvního výskytu ve Zdrojovém dokumentu, které slouží jako stabilní reference.
_Avoid_: ID obrázku, index

**Popis obrázku**:
Alternativní text a umístění (např. číslo slidu) Obrázku, které zůstávají v Markdownu, ne v souboru Obrázku.
_Avoid_: Titulek, caption

**Manifest**:
Strojově čitelný záznam výsledku Převodu: které výstupy vznikly, seznam Obrázků, počet stran nebo slidů a Varování.
_Avoid_: Report, log, metadata

**Varování**:
Zpráva v Manifestu o tom, že Převod uspěl, ale něco ze Zdrojového dokumentu se nepodařilo věrně přenést.
_Avoid_: Upozornění, warning

**Chyba převodu**:
Selhání Převodu, po kterém nevznikl požadovaný výstup; Manifest uvádí její druh (nepodporovaný, poškozený nebo heslem chráněný Zdrojový dokument, překročený čas).
_Avoid_: Výjimka, fail
