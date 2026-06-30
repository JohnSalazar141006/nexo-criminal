# Sistema Contable Margarita

Aplicación con **cerebro propio**: un agente con chat que entiende instrucciones en
lenguaje natural y ejecuta los módulos, con **memoria propia** (SQLite) que guarda
todo internamente, captura de **tasa BCV**, e **importación con detección de cambios**.
Config-driven: cada empresa es configuración, no código. Escala a las ~50 empresas.

## El cerebro — agente con chat

```bash
pip install anthropic && export ANTHROPIC_API_KEY=...
python -m agente.agente
```

Le hablas natural ("genera la comprobación real de Canaima al 28-02", "captura la
tasa BCV", "revisa si papá actualizó algún archivo", "desagregación de Opus con 12
meses de proyección") y entiende + ejecuta usando sus herramientas. La parte de
ejecución ya está validada de punta a punta; solo la conversación necesita tu API key.

## Núcleo (memoria propia)

- `nucleo/almacen.py` — SQLite: tasas BCV, empresas, recetas, importaciones, saldos de
  cierre (arrastre entre años) y bitácora de generados.
- `nucleo/tasas_bcv.py` — captura la tasa BCV (API pública) y la guarda como variable.
  Hoy 30-06-2026 sembrada en 623,02 Bs/$. `sembrar()` permite carga manual sin red.
- `nucleo/importador.py` — importa los archivos del papá; por hash detecta **nuevo** o
  **modificado** y lo registra, para que el sistema "se entere" de las actualizaciones.

## Módulos

### Desagregaciones  ✅ las 4 empresas
Total mensual → reparto por rubro → $ a la tasa → Excel banco (Detallado + Cierres).
Soporta modo **porcentaje** (Canaima/Opus/Von Road) y **por rubro directo** (Plascan,
unidades en KG). Proyección a 12 meses con `factor_meta` para **sobrepasar la meta**
que le interesa al banco. Validado al céntimo contra los archivos reales.

### Balances  ✅ cuadra
- **Cierre** (`balances/motor.py`): Estado de Resultados + Situación Financiera, fiscal
  Bs y real $. Validado contra Canaima 31-12-2025 (cuadra exacto).
- **Comprobación REAL** (`balances/comprobacion.py`): corte a mitad de año con tu lógica:
  ingresos del archivo, inventario inicial = final del año anterior, compras/inv. final/
  gastos/utilidad por **porcentaje** que tú asignas, ISLR y reserva en 0, superávit =
  resultado del corte + acumulado del 31-12 anterior, y **Anticipo a Proveedores como la
  cuenta que cuadra** (plug). Cuadra por construcción.

### Informes  ✅
Motor de plantillas (mail-merge): nombre/RIF o cédula, banco al que va dirigido, fecha y
trámite (solicitud de crédito, trámites correspondientes, actualización de expediente,
apertura de cuenta). Genera Word con membrete y firma.

## Uso por módulo (sin chat)

```bash
python generar.py "Importadora Opus, C.A."                        # desagregación
python generar_comprobacion.py balances/datos/canaima_comprobacion_28-02-2026_real.yaml
python generar_balance.py balances/datos/canaima_31-12-2025_fiscal.yaml
python generar_informe.py informes/datos/constancia_canaima.yaml
```

## Pendientes (los "después arreglamos")

1. Conectar el chat con tu `ANTHROPIC_API_KEY` (o llevarlo a tu stack Next.js + Supabase).
2. Balance fiscal: igual que el real pero con ventas fiscales del archivo y tus %.
3. Anexos del cierre anual: variación patrimonial, flujo de efectivo, notas.
4. Formato de **persona natural** (propiedades, bienes, certificación de ingresos).
5. Proyección de desagregación: afinar la curva base (las fórmulas viven en las hojas).
6. Integrar con `contabilidad-margarita` (extracción + matching ya existentes).

## Estructura

```
agente/        agente.py  (chat con Claude + herramientas)
nucleo/        almacen.py · tasas_bcv.py · importador.py · memoria.db
desagregaciones/  motor.py · generador.py · config/ · datos/ · salidas/
balances/      motor.py · comprobacion.py · generador.py · datos/ · salidas/
informes/      generador.py · plantillas.yaml · datos/ · salidas/
generar*.py    CLIs por módulo
```
