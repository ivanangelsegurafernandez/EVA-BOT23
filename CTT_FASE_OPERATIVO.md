# CTT-Fase Operativo (Candado principal por contexto y desfase)

> Documento lógico (sin código) para operar CTT-Fase como filtro contextual principal.

## 1) Qué es CTT exactamente

CTT es un filtro de contexto (“meteorología” del mercado) construido sobre el resultado reciente del grupo:
- 🟢 **Marea verde**: muchos aciertos recientes.
- 🔴 **Marea roja**: muchos fallos recientes.
- ⚪ **Neutro**: ruido o evidencia insuficiente.

Uso correcto:
- 🔴 bloquea o endurece entradas.
- 🟢 permite/prioriza **solo si** la lógica base ya aprobó.
- CTT **nunca** abre operación por sí solo.

---

## 2) Fase y desfase (sin inventar)

Cada bot tiene ritmo temporal de apertura/cierre. Su fase es su posición en el tiempo respecto al grupo.

Timestamps mínimos por bot:
- `last_open_ts`
- `last_close_ts`

Desfase grupal real:
- si `max(last_close_ts) - min(last_close_ts) > umbral` (10–30s), hay desfase explotable.

---

## 3) Núcleo CTT: línea de tiempo global de cierres

Cada cierre se registra como:
- `(timestamp_real_cierre, bot_id, result)` con `result ∈ {win/check, loss/x}`.

Regla crítica:
- usar timestamp real de cierre, no timestamp de log/print (evita desorden por latencia).

Consenso base en ventana activa:
- `consensus_win = wins / N`
- `consensus_loss = losses / N`

Estados mínimos:
- Verde: `consensus_win >= 0.80`
- Rojo: `consensus_win <= 0.20`
- Neutro: resto

Si `N < N_min`, CTT no opina (Neutro).

---

## 4) Solución al cuello de botella: usar **3 relojes**, no uno

El error típico es mezclar todo en una sola ventana. CTT-Fase necesita 3 capas temporales:

### 4.1 Reloj A: Ventana de ola (`W_wave`)
Pregunta: **¿sigue viva la misma marea?**

- Evalúa consenso colectivo de cierres.
- Para operaciones ~1m: `W_wave = 120–180s`.

### 4.2 Reloj B: Ventana de rezago (`LAG_MIN..LAG_MAX`)
Pregunta: **¿este bot llega tarde pero aún pertenece a la ola?**

- Para `D≈60s`: `LAG_MIN=30–45s`, `LAG_MAX=90–120s`.
- `<LAG_MIN`: diferencia normal (no rezago útil).
- `>LAG_MAX`: rezago viejo/desconectado.

### 4.3 Reloj C: Expiración del permiso (`TTL_wave`)
Pregunta: **¿aunque fue verde, todavía sirve operar?**

- La ola nace con su primer cierre confirmador.
- La ola deja de habilitar cuando vence su TTL.
- Base recomendada: `TTL_wave ≈ W_wave` (o algo más estricto, p. ej. 120–150s si `W_wave=180s`).

---

## 5) Ventana híbrida con frente de ola (mejora clave)

No basta “últimos N” o “últimos W desde ahora”.

Definición robusta:
- `t_front = timestamp del cierre más reciente del grupo`.
- Ola activa = cierres dentro de `[t_front - W_wave, t_front]`.

Luego aplicar límite de cantidad:
- usar hasta `N_target` cierres de esa banda temporal.

Esto evita consenso fósil y desalineaciones por ritmo desigual entre bots.

---

## 6) Confirmadores vs rezagados (columna vertebral)

### Confirmadores
Bots que ya cerraron dentro de la ola activa.

### Rezagados
Bots que aún no cerraron en la ola o quedaron atrás del frente.

### Métrica correcta
No usar `wins / total_bots`.
Usar:
- `confirm_win_rate = wins_confirmadores / n_confirmadores`.

Los rezagados no son evidencia; son candidatos.

---

## 7) Reglas de estado CTT-Fase

Con evidencia mínima `n_confirmadores >= M_min`:
- Verde fuerte: `confirm_win_rate >= T_green`
- Rojo fuerte: `confirm_win_rate <= T_red`
- Neutro: zona intermedia o evidencia insuficiente

Recomendación inicial:
- **6 bots**: `M_min=4–5`, `T_green=0.80–0.85`, `T_red=0.15–0.20`
- **10 bots**: `M_min=7`, `T_green=0.85–0.90`, `T_red=0.10–0.15`

Con 6 bots, exigir 90% rígido suele ser demasiado estricto.

---

## 8) Rezago válido (evitar “bot muerto”)

Definir frente:
- `t_front = max(last_close_ts)`

Lag por bot:
- `lag_bot = t_front - last_close_ts(bot)`

Rezagado válido si:
- bot activo,
- `LAG_MIN <= lag_bot <= LAG_MAX`,
- sigue dentro de `TTL_wave` de la ola activa.

---

## 9) Política operativa: candado asimétrico (rojo > verde)

Orden de compuertas:
1. Sanidad dura (token/saldo/locks/hard guard)
2. Señal individual (trigger/confirm/p_oper)
3. CTT-Fase
4. Ejecución

Reglas:
- 🔴 fuerte + rezagado válido ⇒ **NO-GO duro**.
- 🔴 débil ⇒ endurecer umbral y/o confirmaciones.
- ⚪ neutro ⇒ manda lógica base.
- 🟢 débil ⇒ observación (sin relajación).
- 🟢 fuerte + rezagado válido + ola viva ⇒ permiso/relajación moderada.

CTT modula severidad; no dispara entradas.

---

## 10) Consenso fósil: el error a evitar

Qué es:
- usar cierres “recientes” en cantidad pero viejos en régimen operativo.

Blindaje:
1. banda temporal dura con `W_wave`,
2. ola anclada al `t_front`,
3. expiración por `TTL_wave`.

---

## 11) Bots homogéneos: consenso correlacionado

Con bots similares, el consenso no es independencia estadística plena.

Lectura correcta:
- CTT funciona mejor como detector de régimen que como multiplicador ingenuo de probabilidad.

Mitigación:
- offsets temporales,
- micro-variaciones de umbral/confirmación,
- ventanas de features distintas,
- cooldowns diferentes.

---

## 12) Parámetros iniciales (operación ~1m)

### Escenario 6 bots
- `W_wave = 180s`
- `N_target = 12`
- `N_min = 6`
- `M_min = 4`
- `T_green = 0.80`
- `T_red = 0.20`
- `LAG_MIN = 45s`
- `LAG_MAX = 120s`
- `TTL_wave = 120–180s`

### Escenario 10 bots
- `W_wave = 120–180s`
- `N_target = 20`
- `N_min = 10–12`
- `M_min = 7`
- `T_green = 0.85`
- `T_red = 0.15`
- `LAG_MIN = 45s`
- `LAG_MAX = 120s`
- `TTL_wave = 120–180s`

---

## 13) Martingala C2..C6 (prudencia)

En continuidad de ciclo:
- usar CTT sobre todo como bloqueo rojo,
- evitar boost verde agresivo en escalones avanzados.

---

## 14) Checklist final (sin código)

1. Definir los 3 relojes: `W_wave`, `LAG_MIN..LAG_MAX`, `TTL_wave`.
2. Construir ola con frente `t_front`.
3. Separar confirmadores de rezagados.
4. Exigir evidencia mínima (`N_min`, `M_min`).
5. Calcular estado por `confirm_win_rate`.
6. Invalidar olas expiradas.
7. Aplicar candado asimétrico (rojo > verde).
8. Exigir mínimos base siempre.
9. Auditar rendimiento por régimen (verde/rojo/neutro).

---

## 15) Resumen ejecutivo

CTT-Fase robusto = **Confirmación de ola + Entrada por rezago válido + Expiración temporal**.

- Si la mayoría confirmadora valida ola verde y aún está viva, habilitas rezagados válidos (con mínimos base).
- Si la ola es roja, bloqueas duro.
- Si falta evidencia, CTT queda neutro.

La mejora real no es bajar un umbral fijo.
La mejora real es operar con contexto temporal de fase.
