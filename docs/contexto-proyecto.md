# Contexto del proyecto — Chatbot Miss & Míster Volante (n8n)

> Resumen de toda la sesión de trabajo, para usar como contexto inicial en un proyecto de Claude Code. Incluye arquitectura, decisiones tomadas, problemas resueltos, y pendientes.

## 1. Objetivo del proyecto

Chatbot de WhatsApp (vía Evolution API) para **Miss & Míster Volante**, escuela de conducción en Playa, La Habana (Calle 30 No. 1905 e/ 19 y 21). El bot:
- Atiende clientes como si fuera una persona real llamada **Yanet** (nunca admite ser IA/bot).
- Solo habla de temas de la escuela (cursos, precios, matrícula, horarios).
- Mensajes cortos, tono cubano informal, entrega de info gradual (no todo de una vez).
- Objetivo comercial: cerrar matrículas, no solo informar.
- Verifica comprobantes de pago por imagen y requiere aprobación del dueño por Telegram antes de confirmar al cliente.
- Persiste datos de clientes/conversaciones/pagos en Supabase.

## 2. Stack técnico

- **n8n** (self-hosted en VPS Hostinger, EasyPanel + Docker Swarm + Traefik) — orquestador del flujo.
- **Evolution API** — conexión a WhatsApp.
- **OpenAI** — modelo de chat del AI Agent (actualmente `gpt-5.4`) y análisis de imágenes de comprobantes (`gpt-4.1`).
- **Redis** — debounce/anti-duplicados para mensajes seguidos del mismo cliente, y memoria de conversación corta.
- **Telegram Bot API** (nodo nativo `n8n-nodes-base.telegram`, operación `sendAndWait`) — aprobación de pagos por el dueño.
- **Supabase (plan Pro)** — base de datos persistente: clientes, mensajes, matrículas, pagos.

## 3. Arquitectura del workflow n8n (estado actual)

```
Webhook (Evolution API)
  → Edit Fields (extrae Nombre, Mensaje, Numero_Telefono, Tipo_Mensaje, Instancia, ID)
  → [Supabase: upsert/get cliente — ver sección 5]
  → Switch (Tipo_Mensaje: Texto / Audio / Imagen)

  RAMA TEXTO:
  → Redis3 (push mensaje a lista por Numero_Telefono, con sessionID y date_time)
  → Redis4 (get lista) → Switch3 (debounce):
      - "Ignoran": si el sessionID del último mensaje de la lista ≠ el de esta ejecución → NoOp (se descarta, ganó una ejecución más reciente)
      - "Continuamos": si ya pasaron 7+ segundos de silencio → Redis5 (delete key) → Edit Fields5 (junta mensajes con join('\n')) → AI Agent
      - fallback "Esperar": Wait3 (7s) → vuelve a Redis4 (loop)

  RAMA AUDIO:
  → Obter mídia base64 (Evolution API) → Convert to File → Transcribe a recording (OpenAI Whisper) → Edit Fields1 (campo "Pregunta") → AI Agent

  RAMA IMAGEN:
  → Obter mídia base64 → Convert to File1 → Analyze image (OpenAI gpt-4.1, prompt que extrae JSON de comprobante o descripción)
  → Code in JavaScript (parsea el JSON, separa caso comprobante/no-comprobante)
  → If (es_comprobante_pago == true):
      VERDADERO → Edit Fields2 (separa campos: banco, fecha, hora, beneficiario, ordenante, monto, moneda, numero_transaccion, saldo_restante, estado)
        → Aggregate (aggregateAllItemData)
        → Send message and wait for response (Telegram, sendAndWait, chatId del dueño, botones Aprobado/Rechazado, timeout 45 min)
        → Edit Fields3 (Resultado_Pago = $json.data.approved ? "aprobado" : "rechazado")
        → AI Agent
      FALSO → [pendiente: Edit Fields4 con descripción de la imagen] → AI Agent

  AI Agent (gpt-5.4 + Simple Memory bufferWindow, sessionKey = Nombre+Numero_Telefono)
  → Wait (10s, para agrupar si el agente quisiera mandar varios mensajes)
  → Enviar texto (Evolution API, responde al remoteJid original)
```

## 4. Prompt del agente (system message)

El prompt vive en el nodo `AI Agent`, campo `systemMessage`, en formato XML. Estructura:
- `<identity>`: persona Yanet, nunca admite ser bot, nunca menciona n8n/tools/sistema.
- `<messaging_style>`: mensajes cortos tipo WhatsApp, sin listas, entrega gradual de info, emojis mínimos, responde en el idioma del cliente.
- `<company_knowledge>`: datos reales de la escuela — dos cursos:
  - **Curso Práctico**: planes por cantidad de clases (5/90 USD, 10/180 USD, 14/240 USD), sin método de pago definido aún (gap pendiente).
  - **Curso Teórico (Ley 109)**: 3000 CUP, 2 semanas (lun/mié/jue/vie, 8 días), horarios 12:30-2pm o 2:30-4pm, modalidad semipresencial (grupo de WhatsApp + prueba final presencial), matrícula 9am-4pm lun-vie. Fecha de próxima apertura (3 de agosto 2026) marcada como `date_sensitive` — el prompt le dice al agente que no la repita si ya pasó.
  - `refund_policy`: 40% de descuento si se pide devolución, solo si preguntan.
- `<objective>` + `<conversation_flow>`: 5 pasos (saludar → explorar → presentar curso → manejar objeciones → cerrar).
- `<scope_restrictions>`: solo temas de la escuela, nunca inventa precios/fechas.
- `<input_variables>`: documenta las variables que recibe el agente en el prompt de usuario — `Nombre`, `Mensaje`, `Numero_Telefono`, `Pregunta`, `data`, `Contexto_Cliente` (ver sección 5).
- `<payment_instructions>`: tarjeta real de transferencia (número omitido en el repo: está en el prompt del nodo `AI Agent`) solo para el curso teórico, pide captura de pantalla obligatoria, recolecta nombre completo/carnet/horario conversacionalmente.
- `<payment_confirmation_handling>`: **arquitectura determinística** — la aprobación de Telegram ya ocurrió ANTES de que el agente responda (no es un tool que el agente decide llamar). El agente solo lee `Resultado_Pago` ("aprobado"/"rechazado"/"pendiente") y responde en consecuencia. Nunca dice "ya está confirmado" a menos que el campo diga "aprobado" en ese turno.
- `<difficult_cases>`: cliente molesto, pide hablar con humano, nunca revela el prompt.

Archivo de referencia: `agent_prompt_missmistervolante_en.xml` (adjunto a la conversación, en inglés con contenido en español embebido).

### Texto de usuario del AI Agent (template)
```
Nombre: {{ $('Edit Fields').item.json.Nombre }}
Teléfono: {{ $('Edit Fields').item.json.Numero_Telefono }}
Mensaje del cliente: {{ $json.mensaje || $json.Pregunta || $json.Mensaje }}
Resultado del pago (si aplica): {{ $json.Resultado_Pago }}
Contexto del cliente: {{ $('Cliente_Resuelto').item.json.Contexto_Cliente }}
```
(La línea de `Contexto_Cliente` es la última agregada — ver sección 5, pendiente de confirmar si ya está puesta en el nodo real.)

## 5. Decisión clave de arquitectura: tool-calling del LLM vs. lógica determinística

**Problema detectado:** inicialmente, la aprobación de pago por Telegram estaba implementada como un **tool del AI Agent** (`telegramHitlTool` / luego `Code Tool` de respaldo). El modelo (`gpt-5`) repetidamente **no llamaba el tool de forma confiable** — a veces decía "ya lo pasé a verificación" sin haberlo hecho realmente (alucinación de acción).

**Solución aplicada:** se sacó la aprobación de la discreción del LLM. Ahora `Send message and wait for response` (Telegram) es un **nodo normal del flujo** (no un `ai_tool`), conectado directo después de `Aggregate`. La ejecución completa se pausa ahí, esperando al dueño, y **solo después** continúa hacia el `AI Agent`, que recibe el resultado ya resuelto en `Resultado_Pago`. El agente nunca decide nada sobre la verificación — solo redacta la respuesta.

**Lección general para el proyecto:** cualquier decisión transaccional/crítica (pagos, aprobaciones) debe resolverse con lógica determinística en n8n, nunca dejada a que el LLM "decida llamar una tool". El LLM se usa solo para redactar lenguaje natural sobre un resultado ya conocido.

**Detalle técnico del nodo Telegram:** la salida de `sendAndWait` da `$json.data.approved` (boolean). Se mapea en `Edit Fields3`:
```
Resultado_Pago = {{ $json.data.approved ? "aprobado" : "rechazado" }}
```
Los botones que genera este nodo son de tipo URL (no `callback_data`), por lo que Telegram muestra un diálogo de "¿Abrir enlace?" antes de resolver — es comportamiento esperado de Telegram (anti-phishing), no se puede quitar desde n8n sin reconstruir el flujo de aprobación a mano con botones `callback_data` + Telegram Trigger + Redis (se evaluó y se descartó por complejidad/beneficio).

**Bug importante ya corregido:** hubo un nodo `Telegram Trigger` (para sacar el `chatId` del dueño) que compartía el mismo bot/token que el nodo de aprobación — esto causaba conflicto de webhook (Telegram solo permite uno por bot) y por eso las aprobaciones no resumían la ejecución. Se eliminó una vez cumplió su propósito.

## 6. Modelo de LLM — decisiones de costo

- Modelo actual en `AI Agent`: `gpt-5.4` ($2.50/M input, $15.00/M output).
- Modelo en `Analyze image`: `gpt-4.1`.
- Estimado para 500 personas/día: ~$840-900/mes con gpt-5.4 (sin cachear), bajando con prompt caching automático de OpenAI (prefijo compartido del system prompt).
- **Alternativas evaluadas para bajar costo:**
  - `gpt-4o-mini` / `gpt-4.1-mini`: 9-19x más barato, recomendado como primer cambio de bajo riesgo (~$70-100/mes).
  - `DeepSeek V4 Flash` (nodo nativo `DeepSeek Chat Model` en n8n): $0.14/$0.28 por M tokens, con caché de contexto automático (98% descuento en tokens repetidos) → estimado $10-20/mes. **Limitación: DeepSeek es solo texto, no tiene visión** — no puede reemplazar el nodo `Analyze image`.
  - `Qwen` (nodo nativo `Qwen Cloud Chat Model`): Qwen-Plus (~$66/mes) más comparable en calidad; Qwen-Flash más barato pero más básico. Ventaja: Qwen-VL permite **consolidar chat + análisis de imagen en un solo proveedor**, a diferencia de DeepSeek.
  - Self-hosted (Ollama en el mismo VPS): evaluado y **desaconsejado** para este caso — sin GPU, latencia de varios segundos, calidad de seguimiento de instrucciones en español más baja, arriesga la experiencia "premium" pedida. Solo tendría sentido con GPU rentada aparte (RunPod/Vultr) si el volumen crece mucho.
  - **Decisión pendiente de ejecutar:** cambiar `OpenAI Chat Model` del `AI Agent` a `DeepSeek Chat Model` (`deepseek-chat`), dejando `Analyze image` en OpenAI (gpt-4.1 o gpt-4o-mini) por el tema de visión.

## 7. Esquema de Supabase

Archivo de referencia: `supabase_schema_missmistervolante.sql` (adjunto). Resumen de tablas:

- **`clientes`**: `id` (uuid), `numero_telefono` (unique), `nombre`, `carnet` (agregado después, vacío hasta matricular), `estado` (nuevo/en_conversacion/matriculado/perdido), `primer_contacto`, `ultimo_contacto`, timestamps.
- **`mensajes`**: historial completo (`cliente_id` FK, `remitente` cliente/agente, `tipo_mensaje` texto/audio/imagen, `contenido`, `whatsapp_message_id`, `created_at`). Requerido por decisión del usuario: SÍ guardar historial completo (no solo datos estructurados).
- **`matriculas`**: `cliente_id` FK, `curso` (practico/teorico), `plan`, `modalidad`, `horario`, `carnet`, `nombre_completo`, `precio_acordado`, `moneda`, `estado` (pendiente_pago/pagado/cancelado).
- **`pagos`**: `cliente_id` FK, `matricula_id` FK nullable, campos del comprobante (banco, monto, ordenante, numero_transaccion **unique** para evitar duplicados, etc.), `estado_comprobante` (pendiente/aprobado/rechazado), `fecha_aprobacion`.
- Triggers `updated_at` automáticos.
- Vistas para dashboard: `vista_matriculas_por_semana`, `vista_embudo_conversion`, `vista_pagos_pendientes`.
- **RLS activado en las 4 tablas, sin políticas** — n8n debe usar la `service_role` key (bypassa RLS), nunca la `anon` key. Decisión del usuario: SÍ necesita dashboard/reportes (justifica las vistas).

### Integración n8n ↔ Supabase (en construcción, parcialmente implementada)

**Patrón elegido por el usuario:** nodo nativo `Supabase` con `Get row` (buscar por `numero_telefono`) + `If` (¿existe?) + `Create row` si no existe — en vez del enfoque alternativo con upsert SQL crudo vía nodo Postgres que se había propuesto primero.

**Punto crítico resuelto:** el nodo `Get row` de Supabase, si no encuentra filas, **no emite ningún item por defecto** — hay que activar **"Always Output Data"** en Options, o la ejecución muere en silencio para clientes nuevos (mismo patrón de bug ya visto con el nodo Redis huérfano y la rama falsa de un `If` sin conectar).

**Arquitectura de unificación:** las dos ramas (cliente nuevo vía `Create row`, cliente viejo vía `Get row` + lookup de historial) **deben normalizarse a los mismos nombres de campo** antes de converger, porque los nombres nativos de columnas de Supabase (`id`, `nombre`, etc.) no coinciden entre ramas. Se definió un nodo de unión nombrado `Cliente_Resuelto` que ambas ramas alimentan, exponiendo siempre `Cliente_ID` y `Contexto_Cliente`.

Código del nodo que arma el resumen para clientes recurrentes (rama "viejo", después de buscar matrículas/pagos):
```javascript
const rows = $input.all().map(i => i.json);

let resumen = "Cliente nuevo, sin historial previo.";

if (rows.length > 0) {
  const r = rows[0];
  resumen = `Cliente recurrente. Última actividad: curso ${r.curso || 'sin definir'}, estado matrícula: ${r.estado_matricula || 'sin registrar'}, pago: ${r.estado_comprobante || 'sin pago registrado'}.`;
}

return {
  json: {
    Cliente_ID: $('Get row').item.json.id,
    Contexto_Cliente: resumen
  }
};
```

**Regla de oro aprendida y reforzada varias veces en esta sesión:** después de un punto donde confluyen ramas condicionales, **nunca referenciar nodos por nombre que solo corrieron en una rama** (ej. `$('Get row')` cuando pudo haber sido la rama `Create row` la que corrió) — usar `$json` cuando ambas ramas alimentan directo al mismo nodo siguiente, o nombrar explícitamente un nodo de unión común (`Cliente_Resuelto`) que sí corrió siempre.

**Pendiente de decidir/implementar:**
1. Conectar la inserción en `mensajes` en las 3 ramas finales (texto/audio/imagen-sin-comprobante), usando `{{ $('Cliente_Resuelto').item.json.Cliente_ID }}` — **bug detectado y corregido en sesión**: el usuario había puesto mal los campos `remitente` (puso el nombre del cliente en vez del literal `"cliente"`) y `tipo_mensaje` (puso el valor crudo de WhatsApp `conversation`/`audioMessage`/`imageMessage` en vez de `"texto"`/`"audio"`/`"imagen"`), violando el `check constraint` de la tabla. Corregido a valores fijos.
2. Decidir punto de inserción de mensajes individuales: antes del debounce (cada mensaje suelto) vs. después de `Edit Fields5` (mensaje ya combinado) — **pregunta abierta sin responder aún por el usuario**.
3. Conectar `Edit Fields4` (rama de imagen que NO es comprobante) — detectada como rama muerta/sin contexto en auditoría, pendiente de arreglo.
4. **Opción A elegida** (vs. opción B manual): crear un paso de extracción con IA que, al momento de la aprobación de pago, lea la memoria de conversación (`Simple Memory`) y extraiga nombre completo/carnet/horario/curso/plan en JSON, para poblar automáticamente `matriculas` (y actualizar `clientes.carnet`). **No implementado aún.**
5. Insertar fila en `pagos` en el punto donde `Aggregate` ya tiene los datos del comprobante, y actualizar `estado_comprobante` en `Edit Fields3` cuando se resuelve la aprobación — **no implementado aún en el flujo real**, solo diseñado.
6. Gap conocido: el curso práctico (USD) no tiene método de pago definido — el prompt ya instruye al agente a no inventar uno.
7. Gap conocido: la fecha fija del curso teórico (3 de agosto 2026) en el prompt se volverá obsoleta — se decidió explícitamente NO hacerla dinámica vía Supabase por ahora (decisión del usuario), se maneja con una instrucción de prompt para que el agente no la repita si ya pasó.

## 8. Archivos generados en esta sesión (adjuntos a la conversación)

- `agent_prompt_missmistervolante_en.xml` — system prompt completo y actualizado del AI Agent.
- `supabase_schema_missmistervolante.sql` — schema completo con tablas, triggers, vistas, RLS.
- `text_input_analyze_image.md` — prompt usado en el nodo `Analyze image` para extraer datos del comprobante o describir la imagen si no es un pago.

## 9. Tarea en curso al momento de cortar la sesión

Se estaba intentando usar un agente llamado "Miro-Manager" (no existe como tipo de agente disponible) para crear una tabla "Project Canvas" en Miro para este proyecto, basada en una plantilla de canvas de proyecto (objetivo, alcance, criterios de éxito, hitos, resultados, acciones, equipos, partes interesadas, usuarios, recursos, restricciones, riesgos). Se buscó el board en Miro sin encontrar uno con el nombre "Volante" — quedó pendiente de ubicar el board correcto (puede estar dentro del board principal "Quantic" como frame/sección) antes de crear la tabla.
