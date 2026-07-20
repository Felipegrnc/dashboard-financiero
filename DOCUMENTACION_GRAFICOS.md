# Documentación de Gráficos — Dashboard Financiero

## Librería Utilizada

**Lightweight Charts** (por TradingView)
- CDN: `https://unpkg.com/lightweight-charts/dist/lightweight-charts.standalone.production.js`
- Se consume como variable global `LightweightCharts` desde la que se desestructuran: `createChart`, `CandlestickSeries`, `AreaSeries`, `BaselineSeries`, `LineSeries`, `HistogramSeries`, `LineStyle`, `LineType`.

---

## Resumen de Gráficos Construidos

El dashboard renderiza **3 gráficos sincronizados** verticalmente:

| # | Nombre | Tipo Principal | Datos |
|---|--------|---------------|-------|
| 1 | Balance Diario | Candlestick + Histograma de volumen + Línea MA(20) | `fullWaterfallData` |
| 2 | Flujo de Caja | BaselineSeries (4 series: Ingresos, Gastos, Ahorro, Balance) | `fullCashflowData` |
| 3 | Deuda y Pagos | AreaSeries (2 series: Deuda Acumulada, Pagos) | `fullCreditData` |

Los tres gráficos comparten el **mismo eje temporal** y están sincronizados bidireccionalmente mediante `subscribeVisibleLogicalRangeChange`.

---

## 1. Gráfico Principal — Balance Diario (Candlestick)

### Datos de entrada (API: `/api/financial-analysis/balance-waterfall`)

Cada registro diario contiene:
```json
{ "time": "2025-01-15", "open": float, "high": float, "low": float, "close": float, "volume": float }
```

### Cálculo del Waterfall en el Backend (`process_transactions_to_waterfall`)

1. Se obtienen TODAS las transacciones de Supabase ordenadas por fecha.
2. Se **excluyen** transacciones de tipo `consignación` y con concepto `Abono a Tarjeta` (son movimientos de crédito, no afectan el flujo operativo).
3. Se genera un rango diario continuo desde la primera hasta la última fecha.
4. Para cada transacción se calcula `net_change`: positivo si `type == 'ingreso'`, negativo si es otro tipo.
5. Se agrupa por día y se calcula el balance acumulado (`cumsum`).
6. La vela del día: `open = cierre del día anterior`, `close = balance acumulado del día`, `high = max(open, close)`, `low = min(open, close)`, `volume = suma absoluta de montos del día`.

### Transformación en Frontend (antes de renderizar)

```javascript
const initialBalance = 4298017.67;
```

- Se crea una **barra artificial** el día anterior al primer dato real:
  ```javascript
  { time: previousDayString, open: 0, high: initialBalance, low: 0, close: initialBalance, volume: 0 }
  ```
- El resto de datos se desplaza sumando `initialBalance` a open, high, low y close de cada barra.
- **El gráfico NO arranca desde 0**: arranca desde `4,298,017.67 COP` como saldo inicial.

### Series renderizadas

| Serie | Tipo | Propósito |
|-------|------|-----------|
| Candlestick principal | `CandlestickSeries` | OHLC del balance diario |
| MA(20) | `LineSeries` | Media móvil híbrida de 20 periodos sobre `close` |
| Volumen | `HistogramSeries` | Monto total transado por día (panel inferior, 25% del alto) |
| Líneas verticales (markers) | `CandlestickSeries` transparente | Líneas grises semitransparentes los días con transacciones |

### Configuración de series

**Candlestick:**
- `upColor: 'rgba(24, 177, 29, 0.8)'` (verde, día positivo: close > open)
- `downColor: 'rgba(233, 65, 70, 0.9)'` (rojo, día negativo: close < open)
- `wickUpColor: 'rgb(76, 175, 80)'`, `wickDownColor: 'rgb(239, 83, 80)'`
- `borderVisible: false`

**MA(20):**
- `color: 'rgb(56, 97, 246)'` (azul)
- `lineWidth: 2`, `lineType: LineType.Curved`
- Cálculo: media simple con warmup progresivo (si `i < period`, promedia los datos disponibles hasta ese punto).

**Volumen:**
- `priceScaleId: 'volume_scale'` (escala independiente)
- `scaleMargins: { top: 0.75, bottom: 0 }` → ocupa el 25% inferior del chart
- Color: verde si `close >= open`, rojo si `close < open` (con opacidad ~0.4)

---

## 2. Gráfico Indicador — Flujo de Caja (Baseline)

### Datos de entrada (API: `/api/financial-analysis/cashflow-summary`)

```json
{ "time": "2025-01-15", "ingresos": float, "gastos_totales": float, "ahorro_acumulado": float }
```

### Cálculo en el Backend (`process_transactions_to_cashflow`)

1. Se excluyen transacciones `consignación` y `Abono a Tarjeta`.
2. `ingresos` = suma de montos donde `type == 'ingreso'` ese día.
3. `gastos` = suma de montos donde `type == 'gasto'` ese día.
4. `ahorros` = suma de montos donde `type == 'ahorro'` ese día.
5. `gastos_totales = -(gastos + ahorros)` → siempre negativo (representa salida de dinero).
6. `ahorro_acumulado` = suma acumulada histórica de ahorros con forward-fill.
7. Se rellenan los días sin transacciones con `0` (ingresos/gastos) pero `ahorro_acumulado` mantiene último valor conocido.

### Interpretación de categorías

| Tipo de transacción | Efecto visual |
|---------------------|---------------|
| `ingreso` | Valor positivo en serie verde (por encima de línea base 0) |
| `gasto` | Valor negativo en serie roja (por debajo de línea base 0) |
| `ahorro` | Se cuenta como parte del gasto total (salida) Y se acumula en serie azul celeste separada |
| `consignación` | **Excluida** de este gráfico (va al gráfico 3: crédito) |

### Diferencia Débito vs Crédito

- **Débito (gasto normal):** Cualquier gasto con `payment_method != 'crédito'`. Aparece en este gráfico como gasto (rojo, negativo).
- **Crédito:** Gastos con `payment_method == 'crédito'` aparecen en este gráfico como gasto normal (afectan flujo de caja), PERO TAMBIÉN generan deuda en el Gráfico 3.
- **Consignación:** Es un pago de deuda. Se **excluye** del Gráfico 2 pero aparece en el Gráfico 3 como pago que reduce la deuda. No afecta el flujo de caja operativo.

### Vista "Transacciones" vs "Tendencia"

El gráfico tiene un switch de vistas:

**Vista Transacciones (por defecto):**
- Muestra TODOS los días (incluidos los que tienen $0).
- `incomeData = fullCashflowData.map(d => ({ time: d.time, value: d.ingresos }))`
- `expenseData = fullCashflowData.map(d => ({ time: d.time, value: -Math.abs(d.gastos_totales) }))`
- Balance = `calculateDailyRunningBalance(initialBalance)` → acumula día a día sin excepción.

**Vista Tendencia:**
- Filtra y muestra SOLO los días con actividad (`ingresos !== 0 || gastos_totales !== 0`).
- Balance = `calculateFilteredRunningBalance(initialBalance)` → solo grafica puntos donde hubo movimiento.
- Útil para ver la tendencia sin el "ruido" de los días vacíos.

### Cálculo del Balance (running balance)

```javascript
const initialBalance = 4298017.67;
let currentBalance = initialBalance;
for (const dataPoint of cashflowData) {
    const netFlow = dataPoint.ingresos - Math.abs(dataPoint.gastos_totales);
    currentBalance += netFlow;
    // Se agrega { time, value: currentBalance }
}
```

**El balance arranca desde `4,298,017.67 COP`**, no desde 0.

### Series renderizadas

| Serie | Color | BaseValue | Relleno |
|-------|-------|-----------|---------|
| Ingresos | Verde `rgba(76, 175, 80, 1)` | `{ type: 'price', price: 0 }` | Degradado verde hacia abajo (0.4 → 0) |
| Gastos | Rojo `rgba(239, 83, 80, 1)` | `{ type: 'price', price: 0 }` | Sin relleno (transparent) |
| Balance | Púrpura `rgba(94, 114, 228, 1)` | `{ type: 'price', price: 0 }` | Degradado azulado sutil (0.2 → 0) |
| Ahorro Acum. | Azul `rgba(33, 150, 243, 1)` | `{ type: 'price', price: 0 }` | Degradado azul (0.4 → 0) |

Todas usan `lineWidth: 2`, `lineType: LineType.Curved`.

### Filtros (pills)

El usuario puede mostrar/ocultar cada serie mediante botones pill:
- Ingresos, Gastos, Ahorro, Balance (todos activos por defecto).
- Toggle con `series.applyOptions({ visible: !isVisible })`.

---

## 3. Gráfico Indicador 2 — Deuda y Pagos (Area) — DOCUMENTACIÓN DETALLADA

---

### 3.1 Endpoint y Estructura de Datos

**API:** `GET /api/financial-analysis/credit-summary`

**Respuesta:** Array de objetos, uno por cada día del rango completo de transacciones:

```json
[
  { "time": "2025-01-01", "deuda_acumulada": 0, "pago_diario": 0 },
  { "time": "2025-01-02", "deuda_acumulada": 150000, "pago_diario": 0 },
  { "time": "2025-01-03", "deuda_acumulada": 150000, "pago_diario": 0 },
  { "time": "2025-01-04", "deuda_acumulada": 350000, "pago_diario": 0 },
  { "time": "2025-01-05", "deuda_acumulada": 250000, "pago_diario": 100000 },
  ...
]
```

**Clave para que las líneas coincidan en el eje X:** El backend genera UN registro por CADA DÍA del rango (desde la fecha más antigua de TODAS las transacciones hasta la más reciente). **Ambos campos (`deuda_acumulada` y `pago_diario`) comparten exactamente las mismas fechas.** Esto es lo que permite que las dos series se alineen perfectamente en el gráfico.

---

### 3.2 Cálculo Backend Detallado (`process_transactions_to_credit`)

```python
def process_transactions_to_credit(transactions: List[Dict]) -> List[Dict]:
    import pandas as pd
    if not transactions: return []
    
    df_all = pd.DataFrame(transactions)
    df_all['date'] = pd.to_datetime(df_all['date'])
    if df_all.empty: return []
    
    # 1. RANGO MAESTRO: Genera TODOS los días desde la primera hasta la última transacción.
    #    Esto incluye fines de semana y días sin actividad.
    master_range = pd.date_range(start=df_all['date'].min(), end=df_all['date'].max(), freq='D')
    
    # 2. FILTRADO: Solo nos interesan dos tipos de transacción:
    #    - Gastos a crédito (generan deuda)
    #    - Consignaciones (pagan deuda)
    credit_gastos = (df_all['type'] == 'gasto') & (df_all['payment_method'] == 'crédito')
    consignaciones = df_all['type'] == 'consignación'
    df_credit = df_all[credit_gastos | consignaciones].copy()
    
    # Si no hay transacciones de crédito, devuelve todo en 0 pero CON TODOS LOS DÍAS
    if df_credit.empty:
        return [{'time': d.strftime('%Y-%m-%d'), 'deuda_acumulada': 0, 'pago_diario': 0} for d in master_range]
    
    df_credit['amount'] = pd.to_numeric(df_credit['amount'])
    
    # 3. CÁLCULOS POR TRANSACCIÓN:
    #    - debt_change: POSITIVO si es gasto (aumenta deuda), NEGATIVO si es consignación (reduce deuda)
    #    - pago_diario: Solo el monto de las consignaciones (siempre >= 0)
    df_credit['debt_change'] = df_credit.apply(
        lambda row: row['amount'] if row['type'] == 'gasto' else -row['amount'], axis=1
    )
    df_credit['pago_diario'] = df_credit.apply(
        lambda row: row['amount'] if row['type'] == 'consignación' else 0, axis=1
    )
    
    # 4. AGRUPACIÓN DIARIA: Suma debt_change y pago_diario por día
    daily_summary_grouped = df_credit.groupby('date').agg(
        debt_change=('debt_change', 'sum'),
        pago_diario=('pago_diario', 'sum')
    )
    
    # 5. ACUMULADO DE DEUDA: Suma acumulada del debt_change
    daily_summary_grouped['deuda_acumulada'] = daily_summary_grouped['debt_change'].cumsum()
    
    # 6. REINDEX AL RANGO MAESTRO: Expande al rango completo de días
    daily_summary = daily_summary_grouped.reindex(master_range)
    
    # 7. FORWARD-FILL para deuda_acumulada: Los días sin actividad MANTIENEN la deuda del día anterior.
    #    Esto es lo que hace que la línea roja sea CONTINUA y no caiga a 0.
    daily_summary['deuda_acumulada'].ffill(inplace=True)
    
    # 8. FILLNA(0): Los días sin actividad tienen pago_diario = 0 (no NaN).
    #    La deuda ya fue forward-filled, así que solo afecta a pago_diario.
    daily_summary.fillna(0, inplace=True)
    
    # 9. RESULTADO: Retorna solo deuda_acumulada y pago_diario con el time formateado
    result_df = daily_summary[['deuda_acumulada', 'pago_diario']].reset_index()
    result_df.rename(columns={'index': 'time'}, inplace=True)
    result_df['time'] = result_df['time'].dt.strftime('%Y-%m-%d')
    return result_df.to_dict('records')
```

### Puntos CRÍTICOS del backend:

| Aspecto | Comportamiento | Por qué importa |
|---------|---------------|-----------------|
| `master_range` se calcula sobre `df_all` (TODAS las transacciones) | El rango NO depende solo de las transacciones de crédito | Ambas series cubren TODO el periodo de tiempo, no solo los días con crédito |
| `ffill` en `deuda_acumulada` | La deuda se mantiene hasta que algo la cambie | La línea roja es continua y estable entre pagos/compras |
| `fillna(0)` en `pago_diario` | Los días sin pago valen 0, no NaN | La línea verde baja a 0 entre pagos |
| Ambos campos siempre tienen la misma longitud | Mismo número de registros, mismas fechas | Las series se alinean perfectamente en el eje X |

---

### 3.3 Transformación en Frontend (antes de renderizar)

```javascript
// En fetchAllDataAndInitialize():
const firstDate = new Date(originalWaterfallData[0].time);
firstDate.setDate(firstDate.getDate() - 1);
const previousDayString = firstDate.toISOString().split('T')[0];

// Se añade UN punto "fantasma" al inicio con valores en 0
const creditPadding = { time: previousDayString, deuda_acumulada: 0, pago_diario: 0 };
fullCreditData = [creditPadding, ...originalCreditData];
```

**¿Por qué?** Para alinear la longitud con los otros 2 gráficos (que también tienen un punto artificial al inicio). Esto asegura la sincronización temporal perfecta entre los 3 charts.

**Resultado:** `fullCreditData` tiene N+1 registros, donde el primero es artificial con ambos valores en 0.

---

### 3.4 Creación del Chart (paso a paso)

```javascript
// 1. CREAR INSTANCIA DEL CHART
chartIndicator2 = createChart(indicatorChartContainer2, {
    ...polishedChartOptions,
    watermark: {
        visible: true,
        color: 'rgba(0, 0, 0, 0.04)',
        text: 'DEUDA Y PAGOS',
        fontSize: 48,
        horzAlign: 'center',
        vertAlign: 'center'
    }
});

// 2. APLICAR OPCIONES GLOBALES
chartIndicator2.applyOptions({
    localization: { priceFormatter: priceAxisFormatter }
});
chartIndicator2.priceScale('right').applyOptions({ minimumWidth: 32 });
chartIndicator2.timeScale().applyOptions({ fixLeftEdge: true, fixRightEdge: true });
```

---

### 3.5 Líneas Verticales (Markers) — Serie auxiliar

```javascript
// Determina qué días tienen actividad crediticia
const creditTransactionDaysData = fullCreditData.map((d, i, arr) => {
    // ¿Cambió la deuda respecto al día anterior?
    const debtChanged = i > 0 
        ? d.deuda_acumulada !== arr[i - 1].deuda_acumulada 
        : d.deuda_acumulada !== 0;
    // ¿Hubo un pago ese día?
    const hasPayment = d.pago_diario > 0;
    
    // Solo genera un marker si hubo actividad
    return (debtChanged || hasPayment) 
        ? { time: d.time, open: 100, high: 100, low: 0, close: 100 } 
        : null;
}).filter(Boolean);  // Elimina los null

// Escala oculta para los markers
chartIndicator2.priceScale('markers_scale_credit').applyOptions({
    visible: false,
    scaleMargins: { top: 0.1, bottom: 0.1 }
});

// Serie de candlestick TRANSPARENTE — solo se ven los wicks como líneas verticales
const markerSeriesCredit = chartIndicator2.addSeries(CandlestickSeries, {
    priceScaleId: 'markers_scale_credit',
    upColor: 'transparent',
    downColor: 'transparent',
    borderVisible: false,
    wickUpColor: 'rgba(180, 180, 180, 0.3)',   // Gris semitransparente
    wickDownColor: 'rgba(180, 180, 180, 0.3)',
    crosshairMarkerVisible: false,
    lastValueVisible: false,
    priceLineVisible: false,
});
markerSeriesCredit.setData(creditTransactionDaysData);
```

**Efecto visual:** Líneas verticales grises muy tenues solo en los días donde hubo un cambio de deuda o un pago. Sirven como guía visual para ubicar los eventos.

---

### 3.6 Serie Principal: Deuda Acumulada (Roja)

```javascript
const debtSeries = chartIndicator2.addSeries(AreaSeries, {
    // COLOR DE LA LÍNEA
    lineColor: 'rgba(239, 83, 80, 1)',      // Rojo sólido
    
    // RELLENO: Degradado de arriba a abajo
    topColor: 'rgba(239, 83, 80, 0.4)',     // Rojo con 40% opacidad (pegado a la línea)
    bottomColor: 'rgba(239, 83, 80, 0)',    // Transparente (en el borde inferior)
    
    // ESTILO DE LÍNEA
    lineWidth: 2,
    lineType: LineType.Curved,              // Curvas suavizadas entre puntos
    
    // CROSSHAIR
    crosshairMarkerVisible: true,           // Muestra punto al pasar el mouse
});

// DATOS: Mapeamos TODOS los registros de fullCreditData
debtSeries.setData(fullCreditData.map(d => ({
    time: d.time,
    value: d.deuda_acumulada
})));
```

**Comportamiento de la línea roja:**
- Sube cuando hay gastos a crédito.
- Baja cuando hay consignaciones (pagos).
- Se mantiene PLANA entre eventos gracias al `ffill` del backend.
- Cubre TODO el rango temporal (desde el día artificial hasta la última fecha de transacciones).

---

### 3.7 Serie Secundaria: Pagos Diarios (Verde)

```javascript
const paymentSeries = chartIndicator2.addSeries(AreaSeries, {
    // COLOR DE LA LÍNEA
    lineColor: 'rgba(76, 175, 80, 1)',      // Verde sólido
    
    // RELLENO
    topColor: 'rgba(76, 175, 80, 0.5)',     // Verde con 50% opacidad (arriba)
    bottomColor: 'rgba(76, 175, 80, 0)',    // Transparente (abajo) ← IMPORTANTE
    
    // ESTILO DE LÍNEA
    lineWidth: 2,
    lineType: LineType.Curved,
    
    // CROSSHAIR Y LABELS
    crosshairMarkerVisible: true,
    priceLineVisible: false,                // No muestra línea de precio a la derecha
    lastValueVisible: false,                // No muestra label del último valor
});

// DATOS: Mapeamos TODOS los registros
paymentSeries.setData(fullCreditData.map(d => ({
    time: d.time,
    value: d.pago_diario
})));
```

**Comportamiento de la línea verde:**
- Vale 0 la mayoría de los días (la línea va pegada al eje X).
- Salta a un valor positivo SOLO los días que hubo un pago (consignación).
- Vuelve inmediatamente a 0 el día siguiente.
- Crea "picos" puntuales.

---

### 3.8 POR QUÉ LAS LÍNEAS COINCIDEN EN ESCALA Y POSICIÓN

**Problema que describes:** "Las líneas roja y verde no coinciden en sus alturas."

**Razón por la que coinciden en el original:**

1. **AMBAS SERIES COMPARTEN LA MISMA `priceScale` (`'right'`)**. No tienen `priceScaleId` definido explícitamente, por lo que ambas usan la escala de precios derecha por defecto. Esto significa que un valor de `500,000` en la serie roja se dibuja a la MISMA altura que un valor de `500,000` en la serie verde.

2. **NO hay `scaleMargins` diferentes entre las dos series.** Ambas se renderizan en el mismo espacio vertical con las mismas márgenes. La escala se auto-ajusta al rango total de datos (el max entre `deuda_acumulada` y `pago_diario`).

3. **Los datos se alimentan con el MISMO array de fechas.** Ambas series reciben `fullCreditData.map(...)`, por lo que tienen exactamente las mismas coordenadas X (time). No hay desalineamiento temporal.

**Si en tu réplica las alturas no coinciden, verifica:**
- ¿Estás poniendo las dos series en la misma `priceScaleId`? Si una tiene `priceScaleId: 'left'` y otra `'right'`, tendrán escalas independientes.
- ¿Los datos de ambas series tienen exactamente las mismas fechas? Si una tiene más/menos registros, el eje X se desalinea.
- ¿Estás usando `priceScale` con márgenes diferentes para alguna? No debes.

---

### 3.9 POR QUÉ EL GRÁFICO CONTINÚA MÁS ALLÁ DE LA ÚLTIMA FECHA

**Problema que describes:** "Se me queda pegado a la última fecha del registro."

**Razón por la que en el original el chart continúa sin problema:**

1. **`fixRightEdge: true`** — Esto NO limita el gráfico. Solo evita que el usuario pueda scrollear infinitamente hacia la derecha. Pero los datos se siguen mostrando correctamente.

2. **El rango temporal se define por el `master_range` del backend** que va desde la fecha mínima hasta la fecha MÁXIMA de TODAS las transacciones (no solo las de crédito). Si el gráfico 1 tiene datos hasta el 15 de julio y el crédito solo tiene datos hasta el 10, el `master_range` todavía cubre hasta el 15 porque se calcula sobre `df_all` (todas las transacciones).

3. **La sincronización temporal fuerza el mismo rango visible.** Cuando el gráfico 1 (candlestick) tiene datos hasta una fecha, y haces zoom o scroll, el gráfico 3 muestra ese mismo rango temporal gracias a:
   ```javascript
   chartMain.timeScale().subscribeVisibleLogicalRangeChange(logicalRange => {
       chartIndicator2.timeScale().setVisibleLogicalRange(logicalRange);
   });
   ```

4. **El padding inicial (día artificial)** asegura que los 3 gráficos empiezan en la misma fecha.

5. **Espacio "vacío" a la derecha:** Lightweight Charts naturalmente muestra espacio vacío después del último dato si el timeScale sincronizado lo requiere. No se "pega" al último punto.

**Si en tu réplica se queda pegado al último dato, verifica:**
- ¿Estás sincronizando el timeScale con los otros gráficos? Si no lo estás haciendo, el chart 3 se ajustará solo a SUS datos.
- ¿El `master_range` de tu backend usa `df_all['date'].min()` y `df_all['date'].max()` (de TODAS las transacciones), no solo las de crédito?
- ¿Tienes `fixRightEdge: true` + `fixLeftEdge: true`?
- ¿Estás llamando `fitContent()` o `applyZoomByInterval('All')` después de setear los datos? Esto fuerza al chart a mostrar TODO el rango.
- ¿Los tres datasets (`fullWaterfallData`, `fullCashflowData`, `fullCreditData`) tienen exactamente la misma primera y última fecha?

---

### 3.10 Ejemplo Completo de Datos Esperados

Para que el gráfico funcione correctamente, `fullCreditData` debe verse así:

```json
[
  {"time": "2025-01-14", "deuda_acumulada": 0, "pago_diario": 0},        // ← Padding artificial
  {"time": "2025-01-15", "deuda_acumulada": 0, "pago_diario": 0},        // ← Primer día real, sin crédito
  {"time": "2025-01-16", "deuda_acumulada": 0, "pago_diario": 0},
  {"time": "2025-01-17", "deuda_acumulada": 200000, "pago_diario": 0},   // ← Compra a crédito
  {"time": "2025-01-18", "deuda_acumulada": 200000, "pago_diario": 0},   // ← ffill: se mantiene
  {"time": "2025-01-19", "deuda_acumulada": 200000, "pago_diario": 0},   // ← ffill
  {"time": "2025-01-20", "deuda_acumulada": 450000, "pago_diario": 0},   // ← Otra compra
  {"time": "2025-01-21", "deuda_acumulada": 450000, "pago_diario": 0},   // ← ffill
  {"time": "2025-01-22", "deuda_acumulada": 350000, "pago_diario": 100000}, // ← Pago parcial
  {"time": "2025-01-23", "deuda_acumulada": 350000, "pago_diario": 0},   // ← ffill, pago vuelve a 0
  ...
  {"time": "2025-07-15", "deuda_acumulada": 120000, "pago_diario": 0}    // ← Última fecha = última transacción global
]
```

**Observa:**
- Hay un registro para CADA día del rango, sin saltos.
- `deuda_acumulada` se mantiene constante entre eventos (forward-fill).
- `pago_diario` solo es > 0 los días con consignación; el resto es 0.
- La última fecha coincide con la última fecha de los otros gráficos.

---

### 3.11 Escala de Precios Compartida

Las dos series (`debtSeries` y `paymentSeries`) NO declaran `priceScaleId`, por lo que ambas usan `'right'` por defecto. Esto es crucial:

```
┌─────────────────────────────────────────┐
│                                         │  ← deuda_acumulada: 500,000
│   ─────── (línea roja estable)          │
│                                         │  ← 400,000
│              ╱╲                          │
│             ╱  ╲ (pico verde: pago)      │  ← 300,000
│            ╱    ╲                        │
│           ╱      ╲                       │  ← 200,000
│          ╱        ╲                      │
│─────────╱──────────╲────────── (verde)   │  ← 100,000
│                                         │
│─────────────────────────────────────────│  ← 0
└─────────────────────────────────────────┘
```

Ambas escalas se miden con el MISMO eje Y. Si la deuda es de $500,000 y un pago es de $100,000, el pago se ve como un pico pequeño relativo a la deuda. Esto es correcto y esperado.

---

### 3.12 Leyenda Dinámica (Crosshair)

```javascript
function updateCreditLegend(param) {
    if (!param || !param.time || !param.seriesData.size) {
        creditLegend.innerHTML = `<div class="legend-title">Resumen de Crédito</div>`;
        return;
    }
    const debtData = param.seriesData.get(debtSeries);
    const paymentData = param.seriesData.get(paymentSeries);

    const deuda = debtData ? currencyFormatter(debtData.value) : 'N/A';
    const pago = paymentData && paymentData.value > 0 ? currencyFormatter(paymentData.value) : 'N/A';

    // Solo muestra "Pago del Día" si hubo un pago ese día (value > 0)
    creditLegend.innerHTML = `
        <div class="legend-title">Resumen de Crédito</div>
        <div class="legend-item-container">
            <div class="legend-item">
                <span class="legend-label">Deuda Acumulada:</span>
                <span class="legend-value" style="color: rgb(239, 83, 80);">${deuda}</span>
            </div>
            ${pago !== 'N/A' ? `
            <div class="legend-item">
                <span class="legend-label">Pago del Día:</span>
                <span class="legend-value" style="color: rgb(76, 175, 80);">${pago}</span>
            </div>` : ''}
        </div>`;
}
chartIndicator2.subscribeCrosshairMove(updateCreditLegend);
updateCreditLegend(null);  // Estado inicial sin crosshair
```

---

### 3.13 Watermark

```javascript
watermark: {
    visible: true,
    color: 'rgba(0, 0, 0, 0.04)',  // Casi invisible, solo contexto visual
    text: 'DEUDA Y PAGOS',
    fontSize: 48,
    horzAlign: 'center',
    vertAlign: 'center'
}
```

---

### 3.14 Checklist para Replicar Correctamente

| # | Verificación | Detalle |
|---|-------------|---------|
| 1 | ¿El backend genera un registro por CADA día del rango? | Usa `pd.date_range(min, max, freq='D')` + `reindex` |
| 2 | ¿El rango se calcula sobre TODAS las transacciones? | `master_range` usa `df_all['date'].min/max()`, no `df_credit` |
| 3 | ¿`deuda_acumulada` usa forward-fill? | `.ffill()` después del `reindex` |
| 4 | ¿`pago_diario` se llena con 0 donde no hay datos? | `.fillna(0)` |
| 5 | ¿Se añade el punto padding al inicio? | `{ time: previousDayString, deuda_acumulada: 0, pago_diario: 0 }` |
| 6 | ¿Ambas series usan la misma `priceScaleId` (default `'right'`)? | NO declarar `priceScaleId` en ninguna de las dos, o poner `'right'` en ambas |
| 7 | ¿Ambas series reciben datos con las MISMAS fechas? | `fullCreditData.map(d => ...)` para ambas |
| 8 | ¿El timeScale está sincronizado con los otros charts? | `subscribeVisibleLogicalRangeChange` bidireccional |
| 9 | ¿Se usa `fixLeftEdge: true, fixRightEdge: true`? | En la configuración del timeScale |
| 10 | ¿Se llama `fitContent()` o `applyZoomByInterval()` después de cargar datos? | Para que el chart muestre el rango completo |
| 11 | ¿La serie de markers tiene su propia escala OCULTA? | `priceScaleId: 'markers_scale_credit'` con `visible: false` |
| 12 | ¿`lineType: LineType.Curved`? | Suaviza las transiciones entre puntos |

---

## Variables de Inicialización

| Variable | Valor | Uso |
|----------|-------|-----|
| `initialBalance` | `4298017.67` COP | Saldo inicial desde el que arranca el Gráfico 1 (candlestick) y el cálculo del balance en el Gráfico 2 |

**Los gráficos NO arrancan desde 0.** El saldo inicial representa el patrimonio disponible al momento de comenzar a registrar transacciones. Todos los cálculos acumulativos parten de este valor.

Para el Gráfico 3 (crédito), la deuda sí arranca desde 0 porque el historial de deuda se construye progresivamente desde las transacciones.

---

## Sistema de Períodos (Zoom Temporal)

### Intervalos disponibles

| Botón | Barras visibles | Cálculo |
|-------|----------------|---------|
| 1M | 30 | `barSpacing = chartWidth / 30` |
| 3M | 90 | `barSpacing = chartWidth / 90` |
| 6M | 180 | `barSpacing = chartWidth / 180` |
| 1Y | 252 | `barSpacing = chartWidth / 252` (días hábiles aprox.) |
| All | Todos | `timeScale.fitContent()` |

### Comportamiento

- **Móvil (≤ 800px):** Arranca en `3M` por defecto.
- **Desktop (> 800px):** Arranca en `All` por defecto.
- La transición de zoom se anima con `easeOutQuad` durante 300ms.
- Después de hacer zoom, se scrollea al último dato: `scrollToPosition(dataLength - 1, false)`.
- Cada gráfico aplica `fixLeftEdge: true, fixRightEdge: true` para anclar bordes.

---

## Sincronización de Gráficos

Los 3 gráficos están vinculados bidireccionalmente:

```javascript
chartMain.timeScale().subscribeVisibleLogicalRangeChange(logicalRange => {
    chartIndicator.timeScale().setVisibleLogicalRange(logicalRange);
    chartIndicator2.timeScale().setVisibleLogicalRange(logicalRange);
});
// (Ídem para chartIndicator → los otros dos, y chartIndicator2 → los otros dos)
```

Cualquier scroll o zoom en uno se replica instantáneamente en los demás.

---

## Líneas Verticales (Markers)

En los Gráficos 2 y 3 se dibujan líneas verticales semitransparentes los días con actividad. Se implementan como **CandlestickSeries transparentes** con wicks visibles:

- `wickUpColor: 'rgba(180, 180, 180, 0.3)'`
- `wickDownColor: 'rgba(180, 180, 180, 0.3)'`
- Los cuerpos son transparentes (`upColor/downColor: 'transparent'`).
- Se colocan en una escala oculta (`visible: false`) con valores fijos `open: 100, high: 100, low: 0, close: 100`.

El Gráfico 1 usa una técnica similar pero con dos series separadas: una para el panel de precio (pane 0) y otra para el panel de volumen (pane 1), cada una con valores escalados a su respectivo rango.

---

## Configuración Global del Chart

```javascript
const polishedChartOptions = {
    layout: {
        background: { color: '#ffffff' },
        textColor: '#1e222d',
        fontSize: getDynamicFontSize(1.15),  // ~11.5px basado en root font-size
        fontFamily: 'Roboto'
    },
    grid: {
        vertLines: { color: '#f0f3fa' },
        horzLines: { color: '#f0f3fa' }
    },
    rightPriceScale: { borderVisible: false },
    timeScale: {
        borderVisible: false,
        timeVisible: false,
        secondsVisible: false
    },
    crosshair: {
        horzLine: {
            visible: true,
            labelVisible: true,
            style: LineStyle.Dashed,
            width: 1,
            color: '#5d606b',
            labelBackgroundColor: '#1e222d'
        }
    },
    handleScale: false  // Desactiva zoom por scroll del usuario
};
```

### Watermarks

| Gráfico | Texto | fontSize | Color |
|---------|-------|----------|-------|
| 1 (Balance) | `'Balance Diario'` | 48 | `rgba(0, 0, 0, 0.04)` |
| 3 (Crédito) | `'DEUDA Y PAGOS'` | 48 | `rgba(0, 0, 0, 0.04)` |

### Eje de Precios

- `priceAxisFormatter`: En pantallas ≤ 800px muestra valores abreviados (`1.5M`). En desktop muestra formato completo con separador de miles.
- `scaleMargins` del Gráfico 1: `{ top: 0.1, bottom: 0.25 }` para dejar espacio al volumen.
- `minimumWidth: 32` en todos los ejes de precio.

### Localización

- Formato moneda: `Intl.NumberFormat('es-CO', { style: 'currency', currency: 'COP' })`.
- Se aplica via `chart.applyOptions({ localization: { priceFormatter: priceAxisFormatter } })`.

---

## Leyendas Dinámicas (Crosshair)

Cada gráfico tiene una leyenda overlay (`.chart-legend`) que se actualiza al mover el crosshair:

**Gráfico 1:** Muestra A (Apertura), M (Máximo), m (Mínimo), C (Cierre), Vol (Volumen en miles), MA(20).

**Gráfico 2:** Muestra Ingresos, Gastos, Ahorro Acum., Saldo Final.

**Gráfico 3:** Muestra Deuda Acumulada y Pago del Día (solo si > 0).

---

## Cálculos Adicionales

### Media Móvil Híbrida (MA 20)

```javascript
function calculateHybridMovingAverage(data, period, sourceKey = 'close') {
    // Para i < period: promedia todos los datos disponibles hasta i (warmup progresivo)
    // Para i >= period: media simple estándar de los últimos 'period' valores
}
```

### EMA (Exponential Moving Average)

- `k = 2 / (period + 1)`
- Se usa internamente para calcular MACD (aunque MACD no está renderizado actualmente como serie visible).

### MACD (calculado pero no graficado visiblemente)

- Fast: 12 periodos, Slow: 26, Signal: 9
- Genera `macdLine`, `signalLine` y `histogram` con colores verde/rojo según divergencia.

---

## Resumen: Flujo Completo de Datos

```
Supabase (tabla 'transactions')
    ↓ SELECT * ORDER BY date ASC
Backend (FastAPI)
    ↓ process_transactions_to_waterfall()  → /api/financial-analysis/balance-waterfall
    ↓ process_transactions_to_cashflow()   → /api/financial-analysis/cashflow-summary
    ↓ process_transactions_to_credit()     → /api/financial-analysis/credit-summary
Frontend (chart.js)
    ↓ fetchAllDataAndInitialize()
    ↓ Añadir barra/punto inicial con initialBalance
    ↓ renderCharts()
    ↓ Crear 3 instancias de LightweightCharts
    ↓ Sincronizar timeScale
    ↓ Aplicar zoom por intervalo
```

---

## Consideraciones Especiales

1. **Padding temporal:** Se añade un día artificial (día anterior al primer dato) en los 3 datasets para alinear las longitudes y que la sincronización funcione correctamente.

2. **Exclusiones:** Las transacciones `consignación` y `Abono a Tarjeta` se excluyen de los Gráficos 1 y 2, pero `consignación` SÍ aparece en el Gráfico 3 como pago de deuda.

3. **Forward-fill:** `deuda_acumulada` y `ahorro_acumulado` mantienen su último valor en días sin movimiento (no vuelven a 0).

4. **handleScale: false:** El zoom por scroll está desactivado; el usuario solo puede cambiar rango con los botones de intervalo.

5. **Responsive:** El `priceAxisFormatter` cambia formato según ancho de pantalla. El intervalo por defecto también cambia (3M en móvil, All en desktop).

6. **ResizeObserver:** Los gráficos se redimensionan automáticamente cuando cambia el tamaño del contenedor.

7. **Recarga suave:** El botón "Recargar" hace `fetchAllDataAndInitialize()` sin recargar la página, manteniendo el estado de navegación.

8. **BaselineSeries con `baseValue: { type: 'price', price: 0 }`:** Esto define el nivel 0 como referencia visual. Los valores positivos se rellenan hacia arriba con su color, los negativos hacia abajo.
