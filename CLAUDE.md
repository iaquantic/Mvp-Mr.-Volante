# Chatbot Miss & Míster Volante (MVP)

Chatbot de WhatsApp para **Miss & Míster Volante**, escuela de conducción en Playa, La Habana. Atiende a los clientes como **Yanet** (una persona real; nunca admite ser IA), solo habla de la escuela y su objetivo es **cerrar matrículas**, no solo informar. Proyecto de Quantic.

El contexto completo (arquitectura, decisiones, bugs resueltos y pendientes) está en [`docs/contexto-proyecto.md`](docs/contexto-proyecto.md). Léelo antes de proponer cambios.

## Stack

- **n8n** self-hosted (VPS Hostinger, EasyPanel + Docker Swarm + Traefik): orquesta todo el flujo.
- **Evolution API**: conexión con WhatsApp.
- **OpenAI**: `gpt-5.4` en el AI Agent y `gpt-4.1` para analizar comprobantes (imagen). Whisper para audios.
- **Redis**: debounce de mensajes seguidos (7 s) y memoria corta.
- **Telegram** (`sendAndWait`): el dueño aprueba o rechaza cada pago.
- **Supabase (Pro)**: tablas `clientes`, `mensajes`, `matriculas`, `pagos` y vistas para dashboard. RLS activado sin políticas: n8n usa la `service_role` key, nunca la `anon`.

## Reglas del proyecto

1. **Lo transaccional nunca lo decide el LLM.** Pagos y aprobaciones se resuelven con nodos deterministas de n8n; el agente solo redacta la respuesta sobre un resultado ya conocido (`Resultado_Pago`).
2. **Tras unir ramas condicionales, no referenciar nodos que solo corrieron en una rama.** Usar `$json` o un nodo de unión común (`Cliente_Resuelto`).
3. **`Get row` de Supabase necesita "Always Output Data"**, o la ejecución muere en silencio con clientes nuevos.
4. **Valores fijos en los `check constraint`:** `remitente` = `cliente`/`agente`; `tipo_mensaje` = `texto`/`audio`/`imagen` (no los valores crudos de WhatsApp).
5. El agente **nunca inventa precios, fechas ni métodos de pago** (el curso práctico aún no tiene método de pago).
6. No guardar secretos (tokens, keys, número de tarjeta) en el repo.

## Estado

Etapa actual: **Desarrollo**. Pendientes principales (detalle en la sección 7 de `docs/contexto-proyecto.md`):

- Insertar en `mensajes` en las ramas texto/audio/imagen (falta decidir: antes o después del debounce).
- Conectar `Edit Fields4` (imagen que no es comprobante).
- Extracción con IA de nombre/carnet/horario/curso al aprobar un pago para poblar `matriculas`.
- Insertar en `pagos` y actualizar `estado_comprobante` tras la aprobación.
- Cambiar el chat del AI Agent a DeepSeek (`deepseek-chat`) manteniendo OpenAI para visión.

Archivos de referencia mencionados en el contexto que **aún no están en el repo**: `agent_prompt_missmistervolante_en.xml`, `supabase_schema_missmistervolante.sql`, `text_input_analyze_image.md`. Cuando se suban, van en `prompts/` y `supabase/`.

## Miro

El proyecto tiene su **Project Canvas** en el tablero Quantic: <https://miro.com/app/board/uXjVGvwBkuA=/?moveToWidget=3458764686930586489>. Lo mantiene el agente `miro-quantic` del repo [iaquantic/Miro-Manager](https://github.com/iaquantic/Miro-Manager).
