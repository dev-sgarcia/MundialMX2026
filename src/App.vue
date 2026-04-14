<template>
  <main class="contenedor">
    <header class="cabecera">
      <h1>🏆Mundial México 2026🏆</h1>
      <p>Tu Pronóstico de la Jornada</p>
    </header>
    
    <div v-if="cargando" class="mensaje-carga">
      <p>⏳ Trayendo los partidos desde la nube...</p>
    </div>

    <div v-else class="grid-partidos">
      
      <article v-for="partido in partidos" :key="partido.id" class="tarjeta-partido">
        
        <div class="tarjeta-encabezado">
          <span class="fase">{{ partido.stage_category }}</span>
          <span class="fecha">📅 {{ partido.match_date }}</span>
        </div>

        <div class="tarjeta-cuerpo">
          <div class="equipo">
            <span class="nombre">{{ partido.home_team }}</span>
          </div>

          <div class="pronostico">
            <input type="number" min="0" max="15" class="input-gol" placeholder="-" />
            <span class="vs">VS</span>
            <input type="number" min="0" max="15" class="input-gol" placeholder="-" />
          </div>

          <div class="equipo">
            <span class="nombre">{{ partido.away_team }}</span>
          </div>
        </div>

      </article>

    </div>
  </main>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { supabase } from './supabase'

// Variables reactivas
const partidos = ref([])
const cargando = ref(true)

// Función de arranque
onMounted(async () => {
  const { data, error } = await supabase.from('matches').select('*')
  
  if (error) {
    console.error("Hubo un error:", error)
  } else {
    partidos.value = data
  }
  
  cargando.value = false
})
</script>

<style scoped>
/* Contenedor principal de la página */
.contenedor {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  background-color: #f8fafc;
  min-height: 100vh;
}

.cabecera {
  text-align: center;
  margin-bottom: 30px;
}

.cabecera h1 {
  color: #1e3a8a;
  font-size: 2.5rem;
  margin-bottom: 5px;
}

.mensaje-carga {
  text-align: center;
  font-size: 1.2rem;
  color: #64748b;
}

/* La rejilla que acomoda las tarjetas */
.grid-partidos {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

/* El diseño de la TARJETA */
.tarjeta-partido {
  background-color: white;
  border-radius: 12px;
  padding: 15px 20px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.05);
  border: 1px solid #e2e8f0;
}

.tarjeta-encabezado {
  display: flex;
  justify-content: space-between;
  font-size: 0.85rem;
  color: #64748b;
  margin-bottom: 15px;
  border-bottom: 1px solid #f1f5f9;
  padding-bottom: 8px;
}

.fase {
  font-weight: bold;
  color: #2563eb;
  background-color: #dbeafe;
  padding: 2px 8px;
  border-radius: 4px;
}

/* Acomodo de los equipos y las cajitas de goles */
.tarjeta-cuerpo {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.equipo {
  flex: 1;
  text-align: center;
  font-weight: bold;
  font-size: 1.1rem;
  color: #334155;
}

/* Zona central de VS y Cajitas */
.pronostico {
  display: flex;
  align-items: center;
  gap: 10px;
}

.vs {
  font-size: 0.9rem;
  color: #94a3b8;
  font-weight: bold;
}

/* Las cajitas numéricas */
.input-gol {
  width: 50px;
  height: 50px;
  text-align: center;
  font-size: 1.5rem;
  font-weight: bold;
  border: 2px solid #cbd5e1;
  border-radius: 8px;
  color: #1e293b;
  transition: border-color 0.2s;
}

.input-gol:focus {
  outline: none;
  border-color: #3b82f6;
}

/* Quitar las flechitas de los inputs numéricos */
.input-gol::-webkit-outer-spin-button,
.input-gol::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}
</style>