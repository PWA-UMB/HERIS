# HEIRS — Herramienta Educativa Interactiva de Residuos Sólidos

Juego web educativo para clasificar residuos sólidos según las categorías del **PGIRS Colombia** (Resolución 2184 de 2019). Desarrollado como herramienta de investigación para tesis de grado — **UMB**.

---

## Cómo funciona

1. El usuario ingresa su **nombre** y **programa académico**
2. Configura el número de rondas y el tiempo por ítem
3. Clasifica 15–30 residuos arrastrándolos o haciendo clic en el contenedor correcto
4. Al terminar, ve su puntaje, precisión por categoría y clasificación
5. **Todos los resultados se guardan automáticamente en Supabase**

---

## Ver los resultados (Supabase)

### Acceso al dashboard

1. Ve a [supabase.com](https://supabase.com) e inicia sesión
2. Abre el proyecto **recicla-ya**
3. En el menú lateral: **Table Editor → game_sessions**

Ahí verás cada sesión registrada con todos los campos.

---

## Consultas SQL útiles para el análisis de tesis

Ve a **SQL Editor** en el dashboard y ejecuta cualquiera de estas:

### Resumen general de todas las partidas
```sql
SELECT
    nombre,
    programa,
    score,
    total_rounds,
    accuracy,
    avg_response_time,
    max_streak,
    classification,
    created_at
FROM game_sessions
ORDER BY created_at DESC;
```

### Conteo y promedio por clasificación obtenida
```sql
SELECT
    classification,
    COUNT(*)                        AS total_sesiones,
    ROUND(AVG(accuracy), 2)         AS precision_promedio,
    ROUND(AVG(score), 1)            AS puntaje_promedio,
    ROUND(AVG(avg_response_time), 2) AS tiempo_respuesta_prom
FROM game_sessions
GROUP BY classification
ORDER BY total_sesiones DESC;
```

### Desempeño por programa académico
```sql
SELECT
    programa,
    COUNT(*)                  AS participantes,
    ROUND(AVG(accuracy), 2)   AS precision_promedio,
    ROUND(AVG(score), 1)      AS puntaje_promedio
FROM game_sessions
GROUP BY programa
ORDER BY precision_promedio DESC;
```

### Categoría con más errores (global)
```sql
SELECT
    'Aprovechables'         AS categoria, SUM(apr_correct)   AS correctas, SUM(apr_total)   AS total FROM game_sessions
UNION ALL SELECT
    'No Aprovechables',                    SUM(noapr_correct),              SUM(noapr_total) FROM game_sessions
UNION ALL SELECT
    'Orgánicos',                           SUM(org_correct),                SUM(org_total)   FROM game_sessions
UNION ALL SELECT
    'Especiales/Peligrosos',               SUM(esp_correct),                SUM(esp_total)   FROM game_sessions;
```

### Exportar todo a CSV

En el SQL Editor, ejecuta `SELECT * FROM game_sessions ORDER BY created_at DESC;`
y haz clic en el botón **Export → Download CSV** que aparece en los resultados.

---

## Columnas de la tabla `game_sessions`

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | uuid | Identificador único de la sesión |
| `created_at` | timestamptz | Fecha y hora de la partida |
| `nombre` | text | Nombre del participante |
| `programa` | text | Programa académico |
| `score` | integer | Puntaje final |
| `total_rounds` | integer | Ítems jugados |
| `correct_answers` | integer | Respuestas correctas |
| `accuracy` | numeric | Porcentaje de aciertos (0–100) |
| `avg_response_time` | numeric | Tiempo promedio de respuesta (segundos) |
| `max_streak` | integer | Racha máxima de aciertos consecutivos |
| `classification` | text | Clasificación obtenida |
| `timer_seconds` | integer | Segundos configurados por ítem (0 = sin límite) |
| `apr_correct / apr_total` | integer | Aciertos y total en Aprovechables |
| `noapr_correct / noapr_total` | integer | Aciertos y total en No Aprovechables |
| `org_correct / org_total` | integer | Aciertos y total en Orgánicos |
| `esp_correct / esp_total` | integer | Aciertos y total en Especiales/Peligrosos |

---

## Clasificaciones del juego

| Rango de precisión | Clasificación |
|---|---|
| ≥ 90% | Experto en Reciclaje |
| 75–89% | Reciclador Avanzado |
| 60–74% | Nivel Intermedio |
| 40–59% | Reciclador Principiante |
| < 40% | Necesita Reforzar Conceptos |

---

## Stack técnico

- **Frontend:** HTML5 + CSS3 + JavaScript vanilla (sin dependencias)
- **Base de datos:** Supabase (PostgreSQL) — plan gratuito
- **Hosting:** GitHub Pages
- **Seguridad:** Row Level Security (RLS) — escritura anónima, lectura solo autenticada
