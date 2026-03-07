# CTT-Fase Operativo (Candado Contextual por Olas)

Este documento formaliza la lógica para usar **Consenso de Tendencia Temporal (CTT)** como candado inteligente.

## 1) Objetivo

CTT no predice “la próxima operación”.
CTT evalúa si el contexto reciente está en:
- **Verde** (régimen favorable),
- **Rojo** (régimen adverso),
- **Neutro** (ruido/insuficiente evidencia).

Su función es **filtrar** decisiones del bot base, no reemplazarlo.

---

## 2) Arquitectura de decisión recomendada

Orden de compuertas:

1. **Sanidad dura** (token, saldo, locks, hard guards)
2. **Señal individual** (trigger, confirm, probabilidad base)
3. **CTT-Fase (contexto colectivo)**
4. **Ejecución**

Regla de oro:
- CTT tiene más poder para **bloquear en rojo** que para **habilitar en verde**.
- CTT nunca debe “crear entrada” sin aprobación mínima de la lógica base.

---

## 3) Definición de ventana (evitar ruido y “consenso fósil”)

Usar ventana **híbrida**:
- límite por tiempo: `W` segundos,
- límite por cantidad: `N_target` cierres recientes,
- mínimo de evidencia: `N_min`.

Interpretación:
- Se toma el subconjunto más reciente de cierres que cumpla tiempo y cantidad.
- Si no hay evidencia suficiente (`N < N_min`), el estado es **Neutro**.

### Parámetros iniciales sugeridos

Si duración típica por operación `D ≈ 60s`:
- `W = 120–180s`
- `N_target ≈ 2 * B` (B = bots activos)
- `N_min ≈ B` (o `B + 2` para mayor robustez)

Ejemplo con 6 bots:
- `W=180s`, `N_target=12`, `N_min=6–8`.

---

## 4) Confirmadores vs rezagados (núcleo de fase)

Dentro de una ola:

- **Confirmadores**: bots que cerraron dentro de la ventana activa.
- **Rezagados válidos**: bots activos con lag real dentro de un rango útil.

No medir consenso sobre “todos los bots”, sino sobre confirmadores:

- `confirm_win_rate = wins_confirmadores / n_confirmadores`

Esto evita castigar/acreditar bots que aún no cerraron.

---

## 5) Detección de estados CTT

Con `n_confirmadores >= M_min`:

- **Verde fuerte**: `confirm_win_rate >= 0.85` (o 0.90 en grupos más grandes)
- **Verde débil**: `0.75 <= confirm_win_rate < 0.85`
- **Neutro**: zona intermedia o evidencia insuficiente
- **Rojo débil**: `0.15 < confirm_win_rate <= 0.25`
- **Rojo fuerte**: `confirm_win_rate <= 0.15`

Escala sugerida para 6 bots:
- `M_min = 4–5`
- verde fuerte desde `0.80–0.85`
- rojo fuerte en `0.15–0.20`.

---

## 6) Definición de rezago válido

Tomar:
- `t_front = max(last_close_ts)` del grupo.
- `lag_bot = t_front - last_close_ts(bot)`.

Un bot es rezagado válido si:
- está activo,
- `LAG_MIN <= lag_bot <= LAG_MAX`.

Sugerencia para `D≈60s`:
- `LAG_MIN = 30–45s`
- `LAG_MAX = 90–120s`

Esto excluye bots “muertos” o demasiado atrasados.

---

## 7) Expiración de ola

Una ola debe tener vencimiento:
- si desde el primer cierre de la ola pasan más de `W`, la ola **expira**.
- una ola expirada no habilita entradas, aunque la razón de aciertos histórica siga alta.

Objetivo: evitar entradas tardías por eco estadístico.

---

## 8) Cómo modular el candado (sin bajar todo a ciegas)

En vez de umbral fijo global, usar modulación contextual:

- **Rojo fuerte**: bloqueo duro (NO-GO)
- **Rojo débil**: endurecer umbral de entrada
- **Neutro**: umbral base
- **Verde débil**: observación (sin relajación)
- **Verde fuerte + rezagado válido**: relajación moderada del umbral

Ejemplo conceptual sobre umbral base 60%:
- Rojo: 63–66%
- Neutro: 60%
- Verde fuerte + rezago válido: 56–58%

CTT no dispara entradas; solo cambia severidad de la compuerta.

---

## 9) Manejo de redundancia entre bots

Si varios bots son casi clones, su consenso puede inflar falsas certezas.

Agregar una penalización de confianza cuando:
- múltiples confirmadores provienen de perfiles altamente redundantes.

Lectura operativa:
- consenso diverso = más confiable,
- consenso clonado = aplicar “verde con asterisco” (menos relajación).

---

## 10) Diagnóstico operativo para el estado actual

Con warmup y confiabilidad inmadura:
- la probabilidad individual no debe ser llave principal,
- CTT-Fase debe usarse para control de régimen,
- priorizar prevención de “olas rojas” sobre persecución agresiva de verdes.

Esto reduce bloqueos ciegos por umbral fijo y evita sobreinterpretar muestras pequeñas.

---

## 11) Checklist de implementación lógica (sin código)

1. Definir `W`, `N_target`, `N_min`, `M_min`.
2. Separar confirmadores y rezagados válidos.
3. Calcular estado CTT en escala de 5 niveles.
4. Aplicar expiración de ola.
5. Modular umbral por estado CTT (rojo>verde en peso).
6. Exigir mínimos base siempre (token/saldo/trigger/hard guard).
7. Penalizar consenso redundante.
8. Auditar impacto por estado (win-rate y drawdown por régimen).

---

## 12) Conclusión

La mejora clave no es “bajar candado” de forma fija.
La mejora es volver el candado **contextual por fase**:
- más duro en rojo,
- más oportunista en verde confirmado,
- disciplinado con evidencia mínima y expiración temporal.

Así el sistema deja de depender de un único número de probabilidad y aprovecha mejor el comportamiento colectivo real de los bots.
