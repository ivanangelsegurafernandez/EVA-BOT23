# CTT-Fase Operativo (Candado principal por contexto y desfase)

> Documento lógico (sin código) para operar el **Consenso de Tendencia Temporal (CTT)** como filtro principal de contexto.

## 0) Definición corta (sin maquillaje)

CTT es un **filtro de clima de mercado** basado en cierres recientes del grupo de bots.

- 🟢 **Marea verde**: predominan aciertos recientes.
- 🔴 **Marea roja**: predominan fallos recientes.
- ⚪ **Neutro**: ruido o evidencia insuficiente.

Uso correcto:
- En 🔴: CTT **bloquea/endurece**.
- En 🟢: CTT **permite/prioriza** solo si la lógica base ya dijo GO.
- CTT **nunca** debe abrir una operación por sí solo.

---

## 1) Qué significa “fase” y “desfase” entre bots

Cada bot tiene ritmo propio de apertura/cierre.
La fase es la posición temporal relativa entre bots.

Se detecta desfase real con dos timestamps por bot:
- `last_open_ts`
- `last_close_ts`

Criterio mínimo de desfase grupal:
- si `max(last_close_ts) - min(last_close_ts)` supera umbral (10–30s), hay desfase explotable.

Esto evita inventar “sincronías” donde no existen.

---

## 2) Núcleo operativo CTT (línea de tiempo global)

### 2.1 Registro único de cierres

Cada cierre del grupo se guarda como evento:
- `(timestamp_real_cierre, bot_id, result)`
- `result ∈ {win/check, loss/x}`

**Importante**: usar timestamp real de cierre, no timestamp de impresión/log.

### 2.2 Consenso básico

En la ventana activa:
- `consensus_win = wins / N`
- `consensus_loss = losses / N`

Umbrales base recomendados:
- Verde si `consensus_win >= 0.80`
- Rojo si `consensus_win <= 0.20`
- Neutro en zona intermedia

### 2.3 Confianza mínima (anti-muestra chica)

Si `N < N_min`, CTT no opina:
- estado = ⚪ Neutro.

Regla práctica robusta:
- con 6 bots: `N_min = 6–8`
- con 10 bots: `N_min = 10–12`

---

## 3) Ventanas de tiempo: el cuello de botella real

CTT no mira velas; mira cierres en tiempo común.

### 3.1 Modelos de ventana

1. **Solo tiempo (`W`)**: últimos `W` segundos.
2. **Solo cantidad (`N`)**: últimos `N` cierres.
3. **Híbrida (recomendada)**: últimos `N_target` cierres, siempre que estén dentro de `W`.

La híbrida evita dos trampas:
- consenso fósil (cierres viejos),
- consenso sin evidencia (pocos cierres en `W`).

### 3.2 Cómo fijar W y N sin adivinar

Atar parámetros a:
- `D` = duración típica de operación,
- `B` = bots activos.

Reglas:
- `W ≈ 2D a 3D`
- `N_target ≈ 2B`
- `N_min ≈ B`

Ejemplo `D=60s`:
- 6 bots: `W=180s`, `N_target=12`, `N_min=6–8`
- 10 bots: `W=120–180s`, `N_target=20`, `N_min=10–12`

---

## 4) Confirmadores vs rezagados (CTT-Fase real)

### 4.1 Confirmadores
Bots que ya cerraron dentro de la ola (`W`).

### 4.2 Rezagados
Bots que aún no cerraron en esa ola o cuyo último cierre quedó retrasado respecto al frente.

### 4.3 Métrica correcta de confirmación

No usar porcentaje sobre todos los bots.
Usar:
- `confirm_win_rate = wins_confirmadores / n_confirmadores`

Condición de ola verde confirmada:
- `n_confirmadores >= M_min`
- `confirm_win_rate >= T_green`

Condición de ola roja confirmada:
- `n_confirmadores >= M_min`
- `confirm_win_rate <= T_red`

Sugerencia inicial:
- 6 bots: `M_min=4–5`, `T_green=0.80–0.85`, `T_red=0.15–0.20`
- 10 bots: `M_min=7`, `T_green=0.85–0.90`, `T_red=0.10–0.15`

---

## 5) Rezago válido (para no confundir bot muerto con oportunidad)

Definir frente del grupo:
- `t_front = max(last_close_ts)`

Lag por bot:
- `lag_bot = t_front - last_close_ts(bot)`

Bot rezagado válido si:
- está activo,
- `LAG_MIN <= lag_bot <= LAG_MAX`.

Sugerencia `D≈60s`:
- `LAG_MIN = 30–45s`
- `LAG_MAX = 90–120s` (o hasta `2W` según régimen)

---

## 6) Expiración de ola (pieza crítica)

La ola no dura infinito.

Regla:
- si desde el primer cierre de la ola pasaron más de `W`, la ola expira.
- ola expirada no habilita entradas, aunque su ratio histórico se vea bonito.

Esto evita entrar tarde al final de una racha.

---

## 7) CTT como candado principal: política operativa

Orden recomendado:
1. Sanidad dura (token, saldo, locks, hard guards)
2. Señal individual (trigger, confirm, p_oper)
3. CTT-Fase (contexto)
4. Ejecución

Reglas:
- 🔴 + rezagado válido ⇒ **NO-GO duro**
- 🟢 + rezagado válido ⇒ **permiso condicionado** (solo si base ya aprobó)
- ⚪ Neutro ⇒ lógica base normal

Principio:
- CTT tiene más peso para **bloquear en rojo** que para “empujar” en verde.

---

## 8) Modulación del candado (sin aflojar a martillazos)

No usar un único umbral fijo ciego.
Usar umbral contextual:

- Rojo fuerte: endurecer fuerte / bloqueo
- Rojo débil: endurecer moderado
- Neutro: umbral base
- Verde débil: observación (sin regalo)
- Verde fuerte + rezago válido: relajación moderada

Ejemplo conceptual sobre base 60%:
- Rojo: 63–66%
- Neutro: 60%
- Verde fuerte + rezago válido: 56–58%

CTT modula severidad; no crea entradas.

---

## 9) Bots clones y redundancia (riesgo silencioso)

Si varios bots son muy parecidos, pueden “votar igual” por la misma causa.
Eso puede inflar consenso falso.

Mitigación lógica:
- consenso diverso = mayor confianza,
- consenso redundante = “verde con asterisco” (menos relajación).

Cómo ganar valor con más bots sin cambiar estrategia base:
- offsets temporales,
- pequeñas variaciones de confirmación,
- ventanas de features distintas,
- cooldowns diferentes.

---

## 10) Martingala C2..C6 (criterio prudente)

Para continuidad de ciclo:
- aplicar CTT sobre todo como **bloqueo rojo**,
- evitar usar verde como “boost” agresivo en escalones avanzados.

Motivo: proteger capital cuando el régimen se vuelve adverso.

---

## 11) Parámetros iniciales listos para operar

### Escenario A: 6 bots (actual)
- `W = 180s`
- `N_target = 12`
- `N_min = 6–8`
- `M_min = 4–5`
- `LAG_MIN = 45s`
- `LAG_MAX = 90–120s`
- Verde: `>=0.80` (fuerte desde `0.85`)
- Rojo: `<=0.20` (fuerte desde `0.15`)

### Escenario B: 10 bots (escalado)
- `W = 120–180s`
- `N_target = 20`
- `N_min = 10–12`
- `M_min = 7`
- `LAG_MIN = 45s`
- `LAG_MAX = 90–120s`
- Verde fuerte: `>=0.85–0.90`
- Rojo fuerte: `<=0.10–0.15`

---

## 12) Qué hacer en estado de warmup / confiabilidad baja

Cuando el modelo aún está inmaduro:
- no usar prob IA cruda como llave única,
- usar CTT-Fase para control de régimen,
- priorizar evitar rojos sobre perseguir verdes.

Traducción operativa:
- primero reducir errores de contexto,
- luego exprimir precisión de probabilidad individual.

---

## 13) Checklist de implementación mental (sin código)

1. Definir `W`, `N_target`, `N_min`, `M_min`, `LAG_MIN`, `LAG_MAX`.
2. Construir línea de cierres con timestamp real.
3. Separar confirmadores y rezagados.
4. Calcular estado CTT con evidencia mínima.
5. Aplicar expiración de ola.
6. Modular umbral según estado CTT (rojo > verde en peso).
7. Exigir mínimos base siempre.
8. Penalizar consenso redundante.
9. Auditar rendimiento por régimen (verde/rojo/neutro).

---

## 14) Resumen ejecutivo

CTT-Fase correcto = **Confirmación de ola + Entrada por rezago válido**.

- Mayoría de confirmadores valida el régimen.
- Solo rezagados válidos pueden aprovechar esa ola.
- Si aparece ola roja, se bloquea duro.
- Si falta evidencia, CTT no opina (neutro).

Conclusión:
La mejora real no es bajar el candado fijo.
La mejora real es volverlo **contextual, temporal y disciplinado por fase**.
