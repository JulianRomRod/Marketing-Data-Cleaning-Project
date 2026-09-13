
# Marketing Campaigns Data Cleaning SQL

## Acerca del proyecto
Este proyecto realiza la limpieza y estandarización de una base de datos de campañas de marketing gestionadas por una agencia ficticia para distintas empresas cliente repartidas por las comunidades autónomas de España. El objetivo es transformar un conjunto de datos crudo, inconsistente y con errores intencionados en una tabla fiable, estandarizada y lista para ser utilizada en análisis posteriores.

**Resumen del proyecto:** Limpieza de una base de datos SQL de campañas de marketing con más de 1.200 filas: estandarización de texto, eliminación de duplicados y tratamiento de valores nulos según reglas de negocio definidas.

## Objetivos del Proyecto
El objetivo principal es aplicar un proceso de Data Cleaning completo y documentado sobre una base de datos de campañas de marketing, dejando la información en un estado íntegro, coherente y consistente, sin alterar el significado original de los datos ni introducir sesgos en el proceso de estandarización.

De forma más específica, el proyecto busca:
- Separar los datos crudos de los datos de trabajo, preservando siempre la fuente original.
- Definir y aplicar criterios claros de estandarización de texto (mayúsculas, abreviaturas, símbolos).
- Detectar y eliminar registros duplicados, incluyendo duplicados "disfrazados" bajo distintos formatos.
- Tratar los valores nulos siguiendo reglas de negocio realistas dictadas por la empresa ficticia propietaria de los datos.

## Acerca de los Datos

La fuente de este proyecto es una tabla generada expresamente por Claude (Anthropic), a petición explícita, para simular de forma realista — y de manera intencionadamente exagerada — el tipo de desorden que un analista de datos puede encontrarse en un entorno de trabajo real: formatos de texto inconsistentes, abreviaturas mezcladas con nombres completos, símbolos de moneda dentro de campos numéricos, fechas en distintos formatos, valores nulos disfrazados de texto y duplicados exactos y aproximados.

La tabla cruda (`campaña_marketing_crudo`) contiene **14 columnas y 1.210 registros**, cada uno correspondiente a una campaña de marketing gestionada por la agencia para una empresa cliente:

| Columna              | Descripción                                                          | Tipo de dato   | Errores presentes |
|----------------------|-----------------------------------------------------------------------|----------------|--------------------|
| `id_campana`         | Identificador único de la campaña                                     | VARCHAR(10)    | Ninguno detectado |
| `fecha`              | Fecha de la campaña                                                    | VARCHAR / DATE | Múltiples formatos mezclados (`DD/MM/YYYY`, `YYYY-MM-DD`, `DD-MM-YYYY`, `DD.MM.YYYY`) |
| `empresa`            | Empresa cliente para la que se ejecuta la campaña                      | VARCHAR(100)   | Mayúsculas/minúsculas mezcladas, sufijos legales inconsistentes (S.L., SL, S.A., S.L.U.), espacios extra, valores nulos |
| `comunidad_autonoma` | Comunidad autónoma donde se ejecuta la campaña                         | VARCHAR(50)    | Abreviaturas, mayúsculas/minúsculas mezcladas, nombres alternativos (p. ej. "Murcia", "MU", "Región de Murcia"), valores nulos y nulos disfrazados (`N/A`, `desconocido`, `sin datos`) |
| `sector`             | Sector de actividad de la empresa cliente                              | VARCHAR(50)    | Valores nulos y nulos disfrazados |
| `canal_marketing`    | Canal utilizado en la campaña                                          | VARCHAR(50)    | Abreviaturas y sinónimos mezclados (p. ej. "RRSS", "Redes Sociales", "Social Media") |
| `tipo_campana`       | Tipo u objetivo de la campaña                                          | VARCHAR(50)    | Mayúsculas/minúsculas mezcladas |
| `gasto_campana`      | Gasto invertido en la campaña                                          | VARCHAR / DECIMAL | Formato numérico inconsistente (uso de coma y punto decimal, símbolo "€" incluido como texto), valores nulos |
| `ingresos`           | Ingresos generados por la campaña                                      | VARCHAR / DECIMAL | Mismos errores de formato numérico que `gasto_campana`, valores nulos |
| `impresiones`        | Número de impresiones generadas por la campaña                         | INT            | Valores nulos |
| `clics`              | Número de clics generados por la campaña                               | INT            | Valores nulos |
| `conversiones`       | Número de conversiones generadas por la campaña                        | INT            | Valores nulos |
| `responsable`        | Persona responsable de la gestión de la campaña                        | VARCHAR(100)   | Mayúsculas/minúsculas mezcladas, valores nulos |
| `estado_campana`     | Estado actual de la campaña (Activa, Pausada, Finalizada, Cancelada)   | VARCHAR(20)    | Mayúsculas/minúsculas mezcladas |

Adicionalmente, la tabla contiene **duplicados exactos** y **duplicados aproximados** (mismo `id_campana`, pero con variaciones de formato de texto en columnas como `empresa` o `comunidad_autonoma`), introducidos deliberadamente para poner a prueba el proceso de detección y eliminación de duplicidades.

## Metodología de Limpieza de Datos

El proceso de limpieza se ha estructurado en cuatro fases secuenciales, documentadas y ejecutadas íntegramente en SQL:

**1. Creación de una tabla de trabajo**

Se crea una copia de la tabla original (`campaña_marketing_crudo` → `campaña_clean`) sobre la que se aplican todas las transformaciones. La tabla fuente no se modifica en ningún momento, garantizando trazabilidad y la posibilidad de auditar o revertir el proceso en cualquier fase.

**2. Estandarización de los datos**

Se definen y aplican normas de presentación homogéneas para todas las columnas de texto, cubriendo:
- Uso consistente de mayúsculas y minúsculas (formato "Nombre Propio").
- Unificación de abreviaturas y variantes de nombres (p. ej. comunidades autónomas, canales de marketing) a un único valor canónico.
- Eliminación de espacios innecesarios (`TRIM`).
- Homogeneización de sufijos legales, símbolos, guiones y paréntesis en nombres de empresa.
- Normalización de formatos numéricos (separadores decimales y eliminación de símbolos de moneda dentro de campos numéricos).

**3. Eliminación de duplicados**

Se identifican los registros duplicados mediante la función de ventana `ROW_NUMBER()`, particionando por `id_campana` y ordenando por fecha, lo que permite detectar tanto duplicados exactos como duplicados con variaciones de formato en columnas de texto. Se conservan únicamente los registros con `ROW_NUMBER() = 1` dentro de cada partición.

**4. Tratamiento de valores nulos**

El tratamiento de valores ausentes se realiza conforme a las reglas de negocio establecidas por la empresa propietaria de los datos:

```
a.) Si la empresa no ha sido registrada, eliminamos esa fila; no queremos esos valores en el análisis.
b.) Si no contamos con información en comunidad autónoma, sector, responsable..., no ocurre nada, debemos dejarlo en 'DESCONOCIDO'.
c.) Si de alguna empresa no contamos con 2 de las 5 métricas de análisis (gasto, ingresos, impresiones, clics y conversiones), eliminamos esa fila del análisis.
```

Estas reglas priorizan la integridad del análisis posterior sobre la conservación del volumen total de filas: se descartan aquellos registros que no aportarían valor fiable (empresa desconocida o métricas insuficientes), mientras que la falta de información contextual no crítica se conserva de forma explícita bajo la etiqueta `'DESCONOCIDO'`, evitando así la pérdida de registros que sí contienen métricas válidas.
