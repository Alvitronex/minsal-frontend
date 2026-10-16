# AGENTS.md - Registro de Interacción con Asistente de IA y Criterios Técnicos

## 1. Alcance y Rol del Asistente de IA
El asistente de Inteligencia Artificial se utilizó de forma consultiva y como acelerador de desarrollo para:
- Estructuración rápida de scaffolding en Laravel (migración con `SoftDeletes`, modelo y métodos RESTful).
- Definición de la expresión regular para la detección de la palabra "urgente" insensible a mayúsculas/minúsculas.
- Configuración de directivas de animación scoped con `<TransitionGroup>` en Vue 3 para aislar transiciones.
- Construcción de la especificación de la API bajo el estándar OpenAPI 3.0.

---

## 2. Decisiones Técnicas y Criterios de Arquitectura

### RF-02: Precedencia de la Regla de Negocio en el Backend
- **Criterio:** La asignación automática de prioridad no debe depender exclusivamente de la interfaz gráfica para evitar manipulación del lado del cliente.
- **Implementación:** Se colocó una evaluación con expresión regular `preg_match('/urgente/i', $request->titulo)` en el método `store` de `TareasController`. Si la condición se cumple, la prioridad se sobreescribe como `Alta` con precedencia absoluta sobre el payload enviado por el cliente.

### RF-03: Validación de Fecha Límite y Código HTTP Estricto
- **Criterio:** No permitir fechas anteriores al día actual, permitiendo tareas con fecha del día en curso, respondiendo con un error HTTP 400 Bad Request explícito.
- **Implementación:** Se normalizó la fecha con `Carbon::parse($request->fecha_limite)->startOfDay()` comparándola con `Carbon::today()`. Si la fecha es estrictamente menor, el controlador interrumpe el flujo y retorna de inmediato un código de estado `400` acompañado del mensaje descriptivo solicitado.

### RF-05: Eliminación Lógica (Soft Deletes)
- **Criterio:** Garantizar la persistencia histórica de datos y trazabilidad sin pérdida de información física en base de datos.
- **Implementación:** Se integró el trait `Illuminate\Database\Eloquent\SoftDeletes` en el modelo `Tareas` y la columna `$table->softDeletes()` en la migración. Las consultas por defecto (`index`) omiten automáticamente registros con `deleted_at` no nulo.

### RF-06: Reactividad Aislada y Optimización de Renderizado
- **Criterio:** Al eliminar un elemento, la animación de salida debe ejecutarse únicamente sobre la tarjeta en cuestión, sin disparar re-renderizados ni parpadeos en los demás elementos del listado.
- **Implementación:** Se utilizó `<TransitionGroup>` en Vue 3 enlazado a identificadores únicos (`:key="tarea.id"`), aplicando transiciones CSS exclusivamente en `.fade-tarea-leave-active` y `.fade-tarea-leave-to`.