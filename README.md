# AI Model Drift Monitor

> Sistema de alertas tempranas para degradación de modelos de ML en producción

Pipeline que monitorea métricas de un modelo de ML cada semana, detecta drift comparando contra el baseline histórico y — cuando la degradación supera el umbral — genera un runbook de mitigación de 2 semanas y envía alertas simultáneas a Telegram y Notion.

**Demo:** próximamente  

---

## El problema que resuelve

Un modelo de ML que funciona bien hoy puede degradarse silenciosamente durante semanas. En e-commerce, un modelo de recomendaciones entrenado en invierno falla cuando llegan las tendencias de verano — las métricas caen, la conversión baja, y el equipo se entera cuando el daño ya está hecho. Este sistema detecta el drift antes de que impacte en el negocio.

---

## Concepto: ¿Qué es model drift?

**Model drift** es la degradación gradual del rendimiento de un modelo de ML cuando el mundo real cambia pero el modelo no se actualiza. Las métricas que se monitoran son accuracy, precision, recall, latencia y conversion rate.

**Concept drift** es el caso específico donde la distribución estadística de los datos de entrada cambia con el tiempo — el modelo sigue aplicando las reglas del mundo de ayer.

---

## Arquitectura

```
Schedule Trigger (lunes 8am)
    ↓
HTTP Request → métricas del modelo (Google Sheets CSV)
    ↓
Nodo de detección: calcula drift vs baseline (primeras 5 semanas)
    ↓
    ├── drift ≤ 5% → Log silencioso en Notion ("todo OK")
    └── drift > 5% → 
            ├── GPT-4o-mini genera runbook de 2 semanas
            ├── Alerta en Telegram (resumen ejecutivo)
            └── Runbook completo en Notion
```

---

## Decisiones de diseño clave

**¿Por qué baseline de las primeras 5 semanas?**
Comparar contra la semana anterior detecta cambios bruscos pero no drift gradual. Comparar contra el promedio histórico completo contamina el baseline si ya hay semanas degradadas. Las primeras 5 semanas representan el modelo "sano" — el punto de referencia más limpio.

**¿Por qué umbral del 5%?**
Por debajo del 5% puede ser ruido estadístico normal. Por encima del 10% el sistema escala a severidad CRÍTICO. El 5% permite detección temprana sin generar fatiga de alertas en el equipo.

**¿Por qué alerta dual — Telegram + Notion?**
Son necesidades distintas, no canales redundantes. Telegram entrega el resumen en 5 líneas para decidir si hay que actuar ahora. Notion entrega el runbook completo con diagnóstico, plan de 2 semanas y criterios de recuperación para saber cómo actuar. Velocidad vs profundidad.

**¿Por qué log silencioso cuando no hay drift?**
Un sistema de monitoreo que solo habla cuando hay problemas no genera confianza. El log silencioso registra que el chequeo se realizó y todo está bien — eso es lo que convierte el sistema en confiable.

---

## Stack

| Capa | Herramienta |
|---|---|
| Trigger | n8n Schedule (lunes 8am) |
| Fuente de datos | Google Sheets (CSV público) |
| Detección | Nodo Code con lógica de comparación vs baseline |
| LLM | OpenAI GPT-4o-mini |
| Alerta rápida | Telegram Bot API |
| Runbook | Notion API |

---

## Formato del dataset de métricas

```
semana | accuracy | precision | recall | latencia_ms | conversion_rate
```

El sistema espera al menos 6 semanas de datos: 5 para el baseline + 1 para monitorear.

---

## Variables de entorno necesarias

```env
OPENAI_API_KEY=
NOTION_TOKEN=
NOTION_PAGE_ID=
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
SHEETS_CSV_URL=
```

---

## Resultados del demo

- 8 semanas simuladas de métricas de modelo de recomendaciones e-commerce
- Drift detectado en semana 8: accuracy cayó 12% (0.92 → 0.81), conversión cayó 29%
- Severidad: CRÍTICO
- Runbook generado: diagnóstico + plan semana 1 + plan semana 2 + criterios de recuperación
- Alerta Telegram recibida: < 5 segundos

---

## Lo que aprendí construyendo esto

Un sistema de monitoreo que solo habla cuando hay problemas no es suficiente. Necesitás el log silencioso de que chequeaste y todo está bien — eso es lo que genera confianza en el sistema a lo largo del tiempo.

