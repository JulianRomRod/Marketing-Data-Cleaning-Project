# Proceso de Limpieza de Datos — Consultas SQL

Este documento recoge, paso a paso, todas las consultas SQL utilizadas en el proceso de limpieza de la tabla `campaña_marketing_crudo`, siguiendo la metodología descrita en el `README.md` del repositorio:

1. Creación de una tabla de trabajo
2. Estandarización de variables
3. Eliminación de duplicados
4. Gestión de valores nulos
5. Resultado final

---

## 1. Creación de la tabla de trabajo

Antes de aplicar cualquier transformación, se genera una copia de la tabla fuente. De esta forma, todo el proceso de limpieza se realiza sobre `campaña_clean`, dejando `campaña_marketing_crudo` intacta como referencia original.

```
SELECT*
INTO campaña_clean
FROM campaña_marketing_crudo
```

---

## 2. Estandarización de variables

En esta fase se homogeneiza el formato de todas las columnas de texto. Para cada columna se sigue el mismo patrón de trabajo:

1. Se listan los valores únicos con `DISTINCT` para hacerse una idea inicial del problema.
2. Se fuerza una comparación **case-sensitive** con `COLLATE Latin1_General_100_CS_AS`, ya que por defecto SQL Server no distingue mayúsculas de minúsculas y esto podía ocultar variantes del mismo valor.
3. Se define la regla de estandarización (`LIKE` para variantes con el mismo prefijo, o `IN` para abreviaturas/sinónimos sin relación textual directa).
4. Se aplica el `UPDATE` correspondiente.

### 2.1. Empresa

Se detectan empresas repetidas con mayúsculas/minúsculas mezcladas y distintos sufijos legales (S.L., SL, S.A., S.L.U.). Al compartir siempre el mismo texto inicial, se estandarizan con `LIKE 'nombre%'`.

```
SELECT
	DISTINCT empresa
FROM campaña_clean
ORDER BY 1 ASC

SELECT*
FROM campaña_clean
ORDER BY empresa ASC

SELECT TRIM(empresa) COLLATE Latin1_General_100_CS_AS AS valor, COUNT(*) AS total
FROM campaña_clean
GROUP BY TRIM(empresa) COLLATE Latin1_General_100_CS_AS
ORDER BY valor;

UPDATE campaña_clean
SET
	empresa = CASE WHEN LOWER(TRIM(empresa)) LIKE 'agrocampo%' THEN 'Agrocampo'
	WHEN LOWER(TRIM(empresa)) LIKE 'automotion%' THEN 'AutoMotion'
	WHEN LOWER(TRIM(empresa)) LIKE 'fintech nova%' THEN 'Fintech Nova'
	WHEN LOWER(TRIM(empresa)) LIKE 'grupo peralta%' THEN 'Grupo Peralta'
	WHEN LOWER(TRIM(empresa)) LIKE 'rural gourmet%' THEN 'Rural Gourmet'
	ELSE empresa END
```

### 2.2. Comunidad Autónoma

A diferencia de `empresa`, aquí las variantes **no comparten un prefijo común** (por ejemplo, "Comunidad Valenciana" y "CV" no tienen relación textual entre sí), por lo que `LIKE` no es aplicable. En su lugar, se usa `IN` con la lista explícita de todas las variantes detectadas para cada comunidad. Los nulos disfrazados de texto (`n/a`, `sin datos`, `desconocido`, `-`) se convierten aquí directamente a `NULL`, para poder tratarlos más adelante junto al resto de valores ausentes según las reglas de negocio.

```
SELECT
	DISTINCT comunidad_autonoma
FROM campaña_clean
ORDER BY 1 ASC

SELECT*
FROM campaña_clean
ORDER BY empresa ASC

SELECT TRIM(comunidad_autonoma) COLLATE Latin1_General_100_CS_AS AS valor, COUNT(*) AS total
FROM campaña_clean
GROUP BY TRIM(comunidad_autonoma) COLLATE Latin1_General_100_CS_AS
ORDER BY valor;

UPDATE campaña_clean
SET
	comunidad_autonoma = CASE WHEN TRIM(comunidad_autonoma) IN ('baleares','Baleares','IB', 'BALEARES') THEN 'Islas Baleares'
	WHEN TRIM(comunidad_autonoma) IN ('C. Valenciana','CV', 'comunidad valenciana','Valencia','valencia') THEN 'Comunidad Valenciana'
	WHEN TRIM(comunidad_autonoma) IN ('madrid','Madrid', 'MADRID','MAD') THEN 'Comunidad de Madrid'
	WHEN TRIM(comunidad_autonoma) IN ('Region de Murcia','MURCIA', 'Murcia','MU') THEN 'Región de Murcia'
	WHEN TRIM(comunidad_autonoma) IN ('n/a','N/A', 'sin datos','desconocido','-') THEN NULL
	ELSE comunidad_autonoma END
```

### 2.3. Sector

En esta columna no se detectan variantes de formato relevantes en las categorías propias; el único tratamiento necesario es convertir los nulos disfrazados de texto a `NULL` real, para que sean gestionados junto al resto de valores ausentes en la fase 4.

```
SELECT
	DISTINCT sector
FROM campaña_clean
ORDER BY 1 ASC

SELECT*
FROM campaña_clean
ORDER BY sector ASC

SELECT TRIM(sector) COLLATE Latin1_General_100_CS_AS AS valor, COUNT(*) AS total
FROM campaña_clean
GROUP BY TRIM(sector) COLLATE Latin1_General_100_CS_AS
ORDER BY valor;

UPDATE campaña_clean
SET
	sector = CASE WHEN TRIM(sector) IN ('n/a','N/A','desconocido','-') THEN NULL
	ELSE sector END
```

### 2.4. Canal de Marketing

Se estandarizan abreviaturas y sinónimos del mismo canal (p. ej. "RRSS" / "redes sociales" / "Social Media" → "Redes Sociales"), unificando el criterio a un único valor canónico por canal.

```
SELECT
	DISTINCT canal_marketing
FROM campaña_clean
ORDER BY 1 ASC

SELECT*
FROM campaña_clean
ORDER BY canal_marketing ASC

SELECT TRIM(canal_marketing) COLLATE Latin1_General_100_CS_AS AS valor, COUNT(*) AS total
FROM campaña_clean
GROUP BY TRIM(canal_marketing) COLLATE Latin1_General_100_CS_AS
ORDER BY valor;

UPDATE campaña_clean
SET
	canal_marketing = CASE WHEN TRIM(canal_marketing) IN ('GAds', 'google ads' ) THEN 'Google Ads'
	WHEN TRIM(canal_marketing) IN ('RRSS', 'redes sociales','Social Media') THEN 'Redes Sociales'
	WHEN LOWER(TRIM(canal_marketing)) IN ('tv') THEN 'Televisión'
	ELSE canal_marketing END
```

### 2.5. Tipo de Campaña

Se corrige la mezcla de mayúsculas y minúsculas en las categorías `Fidelización` y `Branding`, que eran las únicas con inconsistencias detectadas.

```
SELECT
	DISTINCT tipo_campana
FROM campaña_clean
ORDER BY 1 ASC

SELECT TRIM(tipo_campana) COLLATE Latin1_General_100_CS_AS AS valor, COUNT(*) AS total
FROM campaña_clean
GROUP BY TRIM(tipo_campana) COLLATE Latin1_General_100_CS_AS
ORDER BY valor;

UPDATE campaña_clean
SET
	tipo_campana = CASE WHEN TRIM(tipo_campana) LIKE 'fidelizacion' THEN 'Fidelización'
	WHEN TRIM(tipo_campana) LIKE 'branding' THEN 'Branding'
	ELSE tipo_campana END
```

### 2.6. Responsable

Al igual que en `empresa`, todas las variantes de cada responsable comparten el mismo nombre como prefijo, por lo que se usa `LIKE 'nombre%'` para capturarlas todas de una vez. Adicionalmente, se aprovecha este `UPDATE` para convertir los nulos disfrazados de texto (`sin datos`, `n/a`, `desconocido`, `-`) a `NULL`.

```
SELECT
	DISTINCT responsable
FROM campaña_clean
ORDER BY 1 ASC

SELECT TRIM(responsable) COLLATE Latin1_General_100_CS_AS AS valor, COUNT(*) AS total
FROM campaña_clean
GROUP BY TRIM(responsable) COLLATE Latin1_General_100_CS_AS
ORDER BY valor;

UPDATE campaña_clean
SET
	responsable = CASE WHEN LOWER(TRIM(responsable)) LIKE 'álvaro gil%' THEN 'Álvaro Gil'
	WHEN LOWER(TRIM(responsable)) LIKE 'ana fernández%' THEN 'Ana Fernández'
	WHEN LOWER(TRIM(responsable)) LIKE 'carlos ruiz%' THEN 'Carlos Ruiz'
	WHEN LOWER(TRIM(responsable)) LIKE 'cristina ortega%' THEN 'Cristina Ortega'
	WHEN LOWER(TRIM(responsable)) LIKE 'diego romero%' THEN 'Diego Romero'
	WHEN LOWER(TRIM(responsable)) LIKE 'elena castro%' THEN 'Elena Castro'
	WHEN LOWER(TRIM(responsable)) LIKE 'javier torres%' THEN 'Javier Torres'
	WHEN LOWER(TRIM(responsable)) LIKE 'laura gómez%' THEN 'Laura Gómez'
	WHEN LOWER(TRIM(responsable)) LIKE 'lucía herrera%' THEN 'Lucía Herrera'
	WHEN LOWER(TRIM(responsable)) LIKE 'marta sánchez%' THEN 'Marta Sánchez'
	WHEN LOWER(TRIM(responsable)) LIKE 'miguel álvarez%' THEN 'Miguel Álvarez'
	WHEN LOWER(TRIM(responsable)) LIKE 'paula serrano%' THEN 'Paula Serrano'
	WHEN LOWER(TRIM(responsable)) LIKE 'sara molina%' THEN 'Sara Molina'
	WHEN LOWER(TRIM(responsable)) LIKE 'david lópez%' THEN 'David López'
	WHEN LOWER(TRIM(responsable)) LIKE 'pablo navarro%' THEN 'Pablo Navarro'
	WHEN LOWER(TRIM(responsable)) IN ('sin datos','n/a','desconocido','-') THEN NULL
	ELSE responsable END
```

### 2.7. Estado de Campaña

Se unifican las variantes de mayúsculas/minúsculas encontradas en los estados `Activa` y `Finalizada`.

```
SELECT
	DISTINCT estado_campana
FROM campaña_clean
ORDER BY 1 ASC

SELECT TRIM(estado_campana) COLLATE Latin1_General_100_CS_AS AS valor, COUNT(*) AS total
FROM campaña_clean
GROUP BY TRIM(estado_campana) COLLATE Latin1_General_100_CS_AS
ORDER BY valor;

UPDATE campaña_clean
SET
	estado_campana = CASE WHEN LOWER(TRIM(estado_campana)) LIKE 'activa%' THEN 'Activa'
	WHEN LOWER(TRIM(estado_campana)) LIKE 'finalizada%' THEN 'Finalizada'
	ELSE estado_campana END
```

---

## 3. Eliminación de duplicados

Con las columnas de texto ya estandarizadas, se procede a identificar y eliminar los registros duplicados. Se utiliza la función de ventana `ROW_NUMBER()`, particionando por `id_campana`: cualquier fila con un número de partición mayor que 1 es una copia duplicada de esa misma campaña. El resultado se materializa en una nueva tabla (`campaña_limpio`) antes de eliminar las filas sobrantes.

```
WITH duplicados_cte as
(
SELECT
	*,
	ROW_NUMBER() OVER(PARTITION BY id_campana ORDER BY id_campana) as duplicados
FROM campaña_clean
)
SELECT
	*
INTO campaña_limpio
FROM duplicados_cte

SELECT*
FROM campaña_limpio

DELETE
FROM campaña_limpio
WHERE duplicados > 1

ALTER TABLE campaña_limpio
DROP COLUMN duplicados
```

---

## 4. Gestión de valores nulos (NULLs)

Con los datos ya estandarizados y sin duplicados, se aplican las tres reglas de negocio definidas por la empresa para el tratamiento de valores ausentes:

> 1. Si la empresa no ha sido registrada, eliminamos esa fila; no queremos esos valores en el análisis.
> 2. Si no contamos con alguna información en comunidad autónoma, sector, responsable..., no ocurre nada, debemos dejarlo en `'DESCONOCIDO'`.
> 3. Si de alguna empresa no contamos con 2 de las 5 métricas de análisis (gasto, ingresos, impresiones, clics y conversiones), eliminamos esa fila del análisis.

**Regla 1 — Empresa no registrada → se elimina la fila.**

```
DELETE
FROM campaña_limpio
WHERE empresa IS NULL
```

**Regla 2 — Información contextual ausente (comunidad autónoma, sector, responsable) → se marca como `'DESCONOCIDO'`.**

```
UPDATE campaña_limpio
SET
	comunidad_autonoma = 'DESCONOCIDO'
WHERE comunidad_autonoma IS NULL

UPDATE campaña_limpio
SET
	sector = 'DESCONOCIDO'
WHERE sector IS NULL

UPDATE campaña_limpio
SET
	responsable = 'DESCONOCIDO'
WHERE responsable IS NULL
```

**Regla 3 — Faltan 2 o más de las 5 métricas de análisis → se elimina la fila.** Para aplicarla, se calcula por fila cuántas de las cinco métricas (`gasto_campana`, `ingresos`, `impresiones`, `clics`, `conversiones`) están vacías, sumando un `CASE WHEN ... IS NULL` por cada una. El resultado se vuelca en la tabla final (`campañas_limpio_final`), se eliminan las filas que incumplen la regla, y finalmente se descarta la columna auxiliar `metricas_faltantes` por no aportar información al análisis.

```
WITH métricas_CTE as
(
SELECT
    *,
    (CASE WHEN gasto_campana IS NULL THEN 1 ELSE 0 END +
     CASE WHEN ingresos       IS NULL THEN 1 ELSE 0 END +
     CASE WHEN impresiones    IS NULL THEN 1 ELSE 0 END +
     CASE WHEN clics          IS NULL THEN 1 ELSE 0 END +
     CASE WHEN conversiones   IS NULL THEN 1 ELSE 0 END) AS metricas_faltantes
FROM campaña_limpio
)
SELECT*
INTO campañas_limpio_final
FROM métricas_CTE

DELETE
FROM campañas_limpio_final
WHERE metricas_faltantes >= 2

ALTER TABLE campañas_limpio_final
DROP COLUMN metricas_faltantes
```

---

## 5. Resultado final

Se comprueba que la tabla ha quedado correctamente estandarizada, sin duplicados y con los nulos tratados según las reglas de negocio. Finalmente, se eliminan las tablas intermedias generadas durante el proceso (`campaña_clean` y `campaña_limpio`), conservando en el repositorio únicamente la tabla fuente y la tabla limpia final.

```
DELETE FROM campaña_clean
DELETE FROM campaña_limpio

SELECT*
FROM campaña_marketing_crudo

SELECT*
FROM campañas_limpio_final
```

### Tablas resultantes del proceso

| Tabla | Rol | ¿Se conserva? |
|---|---|---|
| `campaña_marketing_crudo` | Tabla fuente original, sin modificar | ✅ Sí |
| `campaña_clean` | Tabla de trabajo para la fase de estandarización | ❌ Eliminada al finalizar |
| `campaña_limpio` | Tabla intermedia tras eliminar duplicados | ❌ Eliminada al finalizar |
| `campañas_limpio_final` | Tabla final, estandarizada, sin duplicados y con NULLs tratados | ✅ Sí |
