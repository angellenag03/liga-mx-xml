# Resultados de Liga MX en XML + DTD

Práctica guiada: diseño y validación de resultados de Liga MX con XML y DTD.

## 1. Propósito

Representar en XML los resultados y las estadísticas de los partidos de Liga MX, definir
con un DTD las reglas que debe cumplir cualquier documento de ese formato y comprobarlo
con un caso real: los partidos del **domingo 27 de septiembre de 2026** (Jornada 10 del
Apertura 2026).

```text
liga-mx-xml/
├── README.md
├── xml/
│   ├── resultados.xml            # 3 partidos del domingo 27/09/2026 (válido)
│   └── resultados-invalido.xml   # pruebas negativas (bien formado, NO válido)
└── dtd/
    └── resultados.dtd            # reglas de validación
```

## 2. Análisis de la información (Actividad 1)

| Pregunta | Respuesta |
|---|---|
| ¿Elemento raíz? | `liga`: contiene todo lo demás. |
| ¿Una jornada tiene varios partidos? | Sí, uno o más (`partido+`). |
| ¿Cada partido tiene exactamente dos equipos? | Sí, y con roles distintos: uno local y uno visitante. |
| ¿Cómo se distinguen local y visitante? | Por el nombre del elemento (`equipoLocal` / `equipoVisitante`), no por su posición. |
| ¿Marcador como un dato o goles separados? | Separados: `golesLocal` y `golesVisitante`. Además se registra cada gol como evento. |
| ¿Las estadísticas son del partido o del equipo? | Del equipo (ver sección 5). |
| Obligatorios | `id`, `fecha`, `estado`, los dos equipos con `nombre`, `marcador`, `estadio`. |
| Opcionales | `hora`, `ciudad`, `estadisticas`, `gol`, `tarjeta`, `adicional`. |

## 3. Modelo jerárquico

```text
liga (competencia, torneo, temporada)
└── jornada (numero)
    └── partido (id, fecha, hora?, estado)
        ├── equipoLocal
        │   ├── nombre
        │   ├── estadisticas?  → posesion, tiros, tirosAPuerta, faltas, tirosEsquina
        │   ├── gol*           (jugador, minuto, adicional?, tipo)
        │   └── tarjeta*       (tipo, jugador, minuto, adicional?)
        ├── equipoVisitante    (mismo contenido que equipoLocal)
        ├── marcador           (golesLocal, golesVisitante)
        └── estadio            (ciudad?)
```

Convención para los goles: el `gol` se registra en el equipo al que **le cuenta en el
marcador**. En un autogol (`tipo="autogol"`), el `jugador` pertenece al equipo contrario.

## 4. Elementos y atributos (Actividad 2)

| Información | Elemento / Atributo | Justificación |
|---|---|---|
| Jornada | Elemento `jornada` + atributo `numero` | El elemento agrupa los partidos; el número es un dato simple. |
| Fecha | Atributo `fecha` de `partido` (AAAA-MM-DD) | Dato simple. Va en el partido porque una jornada abarca varios días. |
| ID del partido | Atributo `id` (tipo `ID`) | Es un identificador, no contenido. |
| Equipo local | Elemento `equipoLocal` | Tiene estructura interna (nombre, estadísticas, goles, tarjetas). |
| Equipo visitante | Elemento `equipoVisitante` | Igual que el local. |
| Goles | Atributos `golesLocal` / `golesVisitante` en `marcador`; eventos `gol` con atributos | Son valores atómicos sin subestructura. |
| Estadio | Elemento `estadio` + atributo `ciudad` | El nombre es el contenido; la ciudad es un metadato opcional. |
| Estado del partido | Atributo `estado` enumerado | Conjunto cerrado de valores que el DTD puede validar. |
| Posesión | Elemento `posesion` dentro de `estadisticas` | Cada estadística es un dato propio del equipo y es ampliable. |
| Tarjetas | Elemento `tarjeta` repetible | Puede haber 0..n por equipo, cada una con jugador y minuto. |

## 5. Dónde van las estadísticas (Actividad 4)

Se compararon tres diseños:

| Criterio | A. Dentro de cada equipo (elegido) | B. En el partido con sufijos (`posesionLocal`, `posesionVisitante`…) | C. Genérico (`<estadistica tipo="posesion">`) |
|---|---|---|---|
| Comprensible | Alta: cada dato está junto a su equipo | Media: hay que cruzar nombres | Media |
| Reduce duplicación | Alta: un solo bloque `estadisticas` para ambos equipos | Baja: cada estadística se declara dos veces | Alta |
| Facilita el procesamiento | Alta: `equipoLocal/estadisticas/posesion` | Media | Media: hay que filtrar por `tipo` |
| Permite agregar estadísticas | Una línea en el DTD | Dos líneas por estadística | Sin cambiar el DTD, pero se pierde validación de orden y presencia |

**Decisión:** diseño A. Cada estadística queda ligada sin ambigüedad a su equipo, se declara
una sola vez (el contenido de ambos equipos comparte la entidad `%contenidoEquipo;`) y el DTD
puede exigir que estén todas. El costo es que añadir una estadística nueva implica editar el
DTD; se acepta porque se gana validación estricta.

Estadísticas seleccionadas (las del ejercicio): posesión (%), tiros totales, tiros a puerta,
faltas cometidas y tiros de esquina; las tarjetas se registran como eventos.

## 6. Reglas del DTD (Actividades 5 y 6)

| Regla | Expresión DTD |
|---|---|
| Una liga contiene una o más jornadas | `<!ELEMENT liga (jornada+)>` |
| Una jornada contiene uno o más partidos | `<!ELEMENT jornada (partido+)>` |
| Un partido tiene exactamente un local | `<!ELEMENT partido (equipoLocal, ...)>` (sin operador = exactamente uno) |
| Un partido tiene exactamente un visitante | `<!ELEMENT partido (equipoLocal, equipoVisitante, marcador, estadio)>` |
| Una estadística opcional | `estadisticas?` en `(nombre, estadisticas?, gol*, tarjeta*)` |
| Cero o más tarjetas | `tarjeta*` en el mismo contenido |

Atributos principales: `partido@id` es `ID #REQUIRED`; `partido@estado` es una enumeración
`#REQUIRED`; `partido@hora` y `estadio@ciudad` son `#IMPLIED`; `gol@tipo` es una enumeración
con valor por defecto `normal`; `gol@adicional` y `tarjeta@adicional` (minutos de tiempo
añadido) son `#IMPLIED`.

**¿Qué ventaja tiene declarar el identificador como `ID` en lugar de `CDATA`?**
Con `ID` el validador garantiza que **no se repita** en todo el documento (la prueba 5
lo demuestra) y que sea un nombre XML válido (`P001` sí, `001` no). Con `CDATA` cualquier
texto pasa, incluso duplicado. Además, un `ID` puede ser referenciado después con `IDREF`.

## 7. Asociación XML–DTD (Actividad 7)

El XML está en `xml/` y el DTD en `dtd/`, así que la ruta relativa desde el XML es
`../dtd/resultados.dtd`:

```xml
<!DOCTYPE liga SYSTEM "../dtd/resultados.dtd">
```

## 8. Cómo validar

Requiere `xmllint` (parte de libxml2). Instalación: Linux `sudo apt install libxml2-utils`,
macOS ya lo incluye, Windows mediante WSL o MSYS2.

```bash
# 1) ¿Está bien formado?
xmllint --noout xml/resultados.xml

# 2) ¿Es válido respecto al DTD?  (sin salida = válido)
xmllint --noout --valid xml/resultados.xml

# 3) El archivo de pruebas negativas debe FALLAR en el paso 2
xmllint --noout --valid xml/resultados-invalido.xml
```

Validado con xmllint (libxml 2.9.14): `resultados.xml` es bien formado y válido.

Comprobación adicional que el DTD no puede hacer (el marcador coincide con el número de
`gol` de cada equipo y la posesión suma 100): se verificó con un script y los tres
partidos cumplen. Ejemplo con XPath:

```bash
xmllint --xpath "count(//partido[@id='P003']/equipoLocal/gol)" xml/resultados.xml   # 2
```

## 9. Pruebas negativas (Actividad 8)

`xml/resultados-invalido.xml` contiene seis partidos (P101 a P106), cada uno con un único
error. Además, cada error se probó por separado sobre una copia del XML válido. Resultados
reales de `xmllint`:

| Prueba | ¿Bien formado? | ¿Válido? | Error detectado |
|---|---|---|---|
| Falta visitante | Sí | No | `Element partido content does not follow the DTD, expecting (equipoLocal , equipoVisitante , marcador , estadio), got (equipoLocal marcador estadio )` |
| Dos locales | Sí | No | `... got (equipoLocal equipoLocal marcador estadio )` |
| Orden incorrecto | Sí | No | `... got (equipoLocal equipoVisitante estadio marcador )` |
| Falta atributo obligatorio | Sí | No | `Element partido does not carry attribute estado` |
| ID duplicado | Sí | No | `ID P001 already defined` |
| Elemento desconocido | Sí | No | `No declaration for element arbitro` (más el error de contenido del `partido`) |

> **XML bien formado ≠ XML válido.** Las seis pruebas son documentos *bien formados* que
> incumplen las reglas del DTD. Un documento *mal formado* (por ejemplo con etiquetas
> cruzadas, `<liga><jornada></liga></jornada>`) ni siquiera llega a validarse:
> `Opening and ending tag mismatch: jornada line 2 and liga`.

## 10. La jornada solicitada y el cambio de requisitos (Actividad 9)

`xml/resultados.xml` contiene los **tres partidos del domingo 27 de septiembre de 2026**
(Jornada 10, Apertura 2026). El resto de la jornada se jugó en otros días y no se incluye.

| Partido | Hora (CDMX) | Marcador | Estadio |
|---|---|---|---|
| Pumas UNAM vs Atlético de San Luis | 12:00 | 2 - 3 | Estadio Olímpico Universitario |
| León vs FC Juárez | 19:00 | 2 - 1 | Estadio León |
| Necaxa vs América | 21:10 | 2 - 4 | Estadio Victoria |

Información que apareció al capturar los datos, y qué se decidió:

| Información nueva | ¿Requisito legítimo? | Decisión |
|---|---|---|
| Autogol (Nevárez, de Juárez, anotó para León) | Sí: sin él el marcador y los goleadores no cuadran | `gol@tipo` admite `autogol` |
| Tiempo añadido (45+10, 90+4, 90+9) | Sí: el minuto no basta | Atributo opcional `adicional` |
| Penales (7 de los 14 goles del día) | Sí: dato relevante del gol | `gol@tipo` admite `penal` |
| Alineaciones, cambios, árbitro, decisiones de VAR, pases, xG | No: el ejercicio pide marcador, jornada y seis estadísticas | No se modifica el DTD |
| Diferencias entre fuentes (ver sección 11) | No es un cambio de estructura | Se resuelve eligiendo el dato, no el esquema |

Límites del DTD: no tiene tipos numéricos ni de fecha (`NMTOKEN` solo evita espacios) y
no puede comprobar que el marcador coincida con los goles. Un esquema XSD resolvería lo primero.

## 11. Fuentes y calidad de los datos

El ejercicio indica ESPN México como fuente, pero sus páginas de resultados y de partido se
cargan con JavaScript y no pude leerlas de forma automática. Los datos se tomaron de otras
fuentes públicas y se contrastaron entre sí:

- **Necaxa vs América:** página oficial del partido en clubamerica.com.mx (datos Liga BBVA MX).
- **Pumas vs San Luis:** Proceso, Goles y Cifras (Radiografía), Diario de México, Fox Sports México.
- **León vs Juárez:** Yahoo Sports, Tribuna, SofaScore, Fox Sports México.
- **Cobertura general:** TUDN, Milenio, Liga BBVA MX.

Discrepancias encontradas y criterio aplicado:

| Dato | Lo que dicen las fuentes | Se usó |
|---|---|---|
| Penal de Juninho (Pumas) | 87', 89' o 90' | 90' |
| Gol de David Rodríguez (San Luis) | 54' o 55' | 54' |
| Goles de León-Juárez | 76', 90+4', 90+9' (varias fuentes); otras sumaron el tiempo añadido (78', 93', 99') | 76', 90+4', 90+9' |
| Hora de Necaxa-América | 21:00 o 21:10 | 21:10 |
| Fecha de Necaxa-América | 28/09 en fuentes que usan UTC | 27/09 (hora local) |
| Tarjetas rojas de Pumas-San Luis | una crónica habla de "dos rojas"; tres fuentes estadísticas registran 0 | 0 |
| Estadísticas de tiros y tiros a puerta | varían según el proveedor | las del proveedor citado arriba para cada partido |

**Antes de entregar conviene cotejar marcador y estadísticas con la página de ESPN México**,
porque ESPN puede mostrar cifras ligeramente distintas. Si cambia algún valor, basta editar
el elemento correspondiente en `xml/resultados.xml` y volver a validar.
