<script setup>
import { ref, onMounted, computed } from 'vue';
import axios from 'axios';

const API_URL = 'http://127.0.0.1:8000/api/tasks';

const tareas = ref([]);
const cargando = ref(false);
const errorMsg = ref('');

// Fecha mínima permitida (día actual) en formato YYYY-MM-DD
const hoyStr = computed(() => {
  const hoy = new Date();
  const anio = hoy.getFullYear();
  const mes = String(hoy.getMonth() + 1).padStart(2, '0');
  const dia = String(hoy.getDate()).padStart(2, '0');
  return `${anio}-${mes}-${dia}`;
});

const form = ref({
  titulo: '',
  descripcion: '',
  prioridad: 'Media',
  fecha_limite: new Date().toISOString().split('T')[0]
});

// Detección reactiva de la palabra 'urgente' (RF-02)
const esUrgente = computed(() => /urgente/i.test(form.value.titulo));

const cargarTareas = async () => {
  try {
    const res = await axios.get(API_URL);
    tareas.value = res.data;
  } catch (err) {
    console.error('Error al conectar con la API:', err);
  }
};

const crearTarea = async () => {
  errorMsg.value = '';

  // RF-03: Validación en frontend (no menor a hoy)
  if (form.value.fecha_limite < hoyStr.value) {
    errorMsg.value = 'La fecha límite no puede ser anterior a la fecha actual.';
    return;
  }

  cargando.value = true;
  try {
    const res = await axios.post(API_URL, form.value);
    
    // RF-06: Incorporación reactiva inmediata sin recargar la página
    tareas.value.unshift(res.data);

    // Limpiar formulario
    form.value.titulo = '';
    form.value.descripcion = '';
    form.value.prioridad = 'Media';
    form.value.fecha_limite = hoyStr.value;
  } catch (err) {
    errorMsg.value = err.response?.data?.message || 'Error al registrar la tarea.';
  } finally {
    cargando.value = false;
  }
};

const eliminarTarea = async (id) => {
  if (!confirm('¿Seguro que deseas eliminar esta tarea?')) return;

  try {
    await axios.delete(`${API_URL}/${id}`);
    
    // RF-06: Remueve reactivamente la tarea disparando solo su transición de salida
    tareas.value = tareas.value.filter(t => t.id !== id);
  } catch (err) {
    alert('No se pudo eliminar la tarea en el servidor');
  }
};

onMounted(cargarTareas);
</script>

<template>
  <div class="contenedor">
    <header class="header">
      <h1>TaskAI — Gestión de Tareas</h1>
    </header>

    <!-- FORMULARIO DE REGISTRO (RF-01, RF-02, RF-03) -->
    <section class="tarjeta-formulario">
      <h2>Registrar Nueva Tarea</h2>
      <div v-if="errorMsg" class="alerta-error">{{ errorMsg }}</div>

      <form @submit.prevent="crearTarea">
        <div class="campo">
          <label>Título *</label>
          <input 
            v-model="form.titulo" 
            type="text" 
            placeholder="Ej: Resolver incidente urgente en el servidor" 
            required 
          />
          <small v-if="esUrgente" class="aviso-urgente">
            ⚡ Se detectó la palabra "Urgente": La prioridad se asignará automáticamente como <strong>Alta</strong> en backend.
          </small>
        </div>

        <div class="campo">
          <label>Descripción</label>
          <textarea 
            v-model="form.descripcion" 
            rows="2" 
            placeholder="Detalles sobre la tarea a realizar..."
          ></textarea>
        </div>

        <div class="fila-campos">
          <div class="campo">
            <label>Prioridad</label>
            <select v-model="form.prioridad" :disabled="esUrgente">
              <option value="Baja">Baja</option>
              <option value="Media">Media</option>
              <option value="Alta">Alta</option>
            </select>
          </div>

          <div class="campo">
            <label>Fecha Límite *</label>
            <input 
              v-model="form.fecha_limite" 
              type="date" 
              :min="hoyStr" 
              required 
            />
          </div>
        </div>

        <button type="submit" :disabled="cargando" class="btn-guardar">
          {{ cargando ? 'Guardando...' : 'Crear Tarea' }}
        </button>
      </form>
    </section>

    <!-- LISTADO DINÁMICO DE TAREAS (RF-04, RF-05, RF-06) -->
    <section class="seccion-listado">
      <h2>Listado de Tareas</h2>
      <p v-if="tareas.length === 0" class="sin-datos">No hay tareas pendientes registradas.</p>

      <!-- TransitionGroup anima únicamente la tarjeta individual que desaparece sin re-renderizar el resto -->
      <TransitionGroup name="fade-tarea" tag="div" class="lista-tareas">
        <div 
          v-for="tarea in tareas" 
          :key="tarea.id"
          class="tarjeta-tarea"
          :class="{ 'destacado-urgente': tarea.prioridad === 'Alta' }"
        >
          <div class="tarea-header">
            <h3>{{ tarea.titulo }}</h3>
            <span class="badge" :class="tarea.prioridad ? tarea.prioridad.toLowerCase() : 'media'">
              {{ tarea.prioridad }}
            </span>
          </div>

          <p class="tarea-desc">{{ tarea.descripcion || 'Sin descripción' }}</p>

          <div class="tarea-footer">
            <small>📅 Fecha Límite: {{ tarea.fecha_limite }}</small>
            <button @click="eliminarTarea(tarea.id)" class="btn-eliminar">Eliminar</button>
          </div>
        </div>
      </TransitionGroup>
    </section>
  </div>
</template>

<style scoped>
.contenedor {
  max-width: 760px;
  margin: 2rem auto;
  font-family: system-ui, -apple-system, sans-serif;
  padding: 0 1rem;
  color: #1e293b;
}
.header h1 {
  margin-bottom: 1.5rem;
  font-size: 1.75rem;
  font-weight: 700;
}
.tarjeta-formulario {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1.25rem;
  margin-bottom: 2rem;
  box-shadow: 0 1px 3px rgba(0,0,0,0.06);
}
.tarjeta-formulario h2 {
  margin-top: 0;
  margin-bottom: 1rem;
  font-size: 1.2rem;
}
.campo {
  margin-bottom: 1rem;
  display: flex;
  flex-direction: column;
}
.fila-campos {
  display: flex;
  gap: 1rem;
}
.fila-campos .campo {
  flex: 1;
}
label {
  font-weight: 600;
  margin-bottom: 0.35rem;
  font-size: 0.875rem;
}
input, textarea, select {
  padding: 0.55rem;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  font-size: 0.95rem;
}
input:focus, textarea:focus, select:focus {
  outline: none;
  border-color: #3b82f6;
}
.aviso-urgente {
  color: #dc2626;
  margin-top: 0.35rem;
  font-size: 0.8rem;
}
.alerta-error {
  background: #fee2e2;
  color: #991b1b;
  padding: 0.6rem;
  border-radius: 6px;
  margin-bottom: 1rem;
  font-size: 0.875rem;
}
.btn-guardar {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.65rem 1.3rem;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}
.btn-guardar:hover {
  background: #1d4ed8;
}
.btn-guardar:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
.lista-tareas {
  display: flex;
  flex-direction: column;
  gap: 0.85rem;
}
.tarjeta-tarea {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1rem;
  box-shadow: 0 1px 2px rgba(0,0,0,0.04);
}
/* RF-06: Destacado con borde y fondo rojo suave para prioridad Alta */
.destacado-urgente {
  background-color: #fff5f5 !important;
  border: 1.5px solid #feb2b2 !important;
}
.tarea-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.tarea-header h3 {
  margin: 0;
  font-size: 1.05rem;
}
.badge {
  padding: 0.2rem 0.6rem;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 700;
}
.badge.alta { background: #fee2e2; color: #991b1b; }
.badge.media { background: #fef3c7; color: #92400e; }
.badge.baja { background: #dcfce7; color: #166534; }
.tarea-desc {
  margin: 0.5rem 0;
  color: #475569;
  font-size: 0.92rem;
}
.tarea-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 0.5rem;
}
.btn-eliminar {
  background: #ef4444;
  color: white;
  border: none;
  padding: 0.35rem 0.75rem;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.8rem;
  transition: background 0.2s;
}
.btn-eliminar:hover {
  background: #dc2626;
}
.sin-datos {
  color: #64748b;
  font-style: italic;
}

/* RF-06: Animación aislada que afecta únicamente al ítem eliminado */
.fade-tarea-leave-active {
  transition: all 0.35s ease;
}
.fade-tarea-leave-to {
  opacity: 0;
  transform: translateX(40px);
}
</style>