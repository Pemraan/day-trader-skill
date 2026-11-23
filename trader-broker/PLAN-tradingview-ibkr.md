# Plan: automatizar el day trading con TradingView + Interactive Brokers

Plan de implantación en 6 fases (unas 8 semanas) para pasar de operar a mano
a un sistema donde **TradingView detecta la señal** e **Interactive Brokers
ejecuta**, con el riesgo controlado por código y no por la emoción del momento.

> No es asesoramiento financiero. La mayoría de quienes hacen day trading
> pierden dinero. La automatización no convierte una estrategia perdedora en
> ganadora: solo la ejecuta más rápido. Todo empieza en cuenta **paper**.

---

## 1. La arquitectura en una imagen

```
 TradingView (Pine Script)          Puente (tu VPS, 24/7)                 Interactive Brokers
 ┌─────────────────────┐  webhook  ┌───────────────────────────┐  API    ┌──────────────────┐
 │ estrategia + alerta │ ────────► │ 1. valida secreto + IP    │ ──────► │ IB Gateway       │
 │ JSON con la señal   │  HTTPS    │ 2. idempotencia (id)      │  ib_    │ (paper :4002)    │
 └─────────────────────┘           │ 3. trader-risk (PASS?)    │  async  │ orden bracket:   │
                                   │ 4. tamaño (trader-budget) │         │ entrada+stop+TP  │
                                   │ 5. modo: asistido/semi/auto│ ◄────── │ fills, posiciones│
                                   │ 6. diario + Telegram      │         └──────────────────┘
                                   └───────────────────────────┘
                                         ▲ kill switch, flat a las 15:55 ET
```

**Regla de oro:** TradingView *propone*, el puente *decide*, IBKR *ejecuta*.
El precio, el tamaño y los límites nunca los decide la alerta: los recalcula
el puente con los datos del broker.

### Tres niveles de automatización (se sube de nivel solo con datos)

| nivel | quién pulsa "comprar" | cuándo usarlo |
| --- | --- | --- |
| **1. Asistido** | tú, en el chat (este repo: `/trader-risk` + `/trader-broker`) y luego en IBKR | semanas 1 a 4 y siempre que cambies de estrategia |
| **2. Semi-auto** | tú, con un botón "Aprobar / Rechazar" en Telegram; el puente hace el resto | cuando el paper del nivel 1 sea consistente |
| **3. Automático** | el puente, sin intervención, dentro de límites duros | solo tras cumplir los criterios de la fase 6 |

El conector oficial de IBKR para Claude (`create_order_instruction`) **no envía
órdenes al mercado**: crea una instrucción que apruebas en IBKR. Es perfecto
para el nivel 1, pero los niveles 2 y 3 necesitan el puente con la API de IBKR.

---

## 2. Fase 0: cimientos (semana 1)

| tarea | detalle | hecho |
| --- | --- | --- |
| Cuenta IBKR **paper** | Account Settings > Paper Trading Account. Usuario distinto al real | ☐ |
| Datos de mercado | Suscripción a datos de EE. UU. en la cuenta real (la paper los hereda). Sin datos en tiempo real, la API devuelve datos con retraso o error | ☐ |
| Permisos de API | En TWS/Gateway: *Enable ActiveX and Socket Clients*, desmarcar *Read-Only API*, IP de confianza 127.0.0.1 | ☐ |
| TradingView de pago | Los webhooks exigen plan de pago y **2FA activado** en TradingView | ☐ |
| VPS | Linux pequeño (1 vCPU, 1 a 2 GB RAM) cerca de EE. UU. Este. El PC de casa no vale: se duerme, se actualiza, se cae la wifi | ☐ |
| Dominio + HTTPS | TradingView solo envía webhooks a los puertos 80 y 443 | ☐ |
| Horario | Sesión EE. UU.: **15:30 a 22:00 hora de Madrid** (14:30 a 21:00 las semanas de marzo y octubre/noviembre en que los cambios de hora no coinciden) | ☐ |
| Reglas por escrito | Rellenar `trader/config.json` (ver fase 4) **antes** de escribir una línea de código | ☐ |

---

## 3. Fase 1: la estrategia en Pine Script (semanas 1 y 2)

Una estrategia, un mercado, un marco temporal. Recomendación para empezar:
**ruptura del rango de apertura (ORB) de 15 minutos en velas de 5 minutos**,
que es el mismo setup que ya usan `/trader-scanner` y `/trader-entry`, así
puedes contrastar las señales de TradingView con las del repo.

Reglas mínimas que la estrategia tiene que codificar:

1. **Entrada:** primer cierre de 5 min por encima del máximo del rango 9:30 a 9:45 ET.
2. **Stop:** mitad del rango (el `or-mid` de `/trader-exit`).
3. **Objetivo:** 1R o 1,5R (fijo, nunca "ya veré").
4. **Ventana:** sin entradas antes de 9:45 ni después de 15:30 ET.
5. **Salida forzada:** todo cerrado a las 15:55 ET.
6. **Una entrada por símbolo y día.**

Validación antes de pasar a la fase 2:

- Backtest en el *Strategy Tester* con comisiones y deslizamiento realistas
  (configúralos en las propiedades de la estrategia; sin ellos el backtest miente).
- Mínimo **100 operaciones** en la muestra. Menos es ruido.
- Contraste con `/trader-timemachine` sobre los mismos símbolos.
- Métricas que anotas: esperanza por operación, profit factor, % acierto,
  drawdown máximo, peor racha de pérdidas.
- **Criterio para seguir:** esperanza positiva *después* de costes y un
  drawdown que aguantarías con dinero real sin tocar el sistema.

Lo que **no** se hace: optimizar 15 parámetros hasta que la curva sea perfecta.
Eso es sobreajuste y en real se rompe.

---

## 4. Fase 2: la señal (semana 3)

La alerta de TradingView manda un JSON. Se construye con `alert_message` en
`strategy.entry()` / `strategy.exit()` y la alerta se crea con el mensaje
`{{strategy.order.alert_message}}`.

```json
{
  "secret": "<cadena larga aleatoria, la misma que en el .env del puente>",
  "id": "orb-TSLA-2026-11-23-1",
  "strategy": "orb15",
  "symbol": "{{ticker}}",
  "action": "buy",
  "signal_price": "{{close}}",
  "stop": 358.32,
  "target": 370.04,
  "tv_time": "{{timenow}}"
}
```

Reglas del mensaje:

- **El tamaño NO viaja en la alerta.** Lo calcula el puente (máx. pérdida / distancia al stop).
- `id` único por señal: si TradingView reintenta, el puente no compra dos veces.
- TradingView no permite cabeceras propias, por eso el secreto va en el cuerpo.
- Probar cada alerta primero con un receptor de pruebas antes de conectarla al puente.

---

## 5. Fase 3: el puente (semanas 3 a 5)

Un servicio pequeño en Python en el VPS. Piezas:

| pieza | herramienta | función |
| --- | --- | --- |
| Receptor | FastAPI + Uvicorn detrás de Caddy o Nginx (HTTPS) | recibe el webhook, responde en menos de 3 s y encola |
| Filtro | lista de IPs de TradingView (`52.89.214.238`, `34.212.75.30`, `54.218.53.128`, `52.32.178.7`; comprobar en su documentación) + secreto + `id` ya visto | descarta todo lo demás |
| Riesgo | `trader-risk/check.py` de este repo | PASS o BLOCK con el mismo `config.json` que usas a mano |
| Tamaño | `trader-budget/size.py` | acciones = máx. pérdida / (entrada - stop), redondeado hacia abajo |
| Broker | **IB Gateway** + `ib_async` (sucesor de `ib_insync`) | orden **bracket**: entrada + stop + take profit en un solo envío |
| Sesión | **IBC** | mantiene IB Gateway logado y lo reinicia cada día |
| Avisos | bot de Telegram | señal recibida, BLOCK y motivo, fill, stop, objetivo, errores |
| Diario | SQLite | cada señal, decisión, orden, fill y deslizamiento |

Puertos de IB Gateway: **4002 paper**, 4001 real (TWS: 7497 paper, 7496 real).
El puente arranca **bloqueado en 4002**; pasar a 4001 exige cambiar el `.env`
a mano y reiniciar. Nunca por código ni por alerta.

Flujo por señal:

1. Llega el webhook → valida IP, secreto, esquema e `id`.
2. Pide a IBKR precio actual, posición y efectivo (no se fía del precio de la alerta).
3. Si el precio ya se ha ido más de X % de `signal_price` → descarta (no perseguir).
4. Calcula tamaño → escribe `trader/orders.json` → ejecuta `trader-risk`.
5. BLOCK → Telegram con el motivo, fin.
6. PASS → según el nivel: Telegram con botón (nivel 2) o envío directo (nivel 3).
7. Orden bracket a IBKR. Registra el `orderId`, el fill y el deslizamiento.
8. A las 15:55 ET: cancela los stops/TP pendientes y cierra a mercado lo que quede.

Detalles de IBKR que rompen bots si se ignoran:

- IB Gateway se reinicia a diario; con cuenta real, la autenticación con
  IBKR Mobile (2FA) se pide al menos una vez por semana. El puente debe
  detectar la desconexión y **no operar** hasta reconectar, avisándote.
- Usa `whatIfOrder` en paper para ver comisiones y margen antes de enviar.
- Límite de mensajes de la API: no hagas bucles de consulta agresivos.
- Las órdenes hijas del bracket deben transmitirse juntas (`transmit` solo en la última).

---

## 6. Fase 4: riesgo y kill switch (semana 5, en paralelo)

`trader/config.json` de partida para una cuenta de 5.000 $ (ajústalo a la tuya):

```json
{
  "broker": "ibkr",
  "paper": true,
  "budget": 5000,
  "max_loss_per_trade_usd": 50,
  "max_daily_loss_usd": 150,
  "max_trades_per_day": 3,
  "min_reward_risk": 1.0,
  "require_stop": true,
  "no_entries_before": "09:45",
  "no_entries_after": "15:30",
  "flat_by": "15:55",
  "allow_short": false,
  "allow_margin": false,
  "allow_options": false
}
```

Interruptores de emergencia (todos obligatorios antes del nivel 3):

| disparador | acción |
| --- | --- |
| pérdida diaria ≥ `max_daily_loss_usd` | no más entradas hoy |
| 3 pérdidas seguidas | no más entradas hoy |
| desconexión de IB Gateway o del feed | pausa + aviso; no reabre sola |
| deslizamiento medio > 2x lo previsto | pausa + aviso |
| orden sin stop confirmado en el broker en 10 s | cierra la posición |
| comando `/stop` en Telegram | cancela todo, cierra todo, apaga entradas |
| 15:55 ET | flat obligatorio |

Regla personal: **los límites solo se cambian fuera del horario de mercado**
y con 24 h de espera.

---

## 7. Fase 5: paper trading en vivo (semanas 5 a 8)

- Nivel 1 en semanas 5 a 6, nivel 2 en semanas 7 a 8, todo en paper.
- Cada día, 10 minutos de revisión tras el cierre (22:15 Madrid):
  señales vs. órdenes, motivos de BLOCK, deslizamiento, incidencias.
- Cada viernes: comparar los resultados en paper con el backtest. Si se
  desvían mucho, el problema es la estrategia o la ejecución, no la suerte.

Métricas del diario:

| métrica | por qué |
| --- | --- |
| esperanza por operación (en R y en $) | ¿gana el sistema después de costes? |
| profit factor | ganancias brutas / pérdidas brutas |
| drawdown máximo | lo que tendrás que aguantar |
| deslizamiento medio | diferencia real entre TradingView y el fill |
| latencia señal → orden | si pasa de unos segundos, revisar el puente |
| % señales bloqueadas y por qué | detecta reglas mal puestas |

---

## 8. Fase 6: pasar a real (solo si se cumple TODO)

- ☐ Al menos **20 sesiones** y **50 operaciones** en paper con el puente.
- ☐ Esperanza positiva después de comisiones y deslizamiento.
- ☐ Resultados en paper compatibles con el backtest.
- ☐ Cero incidencias graves (órdenes duplicadas, posiciones sin stop) en las últimas 2 semanas.
- ☐ Kill switch probado a propósito al menos una vez.
- ☐ Normativa revisada (ver abajo).

Arranque en real: **25 % del tamaño** durante 2 semanas en nivel 2, luego
50 %, luego 100 %. Nivel 3 solo después de un mes estable en nivel 2. Si en
real se toca el drawdown máximo del backtest, se vuelve a paper.

---

## 9. Normativa e impuestos (residente en España)

- **Regla PDT:** IBKR puede aplicar la regla de *pattern day trader* a cuentas
  de margen que operan valores de EE. UU. Consulta con IBKR cómo te aplica
  según tu tipo de cuenta antes de operar más de 3 day trades en 5 días.
- **IRPF:** las ganancias y pérdidas van a la base del ahorro. Ojo con la
  **regla de los dos meses** (pérdidas por recomprar el mismo valor en ese
  plazo no se computan aún), muy fácil de activar haciendo day trading.
- **Modelo 720/721:** posible obligación informativa si los saldos en el
  extranjero superan los umbrales.
- Habla con un asesor fiscal antes de empezar en real. Esto no es asesoramiento fiscal.

---

## 10. Calendario resumen

| semana | entrega |
| --- | --- |
| 1 | Fase 0 completa + reglas en `config.json` |
| 2 | Estrategia ORB en Pine con backtest de 100+ operaciones |
| 3 | Alertas JSON probadas + receptor webhook con filtros |
| 4 | Puente conectado a IB Gateway paper, órdenes bracket OK |
| 5 | Kill switch + Telegram + diario; inicio paper nivel 1 |
| 6 | Paper nivel 1 |
| 7 a 8 | Paper nivel 2 (aprobación por Telegram) |
| 9+ | Revisión de criterios de la fase 6; real al 25 % si se cumplen |

## 11. Costes orientativos

| concepto | rango |
| --- | --- |
| TradingView (plan con webhooks) | plan de pago, mensual o anual |
| Datos de mercado IBKR | unos dólares al mes por suscripción; consulta las tarifas vigentes |
| VPS | 5 a 15 € / mes |
| Dominio | unos 10 € / año |
| Comisiones IBKR | según plan *Fixed* o *Tiered*; inclúyelas en el backtest |
