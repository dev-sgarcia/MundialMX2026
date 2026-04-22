<template>
  <div class="pantalla-completa-wrapper">
    
    <div v-if="!sesionActiva" class="caja-login-flotante">
      <h1>🏆 Mundial México 2026 🏆</h1>
      <h2>¡Bienvenido al torneo!</h2>
      <p>Para registrar tus pronósticos y competir por la gloria, necesitas acceder.</p>
      
      <button @click="iniciarSesion" class="btn-google">
        Acceder con Google 🚀
      </button>

      <div class="enlaces-legales">
        <a href="#" @click.prevent="mostrarReglas = true">📜 Reglas del Torneo</a>
      </div>
    </div>

    <div v-if="sesionActiva" class="zona-privada">
      
      <header class="cabecera">
        <h1>🏆 Mundial México 2026 🏆</h1>      
        <div class="perfil-usuario">
          <p>¡Hola, <strong>{{ usuario?.full_name || 'Estratega' }}</strong>!</p>
          <span class="leyenda-jornada">Tu Pronóstico de la Jornada</span>
          <button @click="cerrarSesion" class="btn-salir">Cerrar Sesión</button>
        </div>
      </header>

      <div v-if="vista === 'lobby'" class="vista-lobby">
        <div class="encabezado-lobby">
          <h2>Mis Ligas</h2>
          <button @click="crearLigaPrueba" class="btn-crear">➕ Crear Liga de Prueba</button>
        </div>

        <div v-if="misLigas.length === 0" class="mensaje-vacio">
          <p>Aún no perteneces a ninguna liga. ¡Crea una para empezar!</p>
        </div>

        <div class="grid-ligas">
          <article v-for="membresia in misLigas" :key="membresia.league_id" class="tarjeta-liga" @click="entrarALiga(membresia.leagues)">
            <h3>🏆 {{ membresia.leagues.name }}</h3>
            <p>Código: {{ membresia.leagues.invite_code }}</p>
            <p class="campeon-texto">Tu Campeón: {{ membresia.champion_team || 'Aún sin seleccionar' }}</p>
          </article>
        </div>
      </div>

      <div v-if="vista === 'partidos'" class="vista-partidos">
        <button @click="volverAlLobby" class="btn-volver">⬅️ Volver a mis ligas</button>

        <div class="encabezado-liga">
          <img src="/src/assets/ra_transparent.png" alt="Logo de la Liga" class="logo-liga" />
          <h2 class="titulo-liga-activa">{{ ligaActual?.name }}</h2>
        </div>



        <div v-if="cargando" class="mensaje-carga">
          <p>⏳ Trayendo los partidos desde la nube...</p>
        </div>

        <div v-for="(partidosDelDia, fecha) in partidosAgrupados" :key="fecha" class="bloque-jornada">
          
          <h2 class="titulo-fecha">📅 {{ fecha }}</h2>
          
          <div class="grid-partidos">
            <article v-for="partido in partidosDelDia" :key="partido.id" class="tarjeta-partido">              
              
              <div class="tarjeta-encabezado">
                <span class="fase">
                  {{ partido.group_name || 'Fase de Grupos' }}
                  <span v-if="partido.guardado" class="badge-guardado">✅ Guardado</span>
                </span>
                <span class="fecha">🏟️ {{ partido.stadium }} | 🕒 {{ partido.match_time }}</span>
              </div>

              <div class="cuerpo-partido">
                <div class="fila-equipo">
                  <div class="detalles-equipo">
                    <span class="bandera">🏠</span>
                    <span class="nombre-equipo">{{ partido.home_team }}</span>
                  </div>
                  <input 
                    type="number" min="0" max="15" class="input-gol" placeholder="-" 
                    v-model="partido.home_score" 
                    @change="guardarPronostico(partido)" 
                  />
                </div>

                <div class="fila-equipo">
                  <div class="detalles-equipo">
                    <span class="bandera">✈️</span>
                    <span class="nombre-equipo">{{ partido.away_team }}</span>
                  </div>
                  <input 
                    type="number" min="0" max="15" class="input-gol" placeholder="-" 
                    v-model="partido.away_score" 
                    @change="guardarPronostico(partido)" 
                  />
                </div>
              </div>

            </article>
          </div>
        </div>
      </div>

    </div> <div v-if="mostrarReglas" class="modal-fondo" @click.self="mostrarReglas = false">
      <div class="modal-contenido">
        <h3>📜 Reglas MundialMX2026</h3>
        <div class="reglas-texto">
          <ul>
            <li><strong>3 Puntos:</strong> Si le atinas al marcador exacto (Ej. Dices 2-1 y termina 2-1).</li>
            <li><strong>1 Punto :</strong> Si le atinas al equipo ganador o si es empate, pero el marcador no es exacto (Ej. Dices 2-0 y termina 1-0).</li>
            <li><strong>10 Puntos :</strong> Si eliges al campeón desde el inicio del torneo.</li>
            <li><strong>Tiempo límite:</strong> Tienes hasta 1 hora antes de que inicie cada partido para guardar o cambiar tu resultado.</li>
            <li><strong>Premios por puntos:</strong> 1 Ganador de fases de grupo, 1 Ganador fase finalistas y 1 Ganador puntaje total.</li>
            <li><strong>Desempate :</strong> En caso de empate en puntos, se toma en cuenta el primero que haya registrado su marcador.</li>
          </ul>
        </div>
        <button @click="mostrarReglas = false" class="btn-cerrar-modal">Entendido</button>
      </div>
    </div>

  </div> 
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { supabase } from './supabase'

// Variables reactivas
const partidos = ref([])
const cargando = ref(false)
const sesionActiva = ref(false) 
const usuario = ref(null) // Aquí guardaremos el nombre y datos

// Navegación y Ligas
const vista = ref('lobby') // Puede ser 'lobby' o 'partidos'
const misLigas = ref([])
const ligaActual = ref(null)

const mostrarReglas = ref(false) // Controla la ventana emergente de reglas

// Función para entrar
const iniciarSesion = async () => {
  const { error } = await supabase.auth.signInWithOAuth({
    provider: 'google',
  })
  if (error) console.error("Error al iniciar:", error)
}

// Función para salir
const cerrarSesion = async () => {
  const { error } = await supabase.auth.signOut()
  if (error) console.error("Error al salir:", error)
}

// Traer las ligas a las que pertenece el usuario
const cargarLigas = async (userId) => {
  const { data, error } = await supabase
    .from('league_members')
    .select('league_id, champion_team, leagues(id, name, invite_code)')
    .eq('user_id', userId)
  
  if (!error && data) {
    misLigas.value = data
  }
}

// Función para crear la liga "Mundialito RA"
const crearLigaPrueba = async () => {
  const { data: { session } } = await supabase.auth.getSession()
  if (!session) {
    alert("Error: No se detectó tu sesión de Google.");
    return;
  }
  const userId = session.user.id;

  // 1. Intentar crear la liga
  const { data: nuevaLiga, error: errLiga } = await supabase
    .from('leagues')
    .insert({ 
      name: 'Mundialito RA', 
      invite_code: 'MUNDIALITO-' + Math.floor(Math.random() * 10000), 
      owner_id: userId 
    })
    .select()
    .single()

  // SI HAY ERROR AL CREAR, QUE NOS AVISE:
  if (errLiga) {
    console.error("Error en leagues:", errLiga);
    alert("Fallo al crear la liga: " + errLiga.message);
    return; // Detenemos la función aquí
  }

  // 2. Intentar inscribir al usuario
  if (nuevaLiga) {
    const { error: errMiembro } = await supabase
      .from('league_members')
      .insert({
        league_id: nuevaLiga.id,
        user_id: userId
      })

    // SI HAY ERROR AL INSCRIBIR, QUE NOS AVISE:
    if (errMiembro) {
      console.error("Error en league_members:", errMiembro);
      alert("Liga creada, pero falló la inscripción: " + errMiembro.message);
      return;
    }

    // ¡ÉXITO!
    alert("¡Mundialito RA creado con éxito!");
    await cargarLigas(userId)
  }
}

// Navegar hacia el interior de una liga
const entrarALiga = async (liga) => {
  ligaActual.value = liga
  vista.value = 'partidos'
  // ¡Ahora sí llamamos a la base de datos!
  await cargarPartidos()
}

// Regresar al Lobby
const volverAlLobby = () => {
  ligaActual.value = null
  vista.value = 'lobby'
}

const partidosAgrupados = computed(() => {
  return partidos.value.reduce((grupos, partido) => {
    const fecha = partido.match_date || 'Fecha por definir';
    if (!grupos[fecha]) {
      grupos[fecha] = [];
    }
    grupos[fecha].push(partido);
    return grupos;
  }, {});
});

// Cargar los partidos y los pronósticos de la liga seleccionada
const cargarPartidos = async () => {
  cargando.value = true;
  partidos.value = []; // Limpiamos la lista

  const { data: { session } } = await supabase.auth.getSession();
  if (!session) return;
  
  const userId = session.user.id;
  const leagueId = ligaActual.value.id; // El ID del Mundialito RA

  // 1. Traer el catálogo de partidos oficiales
  const { data: dbMatches, error: errMatches } = await supabase
    .from('matches')
    .select('*')
    .order('id', { ascending: true }); // Ordenados para que salgan en secuencia
    // --- AGREGA ESTA LÍNEA PARA ESPIAR LOS DATOS ---
    console.log("Datos de Supabase:", dbMatches);


  if (errMatches) {
    console.error("Error al cargar partidos:", errMatches);
    cargando.value = false;
    return;
  }

  // 2. Traer TUS pronósticos exclusivos para esta liga
  const { data: misPronosticos, error: errPronosticos } = await supabase
    .from('predictions')
    .select('*')
    .eq('user_id', userId)
    .eq('league_id', leagueId);

  // 3. Fusionar datos: Si ya tenías un resultado, lo ponemos en pantalla
  const partidosFusionados = dbMatches.map(partido => {
    const pronosticoGuardado = misPronosticos?.find(p => p.match_id === partido.id);
    
    return {
      ...partido,
      home_score: pronosticoGuardado ? pronosticoGuardado.home_score : null,
      away_score: pronosticoGuardado ? pronosticoGuardado.away_score : null,
      guardado: !!pronosticoGuardado
    };
  });

  partidos.value = partidosFusionados;
  cargando.value = false;
};

const guardarPronostico = async (partido) => {
  const { data: { session } } = await supabase.auth.getSession()
  if (!session) return

  // Si alguna caja está vacía, no guardamos nada todavía
  if (partido.home_score === null || partido.home_score === '' || 
      partido.away_score === null || partido.away_score === '') {
    return;
  }

  // Mandamos las 3 coordenadas a la base de datos
  const { error } = await supabase
    .from('predictions')
    .upsert({
      user_id: session.user.id,
      match_id: partido.id,
      league_id: ligaActual.value.id, // ¡ESTA ES LA CLAVE DE LA MULTI-LIGA!
      home_score: partido.home_score,
      away_score: partido.away_score
    }, {
      onConflict: 'user_id, match_id, league_id' // Le recordamos a Supabase la regla
    })

  if (!error) {
    partido.guardado = true; // Aparece el check verde
  } else {
    console.error("Error al guardar pronóstico:", error);
  }
}

// Función de arranque para cargar los datos
onMounted(async () => {
  // 1. Revisar sesión
  const { data: { session } } = await supabase.auth.getSession()
  if (session) {
    sesionActiva.value = true
    usuario.value = session.user.user_metadata
    cargarLigas(session.user.id)    
  }

  // 2. Escuchar cambios de sesión
  supabase.auth.onAuthStateChange((_event, session) => {
    sesionActiva.value = !!session
    usuario.value = session ? session.user.user_metadata : null
  })

  // 3. Traer los partidos
  const { data: dataPartidos, error: errorPartidos } = await supabase.from('matches').select('*')
  
  if (!errorPartidos) {
    // 4. NUEVO: Si hay un usuario logueado, traemos SUS pronósticos
    if (session) {
      const { data: misPronosticos } = await supabase
        .from('predictions')
        .select('*')
        .eq('user_id', session.user.id) // Solo traemos los de este usuario

      // Combinamos los partidos con los pronósticos que encontramos
      partidos.value = dataPartidos.map(partido => {
        // Buscamos si hay un pronóstico guardado para este partido en específico
        const pronosticoGuardado = misPronosticos?.find(p => p.match_id === partido.id)
        
        // Si lo encontramos, le inyectamos los goles para que aparezcan en pantalla
        if (pronosticoGuardado) {
          return {
            ...partido,
            home_score: pronosticoGuardado.home_score,
            away_score: pronosticoGuardado.away_score
          }
        }
        return partido
      })
    } else {
      // Si no hay sesión, mostramos los partidos normales vacíos
      partidos.value = dataPartidos
    }
  }
  
  cargando.value = false
})
</script>

<style scoped>
/* 1. CONFIGURACIÓN DEL LIENZO (FONDO TOTAL) */
.pantalla-completa-wrapper {
  width: 100%;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  /* Centra el login cuando no hay sesión activa */
  justify-content: center; 
  align-items: center;
  
  /* FONDO DEL ESTADIO */
  background-image: url('https://images.unsplash.com/photo-1522778119026-d647f0596c20?ixlib=rb-4.0.3&auto=format&fit=crop&w=2000&q=80');
  background-size: cover;
  background-position: center center;
  background-attachment: fixed;
  background-repeat: no-repeat;
  
  padding: 20px;
  box-sizing: border-box;
  font-family: 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
}

/* 2. DISEÑO DEL LOGIN (CAJA BLANCA FLOTANTE) */
.caja-login-flotante {
  background: rgba(255, 255, 255, 0.96);
  padding: 40px;
  border-radius: 20px;
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.3);
  text-align: center;
  width: 100%;
  max-width: 450px;
  animation: entradaSuave 0.5s ease-out;
}

@keyframes entradaSuave {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}

.btn-google {
  background: white;
  color: #374151;
  border: 1px solid #d1d5db;
  padding: 12px 24px;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  width: 100%;
  margin: 20px 0;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
}

.btn-google:hover {
  background: #f9fafb;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}

/* 3. ZONA PRIVADA (LOBBY Y CANCHA) */
.zona-privada {
  width: 100%;
  max-width: 1400px; /* Expandimos para aprovechar tu monitor 1080p */
  align-self: flex-start; /* Evita que el dashboard se centre verticalmente */
  margin: 0 auto;
}

.cabecera {
  background: rgba(255, 255, 255, 0.9);
  padding: 20px;
  border-radius: 15px;
  margin-bottom: 30px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  backdrop-filter: blur(5px);
}

/* EFECTO DORADO PARA EL NOMBRE DE LA LIGA */
.titulo-liga-activa {
  font-size: 2.8rem; /* Letra grande y dominante */
  font-weight: 900;
  text-align: center;
  margin: 15px 0 30px 0;
  text-transform: uppercase; /* Todo en mayúsculas para más impacto */
  letter-spacing: 2px;
  
  /* El secreto del color dorado */
  background: linear-gradient(to right, #fef08a, #f59e0b, #fbbf24);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  
  /* Sombra negra muy fuerte para separarlo del fondo */
  filter: drop-shadow(2px 4px 6px rgba(0, 0, 0, 0.9));
}

.perfil-usuario { text-align: right; }
.btn-salir {
  background: #fee2e2;
  color: #dc2626;
  border: none;
  padding: 8px 15px;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
}

/* 4. GRID DE PARTIDOS (MULTICOLUMNA) */
.grid-partidos {
  display: grid;
  /* La magia: En PC mostrará 3 o 4 columnas, en móvil 1 */
  grid-template-columns: repeat(auto-fill, minmax(360px, 1fr));
  gap: 20px;
  margin-top: 15px;
}

/* FECHAS DE LOS PARTIDOS (Cristal oscuro con acento dorado) */
.titulo-fecha {
  color: #ffffff;
  margin: 40px 0 20px 0;
  font-size: 1.5rem;
  font-weight: 700;
  
  /* Fondo oscuro semitransparente con desenfoque */
  background: rgba(15, 23, 42, 0.85); /* Un azul marino casi negro */
  backdrop-filter: blur(5px);
  
  /* Detalles de diseño */
  display: inline-block;
  padding: 10px 25px;
  border-radius: 8px; /* Bordes modernos */
  border-left: 6px solid #fbbf24; /* Línea dorada a la izquierda */
  box-shadow: 0 8px 15px rgba(0, 0, 0, 0.5);
}

/* 5. TARJETA DE PARTIDO (EQUIPOS ALINEADOS) */
.tarjeta-partido {
  background: white;
  border-radius: 12px;
  padding: 15px;
  display: flex;
  flex-direction: column;
  gap: 12px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}

.cuerpo-partido {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.fila-equipo {
  display: flex;
  justify-content: space-between; /* Nombre a la izquierda, input a la derecha */
  align-items: center;
  background: #f8fafc;
  padding: 10px;
  border-radius: 8px;
}

.detalles-equipo {
  display: flex;
  align-items: center;
  gap: 10px;
  font-weight: bold;
}

.input-gol {
  width: 55px;
  height: 45px;
  text-align: center;
  font-size: 1.4rem;
  font-weight: bold;
  border: 2px solid #cbd5e1;
  border-radius: 8px;
}

/* 6. MODALES Y EXTRAS */
.modal-fondo {
  position: fixed;
  top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(0,0,0,0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-contenido {
  background: white;
  padding: 30px;
  border-radius: 15px;
  max-width: 500px;
  width: 90%;
}

.grid-ligas {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 20px;
}

.tarjeta-liga {
  background: white;
  padding: 20px;
  border-radius: 12px;
  cursor: pointer;
  transition: transform 0.2s;
}

.tarjeta-liga:hover { transform: scale(1.03); }

/* Quitar flechitas de los inputs */
input::-webkit-outer-spin-button,
input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

.encabezado-liga {
  display: flex;
  align-items: center; /* Centra verticalmente logo y texto */
  justify-content: center; /* O flex-start si prefieres alineación a la izquierda */
  gap: 15px; /* Espacio entre el logo y el nombre */
  margin-bottom: 20px;
}

.logo-liga {
  width: 40px; /* Ajusta el tamaño de la imagen */
  height: auto; /* Mantiene la proporción */
  /* Otros estilos si quieres, ej: border-radius: 50%; */
}

/* Asegúrate de ajustar márgenes si es necesario debido al Flexbox */
.titulo-liga-activa {
  margin: 0; /* Flexbox gap maneja el espacio */
  /* ... tus estilos existentes de dorado y sombra ... */
}
</style>