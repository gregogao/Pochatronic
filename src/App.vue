<script setup>
import { reactive, computed, ref, onMounted, watch } from 'vue'

const game = reactive({
  phase: 'menu', 
  numPlayers: 0,
  players: [],
  rounds: [],
  startTime: null
})

const history = ref([])
const activeTab = ref('table') 

// --- LÓGICA DE PERSISTENCIA ---
const saveCurrentGame = () => {
  if (game.phase === 'playing') {
    localStorage.setItem('pocha_current_backup', JSON.stringify(game))
  }
}

const loadAllData = () => {
  const savedHistory = localStorage.getItem('pocha_history_list')
  if (savedHistory) history.value = JSON.parse(savedHistory)
  const backup = localStorage.getItem('pocha_current_backup')
  if (backup) {
    const data = JSON.parse(backup)
    Object.assign(game, data)
  }
}

watch(() => game.rounds, () => saveCurrentGame(), { deep: true })
onMounted(() => loadAllData())

// --- LÓGICA DE JUEGO ---
const dealerByRound = computed(() => {
  return game.rounds.map((_, roundIndex) => {
    const playerIndex = roundIndex % game.numPlayers
    const name = game.players[playerIndex]?.name
    // Reducimos a 2 letras si hay muchos jugadores para ahorrar espacio
    const length = game.numPlayers > 4 ? 2 : 3
    return name ? name.slice(0, length).toLowerCase() : '???'
  })
})

const hotPlayers = computed(() => {
  const hotIndices = new Set()
  game.players.forEach((_, pIndex) => {
    let streak = 0
    for (const round of game.rounds) {
      if (round.cards > 1) {
        if (Number(round.scores[pIndex]) === 10) {
          streak++
          if (streak >= 4) hotIndices.add(pIndex)
        } else {
          streak = 0
        }
      }
    }
  })
  return hotIndices
})

function initSetup(n) {
  game.numPlayers = n
  game.players = Array.from({ length: n }, (_, i) => ({ id: i, name: '' }))
  game.phase = 'setup'
}

function startGame() {
  if (game.players.some(p => !p.name.trim())) {
    alert('Todos los jugadores deben tener nombre')
    return
  }
  let maxCards = game.numPlayers === 3 ? 12 : game.numPlayers === 4 ? 9 : game.numPlayers === 5 ? 7 : 6
  const newRounds = []
  for (let i = 0; i < game.numPlayers; i++) {
    newRounds.push({ cards: 1, scores: Array(game.numPlayers).fill(null) })
  }
  for (let i = 2; i < maxCards; i++) {
    newRounds.push({ cards: i, scores: Array(game.numPlayers).fill(null) })
  }
  for (let i = 0; i < game.numPlayers; i++) {
    newRounds.push({ cards: maxCards, scores: Array(game.numPlayers).fill(null) })
  }
  game.rounds = newRounds
  game.startTime = new Date().toLocaleString('es-ES', { day:'2-digit', month:'2-digit', hour:'2-digit', minute:'2-digit' })
  game.phase = 'playing'
  saveCurrentGame()
}

const totals = computed(() => {
  return game.players.map((player, pIndex) => {
    let negativeRounds = 0
    let bestScore = 0
    const playerScores = game.rounds.map(r => Number(r.scores[pIndex]) || 0)
    const total = playerScores.reduce((sum, s) => {
      if (s < 0) negativeRounds++
      if (s > bestScore) bestScore = s
      return sum + s
    }, 0)
    return { 
      ...player, 
      total, 
      originalIndex: pIndex,
      statLine: `${negativeRounds}R / Max:${bestScore}`
    }
  })
})

const ranking = computed(() => {
  return [...totals.value].sort((a, b) => b.total - a.total)
})

function finishAndSave() {
  if (confirm("¿Finalizar y guardar?")) {
    const snapshot = {
      date: game.startTime || new Date().toLocaleString(),
      winner: ranking.value[0]?.name || '---',
      winnerScore: ranking.value[0]?.total || 0,
      players: game.players.map(p => p.name).join(', ')
    }
    history.value.unshift(snapshot)
    localStorage.setItem('pocha_history_list', JSON.stringify(history.value))
    exitGame()
  }
}

function finishWithoutSaving() {
  if (confirm("¿Cerrar sin guardar?")) exitGame()
}

function exitGame() {
  localStorage.removeItem('pocha_current_backup')
  game.phase = 'menu'
  game.numPlayers = 0
  game.players = []
  game.rounds = []
  activeTab.value = 'table'
}
</script>

<template>
  <div id="app" :class="[`phase-${game.phase}`, { 'is-playing': game.phase === 'playing' }]">
    <header v-if="game.phase !== 'playing'">
      <h1>Pochatronic™ ♠️</h1>
    </header>

    <main v-if="game.phase === 'menu'" class="setup-container">
      <div class="menu-actions">
        <button @click="game.phase = 'setup-players'" class="btn-main-large">🆕 NUEVA PARTIDA</button>
        <button @click="game.phase = 'history'" class="btn-secondary-large">📜 HISTORIAL</button>
      </div>
      <div v-if="game.rounds.length > 0" class="backup-alert" @click="game.phase = 'playing'">
        <p>📍 Partida en curso</p>
        <small>Toca para continuar</small>
      </div>
    </main>

    <main v-else-if="game.phase === 'history'" class="history-container">
      <h2>Historial</h2>
      <div v-for="(h, i) in history" :key="i" class="history-item">
        <div class="h-header"><span>{{ h.date }}</span> <strong>{{ h.winnerScore }} pts</strong></div>
        <div class="h-winner">🏆 {{ h.winner }}</div>
      </div>
      <button @click="game.phase = 'menu'" class="btn-back-menu">Volver</button>
    </main>

    <main v-else-if="game.phase === 'setup-players'" class="setup-container">
      <h2>¿Cuántos?</h2>
      <div class="grid-buttons">
        <button v-for="n in [3, 4, 5, 6]" :key="n" @click="initSetup(n)" class="btn-main">{{ n }}</button>
      </div>
    </main>

    <main v-else-if="game.phase === 'setup'" class="setup-container">
      <h2>Nombres</h2>
      <div v-for="player in game.players" :key="player.id" class="input-group">
        <input v-model="player.name" :placeholder="'Jugador ' + (player.id + 1)" />
      </div>
      <button @click="startGame" class="btn-start">▶️ Empezar</button>
    </main>

    <main v-else-if="game.phase === 'playing'" class="game-container">
      <div v-show="activeTab === 'table'" class="tab-content">
        <div class="table-wrapper">
          <table class="full-width-table" :class="'players-' + game.numPlayers">
            <thead>
              <tr>
                <th colspan="2" class="th-sticky-meta">REP.</th>
                <th v-for="(p, pIndex) in game.players" :key="p.id" class="th-player">
                  {{ p.name.slice(0, 3).toUpperCase() }}
                  <span v-if="hotPlayers.has(pIndex)">💩</span>
                </th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(round, rIndex) in game.rounds" :key="rIndex">
                <td class="td-cards th-sticky-c">{{ round.cards }}</td>
                <td class="td-dealer th-sticky-d">{{ dealerByRound[rIndex] }}</td>
                <td v-for="(score, pIndex) in round.scores" :key="pIndex" class="td-score">
                  <input type="number" v-model.number="round.scores[pIndex]" 
                         :class="{ 'is-negative': round.scores[pIndex] < 0 }" />
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div v-show="activeTab === 'ranking'" class="tab-content ranking-list">
        <div class="ranking-card" v-for="(player, index) in ranking" :key="player.id">
          <div class="rank-pos">{{ index + 1 }}</div>
          <div class="rank-info">
            <span class="rank-name">{{ player.name }}</span>
            <span class="rank-details">{{ player.statLine }}</span>
          </div>
          <div class="rank-score">{{ player.total }}</div>
        </div>
        <div class="ranking-actions">
          <button @click="finishAndSave" class="btn-reset">🔄 GUARDAR</button>
          <button @click="finishWithoutSaving" class="btn-finish-only">❌ Cerrar</button>
        </div>
      </div>

      <nav class="bottom-nav">
        <button :class="{ active: activeTab === 'table' }" @click="activeTab = 'table'">📝 PUNTOS</button>
        <button :class="{ active: activeTab === 'ranking' }" @click="activeTab = 'ranking'">🏆 RANKING</button>
      </nav>
    </main>
  </div>
</template>

<style>
:root {
  --col-cards: 28px;
  --col-dealer: 38px;
  --primary: #2c3e50; --accent: #3498db; --bg: #f4f7f6; --danger: #e74c3c;
  --nav-height: 60px;
}
body { margin: 0; background: var(--bg); font-family: sans-serif; overflow-x: hidden; }

/* OPTIMIZACIÓN TABLA PARA VERTICAL */
.table-wrapper { width: 100vw; overflow-x: auto; background: white; }
.full-width-table { width: 100%; border-collapse: separate; border-spacing: 0; }

/* Ajuste de columnas fijas (más estrechas) */
.th-sticky-meta { position: sticky; left: 0; z-index: 10; background: #3e4f5f !important; border-right: 1px solid #ccc; font-size: 0.6rem; }
.th-sticky-c { position: sticky; left: 0; z-index: 5; width: var(--col-cards); background: #f8f9fa !important; border-right: 1px solid #eee; }
.th-sticky-d { position: sticky; left: var(--col-cards); z-index: 5; width: var(--col-dealer); background: #f1f3f4 !important; border-right: 1px solid #ddd; }

/* Reglas específicas cuando hay 5 o 6 jugadores en móvil vertical */
@media screen and (max-width: 480px) {
  .players-5 .th-player, .players-5 .td-score,
  .players-6 .th-player, .players-6 .td-score {
    min-width: 42px !important;
    font-size: 0.65rem;
  }
  
  .players-5 input, .players-6 input {
    width: 38px !important;
    height: 32px !important;
    font-size: 0.9rem !important;
    padding: 0;
  }

  .th-sticky-meta { width: 60px; }
  .td-cards, .td-dealer { font-size: 0.65rem !important; }
}

/* ESTILOS DE COMPONENTES */
th { background: var(--primary); color: white; padding: 8px 2px; }
td { border-bottom: 1px solid #eee; padding: 5px 0; text-align: center; }
input[type=number] { width: 45px; height: 35px; border: 1px solid #ddd; border-radius: 4px; text-align: center; font-size: 1rem; }
input.is-negative { background: #fee2e2; color: var(--danger); }

/* NAVEGACIÓN Y OTROS */
.bottom-nav { position: fixed; bottom: 0; width: 100%; display: flex; background: white; height: var(--nav-height); border-top: 1px solid #ddd; z-index: 100; }
.bottom-nav button { flex: 1; border: none; background: none; font-weight: bold; color: #999; }
.bottom-nav button.active { color: var(--accent); background: #f0f9ff; }

.setup-container { padding: 20px; text-align: center; }
.btn-main-large { width: 100%; padding: 20px; background: var(--accent); color: white; border: none; border-radius: 10px; margin-bottom: 10px; font-weight: bold; }
.grid-buttons { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
.btn-main { padding: 20px; border: 2px solid var(--accent); border-radius: 10px; background: white; font-weight: bold; }
.ranking-card { display: flex; padding: 15px; background: white; margin: 10px; border-radius: 10px; align-items: center; }
.rank-pos { width: 30px; font-weight: bold; color: var(--accent); }
.rank-info { flex: 1; display: flex; flex-direction: column; }
.rank-name { font-weight: bold; }
.rank-details { font-size: 0.7rem; color: #888; }
.rank-score { font-size: 1.2rem; font-weight: bold; }
</style>