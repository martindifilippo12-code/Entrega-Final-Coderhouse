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

