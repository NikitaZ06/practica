<template>
  <!-- Основной контейнер симуляции -->
  <div class="world-container">
    <!-- Холст, на котором всё рисуется -->
    <canvas ref="canvasRef" :width="width" :height="height" class="world"></canvas>

    <!-- Панель управления параметрами (реактивно связана с настройками) -->
    <div class="control-panel">
      <h3>Параметры</h3>

      <!-- Ползунки для динамического изменения характеристик -->
      <label>Скорость муравьёв: {{ cfg.antSpeed.toFixed(1) }}</label>
      <input type="range" v-model.number="cfg.antSpeed" min="0.1" max="2.5" step="0.1" />

      <label>Радиус восприятия: {{ cfg.perceptionRadius }}</label>
      <input type="range" v-model.number="cfg.perceptionRadius" min="10" max="80" step="1" />

      <label>Скорость пауков: {{ cfg.enemySpeed.toFixed(1) }}</label>
      <input type="range" v-model.number="cfg.enemySpeed" min="0.1" max="2" step="0.1" />

      <label>Радиус атаки пауков: {{ cfg.enemyAttackRadius }}</label>
      <input type="range" v-model.number="cfg.enemyAttackRadius" min="5" max="30" step="1" />

      <label>Радиус атаки солдат: {{ cfg.soldierAttackRadius }}</label>
      <input type="range" v-model.number="cfg.soldierAttackRadius" min="10" max="50" step="1" />

      <label>Макс. врагов: {{ cfg.maxEnemies }}</label>
      <input type="range" v-model.number="cfg.maxEnemies" min="0" max="10" step="1" />

      <label>Интервал яйцекладки (сек): {{ cfg.queenEggInterval }}</label>
      <input type="range" v-model.number="cfg.queenEggInterval" min="1" max="8" step="0.5" />

      <hr />
      <label><input type="checkbox" v-model="autoBuild" /> Автостройка камер</label>
      <label>Стоимость камеры: {{ buildCost }}</label>
      <input type="range" v-model.number="buildCost" min="5" max="30" step="1" />
      <label>Время постройки (сек): {{ buildTimeSec }}</label>
      <input type="range" v-model.number="buildTimeSec" min="2" max="15" step="0.5" />

      <hr />
      <label>Макс. ферм с тлёй: {{ cfg.maxFarms }}</label>
      <input type="range" v-model.number="cfg.maxFarms" min="0" max="5" step="1" />
      <label>Интервал дойки в ферме (сек): {{ cfg.farmMilkInterval }}</label>
      <input type="range" v-model.number="cfg.farmMilkInterval" min="1" max="10" step="0.5" />
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted } from 'vue'

//  ГЛОБАЛЬНЫЕ ПЕРЕМЕННЫЕ И КОНСТАНТЫ 
const width = window.innerWidth
const height = window.innerHeight
const canvasRef = ref(null)
let ctx                      // Контекст рисования canvas
const nest = { x: width / 2, y: height / 2 }   // Центр гнезда

// Реактивная конфигурация (настройки, меняемые ползунками)
const cfg = reactive({
  antSpeed: 1.2,            // Скорость движения муравьёв (пикселей/кадр при 60 fps)
  perceptionRadius: 40,     // Дальность обзора муравья (еда, феромоны)
  foodPickupRadius: 10,     // Расстояние, на котором муравей подбирает еду
  enemySpeed: 0.8,          // Скорость пауков
  enemyAttackRadius: 15,    // Радиус атаки паука (пиксели)
  soldierAttackRadius: 30,  // Радиус атаки солдата (пиксели)
  maxEnemies: 6,            // Максимальное число врагов на поле
  queenEggInterval: 3,      // Интервал кладки яиц маткой (сек)
  maxFarms: 3,              // Максимальное количество подземных ферм для тли
  farmMilkInterval: 3       // Интервал сбора пади с тли в ферме (сек)
})

// Отдельные реактивные настройки строительства
const autoBuild = ref(true)  // Автоматически строить новые камеры?
const buildCost = ref(10)    // Трата еды на старт постройки
const buildTimeSec = ref(5)  // Длительность постройки (секунд)

// константы (не меняются во время симуляции)
const NEST_SIZE = 35         // Радиус наземного гнезда
const ANT_SIZE = 5           // Размер муравья (по сути радиус тела)
const CHAMBER_RADIUS = 18    // Радиус подземной камеры
const CHAMBER_CAPACITY = 5   // Вместимость одной камеры (сколько яиц/личинок/куколок можно хранить)
const SUBTERRANEAN_BASE_Y = NEST_SIZE + 20   // Сдвиг начала подземелья относительно центра гнезда (ниже гнезда)
const CHAMBER_Y_OFFSET = 40  // Начальное смещение первой камеры от центра гнезда вниз (центр первой камеры)
const SUBTERRANEAN_CENTER = { x: nest.x, y: nest.y + CHAMBER_Y_OFFSET }  // Центр подземной зоны (для блуждания рабочих)
const MIN_CHAMBER_DISTANCE = CHAMBER_RADIUS * 2 + 5  // Минимальное расстояние между центрами камер, чтобы не накладывались

//  ИГРОВЫЕ МАССИВЫ И ПЕРЕМЕННЫЕ 
let foods = []            // Источники еды: { id, x, y, amount, maxAmount, aphids }
let ants = []             // Муравьи: { id, x, y, angle, hasFood, role, lastPheromoneTime, lastFeedTime, lastAttackTime, roleAssignedTime, lastMilkingTime }
let pheromones = []       // Феромоны: { x, y, strength, angle }
let enemies = []          // Пауки: { id, x, y, angle, health, maxHealth, lastAttackTime }
let chambers = []         // Подземные камеры: { id, x, y, type: 'brood' | 'farm', brood: [], aphids?: number, building? }
let tunnels = []          // Туннели между камерами: { from, to }
let colonyFood = 20       // Запас еды в муравейнике
let lastEggTime = 0       // Время последней кладки яйца (timestamp)
let lastRoleReassign = 0  // Время последнего пересмотра ролей (timestamp)
let lastFrameTime = 0     // Время предыдущего кадра (timestamp)

let foodRespawnTimer      // Таймер появления новой еды
let animationId           // ID requestAnimationFrame

//  ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ 
// Расстояние между двумя точками (евклидово)
const dist = (x1, y1, x2, y2) => Math.hypot(x2 - x1, y2 - y1)
// Ограничение числа диапазоном
const clamp = (v, min, max) => Math.max(min, Math.min(max, v))
// Случайное число в интервале [min, max)
const rnd = (min, max) => Math.random() * (max - min) + min
// Плавный поворот угла от a к b с коэффициентом t (0 – без поворота, 1 – мгновенный поворот)
const lerpAngle = (a, b, t) => {
  let diff = b - a
  // Учитываем переход через PI, чтобы поворот всегда был кратчайшим
  while (diff > Math.PI) diff -= 2 * Math.PI
  while (diff < -Math.PI) diff += 2 * Math.PI
  return a + diff * t
}
// Случайная позиция внутри наземного гнезда
const randPosInNest = () => ({
  x: nest.x + rnd(-NEST_SIZE / 2, NEST_SIZE / 2),
  y: nest.y + rnd(-NEST_SIZE / 2, NEST_SIZE / 2)
})
// Случайная позиция внутри подземной камеры (примерно по площади)
const randChamberPos = (ch) => ({
  x: ch.x + rnd(-CHAMBER_RADIUS * 0.7, CHAMBER_RADIUS * 0.7),
  y: ch.y + rnd(-CHAMBER_RADIUS * 0.7, CHAMBER_RADIUS * 0.7)
})

//  ИНИЦИАЛИЗАЦИЯ ИГРОВЫХ ОБЪЕКТОВ 
// Создание источников еды (некоторые с тлёй на поверхности)
function initFoods() {
  foods = []
  for (let i = 0; i < 8; i++) addRandomFoodSource()
}

// Добавление одного случайного источника еды
function addRandomFoodSource() {
  let x, y
  do {
    x = rnd(20, width - 20)
    y = rnd(20, height - 20)
  } while (dist(x, y, nest.x, nest.y) < NEST_SIZE + 30) // не слишком близко к гнезду

  // Тля появляется с некоторым шансом (30% по умолчанию)
  const hasAphids = Math.random() < 0.3
  foods.push({
    id: Date.now() + Math.random(),
    x, y,
    amount: rnd(20, 50),            // текущий запас
    maxAmount: 50,                  // максимальный запас (для размера)
    aphids: hasAphids ? Math.floor(rnd(1, 4)) : 0   // количество тли (0-3)
  })
}

// Создание врагов 
function initEnemies() {
  enemies = []
  for (let i = 0; i < 3; i++) addRandomEnemy()
}

// Добавление одного паука в случайном месте
function addRandomEnemy() {
  let x, y
  do {
    x = rnd(20, width - 20)
    y = rnd(20, height - 20)
  } while (dist(x, y, nest.x, nest.y) < NEST_SIZE + 60) // не возле гнезда
  enemies.push({
    id: Date.now() + Math.random(),
    x, y,
    angle: rnd(0, 2 * Math.PI),
    health: 5,
    maxHealth: 5,
    lastAttackTime: 0
  })
}

// Создание подземных камер (одна главная для расплода)
function initChambers() {
  chambers = [{ id: 0, x: nest.x, y: nest.y + CHAMBER_Y_OFFSET, type: 'brood', brood: [] }]
  tunnels = []
}

// Добавление нового муравья с заданной ролью и позицией
function addAnt(role, x, y) {
  ants.push({
    id: Date.now() + Math.random(),
    x, y,
    angle: rnd(0, 2 * Math.PI),
    hasFood: false,
    role,
    lastPheromoneTime: 0,
    lastFeedTime: 0,
    lastAttackTime: 0,
    roleAssignedTime: performance.now(),
    lastMilkingTime: 0       // время последней дойки (для фуражиров)
  })
}

// Инициализация муравьёв: фуражиры и солдаты в гнезде, рабочие в главной камере
function initAnts() {
  ants = []
  const main = chambers[0]
  const foragers = Math.floor(30 * 0.6)    // 60% фуражиров
  const workers = Math.floor(30 * 0.25)    // 25% рабочих
  const soldiers = 30 - foragers - workers  // оставшиеся – солдаты

  for (let i = 0; i < foragers; i++) addAnt('forager', ...Object.values(randPosInNest()))
  for (let i = 0; i < soldiers; i++) addAnt('soldier', ...Object.values(randPosInNest()))
  for (let i = 0; i < workers; i++) addAnt('worker', ...Object.values(randChamberPos(main)))
}

// Попытка добавить расплод (яйцо/личинку/куколку) в первую доступную камеру для расплода
function addBroodToChamber(item) {
  for (const ch of chambers) {
    // Расплод можно класть только в камеры типа 'brood'
    if (ch.type === 'brood' && ch.brood.length < CHAMBER_CAPACITY) {
      ch.brood.push({
        ...item,
        offX: rnd(-CHAMBER_RADIUS * 0.7, CHAMBER_RADIUS * 0.7),
        offY: rnd(-CHAMBER_RADIUS * 0.7, CHAMBER_RADIUS * 0.7)
      })
      return true
    }
  }
  return false
}

// Проверка, не слишком ли близко новая позиция к уже существующим камерам или строящимся
function isPositionFree(x, y) {
  for (const ch of chambers) {
    if (dist(x, y, ch.x, ch.y) < MIN_CHAMBER_DISTANCE) return false
    // Также проверяем строящиеся камеры, чтобы не выбрать место рядом с будущей камерой
    if (ch.building && dist(x, y, ch.building.newX, ch.building.newY) < MIN_CHAMBER_DISTANCE) return false
  }
  return true
}

// Начать строительство новой камеры указанного типа (по умолчанию 'brood')
// Возвращает true, если строительство начато, иначе false
function startBuilding(type = 'brood') {
  if (colonyFood < buildCost.value || !autoBuild.value) return false

  // Найти родительскую камеру без активного строительства
  const available = chambers.find(c => !c.building) 
  if (!available) return false

  // Попытаемся найти место для новой камеры (до 10 попыток)
  for (let attempt = 0; attempt < 10; attempt++) {
    const angle = rnd(0, 2 * Math.PI)
    const distance = rnd(50, 100)  // чуть увеличим диапазон, чтобы было больше шансов
    const newX = available.x + Math.cos(angle) * distance
    const newY = available.y + Math.sin(angle) * distance

    if (isPositionFree(newX, newY)) {
      // Место подходит – начинаем строительство
      available.building = {
        progress: 0,
        newX,
        newY,
        newId: Date.now(),
        newType: type
      }
      colonyFood -= buildCost.value  // сразу тратим еду
      return true
    }
  }

  // Не удалось найти место – строительство отменяется
  return false
}

//  ПЕРЕРАСПРЕДЕЛЕНИЕ РОЛЕЙ 
// Вызывается каждые 5 секунд, подстраивает численность солдат и рабочих под нужды колонии
function reassignRoles(ts) {
  if (ts - lastRoleReassign < 5000) return  // не чаще раза в 5 секунд
  lastRoleReassign = ts

  const totalAnts = ants.length
  if (!totalAnts) return

  // Считаем потребности: сколько личинок, сколько врагов
  const larvae = chambers.reduce((s, c) => s + c.brood.filter(b => b.type === 'larva').length, 0)
  const enemyCount = enemies.length

  // Целевое количество солдат: 0 если врагов нет, иначе минимум 2, максимум 30% от всех
  const targetSoldiers = enemyCount
    ? Math.min(Math.floor(totalAnts * 0.3), Math.max(2, Math.ceil(enemyCount * 1.5)))
    : 0
  // Целевое количество рабочих: зависит от числа личинок (минимум 1 дежурный)
  const targetWorkers = larvae
    ? Math.min(Math.floor(totalAnts * 0.4), Math.ceil(larvae / 3)) || 1
    : 1

  // Текущие количества
  const cur = {
    s: ants.filter(a => a.role === 'soldier').length,
    w: ants.filter(a => a.role === 'worker').length,
    f: ants.filter(a => a.role === 'forager').length
  }

  // Можно ли муравью менять роль (не несёт еду, не менял роль последние 3 сек)
  const canChange = (a) => !a.hasFood && ts - (a.roleAssignedTime || 0) > 3000

  // Вспомогательная смена роли: ищет подходящего муравья роли `from` и меняет на `to`
  // mover — функция, возвращающая новую позицию (например, переселение в камеру)
  const swap = (from, to, mover) => {
    const candidate = ants.find(a => a.role === from && canChange(a))
    if (!candidate) return false
    candidate.role = to
    candidate.hasFood = false
    candidate.roleAssignedTime = ts
    if (mover) { const pos = mover(); candidate.x = pos.x; candidate.y = pos.y }
    return true
  }

  // Увеличиваем солдат за счёт фуражиров
  while (cur.s < targetSoldiers && cur.f > 0) {
    if (swap('forager', 'soldier', randPosInNest)) { cur.s++; cur.f-- } else break
  }
  // Увеличиваем рабочих за счёт фуражиров
  while (cur.w < targetWorkers && cur.f > 0) {
    if (swap('forager', 'worker', () => randChamberPos(chambers[0]))) { cur.w++; cur.f-- } else break
  }
  // Лишних солдат превращаем в фуражиров
  while (cur.s > targetSoldiers) {
    if (swap('soldier', 'forager', randPosInNest)) { cur.s--; cur.f++ } else break
  }
  // Лишних рабочих превращаем в фуражиров
  while (cur.w > targetWorkers) {
    if (swap('worker', 'forager', randPosInNest)) { cur.w--; cur.f++ } else break
  }
}

// ГЛАВНЫЙ ИГРОВОЙ ЦИКЛ 
function update(ts) {
  if (!lastFrameTime) lastFrameTime = ts
  const dt = ts - lastFrameTime   // дельта времени с предыдущего кадра (в миллисекундах)
  lastFrameTime = ts

  // 1. Испарение феромонов (уменьшаем силу, удаляем почти невидимые)
  pheromones = pheromones
    .map(p => ({ ...p, strength: p.strength - 0.008 }))
    .filter(p => p.strength > 0.05)

  // 2. Размножение тли в подземных фермах и на поверхности
  // В фермах: каждая тля может породить новую с малым шансом, но не превышая 5 особей на камеру
  for (const ch of chambers) {
    if (ch.type === 'farm' && ch.aphids !== undefined && ch.aphids < 5 && Math.random() < 0.002) {
      ch.aphids++
    }
  }
  // На поверхности: тля тоже медленно размножается
  for (const f of foods) {
    if (f.aphids > 0 && f.aphids < 5 && Math.random() < 0.002) {
      f.aphids++
    }
  }

  // 3. Матка откладывает яйца с заданным интервалом, тратя еду
  if (ts - lastEggTime > cfg.queenEggInterval * 1000 && colonyFood >= 5) {
    colonyFood -= 5
    lastEggTime = ts
    // Пытаемся разместить яйцо; если нет места – инициируем постройку камеры для расплода
    if (!addBroodToChamber({ id: Date.now() + Math.random(), type: 'egg', spawnTime: ts })) {
      startBuilding('brood')
    }
  }

  // 4. Развитие расплода в камерах (только в 'brood')
  for (const ch of chambers) {
    if (ch.type !== 'brood') continue
    for (let i = ch.brood.length - 1; i >= 0; i--) {
      const b = ch.brood[i]
      if (b.type === 'egg' && ts - b.spawnTime > 8000) {
        // Яйцо становится личинкой (через 8 сек)
        ch.brood[i] = { ...b, type: 'larva', hunger: 10, feedCount: 0, lastFedTime: ts }
      } else if (b.type === 'larva') {
        // Голодание личинки (линейно теряет сытость, умирает при 0)
        b.hunger = 10 - (ts - b.lastFedTime) / 15000 * 10
        if (b.hunger <= 0) { ch.brood.splice(i, 1); continue }
        // Если достаточно кормлений – окукливание (после 5 кормлений)
        if (b.feedCount >= 5) ch.brood[i] = { id: b.id, type: 'pupa', spawnTime: ts }
      } else if (b.type === 'pupa' && ts - b.spawnTime > 12000) {
        // Куколка становится взрослым муравьём (через 12 сек)
        addAnt('forager', ...Object.values(randPosInNest()))
        ch.brood.splice(i, 1)
      }
    }
  }

  // 5. Строительство камер (обновление прогресса)
  for (const ch of chambers) {
    if (ch.building) {
      // Прогресс строительства зависит от реального времени (buildTimeSec)
      ch.building.progress += dt / (buildTimeSec.value * 1000)
      if (ch.building.progress >= 1) {
        // Завершение – создаём новую камеру с заданным типом
        const newCh = {
          id: ch.building.newId,
          x: ch.building.newX,
          y: ch.building.newY,
          type: ch.building.newType || 'brood',
          brood: [],
          aphids: ch.building.newType === 'farm' ? 1 : undefined   // в новой ферме изначально 1 тля
        }
        chambers.push(newCh)
        tunnels.push({ from: ch.id, to: newCh.id })
        ch.building = null
      }
    }
  }

  // 6. Перераспределение ролей (солдаты/рабочие/фуражиры)
  reassignRoles(ts)

  // 7. Обновление врагов (пауков)
  enemies = enemies.map(en => {
    let { x, y, angle } = en
    // Поиск ближайшего муравья в радиусе атаки
    let closestAnt = null
    let closestDist = cfg.enemyAttackRadius
    for (const a of ants) {
      const d = dist(x, y, a.x, a.y)
      if (d < closestDist) { closestDist = d; closestAnt = a }
    }

    if (closestAnt) {
      // Атака муравья, если прошло достаточно времени (каждые 600 мс)
      if (ts - en.lastAttackTime > 600) {
        ants = ants.filter(a => a.id !== closestAnt.id)
        en.lastAttackTime = ts
      }
      // Двигаемся к цели
      angle = Math.atan2(closestAnt.y - y, closestAnt.x - x)
    } else {
      // Случайное блуждание с избеганием гнезда (чтобы не «залипал» в гнезде)
      angle += rnd(-0.15, 0.15)
      if (dist(x, y, nest.x, nest.y) < NEST_SIZE + 15) {
        angle = Math.atan2(y - nest.y, x - nest.x)
      }
    }

    // Перемещение с отражением от стен (как бильярдный шар)
    let nx = x + Math.cos(angle) * cfg.enemySpeed
    let ny = y + Math.sin(angle) * cfg.enemySpeed
    if (nx < 15 || nx > width - 15) {
      angle = Math.PI - angle          // отражение по горизонтали
      nx = clamp(nx, 15, width - 15)
      angle += rnd(-0.2, 0.2)         // небольшой шум, чтобы не зацикливался
    }
    if (ny < 15 || ny > height - 15) {
      angle = -angle                   // отражение по вертикали
      ny = clamp(ny, 15, height - 15)
      angle += rnd(-0.2, 0.2)
    }

    return { ...en, x: nx, y: ny, angle }
  })

  // 8. Атаки солдат
  for (const a of ants) {
    if (a.role !== 'soldier') continue
    let closestEnemy = null
    let closestDist = cfg.soldierAttackRadius
    for (const en of enemies) {
      const d = dist(a.x, a.y, en.x, en.y)
      if (d < closestDist) { closestDist = d; closestEnemy = en }
    }
    if (closestEnemy && ts - a.lastAttackTime > 800) {
      closestEnemy.health--   // солдат наносит 1 урона
      a.lastAttackTime = ts
      a.angle = Math.atan2(closestEnemy.y - a.y, closestEnemy.x - a.x)
    }
  }
  enemies = enemies.filter(e => e.health > 0)  // удаляем мёртвых пауков

  // 9. Появление новых врагов, если их меньше максимума
  if (enemies.length < cfg.maxEnemies && Math.random() < 0.001) {
    addRandomEnemy()
  }

  // 10. Обновление муравьёв (фуражиры, рабочие, солдаты) – самая объёмная часть логики
  ants = ants.map(a => {
    let { x, y, angle, hasFood, role, lastPheromoneTime, lastFeedTime } = a

    if (role === 'worker') {
      // ----- РАБОЧИЙ -----
      let fed = false
      // Приоритет – кормление личинок (если есть еда и прошёл интервал 1500 мс)
      if (colonyFood >= 1 && ts - lastFeedTime > 1500) {
        for (const ch of chambers) {
          if (ch.type !== 'brood') continue
          const larva = ch.brood.find(b => b.type === 'larva' && b.hunger < 10)
          if (larva) {
            const d = dist(x, y, ch.x, ch.y)
            if (d < CHAMBER_RADIUS + 5) {
              // Кормление личинки: восстанавливает сытость и увеличивает счётчик кормлений
              larva.lastFedTime = ts
              larva.hunger = 10
              larva.feedCount++
              colonyFood--
              lastFeedTime = ts
              fed = true
              break
            } else {
              // Двигаемся к камере с голодной личинкой (чуть быстрее, чем блуждание)
              angle = lerpAngle(angle, Math.atan2(ch.y - y, ch.x - x), 0.1)
              x += Math.cos(angle) * cfg.antSpeed * 0.7
              y += Math.sin(angle) * cfg.antSpeed * 0.7
            }
          }
        }
      }

      // Если не кормили личинок, пытаемся подоить тлю в фермах
      if (!fed && colonyFood >= 0) {  // дойка бесплатна
        for (const ch of chambers) {
          if (ch.type === 'farm' && ch.aphids > 0) {
            const d = dist(x, y, ch.x, ch.y)
            if (d < CHAMBER_RADIUS + 5) {
              // Дойка тли: получаем 1 еды, если прошёл интервал farmMilkInterval
              if (ts - a.lastMilkingTime > cfg.farmMilkInterval * 1000) {
                colonyFood++
                a.lastMilkingTime = ts
              }
              // Рабочий остаётся в камере, медленно блуждает
              angle += rnd(-0.3, 0.3)
              x += Math.cos(angle) * 0.3
              y += Math.sin(angle) * 0.3
              fed = true
              break
            } else {
              // Двигаемся к ферме
              angle = lerpAngle(angle, Math.atan2(ch.y - y, ch.x - x), 0.1)
              x += Math.cos(angle) * cfg.antSpeed * 0.7
              y += Math.sin(angle) * cfg.antSpeed * 0.7
            }
          }
        }
      }

      if (!fed) {
        // Если никого не кормили – свободное блуждание по подземной зоне
        angle += rnd(-0.15, 0.15)
        const nx = x + Math.cos(angle) * cfg.antSpeed * 0.5
        const ny = y + Math.sin(angle) * cfg.antSpeed * 0.5
        // Не уходить далеко от центра подземелья (радиус 150 пикселей)
        if (dist(nx, ny, SUBTERRANEAN_CENTER.x, SUBTERRANEAN_CENTER.y) > 150) {
          angle = Math.atan2(SUBTERRANEAN_CENTER.y - ny, SUBTERRANEAN_CENTER.x - nx)
        }
        x = nx
        y = ny
      }

      // Инициирование строительства, если все камеры для расплода заполнены
      if (
        autoBuild.value &&
        !chambers.some(c => c.building) &&
        chambers.filter(c => c.type === 'brood').reduce((s, c) => s + c.brood.length, 0) >= chambers.filter(c => c.type === 'brood').length * CHAMBER_CAPACITY &&
        colonyFood >= buildCost.value
      ) {
        startBuilding('brood')
      }
    } else if (role === 'soldier') {
      // ----- СОЛДАТ -----
      let closestEnemy = null
      let closestDist = cfg.soldierAttackRadius * 1.5  // зона обнаружения для патруля чуть больше
      for (const en of enemies) {
        const d = dist(x, y, en.x, en.y)
        if (d < closestDist) { closestDist = d; closestEnemy = en }
      }

      if (closestEnemy) {
        // Преследует врага с более высокой скоростью
        angle = lerpAngle(angle, Math.atan2(closestEnemy.y - y, closestEnemy.x - x), 0.1)
        x += Math.cos(angle) * cfg.antSpeed * 0.8
        y += Math.sin(angle) * cfg.antSpeed * 0.8
      } else {
        // Патруль по кругу вокруг гнезда (радиус NEST_SIZE * 1.7)
        const patrolR = NEST_SIZE * 1.7
        const dxN = x - nest.x
        const dyN = y - nest.y
        const distN = Math.hypot(dxN, dyN)
        // Движение по касательной к окружности
        const tangent = Math.atan2(dyN, dxN) + Math.PI / 2
        angle = tangent + rnd(-0.05, 0.05)
        // Коррекция, если отклонился от радиуса патруля
        if (distN > patrolR + 2) angle = Math.atan2(-dyN, -dxN)
        else if (distN < patrolR - 2) angle = Math.atan2(dyN, dxN)
        x += Math.cos(angle) * cfg.antSpeed * 0.6
        y += Math.sin(angle) * cfg.antSpeed * 0.6
      }
      hasFood = false   // солдаты не носят еду
    } else {
      // ----- ФУРАЖИР -----
      if (hasFood) {
        // Несёт еду домой
        if (dist(x, y, nest.x, nest.y) < NEST_SIZE / 2) {
          colonyFood++   // сдал еду в хранилище
          hasFood = false
        } else {
          // Двигается к гнезду
          angle = lerpAngle(angle, Math.atan2(nest.y - y, nest.x - x), 0.08)
          x += Math.cos(angle) * cfg.antSpeed
          y += Math.sin(angle) * cfg.antSpeed
          // Оставляет феромонный след (каждые 100 мс)
          if (ts - lastPheromoneTime > 100) {
            pheromones.push({ x, y, strength: 1, angle })
            lastPheromoneTime = ts
          }
        }
      } else {
        // Поиск ближайшей еды в радиусе восприятия
        let closestFoodIdx = -1
        let closestFoodDist = cfg.perceptionRadius
        for (let i = 0; i < foods.length; i++) {
          const d = dist(x, y, foods[i].x, foods[i].y)
          if (d < closestFoodDist) { closestFoodDist = d; closestFoodIdx = i }
        }

        if (closestFoodIdx !== -1) {
          const f = foods[closestFoodIdx]
          // Находится ли муравей в радиусе доступа к источнику?
          if (closestFoodDist < cfg.foodPickupRadius) {
            // Если на источнике есть тля и можем построить ферму (число ферм < максимум)
            if (f.aphids > 0 && chambers.filter(c => c.type === 'farm').length < cfg.maxFarms && colonyFood >= buildCost.value && !chambers.some(c => c.building)) {
              // Забираем 1 тлю с поверхности для строительства фермы
              f.aphids--
              // Инициируем строительство фермы (функция startBuilding проверит возможность)
              startBuilding('farm')
              // Муравей не собирает обычную еду, он занят переносом тли (запускает стройку)
            } else {
              // Обычный сбор еды (истощение источника)
              f.amount--
              if (f.amount <= 0) {
                foods.splice(closestFoodIdx, 1)  // источник исчерпан
              }
              hasFood = true
              angle = Math.atan2(nest.y - y, nest.x - x)
            }
          } else {
            // Двигаемся к источнику с адаптивной скоростью (чем ближе, тем точнее поворот и медленнее скорость)
            const target = Math.atan2(f.y - y, f.x - x)
            const ratio = closestFoodDist / cfg.foodPickupRadius
            angle = lerpAngle(angle, target, 0.05 + 0.15 / ratio)
            x += Math.cos(angle) * cfg.antSpeed * Math.min(1, ratio * 0.8)
            y += Math.sin(angle) * cfg.antSpeed * Math.min(1, ratio * 0.8)
          }
        } else {
          // Еды не видно – следуем за феромонами или блуждаем
          let sumX = 0, sumY = 0
          for (const p of pheromones) {
            const d = dist(x, y, p.x, p.y)
            if (d < cfg.perceptionRadius) {
              const rev = p.angle + Math.PI   // идём против направления следа (к еде)
              const w = p.strength / (d + 1)
              sumX += Math.cos(rev) * w
              sumY += Math.sin(rev) * w
            }
          }
          if (sumX || sumY) {
            angle = lerpAngle(angle, Math.atan2(sumY, sumX), 0.05)
          } else {
            angle += rnd(-0.2, 0.2)  // случайное блуждание
          }
          x += Math.cos(angle) * cfg.antSpeed
          y += Math.sin(angle) * cfg.antSpeed
        }
      }
      // Отражение от краёв экрана для фуражира (оставлено для плавности движения)
      const m = 5
      if (x < m || x > width - m) { angle = Math.PI - angle; x = clamp(x, m, width - m); angle += rnd(-0.15, 0.15) }
      if (y < m || y > height - m) { angle = -angle; y = clamp(y, m, height - m); angle += rnd(-0.15, 0.15) }
    }

    // Принудительное ограничение координат всех муравьёв, чтобы никто не пропадал за границами
    x = clamp(x, 0, width)
    y = clamp(y, 0, height)

    return { ...a, x, y, angle, hasFood, role, lastPheromoneTime, lastFeedTime, lastAttackTime: a.lastAttackTime, lastMilkingTime: a.lastMilkingTime }
  })

  // 11. Пополнение еды, если источников мало
  if (foods.length < 15 && Math.random() < 0.005) addRandomFoodSource()

  draw()
  animationId = requestAnimationFrame(update)
}

//  ОТРИСОВКА 
function draw() {
  ctx.clearRect(0, 0, width, height)
  ctx.fillStyle = '#3b5e2b'
  ctx.fillRect(0, 0, width, height)  // фон

  // --- Феромоны ---
  for (const p of pheromones) {
    ctx.beginPath()
    ctx.moveTo(p.x, p.y)
    ctx.lineTo(p.x + Math.cos(p.angle) * 3, p.y + Math.sin(p.angle) * 3)
    ctx.strokeStyle = `rgba(180,220,255,${Math.min(p.strength, 0.6)})`
    ctx.lineWidth = 1.5
    ctx.stroke()
  }

  //  Источники еды (с тлёй или без) 
  for (const f of foods) {
    const r = 5 + (f.amount / f.maxAmount) * 8
    const hue = 120 * (f.amount / f.maxAmount)  // цвет от зелёного (полный) до красного (пустой)
    ctx.beginPath()
    ctx.arc(f.x, f.y, r, 0, 2 * Math.PI)
    ctx.fillStyle = `hsl(${hue},80%,50%)`
    ctx.fill()

    // Рисуем тлю на поверхности, если есть (маленькие зелёные точки вокруг)
    if (f.aphids > 0) {
      ctx.fillStyle = '#a0ffa0'
      for (let i = 0; i < f.aphids; i++) {
        const angle = (i / f.aphids) * 2 * Math.PI
        const ax = f.x + Math.cos(angle) * (r + 4)
        const ay = f.y + Math.sin(angle) * (r + 4)
        ctx.beginPath()
        ctx.arc(ax, ay, 1.5, 0, 2 * Math.PI)
        ctx.fill()
      }
    }
  }

  //  Наземное гнездо и матка 
  ctx.beginPath()
  ctx.arc(nest.x, nest.y, NEST_SIZE, 0, 2 * Math.PI)
  ctx.fillStyle = '#8b5a2b'
  ctx.fill()
  ctx.strokeStyle = '#5a3a1a'
  ctx.lineWidth = 3
  ctx.stroke()
  // Матка (золотая точка в центре)
  ctx.beginPath()
  ctx.arc(nest.x, nest.y, 4, 0, 2 * Math.PI)
  ctx.fillStyle = '#FFD700'
  ctx.fill()

  //  Подземные камеры и туннели 
  const sy = nest.y + SUBTERRANEAN_BASE_Y
  ctx.save()
  ctx.translate(0, sy)   // смещаемся в подземную область

  // Туннели
  ctx.strokeStyle = '#8b7355'
  ctx.lineWidth = 2
  for (const t of tunnels) {
    const from = chambers.find(ch => ch.id === t.from)
    const to = chambers.find(ch => ch.id === t.to)
    if (from && to) {
      ctx.beginPath()
      ctx.moveTo(from.x, from.y - sy)
      ctx.lineTo(to.x, to.y - sy)
      ctx.stroke()
    }
  }

  // Камеры
  for (const ch of chambers) {
    const cy = ch.y - sy   // экранная координата Y в подземной системе
    ctx.beginPath()
    ctx.arc(ch.x, cy, CHAMBER_RADIUS, 0, 2 * Math.PI)
    // Цвет камеры зависит от типа: зеленоватый для фермы, коричневый для расплода
    if (ch.type === 'farm') {
      ctx.fillStyle = '#3d5c3a'
    } else {
      ctx.fillStyle = '#5c4033'
    }
    ctx.fill()
    ctx.strokeStyle = '#8b7355'
    ctx.stroke()

    // Прогресс строительства (если есть)
    if (ch.building) {
      ctx.fillStyle = '#333'
      ctx.fillRect(ch.x - 10, cy - 8, 20, 6)    // фон
      ctx.fillStyle = '#aaa'
      ctx.fillRect(ch.x - 10, cy - 8, 20 * ch.building.progress, 6) // заполнение
    }

    // Содержимое камеры
    if (ch.type === 'farm' && ch.aphids !== undefined) {
      // Рисуем тлю (зелёные точки)
      ctx.fillStyle = '#90ee90'
      for (let i = 0; i < ch.aphids; i++) {
        const angle = (i / Math.max(ch.aphids, 1)) * 2 * Math.PI
        const ax = ch.x + Math.cos(angle) * (CHAMBER_RADIUS * 0.6)
        const ay = cy + Math.sin(angle) * (CHAMBER_RADIUS * 0.6)
        ctx.beginPath()
        ctx.arc(ax, ay, 2, 0, 2 * Math.PI)
        ctx.fill()
      }
    } else {
      // Расплод в камере (яйца, личинки, куколки)
      for (const b of ch.brood) {
        const bx = ch.x + b.offX
        const by = cy + b.offY
        if (b.type === 'egg') {
          ctx.beginPath()
          ctx.arc(bx, by, 1.5, 0, 2 * Math.PI)
          ctx.fillStyle = 'white'
          ctx.fill()
        } else if (b.type === 'larva') {
          ctx.beginPath()
          ctx.arc(bx, by, 2.5, 0, 2 * Math.PI)
          ctx.fillStyle = 'white'
          ctx.fill()
        } else { // pupa
          ctx.save()
          ctx.translate(bx, by)
          ctx.beginPath()
          ctx.ellipse(0, 0, 3, 1.5, 0, 0, 2 * Math.PI)
          ctx.fillStyle = '#a0522d'
          ctx.fill()
          ctx.restore()
        }
      }
    }
  }
  ctx.restore()  // возвращаемся в обычную систему координат

  //  Враги (пауки) 
  for (const en of enemies) {
    ctx.save()
    ctx.translate(en.x, en.y)
    ctx.rotate(en.angle)

    // Тело паука (головогрудь и брюшко)
    ctx.fillStyle = '#4a1515'
    ctx.beginPath()
    ctx.ellipse(3.5, 0, 3, 2.5, 0, 0, 2 * Math.PI)
    ctx.fill()
    ctx.fillStyle = '#5a2020'
    ctx.beginPath()
    ctx.ellipse(-4, 0, 5, 4, 0, 0, 2 * Math.PI)
    ctx.fill()

    // Ноги (4 пары)
    ctx.strokeStyle = '#2a0808'
    ctx.lineWidth = 1.5
    const legPairs = [
      { b: 0.7, l: 9, j: 0.4 },
      { b: 1.1, l: 11, j: 0.5 },
      { b: 1.5, l: 11, j: 0.5 },
      { b: 1.9, l: 9, j: 0.4 }
    ]
    for (const p of legPairs) {
      const drawLeg = (side) => {
        const a1 = p.b * side, a2 = (p.b + p.j) * side
        const mx = 1.5 + Math.cos(a1) * p.l * 0.5
        const my = Math.sin(a1) * p.l * 0.5
        ctx.beginPath()
        ctx.moveTo(1.5, 0)
        ctx.lineTo(mx, my)
        ctx.lineTo(mx + Math.cos(a2) * p.l * 0.5, my + Math.sin(a2) * p.l * 0.5)
        ctx.stroke()
      }
      drawLeg(-1)
      drawLeg(1)
    }

    // Глаза и хелицеры
    ctx.fillStyle = 'white'
    ctx.beginPath(); ctx.arc(5, -1.2, 0.8, 0, 2 * Math.PI); ctx.fill()
    ctx.beginPath(); ctx.arc(5, 1.2, 0.8, 0, 2 * Math.PI); ctx.fill()
    ctx.strokeStyle = '#1a0505'
    ctx.lineWidth = 0.8
    ctx.beginPath(); ctx.moveTo(5.5, -0.8); ctx.lineTo(7, -1.5); ctx.moveTo(5.5, 0.8); ctx.lineTo(7, 1.5); ctx.stroke()

    ctx.restore()

    // Полоска здоровья
    ctx.fillStyle = 'red'
    ctx.fillRect(en.x - 5, en.y - 12, 10 * en.health / en.maxHealth, 2)
  }

  //  Статистика (текст) 
  ctx.font = 'bold 13px Arial'
  ctx.fillStyle = 'white'
  ctx.fillText(`Еда: ${colonyFood}`, 15, 25)
  const eggs = chambers.reduce((s, c) => s + c.brood.filter(b => b.type === 'egg').length, 0)
  const larvae = chambers.reduce((s, c) => s + c.brood.filter(b => b.type === 'larva').length, 0)
  const pupae = chambers.reduce((s, c) => s + c.brood.filter(b => b.type === 'pupa').length, 0)
  ctx.fillText(`Яиц: ${eggs}  Личинок: ${larvae}  Куколок: ${pupae}`, 15, 45)
  ctx.fillText(`Муравьёв: ${ants.length}`, 15, 65)
  ctx.fillText(`Врагов: ${enemies.length}`, 15, 85)
  ctx.fillText(`Камер: ${chambers.length} (ферм: ${chambers.filter(c => c.type === 'farm').length})`, 15, 105)

  //  Муравьи 
  for (const a of ants) {
    ctx.save()
    ctx.translate(a.x, a.y)
    ctx.rotate(a.angle)

    // Цвет зависит от роли и наличия еды
    ctx.fillStyle = a.role === 'worker' ? '#4488ff' :
                    a.role === 'soldier' ? '#ff4444' :
                    a.hasFood ? '#ffaa00' : '#222'
    const sz = a.role === 'soldier' ? 1.3 : 1
    ctx.beginPath()
    ctx.ellipse(0, 0, ANT_SIZE * 1.2 * sz, ANT_SIZE * 0.7 * sz, 0, 0, 2 * Math.PI)
    ctx.fill()

    // Усики (показывают направление)
    ctx.beginPath()
    ctx.moveTo(ANT_SIZE * 0.8, -2)
    ctx.lineTo(ANT_SIZE * 1.6, -4)
    ctx.moveTo(ANT_SIZE * 0.8, 2)
    ctx.lineTo(ANT_SIZE * 1.6, 4)
    ctx.strokeStyle = '#555'
    ctx.lineWidth = 1
    ctx.stroke()

    ctx.restore()
  }
}

//  ЗАПУСК И ОСТАНОВКА 
onMounted(() => {
  ctx = canvasRef.value.getContext('2d')
  initEnemies()   // врагов создаём до муравьёв, чтобы initAnts мог использовать их количество
  initChambers()
  initFoods()
  initAnts()
  // Таймер пополнения еды (каждые 5 сек, если источников меньше 15)
  foodRespawnTimer = setInterval(() => {
    if (foods.length < 15) addRandomFoodSource()
  }, 5000)
  lastFrameTime = performance.now()
  animationId = requestAnimationFrame(update)
})

onUnmounted(() => {
  cancelAnimationFrame(animationId)
  clearInterval(foodRespawnTimer)
})
</script>

<style scoped>
.world-container { position: relative; width: 100vw; height: 100vh; }
.world { display: block; background-color: #3b5e2b; }
.control-panel {
  position: absolute; top: 10px; right: 10px;
  background: rgba(0, 0, 0, 0.7); color: white;
  padding: 10px; border-radius: 8px; font-size: 12px; width: 210px;
  user-select: none; z-index: 10;
}
.control-panel h3 { margin: 0 0 6px; font-size: 14px; }
.control-panel label { display: block; margin: 4px 0 2px; }
.control-panel input { width: 100%; }
.control-panel input[type="checkbox"] { width: auto; margin-right: 6px; }
.control-panel hr { border-color: #666; margin: 6px 0; }
</style>