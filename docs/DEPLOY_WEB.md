# Despliegue web — Next.js + Supabase

La app `SistemaContableMargarita.jsx` ya es funcional. Para producción:

## 1. Lógica de módulos (sin cambios)
Todo el cálculo (comprobación, cierre, persona, desagregación, informes) ya está en
JavaScript dentro del .jsx → pasa tal cual a `lib/contable.js`. Es el mismo motor que
el repo Python, ya validado al céntimo contra los archivos reales.

## 2. El chat → API route (protege tu key)
Mueve la llamada a Claude del navegador a un route del servidor:

```
// app/api/chat/route.ts
import Anthropic from "@anthropic-ai/sdk";
const a = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
export async function POST(req) {
  const { messages } = await req.json();
  const r = await a.messages.create({ model:"claude-sonnet-4-6", max_tokens:1500,
    system: SISTEMA, tools: TOOLS, messages });
  return Response.json(r);
}
```
El bucle de tool_use (ya está en el .jsx) corre en el cliente o en el route.

## 3. Memoria → Supabase (en vez de window.storage)
Aplica `supabase_schema.sql`. Reemplaza `window.storage.get/set` por consultas a
Supabase (`@supabase/supabase-js`). Tablas: tasas_bcv, empresas, recetas,
saldos_cierre, importaciones, generados (mismas que el almacén SQLite del repo Python).

## 4. Tasa BCV automática (server-side)
En el navegador no se puede por CORS; en un route o cron de Next.js sí:
```
const r = await fetch("https://pydolarve.org/api/v1/dollar?page=bcv");
const tasa = (await r.json()).monitors.bcv.price;  // guardar en Supabase
```
Programa un cron diario (Vercel Cron) que capture y guarde la tasa.

## 5. Importación de archivos del papá
Subida a Supabase Storage; un route lee el .xlsx (SheetJS), calcula hash y compara
con `importaciones` para detectar nuevo/modificado (igual que `nucleo/importador.py`).

## Variables de entorno
ANTHROPIC_API_KEY, NEXT_PUBLIC_SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY
