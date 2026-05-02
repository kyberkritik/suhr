# Cómo añadir libros al catálogo

El sitio (`index.html`, `catalogo.html`, `rockola.html`) se **genera** a partir de
`Bibliotheca Cardiel.rdf` mediante `build_library_page.py`. Estos dos archivos
viven solo en local (están en `.gitignore`); en GitHub se publica únicamente
el resultado renderizado.

## Resumen del flujo (cada nuevo libro)

1. Sacarle dos fotos al libro:
   - **Portada completa** (lo más recta posible, fondo oscuro y uniforme).
   - **Página de imprenta / impresum** con ISBN, año, editorial, número de serie.
2. Procesar la portada con `local/tools/process_cover.py` (recorta, escala a
   800 px de ancho y elimina metadatos EXIF).
3. Guardar el JPG resultante en `assets/local-covers/<ISBN>.jpg`.
4. Añadir un `<bib:Book>` al final de `Bibliotheca Cardiel.rdf` y registrar el
   ISBN en `<z:Collection rdf:about="#collection_170">`.
5. Ejecutar `build_library_page.py`.
6. Verificar en navegador y commitear los HTML + el cover.

## 1. Foto y procesado de la portada

Las fotos pueden venir directo del teléfono. El script las limpia:

```bash
/tmp/imgenv/bin/python local/tools/process_cover.py \
  local/SUHR/IMG_1234.png \
  assets/local-covers/<ISBN>.jpg \
  --crop 0.05,0.04,0.85,0.95
```

`--crop` acepta coordenadas en píxeles **o** fracciones (0–1) del ancho/alto
original: `LEFT,TOP,RIGHT,BOTTOM`. Conviene revisar el JPG resultante y
afinar las fracciones hasta que solo se vea la portada (sin mesa, sin dedos,
sin pies).

`--max-width` (por defecto 800) e `--quality` (85) son opcionales.

> El entorno virtual con Pillow se creó una sola vez:
> `python3 -m venv /tmp/imgenv && /tmp/imgenv/bin/pip install Pillow`.
> Si /tmp se borra, hay que rehacerlo.

## 2. Entrada en el RDF

Plantilla mínima (taschenbuch con autor único). Pegarla **antes** de
`<z:Collection rdf:about="#collection_170">`:

```xml
<bib:Book rdf:about="urn:isbn:978-3-518-XXXXX-X">
    <z:itemType>book</z:itemType>
    <dcterms:isPartOf>
        <bib:Series>
            <dc:title>Suhrkamp Taschenbuch Wissenschaft</dc:title>
            <dc:identifier>NÚMERO_DE_SERIE</dc:identifier>
        </bib:Series>
    </dcterms:isPartOf>
    <dc:publisher>
        <foaf:Organization>
            <vcard:adr>
                <vcard:Address>
                   <vcard:locality>Frankfurt am Main</vcard:locality>
                </vcard:Address>
            </vcard:adr>
            <foaf:name>Suhrkamp</foaf:name>
        </foaf:Organization>
    </dc:publisher>
    <bib:authors>
        <rdf:Seq>
            <rdf:li>
                <foaf:Person>
                    <foaf:surname>APELLIDO</foaf:surname>
                    <foaf:givenName>NOMBRE</foaf:givenName>
                </foaf:Person>
            </rdf:li>
        </rdf:Seq>
    </bib:authors>
    <dc:title>TÍTULO COMPLETO</dc:title>
    <dc:date>AÑO_DE_ESTA_EDICIÓN</dc:date>
    <z:language>ger</z:language>
    <z:libraryCatalog>Suhrkamp</z:libraryCatalog>
    <dc:identifier>ISBN 978-3-518-XXXXX-X</dc:identifier>
    <prism:edition>Erste Auflage</prism:edition>
    <z:numPages>NÚMERO_DE_PÁGINAS</z:numPages>
</bib:Book>
```

Variantes:

- **Antología con editor (Hrsg.)**: cambiar `<bib:authors>` por `<bib:editors>`
  con la misma estructura interna.
- **Traducción**: añadir bloque `<z:translators>` (mismo formato que
  `<bib:authors>`) tras los autores.
- **Series posibles** (`<dc:title>` dentro de `<bib:Series>`):
  *Suhrkamp Taschenbuch Wissenschaft*, *Suhrkamp Taschenbuch*,
  *Edition Suhrkamp*, *Bibliothek Suhrkamp*, *Suhrkamp-BasisBibliothek*.

Después de añadir el `<bib:Book>`, sumar la línea correspondiente al colección
final del RDF:

```xml
<dcterms:hasPart rdf:resource="urn:isbn:978-3-518-XXXXX-X"/>
```

## 3. Año y editor original (opcional)

Si la edición Suhrkamp es traducción o reedición, se puede añadir el dato a
`original-years.json` con la clave del ISBN:

```json
"978-3-518-XXXXX-X": {
  "year": 1965,
  "original_publisher": "Editions du Seuil",
  "original_language": "fr"
}
```

## 4. Reconstruir el sitio

```bash
/tmp/imgenv/bin/python build_library_page.py
```

Esto regenera **los tres** HTML. Si solo se actualiza una página manualmente
el siguiente build pisa el cambio (eso fue justo el bug del PR de Codex
mergeado en mayo 2026: tocó `catalogo.html` y `rockola.html` sin actualizar
el RDF, así que `index.html` quedó con menos libros y al rehacer el build
todos los cambios manuales habrían desaparecido).

## 5. Verificar y commitear

Abrir `index.html` en el navegador y revisar:

- Aparece la nueva tarjeta con su portada.
- El recuento total cuadra (`X portadas`).
- En `rockola.html` la portada se muestra a buen contraste y nitidez.

Archivos a commitear:

- `index.html`, `catalogo.html`, `rockola.html`
- `assets/local-covers/<ISBN>.jpg`

Lo demás (RDF, scripts, JSON de overrides, fotos originales en `local/SUHR/`)
queda solo en local.

## Estructura del repo

```
.
├── index.html              # Mosaico + buscador (entrada principal)
├── catalogo.html           # Vista tabular completa
├── rockola.html            # Vista carrusel
├── assets/
│   ├── covers/             # Portadas oficiales (descargadas del editor)
│   └── local-covers/       # Portadas tomadas con teléfono
├── ADDING_BOOKS.md         # Esta guía
├── .gitignore
│
├── Bibliotheca Cardiel.rdf # FUENTE — solo local
├── build_library_page.py   # Generador — solo local
├── original-years.json     # Datos de edición original — solo local
├── cover-overrides.json    # Overrides de portadas — solo local
└── local/                  # Fotos en bruto, originales, screenshots, herramientas
    ├── SUHR/               # Fotos del teléfono (cover + impresum)
    ├── original-covers/    # Versiones sin recortar
    ├── screenshots-archive/
    └── tools/
        └── process_cover.py
```
