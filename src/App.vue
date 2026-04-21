<template>
  <main class="contenedor">
    <header class="cabecera">
      <h1>🏆 Mundial México 2026 🏆</h1>      
      <div v-if="sesionActiva" class="perfil-usuario">
        <p>¡Hola, <strong>{{ usuario?.full_name || 'Estratega' }}</strong>!</p>
        <span class="leyenda-jornada">Tu Pronóstico de la Jornada</span>
        <button @click="cerrarSesion" class="btn-salir">Cerrar Sesión</button>
      </div>
    </header>    

      <div v-if="mostrarReglas" class="modal-fondo" @click.self="mostrarReglas = false">
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
    
    <div v-if="!sesionActiva" class="pantalla-login">
      <div class="login-tarjeta">
        <h2>¡Bienvenido al torneo!</h2>
        <p>Para registrar tus pronósticos y competir por la gloria, necesitas acceder.</p>
        
        <button @click="iniciarSesion" class="btn-google">
          Acceder con Google 🚀
        </button>

        <div class="enlaces-legales">
          <a href="#" @click.prevent="mostrarReglas = true">📜 Reglas del Torneo</a>
        </div>
      </div>
    </div>

    <div v-if="sesionActiva">   
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
        <h2 class="titulo-liga-activa">{{ ligaActual?.name }}</h2>

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
      
      
    </div>

  </main>
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

// Función rápida para crear una liga provisional
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
/* (El CSS se mantiene igual, solo agregamos el estilo del botón) */
.contenedor {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  background-color: #f8fafc;
  min-height: 100vh;
}

/* Estilos para el perfil y botón de salir */
.perfil-usuario {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
}
.btn-salir {
  background-color: #fee2e2;
  color: #ef4444;
  border: 1px solid #f87171;
  padding: 5px 15px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.2s;
}
.btn-salir:hover {
  background-color: #fecaca;
}
.perfil-usuario p {
  color: #1e293b; /* Un gris casi negro, muy nítido */
  font-size: 1.2rem; /* Un poco más grande */
  margin-bottom: 5px;
}
.perfil-usuario strong {
  color: #2563eb; /* El azul vibrante de tu tema */
  font-weight: 800; /* Negrita más gruesa */
}

.cabecera { text-align: center; margin-bottom: 30px; }
.cabecera h1 { color: #1e3a8a; font-size: 2.5rem; margin-bottom: 5px; }
.mensaje-carga { text-align: center; font-size: 1.2rem; color: #64748b; }

.tarjeta-partido { background-color: white; border-radius: 12px; padding: 15px 20px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); border: 1px solid #e2e8f0; }
.tarjeta-encabezado { display: flex; justify-content: space-between; font-size: 0.85rem; color: #64748b; margin-bottom: 15px; border-bottom: 1px solid #f1f5f9; padding-bottom: 8px; }
.fase { font-weight: bold; color: #2563eb; background-color: #dbeafe; padding: 2px 8px; border-radius: 4px; }
.tarjeta-cuerpo { display: flex; justify-content: space-between; align-items: center; }
/*.equipo { flex: 1; text-align: center; font-weight: bold; font-size: 1.1rem; color: #334155; }*/

.vs { font-size: 0.9rem; color: #94a3b8; font-weight: bold; }
.input-gol { width: 50px; height: 50px; text-align: center; font-size: 1.5rem; font-weight: bold; border: 2px solid #cbd5e1; border-radius: 8px; color: #1e293b; transition: border-color 0.2s; }
.input-gol:focus { outline: none; border-color: #3b82f6; }
.input-gol::-webkit-outer-spin-button, .input-gol::-webkit-inner-spin-button { -webkit-appearance: none; margin: 0; }

/* NUEVOS ESTILOS PARA EL LOGIN */
.pantalla-login {
  text-align: center;
  background: white;
  padding: 40px 20px;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.05);
  border: 1px solid #e2e8f0;
}
.pantalla-login h2 { color: #334155; margin-bottom: 10px; }
.pantalla-login p { color: #64748b; margin-bottom: 25px; }

.btn-google {
  background-color: #ffffff;
  color: #334155;
  border: 1px solid #cbd5e1;
  padding: 12px 24px;
  font-size: 1.1rem;
  font-weight: bold;
  border-radius: 8px;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
  transition: all 0.2s;
}
.btn-google:hover {
  background-color: #f8fafc;
  border-color: #94a3b8;
}

/* --- ESTILOS DE LA NUEVA PORTADA --- */
.pantalla-login {
  /* Imagen de estadio gratuita de alta calidad */
  background-image: linear-gradient(rgba(0, 0, 0, 0.6), rgba(0, 0, 0, 0.7)), url('https://images.unsplash.com/photo-1522778119026-d647f0596c20?ixlib=rb-4.0.3&auto=format&fit=crop&w=2000&q=80');  
  background-size: cover;
  background-position: center;
  border-radius: 12px;
  padding: 60px 20px;
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 400px;
  margin-top: 20px;
}

.login-tarjeta {
  background: rgba(255, 255, 255, 0.95);
  padding: 40px;
  border-radius: 16px;
  text-align: center;
  max-width: 400px;
  box-shadow: 0 10px 25px rgba(0,0,0,0.2);
}

.login-tarjeta h2 { color: #1e293b; margin-bottom: 15px; font-size: 1.8rem; }
.login-tarjeta p { color: #475569; margin-bottom: 25px; line-height: 1.5; }

.enlaces-legales { margin-top: 25px; }
.enlaces-legales a {
  color: #2563eb;
  text-decoration: none;
  font-size: 0.9rem;
  font-weight: bold;
  transition: color 0.2s;
}
.enlaces-legales a:hover { color: #1d4ed8; text-decoration: underline; }

/* --- ESTILOS DE LA VENTANA EMERGENTE (MODAL) --- */
.modal-fondo {
  position: fixed;
  top: 0; left: 0; width: 100%; height: 100%;
  background-color: rgba(0, 0, 0, 0.5); /* Fondo oscuro semitransparente */
  backdrop-filter: blur(4px); /* Efecto cristal */
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000; /* Para que quede por encima de todo */
}

.modal-contenido {
  background-color: white;
  padding: 30px;
  border-radius: 12px;
  max-width: 500px;
  width: 90%;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
}

.modal-contenido h3 { color: #1e3a8a; margin-bottom: 20px; border-bottom: 2px solid #f1f5f9; padding-bottom: 10px; }
.reglas-texto ul { text-align: left; color: #334155; line-height: 1.6; padding-left: 20px; margin-bottom: 25px; }
.reglas-texto li { margin-bottom: 10px; }

.btn-cerrar-modal {
  background-color: #1e293b;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 6px;
  font-weight: bold;
  cursor: pointer;
  width: 100%;
  transition: background-color 0.2s;
}
.btn-cerrar-modal:hover { background-color: #334155; }

/* --- ESTILOS NUEVOS PARA EL FEEDBACK VISUAL --- */
.leyenda-jornada {
  color: #64748b;
  font-size: 1rem;
  margin-bottom: 15px;
  font-weight: 500;
}

.badge-guardado {
  margin-left: 10px;
  color: #059669;
  background-color: #d1fae5;
  padding: 4px 8px;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: bold;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
  display: inline-block;
  animation: latido 0.3s ease-out;
}

/* Animación que hace que el botoncito brinque un poco al aparecer */
@keyframes latido {
  0% { transform: scale(0.8); opacity: 0; }
  50% { transform: scale(1.1); }
  100% { transform: scale(1); opacity: 1; }
}


/* --- CONTENEDOR RESPONSIVO --- */
.vista-partidos {
  max-width: 1200px; /* Le damos permiso de estirarse más en PC */
  margin: 0 auto;
  padding: 0 15px;
}

.grid-partidos {
  display: grid;
  /* Esta línea es la magia: tarjetas de mínimo 320px, y si sobra espacio, caben más en la misma fila */
  grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
  gap: 20px;
}

/* --- DISEÑO DE LA TARJETA --- */
.tarjeta-partido {
  background: white;
  border-radius: 12px;
  padding: 15px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.05);
  border: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
  gap: 15px;
  transition: transform 0.2s;
}

.tarjeta-partido:hover {
  box-shadow: 0 8px 15px rgba(0,0,0,0.1);
}

.tarjeta-encabezado {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
  border-bottom: 1px solid #f1f5f9;
  padding-bottom: 8px;
}

.fase { font-weight: bold; color: #2563eb; }

/* --- FILAS DE EQUIPOS Y CASILLEROS --- */
.cuerpo-partido {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.fila-equipo {
  display: flex;
  justify-content: space-between; /* Empuja el nombre a la izquierda y el input a la derecha */
  align-items: center;
  background-color: #f8fafc;
  padding: 8px 12px;
  border-radius: 8px;
}

.detalles-equipo {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 1.1rem;
  font-weight: 600;
  color: #1e293b;
}

.input-gol {
  width: 50px;
  height: 45px;
  text-align: center;
  font-size: 1.3rem;
  font-weight: bold;
  border: 2px solid #cbd5e1;
  border-radius: 8px;
  color: #0f172a;
  background: white;
  outline: none;
  transition: all 0.2s;
}

.input-gol:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.2);
}

/* Opcional: Oculta las flechitas de subir/bajar del input de números para que se vea más limpio */
.input-gol::-webkit-inner-spin-button, 
.input-gol::-webkit-outer-spin-button { 
  -webkit-appearance: none; 
  margin: 0; 
}


.vista-lobby { max-width: 800px; margin: 0 auto; }
.encabezado-lobby { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.btn-crear { background: #2563eb; color: white; border: none; padding: 10px 15px; border-radius: 8px; cursor: pointer; }
.grid-ligas { display: grid; gap: 15px; grid-template-columns: repeat(auto-fill, minmax(250px, 1fr)); }
.tarjeta-liga { background: white; padding: 20px; border-radius: 12px; border-left: 5px solid #2563eb; cursor: pointer; transition: transform 0.2s; box-shadow: 0 4px 6px rgba(0,0,0,0.05); text-align: left; }
.tarjeta-liga:hover { transform: translateY(-3px); }
.campeon-texto { font-size: 0.85rem; color: #64748b; margin-top: 10px; }
.btn-volver { background: transparent; border: 1px solid #cbd5e1; padding: 8px 12px; border-radius: 6px; cursor: pointer; margin-bottom: 20px; }
.titulo-liga-activa { color: #1e293b; margin-bottom: 30px; font-size: 2rem; }
</style>