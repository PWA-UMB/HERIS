# Correcciones HERIS — Instrumento de recolección (Etapa 3)

**Proyecto:** EcoUMB — Trabajo de grado UMB Bogotá
**Repositorio:** `GitHub/HERIS`
**Objetivo:** ajustar HERIS para que sea válido como instrumento de recolección de datos de tesis antes de aplicarlo a la muestra real.

Prioridad de implementación sugerida: 1 → 2 → 3 → 4 → 5 → 6.

---

## 1. Consentimiento informado

**Problema:** el formulario captura nombre completo y programa académico y los envía a Supabase sin ningún paso de consentimiento. Esto no cumple con lo declarado en las bases legales del proyecto (Ley 1581/2012, tratamiento de datos personales).

**Corrección requerida (en código, sin documento legal aparte):**

- Agregar un checkbox obligatorio antes de habilitar el botón "Comenzar juego", con un texto breve tipo: *"Acepto que mis datos (nombre, programa académico y resultados de esta actividad) sean usados con fines exclusivamente académicos para el trabajo de grado EcoUMB."*
- El botón `Comenzar juego` debe quedar deshabilitado (`disabled`) hasta que el checkbox esté marcado.
- Guardar el estado del consentimiento como un campo adicional en el `INSERT` a `game_sessions` (por ejemplo `consent_accepted: true`), para dejar trazabilidad de que cada registro tuvo consentimiento explícito.
- Si el checkbox no está marcado y el usuario intenta enviar, mostrar el mismo tipo de mensaje de error que ya usan para nombre/programa (`reg-error`).

---

## 2. Control de duplicados / identidad

**Problema:** no existe ninguna restricción que impida que una misma persona juegue varias veces con datos distintos (o el mismo nombre repetido), lo que puede inflar artificialmente la muestra (N = 80-120) y sesgar el análisis estadístico.

**Corrección requerida:**

- Agregar un campo de **correo institucional UMB** (validado por dominio, ej. `@umb.edu.co`) como identificador único del participante, además de nombre y programa.
- Antes de insertar en `game_sessions`, hacer una consulta a Supabase para verificar si ese correo ya tiene un registro. Si ya existe:
  - No permitir una nueva partida "oficial" (o marcarla como `is_retake: true` si se decide permitir reintentos, pero excluyendo esos registros del análisis principal).
- Alternativa mínima si no se quiere pedir correo: generar un hash de dispositivo/navegador (ej. combinación de `localStorage` con un UUID persistente) y bloquear más de una partida por dispositivo. Es menos robusto que el correo, pero mejor que nada.
- Documentar en el README de HERIS cuál mecanismo se implementó, para poder justificarlo en el Capítulo III (técnicas de recolección) frente al asesor.

---

## 3. Distribución balanceada de ítems por partida

**Problema:** los ítems se seleccionan con `Math.random()` sobre todo `WASTE_DB` sin garantizar proporción por categoría. Dos participantes pueden jugar el mismo número de rondas pero con mezclas de dificultad/categoría completamente distintas, lo que hace que el `accuracy` no sea comparable entre personas.

**Corrección requerida:**

- Antes de iniciar la partida, construir el mazo de ítems garantizando una distribución proporcional fija por categoría según el número de rondas seleccionado (10/15/20/30). Ejemplo para 15 rondas: ~4 aprovechables, ~4 no aprovechables, ~4 orgánicos, ~3 especiales (ajustar proporción si se resuelve el punto de categorías del informe anterior).
- Dentro de cada categoría, mezclar aleatoriamente los ítems disponibles (sí se puede usar `Math.random()` para el orden, pero no para la selección de categorías).
- Si el número de rondas es menor al número de categorías, documentar la regla de prioridad (por ejemplo, priorizar cubrir las 3-4 categorías al menos una vez antes de repetir).

---

## 4. Distinguir "incorrecta" de "no respondida por tiempo"

**Problema:** actualmente el timeout aplica una penalización fija de -3 puntos y se contabiliza igual que un error de clasificación, mezclando dos fenómenos distintos: falta de conocimiento vs. presión de tiempo.

**Corrección requerida:**

- En el objeto que se guarda por cada respuesta (`STATE.answers`), agregar un campo `answer_type` con valores: `'correct'`, `'incorrect'`, `'timeout'`.
- Al calcular `accuracy` para el resultado final y para el registro en `game_sessions`, mantener el cálculo actual (timeout cuenta como fallo), pero agregar un campo adicional agregado por partida, por ejemplo `timeout_count`, para que en el análisis estadístico se pueda filtrar o reportar por separado.
- Esto no cambia el puntaje ni la mecánica de juego, solo el detalle que se persiste para el análisis posterior.

---

## 5. Accesibilidad de los bins (contenedores de categoría)

**Problema:** los bins son `<div>` con `onclick`, sin atributos ARIA ni soporte real de foco/teclado más allá de los atajos numéricos globales; además hay combinaciones de color con contraste no verificado (ej. chip negro sobre fondo oscuro, chip rojo oscuro sobre blanco).

**Corrección requerida:**

- Cambiar los `div.bin` a elementos `<button>` (o agregar `role="button"`, `tabindex="0"` y manejar `keydown` para Enter/Espacio) para que sean navegables por teclado y anunciables por lectores de pantalla.
- Agregar `aria-label` descriptivo a cada bin (ej. `aria-label="Aprovechables: papel, cartón, plástico, vidrio, metal"`).
- Verificar contraste de los `cat-chip` y `bin` contra WCAG 2.1 AA (mínimo 4.5:1 para texto normal). Ajustar colores donde no cumpla, sin perder la paleta actual del proyecto.

---

## 6. Confirmación al salir / pausa de partida

**Problema:** no hay aviso al intentar salir a mitad de partida (se pierde el progreso sin advertencia) ni forma de pausar, lo cual es relevante porque las partidas se van a jugar en el campus con interrupciones frecuentes (clases, llamadas, etc.), generando sesiones incompletas que ensucian los datos.

**Corrección requerida:**

- Agregar un listener `beforeunload` (o equivalente dentro de la SPA) que muestre confirmación si el usuario intenta cerrar/recargar la pestaña durante una partida activa.
- Agregar un botón de "Pausar" visible durante el juego que detenga el temporizador (`STATE.timerSec`) y muestre un overlay simple con opciones "Continuar" / "Salir sin guardar".
- Las partidas abandonadas (cerradas sin completar) **no deben** generar un registro parcial en `game_sessions` — solo se inserta el registro al llegar a la pantalla de resultados, como ya ocurre; solo falta evitar que el usuario pierda el progreso sin saberlo.

---

## 7. Verificación de políticas RLS en Supabase

**Problema:** el README indica "escritura anónima, lectura solo autenticada", pero no está confirmado que la política de Row Level Security realmente restrinja `UPDATE` y `DELETE` para el rol `anon` — solo se menciona que la lectura está protegida. Si `UPDATE`/`DELETE` no están explícitamente bloqueados, cualquier persona con acceso a la consola del navegador (usando la anon key ya expuesta en el código, lo cual es normal en Supabase) podría alterar o borrar registros de otros participantes.

**Corrección requerida:**

- En el dashboard de Supabase (proyecto `recicla-ya`), revisar las políticas RLS de la tabla `game_sessions` y confirmar que existan reglas separadas para cada operación:
  - `INSERT`: permitido para rol `anon` (esto ya debe estar funcionando).
  - `SELECT`: bloqueado para `anon` (según el README, ya está así — confirmar).
  - `UPDATE`: bloqueado para `anon`.
  - `DELETE`: bloqueado para `anon`.
- Si alguna de estas políticas no existe explícitamente, crearla con condición `false` para el rol anónimo.
- Documentar las políticas finales en el README de HERIS (sección de seguridad) para que quede como evidencia en la tesis.

---

## Resumen para seguimiento

| # | Corrección | Área | Bloqueante antes de aplicar a la muestra real |
|---|---|---|---|
| 1 | Consentimiento informado en el flujo | Datos/legal | Sí |
| 2 | Control de duplicados (correo o device hash) | Datos/muestra | Sí |
| 3 | Distribución balanceada de ítems por partida | Metodología | Sí |
| 4 | Distinguir incorrecta vs. timeout en el registro | Metodología | Recomendado |
| 5 | Accesibilidad de los bins (ARIA, teclado, contraste) | Front-end/UX | Recomendado |
| 6 | Confirmación de salida / pausa | Front-end/UX | Recomendado |
| 7 | Verificar políticas RLS (UPDATE/DELETE bloqueados) | Backend/seguridad | Sí |
