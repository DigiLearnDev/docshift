---
status: accepted
---

# Engine: LibreOffice pro PDF, MarkItDown pro Markdown

PDF vzniká přes LibreOffice headless (jediný volný engine pro Windows i Linux, MPL 2.0). Markdown vzniká přes MarkItDown (MIT, lehký, bez modelů, u PPTX dává čísla slidů, alt text, tabulky a poznámky). MarkItDown neukládá Obrázky do souborů, takže extrakci a číslování Obrázků včetně přepisu odkazů v Markdownu dělá docshift sám. Formáty, které MarkItDown nečte (ODT, ODP), se nejdřív převedou LibreOffice.

Uživatel výběr přenechal na nás s cílem, aby výstup co nejlépe zpracoval LLM.

## Considered Options

- **LibreOffice + Docling**: ukládá Obrázky do souborů a čte i ODF, ale závisí na PyTorch a u poznámek, alt textu a pojmenování Obrázků to výzkum neověřil.
- **Vlastní čtení OOXML/ODF**: plná kontrola, nejvíc kódu a údržby.
- **pandoc**: PPTX čte jen novou čtečkou a licence je GPL v2.

Detaily a zdroje: [research](../research/office-konverze-pdf-a-markdown.md).

## Consequences

- Runtime zahrnuje LibreOffice i Python s MarkItDown.
- Pokud MarkItDown přestane stačit u věrnosti Markdownu, výměna za Docling je lokální změna za rozhraním CLI.
