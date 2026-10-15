# Entrega Final Coderhouse
Caso de uso: Triage automático de pedidos de cotización para un productor asesor de seguros generales. Un cliente pide una cotización; la IA interpreta el pedido, extrae los datos, detecta lo que falta, asigna una prioridad y redacta la respuesta. El productor aprueba o rechaza con un clic y recién ahí se contacta al cliente.

## Enlances de Entrega 
1. Video Demo (3 min)
2. Dashboard de control (KPIs)
3. Base de datos (solo lectura)
4. Flujo Principal (JSON)
5. Workflow de manejo de errores(JSON)
6. Esquema JSON de Integracion
7. Capturas de Evidencia

## Problemas y Propuesta 
Situación actual (manual): el productor recibe pedidos de cotización por distintos canales, con texto libre y casi siempre incompletos. Por cada pedido tiene que leerlo, identificar el tipo de seguro, ver qué datos faltan, decidir a quién responder primero y escribir un mail. Los pedidos urgentes (pólizas por vencer, clientes que vienen de otra compañía) se mezclan con consultas generales.

Propuesta: un sistema que procesa cada pedido de punta a punta sin intervención manual, salvo un único punto de control humano antes de contactar al cliente:

✔️ Registra Cliente y Solicitud en una base de datos con estados. 

✔️ La IA interpreta el lenguaje natural y devuelve datos estructurados (JSON). 

✔️ Prioriza automaticamente (ALTA/MEDIA/BAJA) con un motivo explicado. 
  
✔️ Redacta la respuesta pidiendo exactamente los datos que faltan. 
 
✔️ Human in the loop: el productor aprueba o rechaza desde su Gmail. 
 
✔️ Todo error queda registrado y la solicitud nunca queda perdida. 
  
## Stack Tecnologico (4 Categorias obligatorias)

| Categoría | Herramienta | Uso en el sistema |
| :--- | :--- | :--- |
| **Orquestador** | n8n Cloud | Flujo principal (18 nodos) + workflow de errores separado |
| **Base de datos** | Airtable | Memoria y registro: `Clientes`, `Solicitudes`, `Log Errores` (vinculadas) |
| **Procesamiento IA** | OpenAI `gpt-4o-mini` (nodo nativo de n8n) | Clasificación, extracción de datos, priorización y redacción, con salida JSON estructurada |
| **Canal de salida** | Gmail | Aprobación humana (*Send and Wait*) y respuesta al cliente |

## Diagrama de Arquitectura

<img width="3440" height="1960" alt="00-arquitectura" src="https://github.com/user-attachments/assets/960ffa8e-2d48-4c59-852e-ce2464390760" />

## 4. Cómo funciona el flujo (paso a paso)

| # | Nodo | Tipo | Qué hace |
|---|---|---|---|
| 1 | **Formulario de cotización** | Form Trigger | Trigger inteligente: el flujo solo corre cuando un cliente envía el formulario (no hay polling ni consumo innecesario). |
| 2 | **Config** | Set | Centraliza todas las variables: base de Airtable, nombres de tablas, modelo, `maxTokens`, nombre y email del productor y el prompt. **Ningún otro nodo tiene datos hardcodeados.** |
| 3 | **Validar y normalizar** | Code | Limpia espacios, pasa el email a minúsculas y valida: nombre presente, email con formato válido y mensaje de al menos 15 caracteres. |
| 4 | **¿Datos válidos?** | IF | Filtro: si los datos no sirven, se registra en `Log Errores` y **no se llama a la IA** (ahorro de tokens). |
| 5 | **Crear o actualizar cliente** | HTTP → Airtable | *Upsert* por email: no se duplican clientes. |
| 6 | **Crear solicitud** | HTTP → Airtable | Crea la solicitud vinculada al cliente con `Estado = Pendiente`. |
| 7 | **Analizar con IA** | OpenAI (nativo) | `gpt-4o-mini`, temperatura 0.2, máx. 600 tokens. Devuelve el JSON definido en el prompt. Reintenta 3 veces y tiene salida de error. |
| 8 | **Interpretar respuesta IA** | Code | Valida que la respuesta sea un JSON correcto, normaliza los valores a las listas permitidas y cuenta los tokens. Si falla, sale por la rama de error. |
| 9 | **Guardar análisis IA** | HTTP → Airtable | Guarda tipo, datos extraídos, faltantes, prioridad, respuesta y tokens → `Procesado por IA`. |
| 10 | **Pedir aprobación (HITL)** | Gmail *Send and Wait* | Envía al productor un resumen con botones **Aprobar y enviar / Rechazar**. **La ejecución queda en pausa** hasta la decisión. |
| 11 | **¿Aprobado?** | IF | Bifurca según la decisión humana. |
| 12a | **Marcar aprobado → Enviar respuesta al cliente → Marcar enviado** | Airtable + Gmail | `Aprobado por Humano` (con fecha) → mail al cliente → `Enviado`. |
| 12b | **Marcar rechazado** | HTTP → Airtable | `Rechazado` (con fecha). No se contacta al cliente. |
| E1 | **Registrar error de validación** | HTTP → Airtable | Datos inválidos → `Log Errores`. |
| E2 | **Registrar error IA → Marcar solicitud con error** | HTTP → Airtable | Falla de la API o JSON inválido → `Log Errores` (vinculado a la solicitud) + `Estado = Error`. |
| E3 | **Workflow "Manejo de errores"** | Error Trigger | Configurado como *Error Workflow* del principal: cualquier falla no contemplada también queda registrada. |

## 5. Base de datos (el "cerebro")
Base de Airtable **Cotizaciones**, con tres tablas vinculadas entre sí para evitar datos aislados:

### [`Tabla Clientes`](AIRTABLE%20TABLAS/7-%20AIRTABLE%20TABLA%20CLIENTES.png)
 
| Campo | Tipo | Nota |
|---|---|---|
| Nombre | Texto (principal) | |
| Email | Email | Clave del *upsert* (evita duplicados) |
| Telefono | Teléfono | |
| Solicitudes | Link → Solicitudes | Historial del cliente |

### [`Tabla Solicitudes`](AIRTABLE%20TABLAS/8-AIRTABLE%20TABLA%20SOLICITUDES.png)
| Campo | Tipo | Nota |
|---|---|---|
| Id | Autonumber (principal) | |
| Fecha | Created time | |
| Cliente | Link → Clientes | |
| Mensaje Original | Texto largo | Texto libre del cliente |
| Tipo de Seguro | Selección: Auto, Hogar, Comercio, Vida, Otro | Lo define la IA |
| Datos Extraidos | Texto largo (JSON) | Lo que el cliente informó |
| Datos Faltantes | Texto largo | Lo que falta para cotizar |
| Prioridad | Selección: Alta, Media, Baja | |
| Motivo Prioridad | Texto largo | Explicación de la IA (trazabilidad) |
| Respuesta Sugerida | Texto largo | Borrador para el cliente |
| **Estado** | Selección: **Pendiente → Procesado por IA → Aprobado por Humano → Enviado** · Rechazado · Error | Ciclo de vida de la solicitud |
| Tokens usados | Número | Control de costos |
| Fecha Aprobacion | Fecha y hora | Momento de la decisión humana |
| Log Errores | Link → Log Errores | |
