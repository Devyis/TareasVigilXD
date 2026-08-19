# Cambios: distractores curados en el modo Versus

## El problema original

En el modo ⚔️ Versus, cada pregunta se arma como opción múltiple: 1 respuesta correcta + 3
"distractores" (opciones incorrectas). Antes, esos 3 distractores se sacaban **al azar de las
respuestas de otras tarjetas del mismo mazo** (función `vsBuildQuestions`).

Esto generaba opciones fáciles de descartar a simple vista, porque:
- Un distractor de otro tema se nota (ej. pregunta sobre "Middleware" con una opción sobre
  "Visual Studio").
- La longitud/estilo de la opción correcta no siempre coincidía con los distractores, delatando
  cuál era la real.
- Con mazos chicos, los mismos distractores se repetían seguido y el usuario los memorizaba
  como "esta nunca es la correcta".

## La solución

Se agregó soporte para que cada tarjeta pueda traer **3 distractores diseñados específicamente
para esa pregunta** (mismo tema, longitud y estilo que la respuesta correcta), en vez de reciclar
respuestas de otras tarjetas.

### 1. Nuevo formato de CSV (compatible con el viejo)

El importador de CSV (pestaña ⚙️ Gestionar → Importar CSV) ahora acepta dos formatos:

**Formato simple (como antes, sigue funcionando):**
```
pregunta,respuesta
```

**Formato Versus (nuevo, recomendado):**
```
pregunta,respuesta,distractor1,distractor2,distractor3
```

El parser (`fcParseCsv`) detecta automáticamente cuántas columnas trae cada línea:
- Si hay 5+ columnas → guarda `{ q, a, distractors: [d1, d2, d3] }`.
- Si hay 2-4 columnas → guarda `{ q, a }` (formato viejo, sin distractores curados).

No hace falta convertir mazos existentes: los que ya están cargados sin distractores siguen
funcionando igual que siempre (ver punto 3, fallback).

### 2. `vsBuildQuestions()` usa los distractores curados

Cuando arma las preguntas de una partida Versus, la función ahora revisa cada tarjeta:

- **Si la tarjeta tiene `distractors` (array de 3)** → usa esos 3 tal cual, en vez de sacar
  respuestas de otras cartas al azar.
- **Si no los tiene** → cae al comportamiento anterior (fallback): arma distractores tomando
  respuestas de otras tarjetas del mazo al azar. Así el modo Versus sigue funcionando incluso
  con mazos viejos o tarjetas agregadas a mano sin distractores.

Esto es automático — no hay que activar nada ni migrar datos existentes.

### 3. Compatibilidad hacia atrás

- Mazos ya guardados en Supabase con el formato viejo (`{q, a}`) **no se rompen**: simplemente
  no tienen distractores curados y el Versus sigue armándolos al azar como antes.
- Se puede mezclar dentro de un mismo mazo: algunas tarjetas con distractores curados (subidas
  vía CSV de 5 columnas) y otras sin ellos (agregadas a mano desde "Agregar tarjeta"). Cada una
  usa su propio método al jugar.
- La tarjeta agregada manualmente (`fcAddSingleCard`, un solo par pregunta/respuesta) **no**
  pide distractores — sigue guardando solo `{q, a}` y usará el fallback aleatorio en Versus. Si
  se quiere que una tarjeta puntual tenga distractores curados, hay que subirla vía CSV.

## Cómo generar el CSV con distractores curados

La forma más simple: pedirle a Claude que tome tu CSV simple (`pregunta,respuesta`, el que baja
NotebookLM por ejemplo) y devuelva la versión de 5 columnas, con 3 distractores por pregunta que:
- sean del mismo tema exacto que la pregunta (no de otra parte del mazo),
- tengan longitud y estilo parecido a la respuesta correcta,
- sean claramente incorrectos para alguien que sí estudió, pero no "raros" a simple vista.

Después subís ese CSV de 5 columnas por el importador normal (pestaña Gestionar → Importar CSV)
y listo — el Versus ya usa esos distractores automáticamente.

## Archivos modificados

- `index.html`:
  - `fcParseCsv()` — detecta y parsea el formato de 5 columnas.
  - `vsBuildQuestions()` — prioriza `card.distractors` sobre el método aleatorio.
  - Texto de ayuda (`fc-csv-hint`) en la pestaña Gestionar, actualizado para explicar ambos
    formatos.
