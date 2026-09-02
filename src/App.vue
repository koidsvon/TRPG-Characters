<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import { supabase } from './supabase.js'

// 页面状态：home / room / lobby / character / magic / buff
const page = ref('home')

const slotExpanded = ref('')  // 当前展开的栏位：helmet / chest / legs / amulet / backpack
const backpackOpenId = ref(null)

const currentMission = ref(null)
const missionTitle = ref('')
const missionClient = ref('')
const missionLocation = ref('')
const missionType = ref('')
const missionSummary = ref('')  // 事件简述
const showMissionPanel = ref(false)
// page 增加 'book'
const bookList = ref([])
const editingBook = ref(null) // null=列表；对象=编辑中
const bookForm = ref({
  title: '',
  client: '',
  location: '',
  mission_type: '',
  summary: ''
})

const MAP_W = 50
const MAP_H = 50
const VIEW = 28

const viewX = ref(0)
const viewY = ref(0)
const battleMap = ref(null)
const battleTokens = ref([])
const selectedTokenId = ref('')
const pendingMoveTokenId = ref('')
const addEnemyId = ref('')
const tokenStatIndex = ref({})
const STAT_CYCLE = ['hp', 'atk', 'def', 'res']
const diceLogs = ref([])
const customDice = ref('1d6')

const phantasmList = ref([])
const bookPhantasms = ref([])      // 本战役已选用（带图鉴信息）
const missionPhantasms = ref([])
const editingPhantasm = ref(null)  // 正在编辑的图鉴对象，null=列表
const showPhantasmLib = ref(false)

function emptyPhantasmForm() {
  return {
    name: '',
    category: '常规',
    hp: 0, atk: 0, spd: 0, def: 0, res: 0,
    strength: 0, intelligence: 0, agility: 0,
    attack_range: 1,
    move_range: 4,
    skills: [{ name: '', desc: '' }],
    boss_skills: [{ name: '', desc: '' }, { name: '', desc: '' }],
    token_url: '',
    notes: ''
  }
}
const phantasmForm = ref(emptyPhantasmForm())

const bookNodes = ref([])          // 编辑记录簿时
const missionNodes = ref([])       // 本局
const currentMissionNode = ref(null)
const displayMissionNode = computed(() => {
  const n = currentMissionNode.value
  if (!n) return null
  // GM 看真实内容；玩家看公开节点，隐藏节点显示未知区域
  if (!isGM.value && n.visible_to_players === false) {
    return {
      name: '未知区域',
      description: '（此处尚未对调查员开放）',
      image_url: ''
    }
  }
  return n
})

const announceMessages = ref([])
const chatMessages = ref([])
const announceDraft = ref('')
const chatDraft = ref('')

const nodeForm = ref({
  name: '',
  description: '',
  image_url: '',
  visible_to_players: true,
  parent_id: null
})
const editingNodeId = ref(null)

function getBookNodeDepth(nodeId, nodes) {
  let depth = 1
  let cur = nodes.find(n => n.id === nodeId)
  while (cur && cur.parent_id) {
    depth++
    cur = nodes.find(n => n.id === cur.parent_id)
    if (depth > 10) break
  }
  return depth
}

function resetNodeForm(parentId = null) {
  editingNodeId.value = null
  nodeForm.value = {
    name: '',
    description: '',
    image_url: '',
    visible_to_players: true,
    parent_id: parentId,
    sort_order: 0
  }
}

function beginEditNode(n) {
  editingNodeId.value = n.id
  nodeForm.value = {
    name: n.name || '',
    description: n.description || '',
    image_url: n.image_url || '',
    visible_to_players: n.visible_to_players !== false,
    parent_id: n.parent_id || null,
    sort_order: n.sort_order || 0
  }
}

function beginAddChild(n) {
  resetNodeForm(n.id)
}

function beginAddRoot() {
  resetNodeForm(null)
}

function nodeDepthLabel(n) {
  return getBookNodeDepth(n.id, bookNodes.value)
}

function parentName(n) {
  if (!n.parent_id) return '顶层'
  const p = bookNodes.value.find(x => x.id === n.parent_id)
  return p ? ('子级 ← ' + p.name) : '子级'
}

async function loadBookList() {
  const { data, error } = await supabase
    .from('campaign_book')
    .select('*')
    .order('updated_at', { ascending: false })
  if (!error) bookList.value = data || []
}

function ensureSkillArrays(p) {
  const skills = Array.isArray(p.skills) && p.skills.length
    ? p.skills
    : [{ name: '', desc: '' }]
  const boss_skills = Array.isArray(p.boss_skills) && p.boss_skills.length
    ? p.boss_skills
    : [{ name: '', desc: '' }, { name: '', desc: '' }]
  return { ...p, skills, boss_skills }
}

async function loadPhantasmList() {
  const { data, error } = await supabase
    .from('phantasms')
    .select('*')
    .order('category')
    .order('name')
  if (!error) phantasmList.value = (data || []).map(ensureSkillArrays)
}

async function loadBookPhantasms(bookId) {
  bookPhantasms.value = []
  if (!bookId) return
  const { data, error } = await supabase
    .from('book_phantasms')
    .select('id, phantasm_id, phantasms(*)')
    .eq('book_id', bookId)
  if (error) {
    console.error(error)
    return
  }
  bookPhantasms.value = (data || []).map(row => ({
    linkId: row.id,
    ...ensureSkillArrays(row.phantasms || { name: '未知', category: '常规' })
  }))
}

function startNewPhantasm() {
  editingPhantasm.value = { id: null }
  phantasmForm.value = emptyPhantasmForm()
}

function editPhantasmItem(p) {
  editingPhantasm.value = p
  phantasmForm.value = ensureSkillArrays({
    name: p.name || '',
    category: p.category || '常规',
    hp: p.hp || 0, atk: p.atk || 0, spd: p.spd || 0,
    def: p.def || 0, res: p.res || 0,
    strength: p.strength || 0,
    intelligence: p.intelligence || 0,
    agility: p.agility || 0,
    skills: p.skills,
    boss_skills: p.boss_skills,
    token_url: p.token_url || '',
    notes: p.notes || ''
  })
}

function addInnateSkill() {
  phantasmForm.value.skills.push({ name: '', desc: '' })
}
function removeInnateSkill(i) {
  if (phantasmForm.value.skills.length <= 1) return
  phantasmForm.value.skills.splice(i, 1)
}
function addBossSkill() {
  phantasmForm.value.boss_skills.push({ name: '', desc: '' })
}
function removeBossSkill(i) {
  if (phantasmForm.value.boss_skills.length <= 1) return
  phantasmForm.value.boss_skills.splice(i, 1)
}

async function savePhantasmItem() {
  const name = (phantasmForm.value.name || '').trim()
  if (!name) {
    alert('请输入空想种名称')
    return
  }
  const f = phantasmForm.value
  const payload = {
    name,
    category: f.category || '常规',
    hp: Number(f.hp) || 0,
    atk: Number(f.atk) || 0,
    spd: Number(f.spd) || 0,
    def: Number(f.def) || 0,
    res: Number(f.res) || 0,
    strength: Number(f.strength) || 0,
    intelligence: Number(f.intelligence) || 0,
    agility: Number(f.agility) || 0,     
    attack_range: Number(f.attack_range) || 1,
    move_range: Number(f.move_range) || 4,
    skills: f.skills || [],
    boss_skills: f.category === 'Boss' ? (f.boss_skills || []) : [],
    token_url: f.token_url || '',
    notes: f.notes || '',
    updated_at: new Date().toISOString()
  }
  if (editingPhantasm.value?.id) {
    const { error } = await supabase.from('phantasms').update(payload).eq('id', editingPhantasm.value.id)
    if (error) return alert('保存失败：' + error.message)
  } else {
    const { error } = await supabase.from('phantasms').insert(payload)
    if (error) return alert('创建失败：' + error.message)
  }
  editingPhantasm.value = null
  await loadPhantasmList()
  if (editingBook.value?.id) await loadBookPhantasms(editingBook.value.id)
  alert('空想种已保存到图鉴')
}

async function deletePhantasmItem(p) {
  if (!confirm('删除图鉴中的「' + p.name + '」？已开本局的副本不会删除。')) return
  const { error } = await supabase.from('phantasms').delete().eq('id', p.id)
  if (error) alert('删除失败：' + error.message)
  else {
    await loadPhantasmList()
    if (editingBook.value?.id) await loadBookPhantasms(editingBook.value.id)
  }
}

async function addPhantasmToBook(phantasmId) {
  if (!editingBook.value?.id) {
    alert('请先保存战役，再选用空想种')
    return
  }
  const { error } = await supabase.from('book_phantasms').insert({
    book_id: editingBook.value.id,
    phantasm_id: phantasmId
  })
  if (error) return alert('添加失败：' + error.message)
  await loadBookPhantasms(editingBook.value.id)
}

async function removePhantasmFromBook(linkId) {
  const { error } = await supabase.from('book_phantasms').delete().eq('id', linkId)
  if (error) return alert('移除失败：' + error.message)
  await loadBookPhantasms(editingBook.value.id)
}

function phantasmsByCategory(list, cat) {
  return (list || []).filter(p => p.category === cat)
}

async function loadMissionPhantasms() {
  missionPhantasms.value = []
  if (!currentMission.value?.id) return
  const { data, error } = await supabase
    .from('mission_phantasms')
    .select('*')
    .eq('mission_id', currentMission.value.id)
    .order('category')
  if (!error) missionPhantasms.value = (data || []).map(normalizeMissionPhantasm)
}

function clampView() {
  viewX.value = Math.max(0, Math.min(MAP_W - VIEW, viewX.value))
  viewY.value = Math.max(0, Math.min(MAP_H - VIEW, viewY.value))
}

function tokenAt(x, y) {
  return battleTokens.value.find(t => t.x === x && t.y === y)
}

function canMoveToken(t) {
  if (!t) return false
  if (isGM.value) return true
  if (t.kind !== 'player') return false
  const ch = characters.value.find(c => c.id === t.ref_id)
  return !!(ch && ch.name === myCharacterName.value)
}

function canEditHp(t) {
  if (!t) return false
  if (isGM.value) return true
  if (t.kind !== 'player') return false
  const ch = characters.value.find(c => c.id === t.ref_id)
  return !!(ch && ch.name === myCharacterName.value)
}

function getTokenSource(t) {
  if (!t) return null
  if (t.kind === 'player') return characters.value.find(c => c.id === t.ref_id) || null
  return missionPhantasms.value.find(p => p.id === t.ref_id) || null
}

function defaultEnemyReveal() {
  return {
    hp: false,
    atk: false,
    def: false,
    res: false,
    spd: false,
    strength: false,
    agility: false,
    intelligence: false,
    attack_range: false,
    move_range: false,
    skills: {},
    boss_skills: {}
  }
}

function normalizeMissionPhantasm(p) {
  if (!p) return p
  if (p.attack_range == null) p.attack_range = 1
  if (p.move_range == null) p.move_range = 4
  if (!Array.isArray(p.skills)) p.skills = []
  if (!Array.isArray(p.boss_skills)) p.boss_skills = []
  const r = { ...defaultEnemyReveal(), ...(p.reveal || {}) }
  if (!r.skills || typeof r.skills !== 'object') r.skills = {}
  if (!r.boss_skills || typeof r.boss_skills !== 'object') r.boss_skills = {}
  p.reveal = r
  return p
}

function isEnemyStatPublic(src, key) {
  if (!src) return false
  if (isGM.value) return true
  return !!(src.reveal && src.reveal[key])
}

function shownEnemyStat(src, key) {
  if (!src) return '—'
  if (isEnemyStatPublic(src, key)) {
    if (key === 'hp') return (src.hp_current ?? src.hp ?? 0) + ' / ' + (src.hp ?? 0)
    const v = src[key]
    return v == null || v === '' ? '—' : v
  }
  return '未知'
}

function isEnemySkillPublic(src, listKey, index) {
  if (!src) return false
  if (isGM.value) return true
  return !!(src.reveal && src.reveal[listKey] && src.reveal[listKey][index])
}

function shownEnemySkillDesc(src, listKey, index) {
  const list = src?.[listKey] || []
  const sk = list[index]
  if (!sk) return ''
  if (isEnemySkillPublic(src, listKey, index)) return sk.desc || '—'
  return '未知'
}

async function setEnemyReveal(src, key, value) {
  if (!isGM.value || !src?.id) return
  const reveal = { ...defaultEnemyReveal(), ...(src.reveal || {}) }
  reveal[key] = value
  src.reveal = reveal
  const { error } = await supabase
    .from('mission_phantasms')
    .update({ reveal })
    .eq('id', src.id)
  if (error) alert('更新可见性失败：' + error.message)
  else await loadMissionPhantasms()
}

async function setEnemySkillReveal(src, listKey, index, value) {
  if (!isGM.value || !src?.id) return
  const reveal = { ...defaultEnemyReveal(), ...(src.reveal || {}) }
  reveal[listKey] = { ...(reveal[listKey] || {}) }
  reveal[listKey][index] = value
  src.reveal = reveal
  const { error } = await supabase
    .from('mission_phantasms')
    .update({ reveal })
    .eq('id', src.id)
  if (error) alert('更新可见性失败：' + error.message)
  else await loadMissionPhantasms()
}

function tokenDisplayName(t) {
  const src = getTokenSource(t)
  if (t?.kind === 'player') return src?.name || t.label
  return src?.name || t.label
}

function tokenStatKey(t) {
  return STAT_CYCLE[tokenStatIndex.value[t.id] || 0]
}

function cycleTokenStat(t, dir) {
  const cur = tokenStatIndex.value[t.id] || 0
  tokenStatIndex.value = { ...tokenStatIndex.value, [t.id]: (cur + dir + 4) % 4 }
}

function tokenStatValue(t) {
  const src = getTokenSource(t)
  const key = tokenStatKey(t)
  if (!src) return '—'
  if (t.kind === 'player') {
    if (key === 'hp') return src.hp_current ?? src.hp_max ?? 0
    if (key === 'atk') return src.atk ?? 0
    if (key === 'def') return src.def ?? 0
    if (key === 'res') return src.res ?? 0
    return 0
  }
  if (!isEnemyStatPublic(src, key)) return '未知'
  if (key === 'hp') return src.hp_current ?? src.hp ?? 0
  if (key === 'atk') return src.atk ?? 0
  if (key === 'def') return src.def ?? 0
  if (key === 'res') return src.res ?? 0
  return 0
}

async function loadDiceLogs() {
  diceLogs.value = []
  if (!currentSession.value?.id) return
  const { data } = await supabase
    .from('dice_logs')
    .select('*')
    .eq('session_id', currentSession.value.id)
    .order('created_at', { ascending: false })
    .limit(20)
  diceLogs.value = data || []
}

function parseDiceFormula(raw) {
  const s = String(raw || '').trim().toLowerCase().replace(/\s/g, '')
  const m = s.match(/^(\d+)d(\d+)([+-]\d+)?$/)
  if (!m) return null
  const n = Number(m[1])
  const sides = Number(m[2])
  const mod = m[3] ? Number(m[3]) : 0
  if (n < 1 || n > 20 || sides < 2 || sides > 1000) return null
  return { n, sides, mod, formula: s }
}

function rollLocal(parsed) {
  const parts = []
  let sum = 0
  for (let i = 0; i < parsed.n; i++) {
    const r = 1 + Math.floor(Math.random() * parsed.sides)
    parts.push(r)
    sum += r
  }
  return { result: sum + parsed.mod, detail: parts.join('+') + (parsed.mod ? (parsed.mod > 0 ? '+' + parsed.mod : String(parsed.mod)) : '') }
}

async function rollDice(formula) {
  if (!currentSession.value?.id) {
    alert('请先进入房间')
    return
  }
  const parsed = parseDiceFormula(formula)
  if (!parsed) {
    alert('格式不对，请用 1d20 或 2d6+1')
    return
  }
  const rolled = rollLocal(parsed)
  const roller = isGM.value ? 'GM' : (myCharacterName.value || '玩家')
  const { error } = await supabase.from('dice_logs').insert({
    session_id: currentSession.value.id,
    roller_name: roller,
    formula: parsed.formula,
    result: rolled.result,
    detail: rolled.detail
  })
  if (error) alert('掷骰失败：' + error.message)
  else await loadDiceLogs()
}

async function loadBattleMap() {
  battleMap.value = null
  battleTokens.value = []
  if (!currentMission.value?.id) return
  const { data: maps } = await supabase
    .from('battle_maps')
    .select('*')
    .eq('mission_id', currentMission.value.id)
    .order('created_at', { ascending: true })
    .limit(1)
  battleMap.value = maps?.[0] || null
  if (!battleMap.value) return
  const { data: tokens } = await supabase
    .from('battle_tokens')
    .select('*')
    .eq('map_id', battleMap.value.id)
  battleTokens.value = tokens || []
}

function nextEnemyLabel(category) {
  if (category === 'Boss') {
    const n = battleTokens.value.filter(t => t.kind === 'enemy' && String(t.label).startsWith('BOSS')).length
    return n === 0 ? 'BOSS' : 'BOSS' + (n + 1)
  }
  const n = battleTokens.value.filter(t => t.kind === 'enemy' && String(t.label).startsWith('E')).length
  return 'E' + (n + 1)
}

function emptyCellNear(sx, sy) {
  for (let y = sy; y < MAP_H; y++) {
    for (let x = sx; x < MAP_W; x++) {
      if (!tokenAt(x, y)) return { x, y }
    }
  }
  for (let y = 0; y < MAP_H; y++) {
    for (let x = 0; x < MAP_W; x++) {
      if (!tokenAt(x, y)) return { x, y }
    }
  }
  return null
}

async function addEnemyToMap() {
  if (!isGM.value) return
  if (!battleMap.value?.id) {
    alert('地图还没建好')
    return
  }
  const p = missionPhantasms.value.find(i => i.id === addEnemyId.value)
  if (!p) {
    alert('请选择要添加的空想种')
    return
  }
  const pos = emptyCellNear(viewX.value + 4, viewY.value + 4)
  if (!pos) {
    alert('没有空位')
    return
  }
  const { error } = await supabase.from('battle_tokens').insert({
    map_id: battleMap.value.id,
    kind: 'enemy',
    ref_id: p.id,
    label: nextEnemyLabel(p.category),
    x: pos.x,
    y: pos.y
  })
  if (error) alert('添加失败：' + error.message)
  else {
    addEnemyId.value = ''
    await loadBattleMap()
  }
}

async function removeToken(t) {
  if (!isGM.value) return
  if (!confirm('从地图上移除 ' + t.label + '？')) return
  const { error } = await supabase.from('battle_tokens').delete().eq('id', t.id)
  if (error) alert(error.message)
  else {
    selectedTokenId.value = ''
    await loadBattleMap()
  }
}

async function ensureBattleSetup() {
  if (!currentMission.value?.id) {
    alert('请先开启神秘事件')
    return null
  }
  await loadMissionPhantasms()
  await loadCharacters()
  await loadBattleMap()
  if (battleMap.value) return battleMap.value

  const { data: map, error: mapErr } = await supabase
    .from('battle_maps')
    .insert({
      mission_id: currentMission.value.id,
      name: '战斗地图',
      width: MAP_W,
      height: MAP_H
    })
    .select()
    .single()
  if (mapErr) {
    alert('创建战斗地图失败：' + mapErr.message)
    return null
  }

  const tokens = []
  const players = [...characters.value].sort((a, b) => String(a.name).localeCompare(b.name))
  players.forEach((ch, i) => {
    tokens.push({
      map_id: map.id,
      kind: 'player',
      ref_id: ch.id,
      label: 'P' + (i + 1),
      x: 2 + (i % 8),
      y: 2 + Math.floor(i / 8)
    })
  })
  if (tokens.length) {
    const { error } = await supabase.from('battle_tokens').insert(tokens)
    if (error) alert('生成 token 失败：' + error.message)
  }

  await supabase
    .from('missions')
    .update({ current_battle_map_id: map.id })
    .eq('id', currentMission.value.id)

  await loadBattleMap()
  return battleMap.value
}

async function enterBattle() {
  const map = await ensureBattleSetup()
  if (!map) return
  await supabase.from('missions').update({ show_battle: true }).eq('id', currentMission.value.id)
  currentMission.value.show_battle = true
  viewX.value = 0
  viewY.value = 0
  page.value = 'battle'
  loadDiceLogs()
}

async function exitBattle() {
  if (currentMission.value?.id) {
    await supabase.from('missions').update({ show_battle: false }).eq('id', currentMission.value.id)
    currentMission.value.show_battle = false
  }
  page.value = 'lobby'
}

function onCellClick(x, y) {
  if (pendingMoveTokenId.value) {
    const t = battleTokens.value.find(i => i.id === pendingMoveTokenId.value)
    if (t) moveTokenTo(t, x, y)
    pendingMoveTokenId.value = ''
    return
  }
  const t = tokenAt(x, y)
  selectedTokenId.value = t ? t.id : ''
}

function beginMove(t) {
  if (!canMoveToken(t)) {
    alert('不能移动这个 token')
    return
  }
  pendingMoveTokenId.value = t.id
  selectedTokenId.value = t.id
}

async function moveTokenTo(t, x, y) {
  if (!canMoveToken(t)) return
  if (x < 0 || y < 0 || x >= MAP_W || y >= MAP_H) return
  if (tokenAt(x, y) && tokenAt(x, y).id !== t.id) {
    alert('该格已有单位')
    return
  }
  const { error } = await supabase.from('battle_tokens').update({ x, y }).eq('id', t.id)
  if (error) alert('移动失败：' + error.message)
  else await loadBattleMap()
}

async function updateTokenHp(t, value) {
  if (!canEditHp(t)) {
    alert('不能修改这个单位的 HP')
    return
  }
  const hp = Number(value)
  if (Number.isNaN(hp)) return
  if (t.kind === 'player') {
    const { error } = await supabase.from('characters').update({ hp_current: hp }).eq('id', t.ref_id)
    if (error) alert(error.message)
    else await loadCharacters()
  } else {
    const { error } = await supabase.from('mission_phantasms').update({ hp_current: hp }).eq('id', t.ref_id)
    if (error) alert(error.message)
    else await loadMissionPhantasms()
  }
}

const selectedToken = computed(() => battleTokens.value.find(t => t.id === selectedTokenId.value) || null)

function openBook() {
  editingBook.value = null
  page.value = 'book'
  loadBookList()
  loadPhantasmList()
}

function startNewBook() {
  editingBook.value = { id: null }
  bookForm.value = {
    title: '',
    client: '',
    location: '',
    mission_type: '',
    summary: ''
  }
  bookNodes.value = []
  editingNodeId.value = null
  nodeForm.value = {
    name: '',
    description: '',
    image_url: '',
    visible_to_players: true,
    parent_id: null
  }
}

function editBookItem(item) {
  editingBook.value = item
  bookForm.value = {
    title: item.title || '',
    client: item.client || '',
    location: item.location || '',
    mission_type: item.mission_type || '',
    summary: item.summary || ''
  }
    loadBookNodes(item.id)
      loadPhantasmList()
      loadBookPhantasms(item.id)
}

async function loadBookNodes(bookId) {
  if (!bookId) {
    bookNodes.value = []
    return
  }
  const { data, error } = await supabase
    .from('book_map_nodes')
    .select('*')
    .eq('book_id', bookId)
    .order('sort_order')
  if (error) {
    console.error(error)
    bookNodes.value = []
    return
  }
  bookNodes.value = data || []
}

async function saveBookNode(bookId) {
  if (!bookId) {
    alert('请先保存事件，再添加节点')
    return
  }
  const name = (nodeForm.value.name || '').trim()
  if (!name) {
    alert('请输入节点名称')
    return
  }
  if (nodeForm.value.parent_id) {
    const d = getBookNodeDepth(nodeForm.value.parent_id, bookNodes.value)
    if (d >= 4) {
      alert('最多 4 层节点，无法再添加子节点')
      return
    }
  }

  const payload = {
    book_id: bookId,
    parent_id: nodeForm.value.parent_id || null,
    name,
    description: nodeForm.value.description || '',
    image_url: nodeForm.value.image_url || '',
    visible_to_players: nodeForm.value.visible_to_players !== false,
    sort_order: nodeForm.value.sort_order || 0
  }

  if (editingNodeId.value) {
    const { error } = await supabase
      .from('book_map_nodes')
      .update(payload)
      .eq('id', editingNodeId.value)
    if (error) {
      alert('保存失败：' + error.message)
      return
    }
  } else {
    const { error } = await supabase
      .from('book_map_nodes')
      .insert(payload)
    if (error) {
      alert('创建失败：' + error.message)
      return
    }
  }

  await loadBookNodes(bookId)
  resetNodeForm(null)
  alert('节点已保存')
}

async function deleteBookNode(node) {
  if (!node?.id) return
  if (!confirm('删除节点「' + node.name + '」及其子节点？')) return
  const { error } = await supabase
    .from('book_map_nodes')
    .delete()
    .eq('id', node.id)
  if (error) {
    alert('删除失败：' + error.message)
    return
  }
  if (editingBook.value?.id) {
    await loadBookNodes(editingBook.value.id)
  }
}

async function saveBookItem() {
  const title = (bookForm.value.title || '').trim()
  if (!title) {
    alert('请输入事件标题')
    return
  }
  const payload = {
    title,
    client: bookForm.value.client || '',
    location: bookForm.value.location || '',
    mission_type: bookForm.value.mission_type || '',
    summary: bookForm.value.summary || '',
    updated_at: new Date().toISOString()
  }

  if (editingBook.value?.id) {
    const { error } = await supabase
      .from('campaign_book')
      .update(payload)
      .eq('id', editingBook.value.id)
    if (error) {
      alert('保存失败：' + error.message)
      return
    }
    alert('已保存到记录簿')
    await loadBookList()
    // 保持编辑，方便继续加节点
    loadBookNodes(editingBook.value.id)
  } else {
    const { data, error } = await supabase
      .from('campaign_book')
      .insert(payload)
      .select()
      .single()
    if (error) {
      alert('创建失败：' + error.message)
      return
    }
    alert('已保存到记录簿，可继续添加地图节点')
    editingBook.value = data
    await loadBookList()
    loadBookNodes(data.id)
  }
}

async function deleteBookItem(item) {
  if (!confirm('确认删除「' + item.title + '」？此操作不删除已在房间内开打的副本。')) return
  const { error } = await supabase.from('campaign_book').delete().eq('id', item.id)
  if (error) alert('删除失败：' + error.message)
  else loadBookList()
}

async function startMissionFromBook(bookId) {
  if (!isGM.value) {
    alert('只有 GM 可以选择战役')
    return
  }
  if (!currentSession.value) {
    alert('请先进入房间')
    return
  }
  if (currentSession.value.current_mission_id) {
    alert('当前已有进行中的事件，请先结束本局')
    return
  }

  const book = bookList.value.find(b => b.id === bookId)
  if (!book) {
    alert('找不到该记录')
    return
  }

  const { data, error } = await supabase
    .from('missions')
    .insert({
      session_id: currentSession.value.id,
      source_book_id: book.id,
      title: book.title,
      client: book.client || '',
      location: book.location || '',
      mission_type: book.mission_type || '',
      summary: book.summary || '',
      status: 'active'
    })
    .select()
    .single()

  if (error || !data) {
    alert('开启失败：' + (error?.message || '未知错误'))
    return
  }

  // 复制地图节点
  const { data: srcNodes } = await supabase
    .from('book_map_nodes')
    .select('*')
    .eq('book_id', book.id)
    .order('sort_order')

  const { data: bookLinks } = await supabase
    .from('book_phantasms')
    .select('phantasm_id, phantasms(*)')
    .eq('book_id', book.id)

  for (const row of (bookLinks || [])) {
    const p = row.phantasms
    if (!p) continue
    await supabase.from('mission_phantasms').insert({
      mission_id: data.id,
      source_phantasm_id: p.id,
      name: p.name,
      category: p.category || '常规',
      hp: p.hp || 0,
      hp_current: p.hp || 0,
      atk: p.atk || 0,
      spd: p.spd || 0,
      def: p.def || 0,
      res: p.res || 0,
      strength: p.strength || 0,
      intelligence: p.intelligence || 0,
      agility: p.agility || 0,
      attack_range: p.attack_range ?? 1,
        move_range: p.move_range ?? 4,
        reveal: {},
      skills: p.skills || [],
      boss_skills: p.boss_skills || [],
      token_url: p.token_url || '',
      notes: p.notes || ''
    })
  }

  const idMap = {}
  let remaining = [...(srcNodes || [])]
  let guard = 0
  while (remaining.length && guard < 20) {
    guard++
    const next = []
    for (const n of remaining) {
      if (n.parent_id && !idMap[n.parent_id]) {
        next.push(n)
        continue
      }
      const { data: created, error: nodeErr } = await supabase
        .from('mission_map_nodes')
        .insert({
          mission_id: data.id,
          parent_id: n.parent_id ? idMap[n.parent_id] : null,
          source_node_id: n.id,
          name: n.name,
          description: n.description || '',
          image_url: n.image_url || '',
          visible_to_players: n.visible_to_players !== false,
          sort_order: n.sort_order || 0
        })
        .select()
        .single()
      if (nodeErr) {
        console.error(nodeErr)
        continue
      }
      if (created) idMap[n.id] = created.id
    }
    remaining = next
  }

  // 默认当前节点：第一个顶层
  const roots = (srcNodes || []).filter(n => !n.parent_id)
  const firstRoot = roots[0]
  let currentNodeId = null
  if (firstRoot && idMap[firstRoot.id]) {
    currentNodeId = idMap[firstRoot.id]
    await supabase
      .from('missions')
      .update({ current_node_id: currentNodeId })
      .eq('id', data.id)
  }

  const { error: e2 } = await supabase
    .from('sessions')
    .update({ current_mission_id: data.id })
    .eq('id', currentSession.value.id)

  if (e2) {
    alert('绑定房间失败：' + e2.message)
    return
  }

  currentSession.value = {
    ...currentSession.value,
    current_mission_id: data.id
  }
  currentMission.value = {
    ...data,
    current_node_id: currentNodeId
  }
  saveSessionToLocal()
  await loadMissionNodes()
  alert('已从记录簿复制并开始本局')
  showMissionPanel.value = false
}

async function loadCurrentMission() {
  currentMission.value = null
  missionNodes.value = []
  currentMissionNode.value = null

  if (!currentSession.value?.id) return

  // 先从服务器刷新房间，拿到最新的 current_mission_id
  const { data: sessionRow } = await supabase
    .from('sessions')
    .select('id, current_mission_id, code, name, gm_password')
    .eq('id', currentSession.value.id)
    .maybeSingle()

  if (sessionRow) {
    currentSession.value = {
      ...currentSession.value,
      ...sessionRow
    }
  }

  let missionId = currentSession.value.current_mission_id

  // 兜底：按房间查进行中的 mission
  if (!missionId) {
    const { data: active } = await supabase
      .from('missions')
      .select('*')
      .eq('session_id', currentSession.value.id)
      .eq('status', 'active')
      .order('created_at', { ascending: false })
      .limit(1)
      .maybeSingle()

    if (active) {
      missionId = active.id
      currentMission.value = active
      currentSession.value = {
        ...currentSession.value,
        current_mission_id: active.id
      }
    }
  } else {
    const { data, error } = await supabase
      .from('missions')
      .select('*')
      .eq('id', missionId)
      .maybeSingle()

    if (error || !data) {
      currentMission.value = null
      return
    }
    currentMission.value = data
  }

  if (!currentMission.value) return

  await loadMissionNodes()
  await loadMissionPhantasms()
   await loadBattleMap()
   await loadDiceLogs()
}

async function debugMap() {
  await loadCurrentMission()
  alert(
    'mission=' + (currentMission.value?.id || '无') +
    '\n节点数=' + missionNodes.value.length +
    '\ncurrent_node_id=' + (currentMission.value?.current_node_id || '无') +
    '\n当前节点=' + (currentMissionNode.value?.name || '无')
  )
}

async function toggleMissionNodeVisibility(node) {
  if (!isGM.value || !node?.id) return
  const next = !node.visible_to_players
  const { error } = await supabase
    .from('mission_map_nodes')
    .update({ visible_to_players: next })
    .eq('id', node.id)
  if (error) {
    alert('更新失败：' + error.message)
    return
  }
  await loadMissionNodes()
}

/** 本局节点树（多根） */
const missionNodeTree = computed(() => {
  const list = missionNodes.value || []
  const byParent = {}
  list.forEach(n => {
    const key = n.parent_id ? n.parent_id : 'root'
    if (!byParent[key]) byParent[key] = []
    byParent[key].push(n)
  })
  Object.keys(byParent).forEach(k => {
    byParent[k].sort((a, b) => (a.sort_order || 0) - (b.sort_order || 0))
  })
  function build(parentKey, depth) {
    return (byParent[parentKey] || []).map(n => ({
      node: n,
      depth,
      children: build(n.id, depth + 1)
    }))
  }
  return build('root', 0)
})

const missionNodeFlat = computed(() => {
  const out = []
  function walk(items) {
    for (const item of items) {
      out.push({ node: item.node, depth: item.depth })
      if (item.children?.length) walk(item.children)
    }
  }
  walk(missionNodeTree.value)
  return out
})

/** 路径文案：总节点：子 - 孙 */
function missionNodePathLabel(node) {
  const list = missionNodes.value || []
  const names = [node.name]
  let cur = node
  while (cur.parent_id) {
    const p = list.find(x => x.id === cur.parent_id)
    if (!p) break
    names.unshift(p.name)
    cur = p
  }
  if (names.length === 1) return names[0]
  // 总节点：子节点 - 第二子节点
  return names[0] + '：' + names.slice(1).join(' - ')
}

async function loadMissionNodes() {
  missionNodes.value = []
  currentMissionNode.value = null
  if (!currentMission.value?.id) return

  const { data, error } = await supabase
    .from('mission_map_nodes')
    .select('*')
    .eq('mission_id', currentMission.value.id)
    .order('sort_order')

  if (error) {
    alert('加载地图节点失败：' + error.message)
    return
  }

  missionNodes.value = data || []
  if (!missionNodes.value.length) return

  const cid = currentMission.value.current_node_id
  let node = cid ? missionNodes.value.find(n => n.id === cid) : null
  if (!node) {
    node = missionNodes.value.find(n => !n.parent_id) || missionNodes.value[0]
  }
  currentMissionNode.value = node
}

async function setCurrentNode(nodeId) {
  if (!isGM.value || !currentMission.value) return
  const { error } = await supabase
    .from('missions')
    .update({ current_node_id: nodeId })
    .eq('id', currentMission.value.id)
  if (error) return alert(error.message)
  currentMission.value = { ...currentMission.value, current_node_id: nodeId }
  currentMissionNode.value = missionNodes.value.find(n => n.id === nodeId) || null
}

// 玩家看到的节点名列表
const visibleMissionNodes = computed(() => {
  if (isGM.value) return missionNodes.value
  return missionNodes.value.filter(n => n.visible_to_players)
})

async function createMission() {
  if (!isGM.value) {
    alert('只有 GM 可以创建神秘事件')
    return
  }
  const title = (missionTitle.value || '').trim()
  if (!title) {
    alert('请输入事件标题')
    return
  }
  if (currentSession.value.current_mission_id) {
    alert('当前已有进行中的事件，请先标记完成')
    return
  }

  const { data, error } = await supabase
    .from('missions')
    .insert({
      session_id: currentSession.value.id,
      title,
      client: missionClient.value || '',
      location: missionLocation.value || '',
      mission_type: missionType.value || '',
      summary: missionSummary.value || '',
     status: 'active'
    })
    .select()
    .single()

  if (error) {
    alert('创建失败：' + error.message)
    return
  }

  const { error: e2 } = await supabase
    .from('sessions')
    .update({ current_mission_id: data.id })
    .eq('id', currentSession.value.id)

  if (e2) {
    alert('绑定房间失败：' + e2.message)
    return
  }

  currentSession.value = { ...currentSession.value, current_mission_id: data.id }
  currentMission.value = data
  missionTitle.value = ''
  missionClient.value = ''
  missionLocation.value = ''
  missionType.value = ''
  missionSummary.value = ''
  showMissionPanel.value = false
  saveSessionToLocal()
  alert('神秘事件已创建')
}

async function updateMissionFields() {
  if (!isGM.value || !currentMission.value) return
  const { error } = await supabase
    .from('missions')
    .update({
      title: currentMission.value.title,
      client: currentMission.value.client || '',
      location: currentMission.value.location || '',
      mission_type: currentMission.value.mission_type || '',
      summary: currentMission.value.summary || ''
    })
    .eq('id', currentMission.value.id)
  if (error) alert('保存失败：' + error.message)
  else alert('已保存')
}

async function completeMission() {
  if (!isGM.value || !currentMission.value) return
  if (!confirm('确认完成当前神秘事件？完成后可创建新事件。')) return

  const id = currentMission.value.id
  await supabase
    .from('missions')
    .update({ status: 'completed', completed_at: new Date().toISOString() })
    .eq('id', id)

  await supabase
    .from('sessions')
    .update({ current_mission_id: null })
    .eq('id', currentSession.value.id)

  currentSession.value = { ...currentSession.value, current_mission_id: null }
  currentMission.value = null
  saveSessionToLocal()
  alert('事件已完成')
}

// 检定相关
const skillsExpanded = ref(false)

const passiveSkillList = {
  I: [
    { name: '学院派', desc: 'ATK+5 HP+5 DEF+3 RES+3' },
    { name: '鲁莽射击', desc: '使用枪械攻击时每次将消耗双倍子弹，同时每次射击时获得一次优势' },
    { name: '练金术I', desc: '战斗中获得的PP+10%' },
    { name: '深呼吸', desc: '遭受异常状态时将会回复10%的HP。' },
    { name: '重型装备爱好者', desc: '使用重型武器时总伤害+6。重型枪械现在可以进行近战攻击，距离0格，伤害为射击的一半' },
    { name: '奸商', desc: '在任意商店购物时享有9折优惠，若1d12≥8则提升至8折优惠' },
    { name: '过劳', desc: '现在在剩余施法点不足以使用需求更多的魔术时，可以正常使用。' },
    { name: '训练', desc: '每次长休时可以选择通过锻炼永久+1ATK' },
    { name: '应战号应', desc: '复数敌人同时以你为目标时，DEF+5 造成的伤害总值+5' },
    { name: '重热量', desc: '获得等同于DEF的额外ATK（最大30）减少同等SPD（最大30）' },
    { name: '肌！肉！', desc: '在背包的超重格被使用时，不再减少移动格，同时ATK+10' },
  ],
  II: [
    { name: '激昂', desc: 'DEF减半，获得等同于全额DEF的额外HP' },
    { name: '灼热化', desc: '每2次近战攻击后，下一次攻击还会额外造成一次等同于本次攻击造成的税前伤害的魔术伤害（枪械无效）' },
    { name: '练金术II', desc: '战斗中获得的PP+20% 若同时具备练金术I则变为30%' },
    { name: '霸臣的食桌', desc: '在战斗中进食将会在本次战斗中+25HP最大值。每场战斗限一次' },
    { name: '不择手段', desc: '当背包里“普通”稀有度的素材大于10个时，主手的武器ATK加值变为1.5倍' },
    { name: '超越计算', desc: '每次使用buff时可以在本次战斗中获得2点永久智力' },
    { name: '奔流', desc: '造成近战伤害时，如果判定百分比≥8，则可以再攻击一次' },
    { name: '战略视野', desc: '跳过自己的第一个回合，为所有友军赋予2次优势' },
    { name: '剑戟怒涛', desc: '对同一名敌人每造成伤害就能获得3点速度直到该敌人死亡。同时只有一名敌人能触发此效果，若中途改变目标也会清零' },
    { name: '鱼民的祝福', desc: '在水边环境战斗时，SPD+6。每回合结束时回复10HP' },
    { name: '精灵加护', desc: '受到伤害时，若敌人的投掷判定结果≥8则伤害总值额外减少8点' },
  ],
  III: [
    { name: '拟似铁心', desc: '免疫一切精神异常。每次成功免疫精神异常时获得一次优势' },
    { name: '贰天壹流', desc: '现在可以同时装备并使用两把武器(刺剑,长剑,长枪,手斧,匕首,手枪,冲锋枪)只有主武器可以触发武器技能' },
    { name: '练金术III', desc: '战斗后获得的PP+30% 若同时具备练金术I和II则变为+60%' },
    { name: '刺杀机-再', desc: '战斗中造成近战伤害后可以使用手枪进行一次奖励射击（若有的话）' },
    { name: '暗剑', desc: '位于敌人身后发动攻击时，暴击伤害+300%' },
    { name: '埋伏', desc: '使用枪械时可以放弃本回合的攻击，下回合对同一目标造成双倍伤害同时判定百分比+2，总伤害+10' },
    { name: '写影身画鬼', desc: '使用魔术或巫毒术后，下一回合一个影子将会对同样的目标使用一次效果减半的相同术式，无视距离。若目标死亡则无效' },
    { name: '高速思考', desc: '魔术的魔力消耗减半（向上取整）' },
    { name: '冲刺', desc: '攻击后可以选择是否移动两格' },
    { name: '毅力', desc: '受到致命伤时余留一点HP。限一次' },
    { name: '亡灵之舞', desc: '每次受到攻击时，敌人的投掷判定结果+1（不影响总值）ATK+8，本场战斗中每击杀一名敌人额外+10' },
  ],
}

const heroSkillList = [
  { name: '不动铁心', desc: '每回合减少10层精神异常的层数。暴击触发范围+2 暴击伤害提升至300%' },
  { name: '未踏军神之路', desc: 'ATK+30 EVA+20 RES+20 DEF-20 如果周围12格内有队友则失效' },
  { name: '落文-春夏秋冬题', desc: '每个自己的第一二个回合开始时，为自己以及周围5格内的友军回复14点HP。第三四个回合效果变为赋予自己以及友军一次优势同时ATK+14。结束后循环' },
  { name: '弓穿独霞，彷徨之心', desc: '每次射击时获得2层特殊状态“待机”10层后清除“待机”状态，并且接下来的三回合内每回合可以额外发动一次攻击，额外攻击时投掷判定点数+2。射击时还可以造成暴击，暴击范围为投掷判定结果为10' },
]

function isPassiveIReward(r) {
  const t = r?.content || ''
  return (t.includes('被动页I') || t.includes('被动I')) &&
         !t.includes('被动页II') && !t.includes('被动II') &&
         !t.includes('被动页III') && !t.includes('被动III')
}

function isPassiveIIReward(r) {
  const t = r?.content || ''
  return (t.includes('被动页II') || t.includes('被动II')) &&
         !t.includes('被动页III') && !t.includes('被动III')
}

function isPassiveIIIReward(r) {
  const t = r?.content || ''
  return t.includes('被动页III') || t.includes('被动III')
}

function isHeroReward(r) {
  const t = r?.content || ''
  return t.includes('英雄技能') || t.includes('英名铭刻')
}

// 所有物品库
const itemCatalog = ref([])          
const loadingItems = ref(false)

// 物品种类迷你图标
// 武器：按大类（category）
// 防具/素材/物品：按小类（sub_type）
// 食物：统一一个
const itemTypeIcons = {
  // ===== 武器（大类）=====
  '刀剑': '/items/刀剑.png',
  '长枪': '/items/长枪.png',
  '斧子': '/items/斧子.png',
  '枪械': '/items/枪械.png',
  '魔导用具': '/items/魔导用具.png',
  '盾牌': '/items/盾牌.png',

  // ===== 防具（小类）=====
  '头盔': '/items/头盔.png',
  '胸甲': '/items/胸甲.png',
  '护腿': '/items/护腿.png',
  '背包': '/items/背包.png',

  // ===== 素材（小类）=====
  '常见素材': '/items/常见素材.png',
  '稀有素材': '/items/稀有素材.png',
  'Rarität': '/items/罕世遗物.png',   // 文件名可按你实际命名改

  // ===== 物品（小类）=====
  '药物': '/items/药物.png',
  '军工': '/items/军工.png',
  '任务用品': '/items/任务用品.png',
  '其他': '/items/其他.png',

  // ===== 食物 =====
  '食物': '/items/食物.png',
}

function getItemIcon(item) {
  if (!item) return ''
  // 先匹配小类（头盔、药物、Rarität 等）
  if (item.sub_type && itemTypeIcons[item.sub_type]) {
    return itemTypeIcons[item.sub_type]
  }
  // 再匹配大类（刀剑、枪械、食物等）
  if (item.category && itemTypeIcons[item.category]) {
    return itemTypeIcons[item.category]
  }
  // 物品单独指定的 icon（可选）
  if (item.icon) return item.icon
  return ''
}

// 房间相关
const roomName = ref('')
const gmPassword = ref('')
const joinCode = ref('')
const joinGmPassword = ref('')
const currentSession = ref(null)
const isGM = ref(false)
const message = ref('')

const myCharacterName = ref('')
const claimNameInput = ref('')

const handbookFrom = ref('room')  // 从哪里进入手册，方便返回
const handbookClass = ref(null)   // 当前查看的职业介绍

function openHandbook(from = 'room') {
  handbookFrom.value = from
  handbookClass.value = null
  page.value = 'handbook'
}

function backFromHandbook() {
  if (handbookClass.value) {
    handbookClass.value = null  // 先退回手册目录
    return
  }
  page.value = handbookFrom.value
}

function openClassInHandbook(className) {
  handbookClass.value = className
}

const skillPageTab = ref('I')

function openPassivePage(tab = 'I') {
  skillPageTab.value = tab
  page.value = 'passive'
}

function openHeroPage() {
  page.value = 'hero'
}

function addAbilitySkill(tier, skill) {
  if (!currentCharacter.value) return
  if (!currentCharacter.value.abilitySkills) {
    currentCharacter.value.abilitySkills = { I: [], II: [], III: [], hero: [] }
  }

  const level = currentCharacter.value.level || 1
  const list = currentCharacter.value.abilitySkills[tier] || []
  const maxSlots = { I: 2, II: 2, III: 1, hero: 1 }
  const max = maxSlots[tier] || 1

  if (list.some(s => s.name === skill.name)) {
    alert('已经拥有该技能')
    return
  }
  if (list.length >= max) {
    alert(`「被动${tier === 'hero' ? '英雄技能' : tier}」栏位已满（最多 ${max} 个）`)
    return
  }

  // 按等级解锁对应栏位
  const nextIndex = list.length // 即将填入的第几个（0 起）
  if (tier === 'I') {
    if (nextIndex === 0 && level < 2) { alert('需要达到 2 级才能选择第一个被动 I'); return }
    if (nextIndex === 1 && level < 4) { alert('需要达到 4 级才能选择第二个被动 I'); return }
  }
  if (tier === 'II') {
    if (nextIndex === 0 && level < 6) { alert('需要达到 6 级才能选择第一个被动 II'); return }
    if (nextIndex === 1 && level < 7) { alert('需要达到 7 级才能选择第二个被动 II'); return }
  }
  if (tier === 'III') {
    if (level < 10) { alert('需要达到 10 级才能选择被动 III'); return }
  }
  if (tier === 'hero') {
    if (level < 15) { alert('需要达到 15 级才能选择英雄技能'); return }
  }

  if (!confirm(`确认添加技能「${skill.name}」？`)) return
  list.push({ name: skill.name, desc: skill.desc })
currentCharacter.value.abilitySkills[tier] = list
  alert('已添加')
}

function removeAbilitySkill(tier, index) {
  if (!confirm('确认移除该技能？')) return
  currentCharacter.value.abilitySkills[tier].splice(index, 1)
}

// 角色相关
const characters = ref([])
const newCharacterName = ref('')
const newPlayerName = ref('')
const currentCharacter = ref(null)
const saving = ref(false)

function itemBagBonus(item) {
  if (!item) return { bag: 0, over: 0 }
  const bag = Number(item.bag_slots ?? item.capacity ?? 0) || 0
  const over = Number(item.overweight_slots ?? item.overweight_capacity ?? 0) || 0
  return { bag, over }
}

function equippedSlotBonuses() {
  const eq = currentCharacter.value?.inventory?.equipment
  if (!eq) return { bag: 0, over: 0 }
  let bag = 0
  let over = 0
  for (const slot of ['helmet', 'chest', 'legs', 'mainHand', 'offHand', 'amulet', 'backpack']) {
    const item = getItemById(eq[slot])
    const b = itemBagBonus(item)
    bag += b.bag
    over += b.over
  }
  return { bag, over }
}

const backpackCapacity = computed(() => {
  const extra = equippedSlotBonuses().bag
  return extra > 0 ? extra : 5
})

const overweightCapacity = computed(() => equippedSlotBonuses().over)

const backpackUsed = computed(() => {
  if (!currentCharacter.value?.inventory?.items) return 0
  let total = 0
  const eq = currentCharacter.value.inventory.equipment || {}
  const equippedIds = new Set(Object.values(eq).filter(Boolean).map(id => String(id)))
  for (const entry of currentCharacter.value.inventory.items) {
    if (equippedIds.has(String(entry.item_id))) continue
    const item = getItemById(entry.item_id)
    const per = (item && item.slots) ? Number(item.slots) : 1
    total += per * (entry.quantity || 1)
  }
  return total
})

const normalUsed = computed(() => Math.min(backpackUsed.value, backpackCapacity.value))
const overweightUsed = computed(() => Math.max(0, backpackUsed.value - backpackCapacity.value))
const totalCapacity = computed(() => backpackCapacity.value + overweightCapacity.value)

function canFitItem(itemId, quantity = 1) {
  const item = getItemById(itemId)
  const need = ((item && item.slots) ? Number(item.slots) : 1) * quantity
  return backpackUsed.value + need <= totalCapacity.value
}

// GM 发放物品相关
const showGrantPanel = ref(false)
const grantTargetId = ref('')      // 目标角色 id
const grantItemId = ref('')        // 要发放的物品 id
const grantQuantity = ref(1)
const grantExpandedCategory = ref('')
const grantExpandedSubType = ref('')

// 固定分类树
// 第一层：武器 / 防具 / 素材 / 物品 / 食物
// 武器下面再分：刀剑、长枪、枪械、斧子、魔导用具、盾牌
const categoryTree = {
  '武器': {
    '刀剑': ['长剑', '刺剑', '大剑', '匕首'],
    '长枪': ['长枪', '骑枪'],
    '枪械': ['手枪', '冲锋枪', '霰弹枪', '步枪', '狙击枪'],
    '斧子': ['巨斧', '手斧'],
    '魔导用具': ['魔杖', '活体魔导书', '硬册魔导书', '魔导键'],
    '盾牌': ['长盾', '小盾']
  },
  '防具': {
    '头盔': ['头盔'],
    '胸甲': ['胸甲'],
    '护腿': ['护腿'],
    '背包': ['背包']
},
  '素材': {
    '常见素材': ['常见素材'],
    '稀有素材': ['稀有素材'],
    'Rarität': ['Rarität']
  },
  '物品': {
    '药物': ['药物'],
    '军工': ['军工'],
    '任务用品': ['任务用品'],
    '其他': ['其他']
  },
  '食物': {
    '食物': ['食物']
  }
}

const grantCategories = computed(() => Object.keys(categoryTree))

const grantExpandedWeaponType = ref('')  // 武器下的第二层（刀剑/长枪等）

function getWeaponTypes(topCategory) {
  return Object.keys(categoryTree[topCategory] || {})
}

function getSubTypesByCategory(topCategory, midCategory) {
  return (categoryTree[topCategory] && categoryTree[topCategory][midCategory]) || []
}

// 根据中间分类名（如「刀剑」）和具体小类找物品
// 数据库里 category 存的是「刀剑」「长枪」等
function getItemsByCategoryAndSub(topCategory, midCategory, subType) {
  if (topCategory === '武器') {
    // 武器：数据库 category = 刀剑/长枪/... ，sub_type = 长剑/手枪/...
    return itemCatalog.value.filter(i =>
      i.category === midCategory && (i.sub_type || '未分类') === subType
    )
  }
  // 素材 / 防具 / 物品 / 食物：
  // 数据库 category = 素材/防具/... ，sub_type = Rarität/头盔/...
  return itemCatalog.value.filter(i =>
    i.category === topCategory && (i.sub_type || '未分类') === subType
  )
}

function toggleGrantCategory(cat) {
  if (grantExpandedCategory.value === cat) {
    grantExpandedCategory.value = ''
    grantExpandedWeaponType.value = ''
    grantExpandedSubType.value = ''
  } else {
    grantExpandedCategory.value = cat
    grantExpandedWeaponType.value = ''
    grantExpandedSubType.value = ''
  }
}

function toggleGrantWeaponType(type) {
  if (grantExpandedWeaponType.value === type) {
    grantExpandedWeaponType.value = ''
    grantExpandedSubType.value = ''
  } else {
    grantExpandedWeaponType.value = type
    grantExpandedSubType.value = ''
  }
}

function toggleGrantSubType(sub) {
  grantExpandedSubType.value = grantExpandedSubType.value === sub ? '' : sub
}

function selectGrantItem(itemId) {
  grantItemId.value = itemId
  grantExpandedCategory.value = ''
  grantExpandedWeaponType.value = ''
  grantExpandedSubType.value = ''
}

async function grantItemToCharacter() {
  if (!isGM.value) {
    alert('只有 GM 可以发放物品')
    return
  }
  if (!grantTargetId.value) {
    alert('请选择目标角色')
    return
  }
  if (!grantItemId.value) {
    alert('请选择要发放的物品')
    return
  }
  const qty = grantQuantity.value || 1
  if (qty < 1) {
    alert('数量至少为 1')
    return
  }

  // 找到目标角色
  const target = characters.value.find(c => c.id === grantTargetId.value)
  if (!target) {
    alert('找不到目标角色')
    return
  }

  // 确保 inventory 结构正确
  let inv = target.inventory
  if (!inv || Array.isArray(inv)) {
    inv = { items: [], equipment: { helmet: null, chest: null, legs: null, mainHand: null, offHand: null, amulet: null, backpack: null } }
  }
  if (!inv.items) inv.items = []

  // 如果已有该物品则增加数量，否则新增
  const existing = inv.items.find(i => i.item_id === grantItemId.value)
  if (existing) {
    existing.quantity = (existing.quantity || 0) + qty
  } else {
    inv.items.push({
      id: Date.now(),
      item_id: grantItemId.value,
      quantity: qty
    })
  }

const eq = inv.equipment || {}
let capacity = 0
let overCap = 0
for (const slot of ['helmet', 'chest', 'legs', 'mainHand', 'offHand', 'amulet', 'backpack']) {
  const it = getItemById(eq[slot])
  const b = itemBagBonus(it)
  capacity += b.bag
  overCap += b.over
}
if (capacity <= 0) capacity = 5
const totalCap = capacity + overCap

let used = 0
for (const entry of inv.items) {
  const it = itemCatalog.value.find(i => i.id === entry.item_id)
  used += ((it && it.slots) ? it.slots : 1) * (entry.quantity || 1)
}

const grantItem = itemCatalog.value.find(i => i.id === grantItemId.value)
const need = ((grantItem && grantItem.slots) ? grantItem.slots : 1) * qty

if (used + need > totalCap) {
  alert(`空间不足：需要 ${need} 格，剩余 ${totalCap - used} 格（含超重格）`)
  return
}

  // 保存到数据库
  const { error } = await supabase
    .from('characters')
    .update({ inventory: inv })
    .eq('id', grantTargetId.value)

  if (error) {
    alert('发放失败：' + error.message)
    return
  }

  const grantExpandedCategory = ref('')   // 当前展开的大类
const grantExpandedSubType = ref('')    // 当前展开的小类

// 获取所有大类（去重）
const grantCategories = computed(() => {
  const set = new Set(itemCatalog.value.map(i => i.category))
  return Array.from(set).sort()
})

// 某个大类下的所有小类
function getSubTypesByCategory(category) {
  const set = new Set(
    itemCatalog.value
      .filter(i => i.category === category)
      .map(i => i.sub_type || '未分类')
  )
  return Array.from(set).sort()
}

// 某个大类 + 小类下的物品
function getItemsByCategoryAndSub(category, subType) {
  return itemCatalog.value.filter(i =>
    i.category === category && (i.sub_type || '未分类') === subType
  )
}

function toggleGrantCategory(cat) {
  if (grantExpandedCategory.value === cat) {
    grantExpandedCategory.value = ''
    grantExpandedSubType.value = ''
  } else {
    grantExpandedCategory.value = cat
    grantExpandedSubType.value = ''
  }
}

function toggleGrantSubType(sub) {
  grantExpandedSubType.value = grantExpandedSubType.value === sub ? '' : sub
}

function selectGrantItem(itemId) {
  grantItemId.value = itemId
  // 选中后自动收起，保持界面简洁
  grantExpandedCategory.value = ''
  grantExpandedSubType.value = ''
}

  // 如果当前正在查看这个角色，同步更新界面
  if (currentCharacter.value && currentCharacter.value.id === grantTargetId.value) {
    currentCharacter.value.inventory = inv
  }

  // 刷新角色列表
  await loadCharacters()

  alert('发放成功！')
  grantItemId.value = ''
  grantQuantity.value = 1
  showGrantPanel.value = false
}

// 物品栏相关
const newItemName = ref('')
const newItemQuantity = ref(1)

const mainHandExpanded = ref(false)   // 主手是否展开选择面板
const offHandExpanded = ref(false)    // 副手是否展开选择面板
const otherSlotExpanded = ref('')     // 其他栏位展开状态

// 获取背包中某个栏位实际拥有的大类（排除已被其他栏位装备的物品）
function getOwnedCategories(slot) {
  if (!currentCharacter.value?.inventory?.items) return []
  const equippedIds = Object.values(currentCharacter.value.inventory.equipment || {})
    .filter(id => id && id !== currentCharacter.value.inventory.equipment[slot])

  const cats = new Set()
  currentCharacter.value.inventory.items.forEach(entry => {
    if (equippedIds.includes(entry.item_id)) return  // 已被其他栏位占用
    const item = getItemById(entry.item_id)
    if (!item) return
    if (slot === 'mainHand' && item.slot === 'mainHand') {
      cats.add(item.category)
    } else if (slot === 'offHand') {
      if (['刀剑', '枪械', '斧子', '魔导用具', '盾牌'].includes(item.category)) {
        // 副手刀剑只允许：长剑、刺剑、匕首（按你最初设定）
        if (item.category === '刀剑' && !['长剑', '刺剑', '匕首'].includes(item.sub_type)) return
        cats.add(item.category)
      }
    } else if (item.slot === slot) {
      cats.add(item.category)
    }
  })
  return Array.from(cats)
}

// 获取某大类下可用于该栏位的物品（排除已被装备的）
function getItemsByCategory(slot, category) {
  if (!currentCharacter.value?.inventory?.items) return []
  const equippedIds = Object.values(currentCharacter.value.inventory.equipment || {})
    .filter(id => id && id !== currentCharacter.value.inventory.equipment[slot])

  return currentCharacter.value.inventory.items
    .map(entry => {
      const item = getItemById(entry.item_id)
      return item ? { ...entry, item } : null
    })
    .filter(x => {
      if (!x) return false
      if (equippedIds.includes(x.item_id)) return false
      if (x.item.category !== category) return false
      if (slot === 'mainHand') return x.item.slot === 'mainHand'
      if (slot === 'offHand') {
        if (!['刀剑', '枪械', '斧子', '魔导用具', '盾牌'].includes(x.item.category)) return false
        if (x.item.category === '刀剑' && !['长剑', '刺剑', '匕首'].includes(x.item.sub_type)) return false
        return true
      }
      return x.item.slot === slot
    })
}

// 装备时：如果该物品已在其他栏位，先卸下
function equipItem(slot, itemId) {
  if (!currentCharacter.value?.inventory?.equipment) return
  if (!itemId) {
    currentCharacter.value.inventory.equipment[slot] = null
    return
  }
  // 从其他栏位卸下同一件物品
  const eq = currentCharacter.value.inventory.equipment
  for (const key of Object.keys(eq)) {
    if (key !== slot && eq[key] === itemId) {
      eq[key] = null
    }
  }
  eq[slot] = itemId
}

function discardItem(index) {
  if (!currentCharacter.value?.inventory?.items) return
  const entry = currentCharacter.value.inventory.items[index]
  if (!entry) return
  const item = getItemById(entry.item_id)
  const label = item?.name || '该物品'
  if (!confirm(`确定丢掉「${label}」×${entry.quantity || 1} 吗？丢掉后无法找回。`)) return

  const eq = currentCharacter.value.inventory.equipment || {}
  for (const slot of Object.keys(eq)) {
    if (eq[slot] && String(eq[slot]) === String(entry.item_id)) {
      eq[slot] = null
    }
  }
  currentCharacter.value.inventory.items.splice(index, 1)
}

// 实时订阅
let realtimeChannel = null

// ==================== 职业模板数据 ====================
const classTemplates = {
  '剑士': {
    name: '剑士',
    description: '兼具一定输出能力和防御能力的平衡型职业，面对不同的敌人时可以以不同的架势灵巧应对。',
    mainAttribute: '力量',
    canUseBuff: true,
    status: '完',
    initialATK: 12,
    initialSPD: 50,
    initialRES: 10,
    initialDEF: 1,
    initialHP: 0,
    initialMove: 4,
    initialPP: 75,
    attributeRequirement: '初始力量≤30，次要属性的成长值≥1.6',
    allocatablePoints: 60,
    proficiency: '运动+3；洞悉+5；察觉+5；巧手+2',
    skillBonuses: {
  athletics: 3,   // 运动+3
  insight: 5,     // 洞悉+5
  perception: 5,  // 察觉+5
  sleight: 2      // 巧手+2
},
    equipment: '长剑、轻甲、锁甲、小盾、护身符。最多同时使用一把武器',
    levelRewards: [
      { level: 1, content: '可以使用buff。可以装备长剑、轻甲、锁甲、小盾、护身符。最多同时使用一把武器。\n准备：战斗中以特定架势招架敌人，可在长休时更换。' },
      { level: 2, content: '被动I：可从被动页I中选择一项。' },
      { level: 3, content: '子职I：可选择是否转职为剑士的子职，或继续本职。\n剑士α：解锁反击架势。战斗中若受到伤害，则可以进行一次1d6判定，若结果≥3，则可以进行一次伤害等同于所受税后伤害一半的反击。' },
      { level: 4, content: '被动I：可从被动页I中选择一项。' },
      { level: 5, content: '子职II：\n剑士β：反击架势的反击伤害判定提升至100%。' },
      { level: 6, content: '被动II：可从被动页II中选择一项。' },
      { level: 7, content: '被动II：可从被动页II中选择一项。' },
      { level: 8, content: '主属性增强：力量+10。' },
      { level: 9, content: '子职III：\n剑士γ：反击伤害变为造成敌人造成的伤害而非自身收到的伤害。同时结果大于等于2即可判定为反击成功。' },
      { level: 10, content: '被动III：可从被动页III中选择一项。' },
      { level: 15, content: '英名铭刻：可从英雄技能页中选择一项。' }
    ],
    stances: [
      { name: '学徒的清击', desc: '在攻击时如果1d10的结果≥5，则伤害总值+6点。LVL3提升至8，LVL9提升至14，LVL15提升至25，同时结果只需≥4即可触发。' },
      { name: '尤欧斯', desc: '倘若在自己的回合开始前有敌人以你为目标进行攻击，在其开始进行计算前你可以对该目标发动一次攻击。触发次数上限1次，自己的回合结束后刷新。LVL6时判定百分比+1=10%，LVL12时次数上限+1。' },
      { name: '剑技铸造', desc: '在自己的回合可对自己无消耗使用buff“熔炉的加护”上限3次，短休补1长休补满。LVL6时本次攻击判定百分比+1=10%，LVL13时额外造成一次等同于本次攻击税前伤害一半的物理伤害。' }
    ],
    subclasses: [
  { 
    name: '决斗者', 
    desc: '子职I：不再可以使用长剑，现在可以使用刺剑。若攻击造成伤害则可以在敌人身上留下5层伤口异常「断筋」持续到战斗结束，上限20层。每层「断筋」将会降低目标1点SPD。\n子职II：现在对拥有任意伤口异常的敌人造成伤害时，伤害总值+8点。此外，拥有「断筋」的敌人在它们的回合结束时会传染给5格内最近的另一名敌人5层伤口异常「断筋」。\n子职III：现在对拥有任意伤口异常的敌人造成伤害时，可选择对其发动决斗。若1d12≥8则决斗成功。目标身上每层「断筋」都会使roll点结果+1。决斗持续到一方死亡为止，玩家决斗者每回合可以发动两次攻击。' 
  },
  { 
    name: '重剑士', 
    desc: '子职I：不再可以使用长剑和小盾，现在可以使用大剑。此外，现在可以装备重甲了。ATK+6，SPD-4。\n子职II：现在可以使用大剑进行主动防御了，数值为（3+1d3）。\n子职III：现在对敌人造成伤害时，还会对其身后3*2格内的敌人造成同等伤害。ATK+10。' 
  }
]
  },
  '赏金猎人': {
    name: '赏金猎人',
    description: '以毒辣手段打击目标的输出型职业。完成赏金任务后可以激活报酬，为自己赚取巨大增益。无法使用buff。',
    mainAttribute: '敏捷',
    canUseBuff: false,
    status: '完',
    initialATK: 10,
    initialSPD: 51,
    initialRES: 10,
    initialDEF: 2,
    initialHP: -5,
    initialMove: 4,
    initialPP: 100,
    attributeRequirement: '主属性成长值固定为2.4，次属性成长值固定为2.3',
    allocatablePoints: 60,
    proficiency: '隐匿+3；求生+4；驯兽+4；知识+2；说服+2；调查+5',
    skillBonuses: {
  stealth: 3,
  survival: 4,
  animal: 4,
  knowledge: 2,
  persuasion: 2,
  investigation: 5
},
    equipment: '长剑、手斧、冲锋枪、轻甲。最多同时使用两把武器。额外获得25初始PP',
    levelRewards: [
      { level: 1, content: '无法使用buff。\n赏金猎人的报酬：完成赏金任务后可以激活报酬获得增益。报酬分为「永久生效」和「单场战斗生效的临时报酬」两种类型。' },
      { level: 2, content: '被动I：可从被动页I中选择一项。' },
      { level: 3, content: '赏金猎人的契约：选择任意一项额外效果：\n· 追猎者：对每场战斗中HP最高的随机一名敌人进行追猎。只要追猎目标存活就能获得4点SPD，并且以追猎目标为攻击对象时不会陷入劣势。\n· 赏金猎人的报酬EX1：任意友方角色（包括自己）的HP等于1时激活。随后获得+5 DEF & EVA（永久报酬）。' },
      { level: 4, content: '被动I：可从被动页I中选择一项。' },
      { level: 5, content: '赏金猎人的契约：选择任意一项额外效果：\n· 拷问者：当攻击目标的HP小于等于最大值的一半时，赏金猎人的每次攻击在进行计算前会向目标施加10层降防属性异常，每层降低目标1点DEF。最多叠加25层。\n· 赏金猎人的报酬EX2：若除自己以外的任意友军在战斗中的投掷判定结果小于3时激活。随后自己在本次战斗中剩余的时间每回合获得一次优势（临时报酬）。' },
      { level: 6, content: '被动II：可从被动页II中选择一项。' },
      { level: 7, content: '被动II：可从被动页II中选择一项。' },
      { level: 8, content: '主属性增强：敏捷+10。' },
      { level: 9, content: '赏金猎人的契约：选择任意一项额外效果：\n· 施暴者：暴击的判定范围提升1。每次暴击时获得20PP。\n· 赏金猎人的报酬EX3：持有100以上PP时激活。激活后每1PP都可提供0.1的次要属性（永久报酬）。' },
      { level: 10, content: '被动III：可从被动页III中选择一项。' },
      { level: 15, content: '英名铭刻：可从英雄技能页中选择一项。' }
    ],
    rewards: {
  permanent: [
    { name: '血的味道', desc: '永久+5 ATK。完成击杀类赏金任务后激活。' },
    { name: '猎手的直觉', desc: '永久+3 察觉与隐匿。完成追踪类赏金任务后激活。' },
    { name: '市场决策', desc: '获得3个稀有掉落物后，每场战斗开始时获得一次属性异常免疫。' },
    { name: '初来驾到', desc: '完成2场战斗后，每场战斗结束后额外获得固定的15PP。' },
    { name: '轻车熟路', desc: '完成5场战斗后提升一级等级。' },
    { name: '霸权执行', desc: '造成4次暴击后，暴击伤害增加50%（200%→250%）。' }
  ],
  temporary: [
    { name: '致命一击预备', desc: '下一次攻击必定暴击（1d10视为10）。使用后消失。' },
    { name: '猎杀标记', desc: '指定一名敌人，每次攻击该目标时获得一次优势。持续至战斗结束或目标死亡。' },
    { name: '冷静收割', desc: '本场战斗中每次击杀回复5点HP，最多触发3次。' },
    { name: '太迟了~', desc: '在战斗中第一个行动时，本场战斗中移动格+1。' },
    { name: '猎击瞄准', desc: '战斗中造成3次伤害后，使用枪械时也可以造成暴击了。' }
  ]
},
    stances: [],
    subclasses: []
  },
  '魔术师': {
    name: '魔术师',
    description: 'Caster。使用魔术行驶魔道者。使用多种魔术的全能型职业。',
    mainAttribute: '智力',
    canUseBuff: false,
    status: '完',
    initialATK: 0,
    initialSPD: 48,
    initialRES: 15,
    initialDEF: 1,
    initialHP: 0,
    initialMove: 4,
    initialPP: 70,
    attributeRequirement: '初始智力=31',
    allocatablePoints: 55,
    proficiency: '洞悉+2；巫毒-2；知识+4；医疗+2；调查+3',
    skillBonuses: {
  insight: 2,
  voodoo: -2,
  knowledge: 4,
  medicine: 2,
  investigation: 3
},
    equipment: '魔导书、魔杖、轻甲。需同时装备魔导书和魔杖。',
    levelRewards: [
  { 
    level: 1, 
    content: '无法使用buff。可以装备魔导书、魔杖、轻甲。需同时装备魔导书和魔杖。\n特异点 锁明：从下列特异点中选择一项：\n· 7的战争：智力+7。任何魔力消耗中带有7的魔术将变为无消耗。\n· 龙胭：伟大之存在的庇护。获得2次特殊状态「龙胭」，每层龙胭提供DEF+5。受到一次伤害后减少一层。每次长休时恢复。\n· 厌花之蛇：能够施加任意毒的魔术的魔力消耗减半。每层毒发的伤害+1。' 
  },
  { level: 2, content: '被动I：可从被动页I中选择一项。' },
  { 
    level: 3, 
    content: '魔道歧途：叩问汝心——\n· 于十二月之寒：每次使用造成冰属性伤害的魔术时，向目标施加1层属性异常「刺骨严寒」，每层降低目标1点DEF，最多叠加10层。\n· 于八月之炎：每次使用魔术时在本次战斗中+1 SPD，使用造成火属性伤害的魔术时变为+2 SPD。' 
  },
  { level: 4, content: '被动I：可从被动页I中选择一项。' },
  { 
    level: 5, 
    content: '魔导迥途：进行抉择——\n· 页：只能使用活体魔导书。每场战斗中每次受到伤害时将会先由魔导书抵挡，魔导书不具备任何DEF/RES，其HP为大小。魔导书将会至少残存1点HP，超额伤害由玩家承受。\n· 册：只能使用硬册魔导书。每使用5次魔术，下一次施法前可以获得一个额外施法机会，并在本次施法中魔力消耗减少100点。全局累计。' 
  },
  { level: 6, content: '被动II：可从被动页II中选择一项。' },
  { level: 7, content: '被动II：可从被动页II中选择一项。' },
  { level: 8, content: '主属性增强：智力+10。' },
  { 
    level: 9, 
    content: '于魔道之止境：踏出步伐——\n· 战胜者：（除毒属性伤害以外的）所有造成单次伤害的魔术伤害+30。\n· 求识者：可学习魔术数量+5。每回合使用魔术后获得一次额外施法机会，本回合中可以使用册数比触发魔术小2册的魔术，并且消耗减少10点。' 
  },
  { level: 10, content: '被动III：可从被动页III中选择一项。' },
  { level: 15, content: '英名铭刻：可从英雄技能页中选择一项。' }
],
    stances: [],
    subclasses: []
  },
  // ===== 待补主职（子职嵌套在内）=====
  '长枪兵': {
    name: '长枪兵',
    description: '使用长枪的AoE攻击型角色。可以使用buff。以出色的速度和攻击范围见长。',
    status: '待补',
    canUseBuff: true,
    subclasses: [
      { name: '盾枪兵', desc: '使用长盾和骑枪的坦克型长枪兵子职。可以使用buff。是队友最可靠的守护者，还可通过精准防御为自己提供攻击增益。' },
      { name: '刃枪兵', desc: '将长剑改造为长枪，大剑改造为骑枪的奇异长枪兵子职。可以使用buff。' }
    ]
  },
  '格斗家': {
    name: '格斗家',
    description: '以拳法致胜的近身格斗型角色。可以使用buff。虽然攻击距离较近，但是凭借灵活冲刺的能力可以有效应对远距离的敌人。',
    status: '待补',
    canUseBuff: true,
    subclasses: []
  },
  '射手': {
    name: '射手',
    description: '使用手枪、冲锋枪、步枪、霰弹枪的远程输出型角色。无法使用buff。可携带三种枪械，具备全能的攻击性能。',
    status: '施工中',
    canUseBuff: false,
    subclasses: []
  },
  '狙击手': {
    name: '狙击手',
    description: '只能使用狙击枪、步枪、手枪的远距离打击特化型射手。无法使用buff。可携带两种枪械，擅长在远距离精准打击。',
    status: '待补',
    canUseBuff: false,
    subclasses: []
  },
  '指挥官': {
    name: '指挥官',
    description: '只能使用手枪和霰弹枪和刺剑的近距离作战特化型射手。无法使用buff。可携带两种枪械，能为周围的友军提供团队增益。',
    status: '待补',
    canUseBuff: false,
    subclasses: []
  },
  '斥候': {
    name: '斥候',
    description: '使用手枪和匕首的独行角色。无法使用buff。面对敌人时可以游刃有余的多次倾泻可怖的火力。',
    status: '待补',
    canUseBuff: false,
    subclasses: [
      { name: '忍者', desc: '使用匕首和手里剑的隐秘型斥候子职。可以使用特殊的能力忍术。在面对海量的敌人时依然能从容应对。' }
    ]
  },
  '处刑者': {
    name: '处刑者',
    description: '使用巨斧的弱点特化型职业，可以使用buff。通过攻击造成伤口，进而通过伤口将敌人处决。',
    status: '待补',
    canUseBuff: true,
    subclasses: []
  },
  '术士': {
    name: '术士',
    description: 'Warlock。召唤者。使用魔导书和法杖施法，术士本人无需成为最强，只要使役最强的存在即可。',
    status: '待补',
    canUseBuff: false,
    subclasses: []
  },
  '萨满': {
    name: '萨满',
    description: '使用巫毒术之人。虽然不能使用魔术，但是只要满足条件即可使用特殊的巫毒术，使用不同的物品施法。',
    status: '待补',
    canUseBuff: false,
    subclasses: [
      { name: '堕落旅人', desc: '使用亡灵巫毒术的尸骸。独特的巫毒术给予其极强的生存能力和干扰能力。使用特殊的物品施法。不能直接出现在大多数大型人类活动区。' }
    ]
  },
  'Code Wizard': {
    name: 'Code Wizard',
    description: '虽然是魔术使，却使用buff之人。使用魔导键施法。根据掌握的buff不同定位而异的灵活职业。',
    status: '待补',
    canUseBuff: true,
    subclasses: []
  }
}

// 完整魔术表（按册 + 种类）
const magicList = {
  volume1: {
    title: '一册魔术（日常）',
    spells: [
      { name: '道化机巧，其一', type: '控制', desc: '召唤一个具备30点HP，没有防御力，持续三回合的人型告示牌。告示牌将会嘲讽所有野兽目标，其他类型的单位也可能会视其为敌人。每名以告示牌为目标的单位死亡时额外掉落5PP（7）' },
      { name: '抽丝剥茧', type: '控制&防御', desc: '提取大地中最为优质的岩石，将岩石升至地表形成一堵石墙。石墙具备50点HP并且没有防御力。若有单位处于石墙之中，则将被束缚（无法移动，施法）直到石墙消失。（7）' },
      { name: '卷土重来', type: '防御', desc: '向后移动3格，并在地上留下移动的痕迹。处于移动痕迹上的单位将会减少12点ATK（10）' },
      { name: '钻地术', type: '防御&回复', desc: '钻入地面之中。钻地期间DEF翻倍并获得一次状态异常免疫效果，结束钻地时回复4点HP。钻地持续1回合（10）' },
      { name: '宁愈术', type: '回复', desc: '持续施法1~2回合。持续施法期间每回合回复自身以及周边4格内的所有友军10点HP（11）' },
      { name: '酸腐兵械', type: '增益', desc: '赋予3格内的友军特殊状态“酸腐兵械”每次攻击后固定造成10点腐蚀魔法伤害。持续3回合（7）' },
      { name: '灾厄化', type: '增益', desc: '赋予3格内的友军特殊状态“灾厄化”使其+8DEF -8RES。持续5回合（9）' },
      { name: '伏特压流', type: '伤害', desc: '对最远5格外的单一目标造成10点电属性魔法伤害。每次使用后在本次战斗中+10本魔术造成的伤害上限，+1魔力消耗（8）' },
      { name: '点火术', type: '伤害', desc: '对最远1格外的敌人造成14点火属性魔法伤害（5）' },
      { name: '热熔术', type: '伤害', desc: '持续施法1回合后，召唤一颗熔岩球。随后可将熔岩球中的熔岩射出对最远7格的单一目标造成30点火属性魔法伤害，最多6次。或将完整的熔岩球射出对最远5格外的2×2范围造成10次6点火属性伤害（13）' }
    ]
  },
  volume2: {
    title: '二册魔术（日常）',
    spells: [
      { name: '胧影之潭', type: '伤害', desc: '将最远5格外的2×2区域变为胧影之潭。处于其中的敌人将会减少10点SPD，并且在回合结束时受到6点伤害（16）' },
      { name: '金流', type: '伤害', desc: '召唤一缕锐利的金粉攻击前方4×3范围的敌人，造成40点物理伤害。（17）' },
      { name: '焚化爆弹', type: '伤害', desc: '持续施法1回合凝练出一颗炽热的爆弹，随后爆弹将会对自身周围4格范围内的所有单位（包括友军）造成两次30点火属性魔法伤害。（20）' },
      { name: '火焰箭', type: '伤害', desc: '对最远4格外的敌人造成28点火属性魔法伤害（10）' },
      { name: '修复术', type: '回复', desc: '瞬间恢复自己或最远2格外的一名友军16点HP（17）' },
      { name: '神智之楔', type: '回复', desc: '操控最远6格外的一名友军的神智，持续2回合。生效期间目标友军不会受到致死性伤害（15）' },
      { name: '幻惑术', type: '控制', desc: '创造一个可控的自身幻象。幻象的HP等同于施法者，不具备防御力，移动格+2。敌人可能会以此幻象为目标（15）' },
      { name: '歌达佩斯大饭店之门', type: '控制', desc: '在最远5格处召唤一扇门扉，经过门扉的敌人将强制停在门扉处投掷判定决定其命运。1~3：被眩晕一回合 4~6：被传送到地图上任意地点 7~9：每回合受到15点虚无伤害 10：受到100点物理伤害（22）' },
      { name: '圣光术', type: '增益', desc: '以自身为中心周围3格的所有友军获得3回合的堕落免疫状态（15）' },
      { name: '橡皮术', type: '防御', desc: '三回合内DEF+4，并且以你为目标的敌人在攻击时将会获得一次劣势（18）' }
    ]
  },
  volume3: {
    title: '三册魔术（专业）',
    spells: [
      { name: '毒香徊天', type: '伤害&控制', desc: '以自身为中心，在周围6格内释放酸雾云，酸雾云持续5回合。其中的敌人每在酸雾云内移动1格就会受到3点腐蚀魔法伤害，若敌人在酸雾云中结束回合，则下一次在酸雾云中移动时受到的伤害+1，最多叠加4次。（37）' },
      { name: '希望信标', type: '增益', desc: '在自身所在位置召唤一个拥有10点HP，不具备防御力的信标。信标将会向全图的友军提供增益，使其受到的单次生命恢复效果必定为最大值（33）' },
      { name: '降咒', type: '增益', desc: '持续施法1回合，随后对全场任意一名友方单位降咒。使其+10力量，+5 SPD，+8 ATK，并清除目标的所有精神异常（32）' },
      { name: '战吼术', type: '防御', desc: '以自身为中心，在周边3格内发出战吼。范围内的友军+20 DEF。每15点力量可以增加1格作用范围（35）' },
      { name: '假死术', type: '防御', desc: '对自己造成3点纯粹伤害。然后进入一回合假死状态，期间无法进行任何行动。所有以你为目标的敌人将改变目标（29）' }
    ]
  },
  volume4: {
    title: '四册魔术（军工）',
    spells: [
      { name: '抽丝剥茧-锐', type: '伤害', desc: '将单一目标的心脏从其体内剥出，造成80点魔法伤害。若投掷结果为1d10=9 or 10，则秒杀人类、野兽等活体目标。' }
    ]
  },
  volume5: {
    title: '五册魔术（军工）',
    spells: []   // 目前空缺
  },
  volume6: {
    title: '六册魔术（神域）',
    spells: []   // 目前空缺
  }
}

// Buff表（按附件三色分类）
const buffList = {
  red: [  // 伤害增益 · 红
    { name: '熔炉的加护', desc: '熔炉室的秘技。为近战武器施加熔炉的加护，将下两次攻击的伤害类型变为火属性的魔法伤害（cost10）' },
    { name: '暴烈的果实', desc: '恩赐之实。获得一颗暴烈果实，食用后本场战斗中+20ATK，受到伤害后失效（cost10）' },
    { name: '军神的加护', desc: '被誉为战斗机器的军神的加护。本回合对人类敌人造成伤害时，判定百分比+1（cost5）' },
    { name: '蛇牙', desc: '遥远大陆的巫毒精髓。接下来的三次攻击造成伤害时，给予敌人4层剧毒（cost9）（10层时每回合受到8点剧毒魔法伤害，20层时受到16点…上限30层）' },
    { name: '狐火之霜', desc: '远东之国的秘术。获得冰属性魔法伤害抗性+5。下两次攻击在造成伤害前将永久减少敌人8DEF（cost18）' },
    { name: '诺伦尼尔的祝福', desc: '向命运的祈祷。本场战斗中在自己回合内主动发动的攻击判定结果+1（不影响最大值）（cost20）' },
    { name: '圣怀斯之歌', desc: '歌谣有曰，汝应痛击恶敌。本场战斗中暴击的触发范围+1（cost15）' }
  ],
  green: [  // 恢复增益 · 绿
    { name: '圣母之光', desc: '高尚之光。回复自身15点HP（cost5）' },
    { name: '暖光的果实', desc: '恩赐之实。食用后获得25×2的持续回复状态，造成伤害时失效（cost10）' },
    { name: '不动之墙', desc: '落日遗迹之墙。获得三次DEF+10效果（cost13）' },
    { name: '龙识日', desc: '远东大陆的传说。免疫下一次伤害≤50的魔术伤害（cost10）' },
    { name: '闭目养神', desc: '免转之境。获得三回合EVA+20（cost15）' },
    { name: '英雄之盾', desc: '于荣光尽头。本场战斗中免疫4点及以下的物理伤害（cost15）' },
    { name: '降灵术（防御）', desc: '降灵术的一种。使用后接下来三次受到伤害前，敌人在判定时获得一次劣势（cost20）' }
  ],
  gold: [  // 其他 · 金
    { name: '澄澈之心', desc: '深呼吸，远离恐惧。解除自身30层精神异常（cost10）' },
    { name: '王道宣言', desc: '守护人民，忠于君主。在本场战斗中免疫精神异常“堕落”（cost10）' },
    { name: '过滤', desc: '静思。解除自身所有的异常状态（cost28）' },
    { name: '重整旗鼓', desc: '不屈不挠。解除自身所有的伤口异常，每接触5层个回复2HP（cost15）' },
    { name: '圣洁的祈祷', desc: '向纯洁神明的祈祷。使用后免疫下一次遭受的异常状态' },
    { name: '大腹便便', desc: '远东异兽的加护。使用后将饱食度归零（cost10）' }
  ]
}

const currentClassInfo = computed(() => {
  if (!currentCharacter.value || !currentCharacter.value.class_name) return null
  return classTemplates[currentCharacter.value.class_name] || null
})

// 主属性加值 = 当前主属性 / 10 向下取整
const mainAttrBonus = computed(() => {
  if (!currentCharacter.value || !currentClassInfo.value) return 0
  const main = currentClassInfo.value.mainAttribute
  let val = 0
  if (main === '力量') val = currentCharacter.value.strength || 0
  else if (main === '智力') val = currentCharacter.value.intelligence || 0
  else if (main === '敏捷') val = currentCharacter.value.agility || 0
  return Math.floor(val / 10)
})

// 判断某个技能是否属于当前主属性
function isMainAttrSkill(skillKey) {
  if (!currentClassInfo.value) return false
  const main = currentClassInfo.value.mainAttribute
  const strengthSkills = ['athletics', 'toughness', 'voodoo', 'intimidate']
  const agilitySkills = ['acrobatics', 'sleight', 'stealth', 'survival', 'animal']
  const intelSkills = ['insight', 'medicine', 'perception', 'deception', 'performance', 'persuasion', 'investigation', 'knowledge']
  
  if (main === '力量') return strengthSkills.includes(skillKey)
  if (main === '敏捷') return agilitySkills.includes(skillKey)
  if (main === '智力') return intelSkills.includes(skillKey)
  return false
}

// 判断是否是职业精通技能（用于高亮）
function isProficientSkill(skillKey) {
  if (!currentClassInfo.value || !currentClassInfo.value.skillBonuses) return false
  return currentClassInfo.value.skillBonuses[skillKey] !== undefined
}

function generateCode() {
  const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'
  let code = ''
  for (let i = 0; i < 6; i++) {
    code += chars.charAt(Math.floor(Math.random() * chars.length))
  }
  return code
}

function saveSessionToLocal() {
  if (currentSession.value) {
    localStorage.setItem('rpg_session', JSON.stringify({
      session: currentSession.value,
      isGM: isGM.value
    }))
  }
}

function restoreSessionFromLocal() {
  const saved = localStorage.getItem('rpg_session')
  if (saved) {
    try {
      const data = JSON.parse(saved)
      if (!data.session || !data.session.code) {
        localStorage.removeItem('rpg_session')
        return false
      }
      currentSession.value = data.session
      isGM.value = data.isGM
      myCharacterName.value = localStorage.getItem('rpg_char_' + data.session.code) || ''
      loadCharacters()
      loadRoomMessages()
      startRealtime()
      loadCurrentMission()
      // 已绑定角色名 → 大厅；否则 → 房间选角页
      page.value = myCharacterName.value ? 'lobby' : 'room'
      return true
    } catch (e) {
      localStorage.removeItem('rpg_session')
    }
  }
  return false
}

async function createRoom() {
  if (!roomName.value) {
    message.value = '请输入房间名称'
    return
  }
  const code = generateCode()
  const { data, error } = await supabase
    .from('sessions')
    .insert({
      code: code,
      name: roomName.value,
      gm_password: gmPassword.value || null
    })
    .select()
    .single()

  if (error) {
    message.value = '创建失败：' + error.message
  } else {
    currentSession.value = data
    isGM.value = true
    myCharacterName.value = localStorage.getItem('rpg_char_' + currentSession.value.code) || ''
    page.value = 'room'
    saveSessionToLocal()
    loadCharacters()
    loadRoomMessages()
    loadCurrentMission()
    startRealtime()
  }
}

async function joinRoom() {
  if (!joinCode.value) {
    message.value = '请输入房间码'
    return
  }
  const { data, error } = await supabase
    .from('sessions')
    .select('*')
    .eq('code', joinCode.value.toUpperCase())
    .single()

  if (error || !data) {
    message.value = '找不到这个房间，请检查房间码'
    return
  }

  let gm = false
  if (data.gm_password && joinGmPassword.value && joinGmPassword.value === data.gm_password) {
    gm = true
  }

  currentSession.value = data
  isGM.value = gm
  page.value = 'room'
  saveSessionToLocal()
  loadCharacters()
  loadRoomMessages()
  loadCurrentMission()
  startRealtime()
}

async function loadCharacters() {
  if (!currentSession.value) return
  const { data, error } = await supabase
    .from('characters')
    .select('*')
    .eq('session_id', currentSession.value.id)
    .order('name', { ascending: true })

  if (!error) {
    characters.value = data || []
  }
}

async function loadRoomMessages() {
  announceMessages.value = []
  chatMessages.value = []
  if (!currentSession.value?.id) return
  const { data, error } = await supabase
    .from('room_messages')
    .select('*')
    .eq('session_id', currentSession.value.id)
    .order('created_at', { ascending: true })
  if (error) {
    console.error(error)
    return
  }
  const rows = data || []
  announceMessages.value = rows.filter(m => m.channel === 'announce')
  chatMessages.value = rows.filter(m => m.channel === 'chat')
}

function posterName() {
  if (isGM.value) return myCharacterName.value ? ('GM · ' + myCharacterName.value) : 'GM'
  return myCharacterName.value || '未绑定角色'
}

async function sendRoomMessage(channel) {
  if (!currentSession.value?.id) {
    alert('不在房间内')
    return
  }
  const draft = channel === 'announce' ? announceDraft : chatDraft
  const text = (draft.value || '').trim()
  if (!text) {
    alert('请输入内容')
    return
  }
  if (channel === 'announce' && !isGM.value) {
    alert('只有 GM 可以发公告')
    return
  }
  if (channel === 'chat' && !isGM.value && !myCharacterName.value) {
    alert('请先绑定角色再发言')
    return
  }
  const { error } = await supabase.from('room_messages').insert({
    session_id: currentSession.value.id,
    channel,
    author_name: channel === 'announce' ? 'GM' : posterName(),
    body: text
  })
  if (error) alert('发送失败：' + error.message)
  else draft.value = ''
}

function startRealtime() {
  if (!currentSession.value) return
  if (realtimeChannel) {
    supabase.removeChannel(realtimeChannel)
  }

  realtimeChannel = supabase
    .channel('room-' + currentSession.value.id)
    .on(
      'postgres_changes',
      {
        event: '*',
        schema: 'public',
        table: 'characters',
        filter: `session_id=eq.${currentSession.value.id}`
      },
      (payload) => {
        loadCharacters()

        if (
          currentCharacter.value &&
          payload.new &&
          payload.new.id === currentCharacter.value.id
        ) {
          const updated = { ...payload.new }

          // ---------- inventory 兼容 ----------
          if (!updated.inventory || Array.isArray(updated.inventory)) {
            const oldItems = Array.isArray(updated.inventory) ? updated.inventory : []
            updated.inventory = {
              items: oldItems,
              equipment: {
                helmet: null,
                chest: null,
                legs: null,
                mainHand: null,
                offHand: null,
                amulet: null,
                backpack: null
              }
            }
          } else {
            if (!updated.inventory.items) updated.inventory.items = []
            if (!updated.inventory.equipment) {
              updated.inventory.equipment = {
                helmet: null,
                chest: null,
                legs: null,
                mainHand: null,
                offHand: null,
                amulet: null,
                backpack: null
              }
            } else {
              const eq = updated.inventory.equipment
              ;['helmet', 'chest', 'legs', 'mainHand', 'offHand', 'amulet', 'backpack'].forEach((k) => {
                if (eq[k] === undefined) eq[k] = null
              })
            }
          }

          // ---------- skills 兼容 ----------
          const defaultSkills = {
            athletics: 4, toughness: 4, voodoo: 4, intimidate: 4,
            acrobatics: 4, sleight: 4, stealth: 4, survival: 4, animal: 4,
            insight: 4, medicine: 4, perception: 4, deception: 4,
            performance: 4, persuasion: 4, investigation: 4, knowledge: 4
          }
          if (!updated.skills) {
            updated.skills = { ...defaultSkills }
          } else {
            for (const key in defaultSkills) {
              if (updated.skills[key] === undefined || updated.skills[key] === null) {
                updated.skills[key] = defaultSkills[key]
              }
            }
          }

          // ---------- abilitySkills 兼容 ----------
          if (!updated.abilitySkills && updated.ability_skills) {
            updated.abilitySkills = updated.ability_skills
          }
          if (!updated.abilitySkills) {
            updated.abilitySkills = { I: [], II: [], III: [], hero: [] }
          } else {
            updated.abilitySkills = {
              I: updated.abilitySkills.I || [],
              II: updated.abilitySkills.II || [],
              III: updated.abilitySkills.III || [],
              hero: updated.abilitySkills.hero || []
            }
          }

          // ---------- learnedMagic 兼容 ----------
          if (!updated.learnedMagic && updated.learned_magic) {
            updated.learnedMagic = updated.learned_magic
          }
          if (!updated.learnedMagic) {
            updated.learnedMagic = []
          }

          currentCharacter.value = updated
        }
      }
    )
    .on(
      'postgres_changes',
      {
        event: '*',
        schema: 'public',
        table: 'room_messages',
        filter: `session_id=eq.${currentSession.value.id}`
      },
      () => {
        loadRoomMessages()
      }
    )
    .on(
      'postgres_changes',
      {
        event: '*',
        schema: 'public',
        table: 'missions',
        filter: `session_id=eq.${currentSession.value.id}`
      },
      () => {
        loadCurrentMission()
      }
    )
    .on(
      'postgres_changes',
      { event: '*', schema: 'public', table: 'mission_phantasms' },
      () => { loadMissionPhantasms() }
    )
    .on(
      'postgres_changes',
      { event: '*', schema: 'public', table: 'battle_tokens' },
      () => { loadBattleMap() }
    )
    .subscribe()
}

async function createCharacter() {
  if (!newCharacterName.value) {
    alert('请输入角色名')
    return
  }
  const name = newCharacterName.value.trim()
  if (!name) {
    alert('请输入角色名')
    return
  }

  // 同房间角色名不可重复
  const { data: existing } = await supabase
    .from('characters')
    .select('id')
    .eq('session_id', currentSession.value.id)
    .eq('name', name)

  if (existing && existing.length > 0) {
    alert('该房间已存在同名角色，请换一个名字')
    return
  }

  const { error } = await supabase
    .from('characters')
    .insert({
      session_id: currentSession.value.id,
      name: name,
      player_name: newPlayerName.value || '未命名玩家',
      class_name: '未选择',
      level: 1,
      is_locked: false,
      locked_name: null,
      inventory: {
        items: [],
        equipment: {
          helmet: '', chest: '', legs: '',
          mainHand: '', offHand: '', amulet: '', backpack: ''
        }
      },
      skills: {
        athletics: 4, toughness: 4, voodoo: 4, intimidate: 4,
        acrobatics: 4, sleight: 4, stealth: 4, survival: 4, animal: 4,
        insight: 4, medicine: 4, perception: 4, deception: 4,
        performance: 4, persuasion: 4, investigation: 4, knowledge: 4
      }
    })

  if (error) {
    alert('创建角色失败：' + error.message)
  } else {
    newCharacterName.value = ''
    newPlayerName.value = ''
    loadCharacters()
  }
}


function saveMyCharacterName(name) {
  myCharacterName.value = name
  if (currentSession.value?.code) {
    localStorage.setItem('rpg_char_' + currentSession.value.code, name)
  }
}

async function claimAndEnterCharacter(char) {
  if (char.is_locked) {
    if (myCharacterName.value && myCharacterName.value === char.name) {
      currentCharacter.value = null
      page.value = 'lobby'
      return
    }
    alert('该角色已被其他玩家扮演')
    return
  }

  const { error } = await supabase
    .from('characters')
    .update({
      is_locked: true,
      locked_name: char.name
    })
    .eq('id', char.id)

  if (error) {
    alert('锁定失败：' + error.message)
    return
  }

  saveMyCharacterName(char.name)
  await loadCharacters()
  currentCharacter.value = null
  await enterLobby()
  const myCharacterId = ref(null)
}

async function claimByName() {
  const name = (claimNameInput.value || '').trim()
  if (!name) {
    alert('请输入角色名')
    return
  }
  if (!currentSession.value) {
    alert('请先加入房间')
    return
  }

  const { data, error } = await supabase
    .from('characters')
    .select('*')
    .eq('session_id', currentSession.value.id)
    .eq('name', name)
    .maybeSingle()

  if (error || !data) {
    alert('找不到该角色，请检查名字是否正确')
    return
  }

  if (!data.is_locked) {
    // 未锁定：视为首次扮演
    await claimAndEnterCharacter(data)
    return
  }

  // 已锁定：用名字认领
  saveMyCharacterName(name)
  currentCharacter.value = null
  await enterLobby()
}

// GM 解除锁定
async function unlockCharacter(char) {
  if (!isGM.value) {
    alert('只有 GM 可以解除锁定')
    return
  }
  if (!confirm(`确认解除角色「${char.name}」的锁定？玩家将可以重新选择角色。`)) return

  const { error } = await supabase
    .from('characters')
    .update({ is_locked: false, locked_name: null })
    .eq('id', char.id)

  if (error) {
    alert('解锁失败：' + error.message)
  } else {
    if (myCharacterName.value === char.name) {
      myCharacterName.value = ''
      if (currentSession.value?.code) {
        localStorage.removeItem('rpg_char_' + currentSession.value.code)
      }
    }
    loadCharacters()
    alert('已解除锁定')
  }
}

function enterMyCharacterFromLobby(char) {
  if (!char) return
  // 只允许自己的角色（按名字绑定）
  if (!isGM.value && char.name !== myCharacterName.value) {
    alert('只能查看自己的角色')
    return
  }
  // GM 若也要限制，可去掉 !isGM 例外；目前：玩家只能看自己，GM 可看全部（方便）
  enterCharacter(char)
}

function backToLobby() {
  currentCharacter.value = null
  page.value = 'lobby'
}

function enterCharacter(char) {
  const charCopy = { ...char }



if (!charCopy.learnedMagic) {
  charCopy.learnedMagic = []
}
  // ---------- inventory 新结构 ----------
  // { items: [{ id, item_id, quantity }], equipment: { mainHand: item_id|null, ... } }
  if (!charCopy.inventory || Array.isArray(charCopy.inventory)) {
    // 兼容旧数据
    charCopy.inventory = {
      items: [],
      equipment: {
        helmet: null,
        chest: null,
        legs: null,
        mainHand: null,
        offHand: null,
        amulet: null,
        backpack: null
      }
    }
  } else {
    if (!charCopy.inventory.items) charCopy.inventory.items = []
    if (!charCopy.inventory.equipment) {
      charCopy.inventory.equipment = {
        helmet: null, chest: null, legs: null,
        mainHand: null, offHand: null, amulet: null, backpack: null
      }
    }
  }

  // ---------- skills 兼容处理（保持原来的）----------
  const defaultSkills = {
    athletics: 4, toughness: 4, voodoo: 4, intimidate: 4,
    acrobatics: 4, sleight: 4, stealth: 4, survival: 4, animal: 4,
    insight: 4, medicine: 4, perception: 4, deception: 4,
    performance: 4, persuasion: 4, investigation: 4, knowledge: 4
  }
  if (!charCopy.skills) {
    charCopy.skills = { ...defaultSkills }
  } else {
    for (const key in defaultSkills) {
      if (charCopy.skills[key] === undefined || charCopy.skills[key] === null) {
        charCopy.skills[key] = defaultSkills[key]
      }
    }
  }

if (!charCopy.abilitySkills) {
  charCopy.abilitySkills = { I: [], II: [], III: [], hero: [] }
} else {
  charCopy.abilitySkills = {
    I: charCopy.abilitySkills.I || [],
    II: charCopy.abilitySkills.II || [],
    III: charCopy.abilitySkills.III || [],
    hero: charCopy.abilitySkills.hero || [],
  }
}

  currentCharacter.value = charCopy
  page.value = 'character'
}

function viewCharacterAsGM(char) {
  if (!isGM.value) {
    alert('只有 GM 可以查看其他角色')
    return
  }
  if (!char) return
  enterCharacter(char)
}

function addItem() {
  if (!newItemName.value) {
    alert('请输入物品名称')
    return
  }
  if (!currentCharacter.value.inventory) {
    currentCharacter.value.inventory = { items: [], equipment: {} }
  }
  if (!currentCharacter.value.inventory.items) {
    currentCharacter.value.inventory.items = []
  }
  currentCharacter.value.inventory.items.push({
    id: Date.now(),
    name: newItemName.value,
    quantity: newItemQuantity.value || 1
  })
  newItemName.value = ''
  newItemQuantity.value = 1
}

function removeItem(index) {
  currentCharacter.value.inventory.items.splice(index, 1)
}

async function saveCharacter() {
  if (!currentCharacter.value) return
  saving.value = true
  const { error } = await supabase
    .from('characters')
    .update({
      name: currentCharacter.value.name,
      player_name: currentCharacter.value.player_name,
      class_name: currentCharacter.value.class_name,
      level: currentCharacter.value.level,
      ability_skills: currentCharacter.value.abilitySkills,
      learned_magic: currentCharacter.value.learnedMagic || [],
      strength: currentCharacter.value.strength,
      intelligence: currentCharacter.value.intelligence,
      agility: currentCharacter.value.agility,
      strength_growth: currentCharacter.value.strength_growth,
intelligence_growth: currentCharacter.value.intelligence_growth,
agility_growth: currentCharacter.value.agility_growth,
      hp_current: currentCharacter.value.hp_current,
      hp_max: currentCharacter.value.hp_max,
      atk: currentCharacter.value.atk,
      def: currentCharacter.value.def,
      res: currentCharacter.value.res,
      spd: currentCharacter.value.spd,
      move_range: currentCharacter.value.move_range,
      pp: currentCharacter.value.pp,
      notes: currentCharacter.value.notes,
      inventory: currentCharacter.value.inventory,
      skills: currentCharacter.value.skills,
    })
    .eq('id', currentCharacter.value.id)
  saving.value = false
  if (error) {
    alert('保存失败：' + error.message)
  } else {
    alert('保存成功！')
  }
}

function applyLevelGrowth() {
  if (!currentCharacter.value) return

  const s = Number(currentCharacter.value.strength_growth) || 0
  const i = Number(currentCharacter.value.intelligence_growth) || 0
  const a = Number(currentCharacter.value.agility_growth) || 0

  // 基础属性（保留小数）
  currentCharacter.value.strength = (Number(currentCharacter.value.strength) || 0) + s
  currentCharacter.value.intelligence = (Number(currentCharacter.value.intelligence) || 0) + i
  currentCharacter.value.agility = (Number(currentCharacter.value.agility) || 0) + a

  // 派生属性：用完整小数累计（不在这里取整，让小数能累积）
  currentCharacter.value.hp_max = (Number(currentCharacter.value.hp_max) || 0) + s * 2
  currentCharacter.value.res = (Number(currentCharacter.value.res) || 0) + i * 0.5
  currentCharacter.value.spd = (Number(currentCharacter.value.spd) || 0) + a * 0.1

  // ATK：增加升级后主属性的 1/4（完整小数累计）
  if (currentClassInfo.value) {
    const main = currentClassInfo.value.mainAttribute
    let newMain = 0
    if (main === '力量') newMain = currentCharacter.value.strength
    else if (main === '智力') newMain = currentCharacter.value.intelligence
    else if (main === '敏捷') newMain = currentCharacter.value.agility

    const atkAdd = newMain / 4
    currentCharacter.value.atk = (Number(currentCharacter.value.atk) || 0) + atkAdd
  }

  alert('已应用本级成长值（小数会累计，显示时再向下取整）')
}

// 根据 item_id 从物品库找到物品信息
function getItemById(itemId) {
  if (!itemId) return null
  return itemCatalog.value.find(i => i.id === itemId) || null
}

// 获取角色背包里可用于某个装备位的物品
function getEquippableItems(slot) {
  if (!currentCharacter.value?.inventory?.items) return []
  return currentCharacter.value.inventory.items
    .map(entry => {
      const item = getItemById(entry.item_id)
      return item ? { ...entry, item } : null
    })
    .filter(x => x && (x.item.slot === slot || x.item.slot === 'mainHand' && slot === 'offHand'))
    // 上面简单处理：部分主手物品也可副手（以后可再精细化）
}

// 卸下装备
function unequipItem(slot) {
  if (!currentCharacter.value?.inventory?.equipment) return
  currentCharacter.value.inventory.equipment[slot] = null
}

function leaveRoom() {
  if (realtimeChannel) {
    supabase.removeChannel(realtimeChannel)
    realtimeChannel = null
  }
  localStorage.removeItem('rpg_session')
  page.value = 'home'
  currentSession.value = null
  characters.value = []
  currentCharacter.value = null
  message.value = ''
  joinCode.value = ''
  joinGmPassword.value = ''
}

function openMagicTable() {
  page.value = 'magic'
}

function addMagicSpell(spell, volumeTitle) {
  if (!currentCharacter.value) {
    alert('请先从角色页进入魔术表')
    return
  }
  if (!spell || !spell.name) {
    alert('无效的魔术数据')
    return
  }
  if (!currentCharacter.value.learnedMagic) {
    currentCharacter.value.learnedMagic = []
  }
  const list = currentCharacter.value.learnedMagic
  if (list.some(m => m.name === spell.name)) {
    alert('已经学会该魔术')
    return
  }
  let cap = 2
  try {
    cap = magicCollectionCapacity.value || 2
  } catch (e) {
    cap = 2
  }
  if (list.length >= cap) {
    alert('魔术收藏已满（' + cap + ' 格）')
    return
  }
  if (!confirm('确认学习魔术「' + spell.name + '」？')) {
    return
  }
  list.push({
    name: spell.name,
    type: spell.type || '',
    desc: spell.desc || '',
    volume: volumeTitle || ''
  })
  currentCharacter.value.learnedMagic = list
  alert('已添加：' + spell.name)
}

function removeMagicSpell(index) {
  if (!currentCharacter.value?.learnedMagic) return
  if (!confirm('确认移除？')) return
  currentCharacter.value.learnedMagic.splice(index, 1)
}

function openBuffTable() {
  page.value = 'buff'
}

function backToCharacter() {
  page.value = 'character'
}

onMounted(() => {
  restoreSessionFromLocal()
  loadItemCatalog()
})

async function loadItemCatalog() {
  loadingItems.value = true
  const { data, error } = await supabase
    .from('items')
    .select('*')
    .order('category')
    .order('name')
  
  if (error) {
    console.error('加载物品库失败', error)
    alert('加载物品库失败：' + error.message)
  } else {
    itemCatalog.value = data || []
  }
  loadingItems.value = false
}

onUnmounted(() => {
  if (realtimeChannel) {
    supabase.removeChannel(realtimeChannel)
  }
})
</script>

<template>
  <!-- 首页 -->
  <div v-if="page === 'home'" style="max-width: 500px; margin: 50px auto; font-family: sans-serif;">
    <h1>跑团角色工具</h1>
    <p style="color: #666;">私用多人在线角色卡 · 解体熔炉</p>
    <hr style="margin: 30px 0;" />
    <h2>创建房间</h2>
    <div style="margin-bottom: 15px;">
      <label>房间名称：</label><br />
      <input v-model="roomName" placeholder="例如：第一次冒险" style="width: 100%; padding: 8px; margin-top: 5px;" />
    </div>
    <div style="margin-bottom: 15px;">
      <label>GM 密码（可选）：</label><br />
      <input v-model="gmPassword" type="password" placeholder="设置后只有知道密码的人是GM" style="width: 100%; padding: 8px; margin-top: 5px;" />
    </div>
    <button @click="createRoom" style="padding: 10px 20px; background: #4CAF50; color: white; border: none; cursor: pointer;">创建房间</button>
    <hr style="margin: 40px 0;" />
    <div style="margin: 24px 0; padding: 16px; border: 1px solid #c62828; border-radius: 8px; background: #ffebee;">
      <strong style="color: #c62828;">神秘事件记录簿</strong>
      <p style="margin: 8px 0; font-size: 13px; color: #b71c1c;">
        仅供 GM 编辑与保存战役（含简介、地图、敌人等）。玩家请勿进入。
      </p>
      <button type="button" @click="openBook"
          style="padding: 8px 16px; background: #c62828; color: white; border: none; border-radius: 4px; cursor: pointer;">
          进入记录簿（GM）
      </button>
    </div>
    <h2>加入房间</h2>
    <div style="margin-bottom: 15px;">
      <label>房间码：</label><br />
      <input v-model="joinCode" placeholder="输入6位房间码" style="width: 100%; padding: 8px; margin-top: 5px;" />
    </div>
    <div style="margin-bottom: 15px;">
      <label>GM 密码（可选）：</label><br />
      <input v-model="joinGmPassword" type="password" placeholder="知道密码就是GM" style="width: 100%; padding: 8px; margin-top: 5px;" />
    </div>
    <button @click="joinRoom" style="padding: 10px 20px; background: #2196F3; color: white; border: none; cursor: pointer;">加入房间</button>
    <p style="margin-top: 30px; color: #c00;">{{ message }}</p>
  </div>

      <div v-else-if="page === 'book'" style="max-width: 800px; margin: 30px auto; font-family: sans-serif; padding: 0 20px 40px;">
    <div style="padding: 12px 14px; background: #ffebee; border: 1px solid #e57373; border-radius: 8px; margin-bottom: 20px;">
      <strong style="color: #c62828;">玩家勿入</strong>
      <span style="margin-left: 8px; font-size: 14px; color: #b71c1c;">
        本页为 GM 专用神秘事件记录簿，用于编写与保存战役，请玩家不要进入或修改。
      </span>
    </div>

    <div style="display: flex; justify-content: space-between; align-items: center;">
      <h1>神秘事件记录簿</h1>
      <button type="button" @click="page = 'home'" style="padding: 8px 14px;">返回首页</button>
    </div>

    <!-- 列表 -->
    <div v-if="!editingBook">
      <button type="button" @click="startNewBook"
              style="margin: 16px 0; padding: 8px 16px; background: #3949ab; color: white; border: none; border-radius: 4px; cursor: pointer;">
        新建事件
      </button>
      <div v-if="bookList.length === 0" style="color: #888;">暂无保存的事件</div>
      <div v-for="item in bookList" :key="item.id"
           style="border: 1px solid #ddd; border-radius: 8px; padding: 14px; margin-bottom: 10px;">
        <strong>{{ item.title }}</strong>
        <div style="font-size: 13px; color: #666; margin-top: 6px;">
          {{ item.client || '—' }} ｜ {{ item.location || '—' }} ｜ {{ item.mission_type || '—' }}
        </div>
        <div style="margin-top: 10px; display: flex; gap: 8px;">
          <button type="button" @click="editBookItem(item)" style="padding: 4px 10px; cursor: pointer;">编辑</button>
          <button type="button" @click="deleteBookItem(item)" style="padding: 4px 10px; cursor: pointer; color: #c62828;">删除</button>
        </div>
      </div>
    </div>

    <!-- 编辑表单 -->
    <div v-else style="margin-top: 16px;">
      <h2>{{ editingBook.id ? '编辑事件' : '新建事件' }}</h2>
      <div style="margin-bottom: 8px;">
        <label>标题</label><br />
        <input v-model="bookForm.title" style="width: 100%; padding: 8px; box-sizing: border-box;" />
      </div>
      <div style="margin-bottom: 8px;">
        <label>委托人</label><br />
        <input v-model="bookForm.client" style="width: 100%; padding: 8px; box-sizing: border-box;" />
      </div>
      <div style="margin-bottom: 8px;">
        <label>地点</label><br />
        <input v-model="bookForm.location" style="width: 100%; padding: 8px; box-sizing: border-box;" />
      </div>
      <div style="margin-bottom: 8px;">
        <label>委托类型</label><br />
        <input v-model="bookForm.mission_type" style="width: 100%; padding: 8px; box-sizing: border-box;" />
      </div>
      <div style="margin-bottom: 8px;">
        <label>事件简述</label><br />
        <textarea v-model="bookForm.summary" rows="5" style="width: 100%; padding: 8px; box-sizing: border-box;"></textarea>
      </div>
      <div style="display: flex; gap: 8px;">
        <button type="button" @click="saveBookItem"
                style="padding: 8px 16px; background: #3949ab; color: white; border: none; border-radius: 4px; cursor: pointer;">保存</button>
        <button type="button" @click="editingBook = null; loadBookList()" style="padding: 8px 16px;">返回列表</button>
            <div v-if="editingBook.id" style="margin-top: 24px; border-top: 1px solid #ddd; padding-top: 16px;">
        <h3>地图节点（最多 4 层；每一层可单独设置是否对玩家公开）</h3>

        <button type="button" @click="beginAddRoot"
                style="margin-bottom: 12px; padding: 6px 12px; cursor: pointer;">
          + 新建顶层节点
        </button>

              <div v-if="editingBook.id" style="margin-top: 28px; border-top: 1px solid #ddd; padding-top: 16px;">
        <h3>本战役空想种</h3>
        <p style="font-size: 13px; color: #666;">先在图鉴中创建，再按分类「选用」进本战役。开本局时会复制到本局。</p>

        <div style="margin: 12px 0;">
          <button type="button" @click="startNewPhantasm" style="padding: 6px 12px; cursor: pointer; margin-right: 8px;">新建空想种（图鉴）</button>
          <button type="button" @click="showPhantasmLib = !showPhantasmLib; loadPhantasmList()" style="padding: 6px 12px; cursor: pointer;">
            {{ showPhantasmLib ? '收起图鉴' : '从图鉴选用' }}
          </button>
        </div>

        <!-- 编辑图鉴表单 -->
        <div v-if="editingPhantasm" style="padding: 14px; background: #fff3e0; border-radius: 8px; margin-bottom: 16px;">
          <h4>{{ editingPhantasm.id ? '编辑空想种' : '新建空想种' }}</h4>
          <input v-model="phantasmForm.name" placeholder="名称" style="width: 100%; padding: 8px; margin-bottom: 6px; box-sizing: border-box;" />
          <select v-model="phantasmForm.category" style="width: 100%; padding: 8px; margin-bottom: 8px;">
            <option>常规</option>
            <option>难敌</option>
            <option>Boss</option>
          </select>
          <div style="display: grid; grid-template-columns: repeat(4, 1fr); gap: 6px; margin-bottom: 8px;">
            <label>HP <input type="number" v-model.number="phantasmForm.hp" style="width: 100%;" /></label>
            <label>ATK <input type="number" v-model.number="phantasmForm.atk" style="width: 100%;" /></label>
            <label>SPD <input type="number" v-model.number="phantasmForm.spd" style="width: 100%;" /></label>
            <label>DEF <input type="number" v-model.number="phantasmForm.def" style="width: 100%;" /></label>
            <label>RES <input type="number" v-model.number="phantasmForm.res" style="width: 100%;" /></label>
            <label>力量 <input type="number" v-model.number="phantasmForm.strength" style="width: 100%;" /></label>
            <label>智力 <input type="number" v-model.number="phantasmForm.intelligence" style="width: 100%;" /></label>
            <label>敏捷 <input type="number" v-model.number="phantasmForm.agility" style="width: 100%;" /></label>
            <label>攻击距离 <input type="number" v-model.number="phantasmForm.attack_range" style="width: 100%;" /></label>
            <label>移动格 <input type="number" v-model.number="phantasmForm.move_range" style="width: 100%;" /></label>
          </div>
          <input v-model="phantasmForm.token_url" placeholder="TOKEN 图片 URL（可空）" style="width: 100%; padding: 8px; margin-bottom: 6px; box-sizing: border-box;" />
          <textarea v-model="phantasmForm.notes" rows="2" placeholder="备注" style="width: 100%; padding: 8px; margin-bottom: 8px; box-sizing: border-box;"></textarea>

          <strong>固有技能</strong>
          <div v-for="(s, i) in phantasmForm.skills" :key="'s'+i" style="margin: 6px 0; padding: 8px; background: #fff; border-radius: 4px;">
            <input v-model="s.name" placeholder="技能名" style="width: 100%; margin-bottom: 4px;" />
            <textarea v-model="s.desc" rows="2" placeholder="效果" style="width: 100%;"></textarea>
            <button type="button" @click="removeInnateSkill(i)">删此栏</button>
          </div>
          <button type="button" @click="addInnateSkill">+ 固有技能栏</button>

          <div v-if="phantasmForm.category === 'Boss'" style="margin-top: 12px;">
            <strong>特殊技能（Boss）</strong>
            <div v-for="(s, i) in phantasmForm.boss_skills" :key="'b'+i" style="margin: 6px 0; padding: 8px; background: #fff8e1; border-radius: 4px;">
              <input v-model="s.name" placeholder="特殊技名" style="width: 100%; margin-bottom: 4px;" />
              <textarea v-model="s.desc" rows="2" placeholder="效果" style="width: 100%;"></textarea>
              <button type="button" @click="removeBossSkill(i)">删此栏</button>
            </div>
            <button type="button" @click="addBossSkill">+ 特殊技能栏</button>
          </div>

          <div style="margin-top: 12px;">
            <button type="button" @click="savePhantasmItem" style="padding: 8px 16px; background: #e65100; color: #fff; border: none; border-radius: 4px;">保存到图鉴</button>
            <button type="button" @click="editingPhantasm = null">取消</button>
          </div>
        </div>

        <!-- 图鉴选用 -->
        <div v-if="showPhantasmLib" style="padding: 12px; background: #f5f5f5; border-radius: 8px; margin-bottom: 16px;">
          <div v-for="cat in ['常规','难敌','Boss']" :key="cat" style="margin-bottom: 12px;">
            <strong>{{ cat }}</strong>
            <div v-if="!phantasmsByCategory(phantasmList, cat).length" style="color:#aaa; font-size:13px;">暂无</div>
            <div v-for="p in phantasmsByCategory(phantasmList, cat)" :key="p.id"
                 style="display:flex; justify-content:space-between; align-items:center; padding:6px 0; border-bottom:1px solid #eee;">
              <span>{{ p.name }}　HP{{ p.hp }} / ATK{{ p.atk }}</span>
              <span>
                <button type="button" @click="editPhantasmItem(p)">编辑</button>
                <button type="button" @click="addPhantasmToBook(p.id)">选用到本战役</button>
                <button type="button" @click="deletePhantasmItem(p)" style="color:#c62828;">删图鉴</button>
              </span>
            </div>
          </div>
        </div>

        <!-- 本战役已选用 -->
        <div v-for="cat in ['常规','难敌','Boss']" :key="'sel'+cat" style="margin-top: 10px;">
          <strong>{{ cat }}（本战役）</strong>
          <div v-if="!phantasmsByCategory(bookPhantasms, cat).length" style="color:#aaa; font-size:13px;">未选用</div>
          <div v-for="p in phantasmsByCategory(bookPhantasms, cat)" :key="p.linkId"
               style="display:flex; justify-content:space-between; padding:6px 0;">
            <span>{{ p.name }}</span>
            <button type="button" @click="removePhantasmFromBook(p.linkId)">从本战役移除</button>
          </div>
        </div>
      </div>

        <div v-if="bookNodes.length === 0" style="color: #888; margin-bottom: 12px;">暂无节点</div>

        <div
          v-for="n in bookNodes"
          :key="n.id"
          style="padding: 10px; margin: 8px 0; background: #f5f5f5; border-radius: 6px; border-left: 4px solid #3949ab;"
        >
          <div>
            <strong>{{ n.name }}</strong>
            <span style="font-size: 12px; color: #888; margin-left: 8px;">
              第 {{ nodeDepthLabel(n) }} 层 · {{ parentName(n) }} ·
              {{ n.visible_to_players ? '对玩家公开' : '仅 GM' }}
            </span>
          </div>
          <div style="font-size: 13px; color: #555; margin: 6px 0;">{{ n.description }}</div>
          <div style="display: flex; gap: 8px; flex-wrap: wrap;">
            <button type="button" @click="beginEditNode(n)">编辑此节点</button>
            <button type="button" @click="beginAddChild(n)">在此节点下加子节点</button>
            <button type="button" @click="deleteBookNode(n)" style="color: #c62828;">删除</button>
          </div>
        </div>

        <div style="margin-top: 16px; padding: 14px; background: #e8eaf6; border-radius: 8px;">
          <h4 style="margin-top: 0;">
            {{ editingNodeId ? '编辑节点' : (nodeForm.parent_id ? '新建子节点' : '新建顶层节点') }}
          </h4>
          <p v-if="nodeForm.parent_id" style="font-size: 13px; color: #555;">
            父节点：{{ bookNodes.find(x => x.id === nodeForm.parent_id)?.name || '—' }}
          </p>

          <input
            v-model="nodeForm.name"
            placeholder="节点名称"
            style="width: 100%; padding: 8px; margin-bottom: 6px; box-sizing: border-box;"
          />
          <textarea
            v-model="nodeForm.description"
            placeholder="文字描述"
            rows="3"
            style="width: 100%; padding: 8px; margin-bottom: 6px; box-sizing: border-box;"
          ></textarea>
          <input
            v-model="nodeForm.image_url"
            placeholder="背景图 URL"
            style="width: 100%; padding: 8px; margin-bottom: 6px; box-sizing: border-box;"
          />

          <label style="display: flex; align-items: center; gap: 8px; margin: 8px 0;">
            <input type="checkbox" v-model="nodeForm.visible_to_players" />
            对玩家公开此节点（取消勾选则玩家看到「未知区域」）
          </label>

          <div style="display: flex; gap: 8px; margin-top: 8px;">
            <button
              type="button"
              @click="saveBookNode(editingBook.id)"
              style="padding: 8px 16px; background: #3949ab; color: white; border: none; border-radius: 4px; cursor: pointer;"
            >
              保存节点
            </button>
            <button type="button" @click="resetNodeForm(null)">清空表单</button>
          </div>
        </div>
      </div>

      <p v-else style="margin-top: 16px; color: #888;">请先保存事件，再添加地图节点</p>  
      </div>

      <!-- 地图节点：仅已保存的事件 -->
      <div v-if="editingBook.id" style="margin-top: 24px; border-top: 1px solid #ddd; padding-top: 16px;">
        <h3>地图节点（最多 4 层）</h3>
        <div v-for="n in bookNodes" :key="n.id" style="padding: 8px; margin: 6px 0; background: #f5f5f5; border-radius: 6px;">
          <strong>{{ n.name }}</strong>
          <span style="font-size: 12px; color: #888; margin-left: 8px;">
            {{ n.visible_to_players ? '对玩家公开' : '仅 GM' }}
            {{ n.parent_id ? '（子节点）' : '（顶层）' }}
          </span>
          <div style="font-size: 13px; color: #555;">{{ n.description }}</div>
          <button type="button" @click="nodeForm.parent_id = n.id; editingNodeId = null">加子节点</button>
          <button type="button" @click="editingNodeId = n.id; nodeForm = { name: n.name, description: n.description, image_url: n.image_url, visible_to_players: n.visible_to_players, parent_id: n.parent_id }">编辑</button>
          <button type="button" @click="deleteBookNode(n)">删除</button>
        </div>
        <h4>{{ editingNodeId ? '编辑节点' : '新节点' }}</h4>
        <input v-model="nodeForm.name" placeholder="节点名称" style="width: 100%; padding: 8px; margin-bottom: 6px;" />
        <textarea v-model="nodeForm.description" placeholder="文字描述" rows="3" style="width: 100%; padding: 8px; margin-bottom: 6px;"></textarea>
        <input v-model="nodeForm.image_url" placeholder="背景图 URL" style="width: 100%; padding: 8px; margin-bottom: 6px;" />
        <label>
          <input type="checkbox" v-model="nodeForm.visible_to_players" /> 对玩家公开
        </label>
        <div style="margin-top: 8px;">
          <button type="button" @click="saveBookNode(editingBook.id)">保存节点</button>
        </div>
      </div>
      <p v-else style="margin-top: 16px; color: #888;">请先保存事件，再添加地图节点</p>
    </div>
  </div>

  <!-- 角色列表页 -->
  <div v-else-if="page === 'room'" style="max-width: 800px; margin: 30px auto; font-family: sans-serif; padding: 0 20px;">
    <div style="display: flex; justify-content: space-between; align-items: center;">
      <div>
        <h1>{{ currentSession?.name }}</h1>
        <p>房间码：<strong>{{ currentSession?.code }}</strong>　|　{{ isGM ? '你是 GM' : '你是玩家' }}</p>
      </div>
      <button @click="leaveRoom" style="padding: 8px 16px;">退出房间</button>
    </div>
    <hr style="margin: 20px 0;" />
    <!-- GM 发放物品 -->
<div v-if="isGM" style="margin: 20px 0; padding: 16px; background: #fff3e0; border: 1px solid #ffb74d; border-radius: 8px;">
  <div style="display: flex; justify-content: space-between; align-items: center;">
    <strong style="color: #e65100;">GM 功能：发放物品</strong>
    <button @click="showGrantPanel = !showGrantPanel" style="padding: 6px 12px; background: #ff9800; color: white; border: none; border-radius: 4px; cursor: pointer;">
      {{ showGrantPanel ? '收起' : '打开' }}
    </button>
  </div>
  <div v-if="showGrantPanel" style="margin-top: 14px; display: flex; flex-wrap: wrap; gap: 12px; align-items: flex-end;">
    <div>
      <label style="font-size: 13px; color: #666;">目标角色</label><br />
      <select v-model="grantTargetId" style="padding: 8px; min-width: 160px;">
        <option value="">选择角色</option>
        <option v-for="c in characters" :key="c.id" :value="c.id">
          {{ c.name }}（{{ c.player_name || '未知玩家' }}）
        </option>
      </select>
    </div>
   <div style="min-width: 260px; flex: 1;">
  <label style="font-size: 13px; color: #666;">物品</label>

  <!-- 已选中的物品显示 -->
  <div v-if="grantItemId" style="margin-top: 6px; padding: 8px 10px; background: #fff8e1; border-radius: 6px; display: flex; justify-content: space-between; align-items: center;">
    <span>
      <strong>{{ getItemById(grantItemId)?.name }}</strong>
      <span style="color: #888; margin-left: 6px; font-size: 13px;">
        {{ getItemById(grantItemId)?.category }} · {{ getItemById(grantItemId)?.sub_type }}
      </span>
    </span>
    <button @click="grantItemId = ''" style="padding: 2px 8px; font-size: 12px; background: #eee; border: none; border-radius: 4px; cursor: pointer;">
      清除
    </button>
  </div>

  <!-- 未选择时显示分类树 -->
  <div v-else style="margin-top: 6px; border: 1px solid #e0e0e0; border-radius: 6px; max-height: 360px; overflow-y: auto;">
  <div v-for="top in grantCategories" :key="top">
    <!-- 第一层：武器 / 防具 / 素材 ... -->
    <div @click="toggleGrantCategory(top)"
         style="padding: 8px 12px; background: #f5f5f5; cursor: pointer; display: flex; justify-content: space-between; border-bottom: 1px solid #eee; user-select: none;">
      <span style="font-weight: 600;">{{ top }}</span>
      <span style="color: #888;">{{ grantExpandedCategory === top ? '▲' : '▼' }}</span>
    </div>

    <div v-show="grantExpandedCategory === top" style="background: #fafafa;">
      <!-- 第二层：刀剑 / 头盔 / 常见素材 ... -->
      <div v-for="mid in getWeaponTypes(top)" :key="mid">
        <div @click.stop="toggleGrantWeaponType(mid)"
             style="padding: 6px 12px 6px 20px; cursor: pointer; display: flex; justify-content: space-between; border-bottom: 1px solid #f0f0f0; font-size: 13px; color: #444; user-select: none;">
          <span>{{ mid }}</span>
          <span style="color: #aaa;">{{ grantExpandedWeaponType === mid ? '▲' : '▼' }}</span>
        </div>

        <div v-show="grantExpandedWeaponType === mid" style="background: #fff;">
          <!-- 第三层：具体小类 -->
          <div v-for="sub in getSubTypesByCategory(top, mid)" :key="sub">
            <div @click.stop="toggleGrantSubType(sub)"
                 style="padding: 5px 12px 5px 36px; cursor: pointer; display: flex; justify-content: space-between; border-bottom: 1px solid #f5f5f5; font-size: 12px; color: #666; user-select: none;">
              <span>{{ sub }}</span>
              <span style="color: #ccc;">{{ grantExpandedSubType === sub ? '▲' : '▼' }}</span>
            </div>

            <!-- 第四层：具体物品 -->
            <div v-show="grantExpandedSubType === sub" style="background: #fafafa;">
              <div v-if="getItemsByCategoryAndSub(top, mid, sub).length === 0"
     style="padding: 4px 12px 4px 48px; font-size: 12px; color: #bbb;">
  （暂无物品）
</div>
              <div v-for="item in getItemsByCategoryAndSub(top, mid, sub)" :key="item.id"
     @click.stop="selectGrantItem(item.id)"
     style="padding: 5px 12px 5px 48px; cursor: pointer; border-bottom: 1px solid #f0f0f0; font-size: 13px;">
  {{ item.name }}
</div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
</div>
    <div>
      <label style="font-size: 13px; color: #666;">数量</label><br />
      <input type="number" v-model.number="grantQuantity" min="1" style="padding: 8px; width: 80px;" />
    </div>
    <button @click="grantItemToCharacter" style="padding: 8px 18px; background: #e65100; color: white; border: none; border-radius: 4px; cursor: pointer;">
      确认发放
    </button>
  </div>
</div>
 
    <!-- 用角色名进入（所有人可见，在 GM 面板外面） -->
    <div style="margin: 20px 0; padding: 14px; background: #e3f2fd; border-radius: 8px;">
      <strong>用角色名进入（重新打开网页时）</strong>
      <div style="display: flex; gap: 10px; margin-top: 10px; flex-wrap: wrap;">
        <input v-model="claimNameInput" placeholder="输入你的角色名"
               style="padding: 8px; flex: 1; min-width: 160px;" />
        <button @click="claimByName"
                style="padding: 8px 16px; background: #1976d2; color: white; border: none; border-radius: 4px; cursor: pointer;">
          进入
        </button>
      </div>
      <p v-if="myCharacterName" style="margin: 8px 0 0; font-size: 13px; color: #555;">
        本机上次绑定：{{ myCharacterName }}
      </p>
    </div>

        <h2>角色列表</h2>
    <div v-if="characters.length === 0" style="color: #888; margin: 20px 0;">目前还没有角色，创建一个吧。</div>
    <div
      v-for="char in characters"
      :key="char.id"
      style="border: 1px solid #ddd; padding: 15px; margin-bottom: 10px; border-radius: 8px; display: flex; justify-content: space-between; align-items: center;"
    >
      <div>
        <strong style="font-size: 18px;">{{ char.name }}</strong>
        <span style="color: #666; margin-left: 10px;">（{{ char.player_name }}）</span>
        <span v-if="char.is_locked" style="margin-left: 8px; font-size: 12px; color: #e65100;">使用中</span>
        <div style="font-size: 14px; color: #888; margin-top: 4px;">
          职业：{{ char.class_name || '未选择' }}　|　等级：{{ char.level || 1 }}
        </div>
      </div>
      <div style="display: flex; gap: 8px; align-items: center; flex-wrap: wrap;">
        <button
          v-if="isGM"
          type="button"
          @click="viewCharacterAsGM(char)"
          style="padding: 6px 12px; background: #5c6bc0; color: white; border: none; border-radius: 4px; cursor: pointer;"
        >查看</button>
        <button
          v-if="!char.is_locked || char.locked_name === myCharacterName || char.name === myCharacterName"
          type="button"
          @click="claimAndEnterCharacter(char)"
          style="padding: 6px 12px; background: #2196F3; color: white; border: none; border-radius: 4px; cursor: pointer;"
        >扮演角色</button>
        <span v-else style="font-size: 13px; color: #999;">已被扮演</span>
        <button
          v-if="isGM && char.is_locked"
          type="button"
          @click="unlockCharacter(char)"
          style="padding: 6px 12px; background: #ff9800; color: white; border: none; border-radius: 4px; cursor: pointer;"
        >解除锁定</button>
      </div>
    </div>
    <hr style="margin: 30px 0;" />
    <h3>创建新角色</h3>
    <div style="display: flex; gap: 10px; margin-top: 10px; flex-wrap: wrap;">
      <input v-model="newCharacterName" placeholder="角色名" style="padding: 8px; flex: 1; min-width: 150px;" />
      <input v-model="newPlayerName" placeholder="玩家昵称（可选）" style="padding: 8px; flex: 1; min-width: 150px;" />
      <button @click="createCharacter" style="padding: 8px 20px; background: #4CAF50; color: white; border: none; cursor: pointer;">创建角色</button>
    </div>
  </div>

  <!-- 大厅页 -->
  <div v-else-if="page === 'lobby'" style="max-width: 900px; margin: 30px auto; font-family: sans-serif; padding: 0 20px;">
    <div style="display: flex; justify-content: space-between; align-items: center;">
      <div>
        <h1>大厅</h1>
        <p>
          房间：<strong>{{ currentSession?.name }}</strong>
          （{{ currentSession?.code }}）
          ｜ {{ isGM ? 'GM' : '玩家' }}
          ｜ 你的角色：<strong>{{ myCharacterName || '未绑定' }}</strong>
        </p>
      </div>
      <div style="display: flex; gap: 8px;">
        <button v-if="isGM" type="button" @click="page = 'room'" style="padding: 8px 14px;">房间管理</button>
        <button type="button" @click="leaveRoom" style="padding: 8px 14px;">退出房间</button>
      </div>
    </div>
    <hr style="margin: 20px 0;" />

    <div v-if="isGM && currentMission" style="margin-bottom: 20px; padding: 16px; border: 1px solid #bf360c; border-radius: 10px; background: #fbe9e7;">
      <strong style="color: #bf360c;">本局空想种</strong>
      <div v-if="!missionPhantasms.length" style="margin-top: 8px; color: #888;">本局未复制到空想种</div>
      <div v-for="cat in ['常规','难敌','Boss']" :key="'m'+cat" style="margin-top: 10px;">
        <div style="font-weight: bold; font-size: 14px;">{{ cat }}</div>
        <div v-for="e in phantasmsByCategory(missionPhantasms, cat)" :key="e.id"
             style="padding: 8px; margin: 6px 0; background: #fff; border-radius: 6px; font-size: 13px;">
          <strong>{{ e.name }}</strong>
          <span style="color:#666; margin-left:8px;">HP {{ e.hp_current }}/{{ e.hp }}　ATK {{ e.atk }}　SPD {{ e.spd }}</span>
        </div>
      </div>
    </div>

        <!-- 神秘事件（本局） -->
    <div style="margin-bottom: 20px; padding: 16px; border: 1px solid #5c6bc0; border-radius: 10px; background: #e8eaf6;">
      <strong style="color: #3949ab;">神秘事件（本局）</strong>

      <div v-if="currentMission" style="margin-top: 12px;">
        <div style="font-size: 18px; font-weight: bold;">{{ currentMission.title }}</div>
        <div style="font-size: 14px; margin-top: 8px; line-height: 1.6;">
          <div>委托人：{{ currentMission.client || '—' }}</div>
          <div>地点：{{ currentMission.location || '—' }}</div>
          <div>委托类型：{{ currentMission.mission_type || '—' }}</div>
          <div style="margin-top: 6px; white-space: pre-wrap;">{{ currentMission.summary || '—' }}</div>
        </div>
        <button
          v-if="isGM"
          type="button"
          @click="completeMission"
          style="margin-top: 10px; padding: 6px 12px; cursor: pointer;"
        >
          结束本局
        </button>
                <button
          v-if="isGM"
          type="button"
          @click="enterBattle"
          style="margin-left: 8px; padding: 6px 12px; cursor: pointer;"
        >进入战斗地图</button>
        <button
          v-if="!isGM && currentMission.show_battle"
          type="button"
          @click="enterBattle"
          style="margin-left: 8px; padding: 6px 12px; cursor: pointer;"
        >进入战斗地图</button>
      </div>

      <div v-else style="margin-top: 12px;">
        <div style="color: #666; margin-bottom: 10px;">当前没有进行中的事件</div>
        <div v-if="isGM">
          <button
            type="button"
            @click="loadBookList(); showMissionPanel = !showMissionPanel"
            style="padding: 8px 14px; background: #3949ab; color: white; border: none; border-radius: 4px; cursor: pointer;"
          >
            从记录簿选择
          </button>
          <div v-if="showMissionPanel" style="margin-top: 12px;">
            <div v-if="!bookList.length" style="color: #888; font-size: 14px;">
              记录簿为空，请先到首页「神秘事件记录簿」创建并保存
            </div>
            <div
              v-for="b in bookList"
              :key="b.id"
              style="display: flex; justify-content: space-between; align-items: center; margin: 8px 0; padding: 10px; background: #fff; border-radius: 6px;"
            >
              <span>{{ b.title }}</span>
              <button
                type="button"
                @click="startMissionFromBook(b.id)"
                style="padding: 6px 12px; background: #2e7d32; color: white; border: none; border-radius: 4px; cursor: pointer;"
              >
                开始本局
              </button>
            </div>
          </div>
        </div>
        <div v-else style="font-size: 14px; color: #888;">等待 GM 从记录簿开启事件</div>
      </div>
    </div>

    <div style="margin-bottom: 24px; border: 1px solid #ddd; border-radius: 10px; overflow: hidden;">
      <div
        v-if="displayMissionNode"
        :style="{
          minHeight: '200px',
          backgroundImage: displayMissionNode.image_url ? 'url(' + displayMissionNode.image_url + ')' : 'none',
          backgroundSize: 'cover',
          backgroundPosition: 'center',
          padding: '20px',
          color: '#fff',
          textShadow: '0 1px 3px #000',
          backgroundColor: '#333'
        }"
      >
        <div style="font-size: 20px; font-weight: bold;">{{ displayMissionNode.name }}</div>
        <div style="margin-top: 10px; white-space: pre-wrap;">{{ displayMissionNode.description }}</div>
      </div>
      <div
        v-else
        style="min-height: 160px; display: flex; align-items: center; justify-content: center; color: #888; background: #f5f5f5; padding: 16px; text-align: center;"
      >
        <span v-if="!currentMission">当前没有进行中的事件</span>
        <span v-else-if="!missionNodes.length">本局没有地图节点数据</span>
        <span v-else>当前节点无法显示</span>
      </div>
      <div style="padding: 10px 12px; background: #fafafa; border-top: 1px solid #eee;">
        <button type="button" @click="loadCurrentMission" style="padding: 6px 12px; margin-right: 8px; cursor: pointer;">刷新地图 / 事件</button>
        <button type="button" @click="debugMap" style="padding: 6px 12px; cursor: pointer;">调试地图</button>
      </div>
      <div
        v-if="isGM && currentMission && missionNodes.length"
        style="padding: 12px; background: #fafafa; border-top: 1px solid #eee;"
      >
        <strong style="font-size: 13px;">本局地图节点（GM）</strong>
        <div style="font-size: 12px; color: #666; margin: 6px 0 10px;">
          树形显示；点名称切换地点；公开/隐藏只影响本局
        </div>
        <div
          v-for="row in missionNodeFlat"
          :key="row.node.id"
          :style="{
            display: 'flex',
            flexWrap: 'wrap',
            alignItems: 'center',
            gap: '8px',
            marginBottom: '8px',
            paddingLeft: (8 + row.depth * 20) + 'px',
            paddingTop: '8px',
            paddingBottom: '8px',
            paddingRight: '8px',
            background: currentMissionNode && currentMissionNode.id === row.node.id ? '#e8eaf6' : '#fff',
            borderRadius: '6px'
          }"
        >
          <button
            type="button"
            @click="setCurrentNode(row.node.id)"
            style="padding: 4px 10px; cursor: pointer; border: 1px solid #ccc; border-radius: 4px; background: #fff;"
          >
            {{ missionNodePathLabel(row.node) }}
          </button>
          <span style="font-size: 12px; color: #888;">
            {{ row.node.visible_to_players ? '玩家可见' : '仅 GM' }}
          </span>
          <button
            type="button"
            @click="toggleMissionNodeVisibility(row.node)"
            style="padding: 4px 10px; font-size: 12px; cursor: pointer;"
          >
            {{ row.node.visible_to_players ? '设为隐藏' : '设为公开' }}
          </button>
        </div>
      </div>
    </div>

    <div style="margin: 16px 0; padding: 12px; border: 1px solid #ccc; border-radius: 8px; background: #fafafa;">
      <strong>骰子</strong>
      <div style="display: flex; flex-wrap: wrap; gap: 8px; margin-top: 8px;">
        <button type="button" @click="rollDice('1d6')">1d6</button>
        <button type="button" @click="rollDice('1d10')">1d10</button>
        <button type="button" @click="rollDice('1d12')">1d12</button>
        <button type="button" @click="rollDice('1d20')">1d20</button>
        <input v-model="customDice" placeholder="2d6+1" style="width: 80px; padding: 4px;" />
        <button type="button" @click="rollDice(customDice)">自定义</button>
      </div>
      <div style="margin-top: 8px; font-size: 13px; max-height: 120px; overflow: auto;">
        <div v-for="d in diceLogs" :key="d.id" style="padding: 2px 0;">
          {{ d.roller_name }} 掷 {{ d.formula }} → <strong>{{ d.result }}</strong>
          <span style="color:#888;">（{{ d.detail }}）</span>
        </div>
      </div>
    </div>
    
    <h2>成员名单</h2>
    <div v-if="characters.length === 0" style="color: #888;">暂无角色</div>
    <div
        v-for="char in characters"
        :key="char.id"
        style="display: flex; justify-content: space-between; align-items: center; padding: 8px 0; border-bottom: 1px solid #eee;"
      >
        <div>
          <strong>{{ char.name }}</strong>
          <span style="color:#888; margin-left: 8px;">{{ char.player_name }}</span>
          <span v-if="char.is_locked" style="margin-left: 8px; font-size: 12px; color: #e65100;">使用中</span>
        </div>
        <div style="display: flex; gap: 8px;">
          <button
            v-if="isGM"
            type="button"
            @click="viewCharacterAsGM(char)"
            style="padding: 6px 12px; background: #5c6bc0; color: white; border: none; border-radius: 4px; cursor: pointer;"
          >查看</button>
          <button
            v-if="char.name === myCharacterName"
            type="button"
            @click="enterCharacter(char); page = 'character'"
            style="padding: 6px 12px; background: #2196F3; color: white; border: none; border-radius: 4px; cursor: pointer;"
          >进入角色</button>
        </div>
      </div>
  </div>
  
  <div v-if="page === 'battle'" style="max-width: 1100px; margin: 16px auto; font-family: sans-serif; padding: 0 12px;">
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px;">
      <h1 style="margin: 0;">战斗地图</h1>
      <div>
        <button type="button" @click="exitBattle" style="padding: 8px 14px;">返回大厅</button>
      </div>
    </div>
    <p style="color:#666; font-size:13px;">
      视野 {{ VIEW }}×{{ VIEW }} ／ 全图 {{ MAP_W }}×{{ MAP_H }}
      ｜ 先点 token 再点「移动」，然后点空格
      ｜ 三角切换 HP / ATK / DEF / RES
    </p>

    <div style="margin-bottom: 8px;">
      <button type="button" @click="viewY = viewY - 4; clampView()">上</button>
      <button type="button" @click="viewY = viewY + 4; clampView()">下</button>
      <button type="button" @click="viewX = viewX - 4; clampView()">左</button>
      <button type="button" @click="viewX = viewX + 4; clampView()">右</button>
      <span style="margin-left: 8px; font-size: 13px; color:#666;">原点 ({{ viewX }}, {{ viewY }})</span>
    </div>

    <div v-if="isGM" style="margin: 10px 0; display: flex; flex-wrap: wrap; gap: 8px; align-items: center;">
      <label>添加敌人</label>
      <select v-model="addEnemyId">
        <option value="">选择本局空想种</option>
        <option v-for="e in missionPhantasms" :key="e.id" :value="e.id">
          [{{ e.category }}] {{ e.name }}
        </option>
      </select>
      <button type="button" @click="addEnemyToMap">放到地图</button>
    </div>

        <div style="display: flex; gap: 16px; align-items: flex-start;">
      <div>
        <div
          v-for="row in VIEW"
          :key="'r'+row"
          style="display: flex;"
        >
          <div
            v-for="col in VIEW"
            :key="'c'+col"
            @click="onCellClick(viewX + col - 1, viewY + row - 1)"
            :style="{
              width: '36px',
              height: '36px',
              border: '1px solid #ddd',
              boxSizing: 'border-box',
              position: 'relative',
              background: tokenAt(viewX + col - 1, viewY + row - 1)
                ? (tokenAt(viewX + col - 1, viewY + row - 1).kind === 'player' ? '#e3f2fd' : '#ffebee')
                : ((viewX + col - 1 + viewY + row - 1) % 2 ? '#fafafa' : '#fff'),
              outline: selectedTokenId && tokenAt(viewX + col - 1, viewY + row - 1)?.id === selectedTokenId ? '2px solid #1565c0' : 'none',
              cursor: 'pointer'
            }"
          >
            <template v-if="tokenAt(viewX + col - 1, viewY + row - 1)">
              <div
                style="position: absolute; top: -18px; left: -8px; right: -8px; font-size: 11px; font-weight: bold; text-align: center; white-space: nowrap; z-index: 2;"
                @click.stop
              >
                <button type="button" style="padding:0 2px; font-size:10px;" @click="cycleTokenStat(tokenAt(viewX + col - 1, viewY + row - 1), -1)">◀</button>
                {{ tokenStatKey(tokenAt(viewX + col - 1, viewY + row - 1)).toUpperCase() }}
                {{ tokenStatValue(tokenAt(viewX + col - 1, viewY + row - 1)) }}
                <button type="button" style="padding:0 2px; font-size:10px;" @click="cycleTokenStat(tokenAt(viewX + col - 1, viewY + row - 1), 1)">▶</button>
              </div>
              <div style="font-size: 11px; font-weight: bold; text-align: center; line-height: 36px;">
                {{ tokenAt(viewX + col - 1, viewY + row - 1).label }}
              </div>
            </template>
          </div>
        </div>
      </div>

      <div style="width: 320px; padding: 12px; border: 1px solid #ddd; border-radius: 8px; max-height: 70vh; overflow: auto;">
        <div v-if="!selectedToken" style="color:#888;">点击格子上的 token</div>

        <div v-else-if="selectedToken.kind === 'player'">
          <div style="font-size: 18px; font-weight: bold;">{{ selectedToken.label }}</div>
          <div style="margin: 6px 0;">{{ tokenDisplayName(selectedToken) }}</div>
          <div style="font-size: 13px; color:#555;">
            ATK {{ getTokenSource(selectedToken)?.atk ?? '—' }}
            ｜ DEF {{ getTokenSource(selectedToken)?.def ?? '—' }}
            ｜ RES {{ getTokenSource(selectedToken)?.res ?? '—' }}
          </div>
          <div v-if="canEditHp(selectedToken)" style="margin-top: 10px;">
            <label>HP</label>
            <input
              type="number"
              :value="getTokenSource(selectedToken)?.hp_current ?? 0"
              @change="updateTokenHp(selectedToken, $event.target.value)"
              style="width: 80px; margin-left: 6px;"
            />
          </div>
          <button v-if="canMoveToken(selectedToken)" type="button" @click="beginMove(selectedToken)" style="margin-top: 10px; padding: 6px 12px;">
            {{ pendingMoveTokenId === selectedToken.id ? '点空格完成移动' : '移动' }}
          </button>
          <button v-if="isGM" type="button" @click="removeToken(selectedToken)" style="margin-top: 8px; color: #c62828;">从地图移除</button>
        </div>

        <div v-else>
          <div style="font-size: 18px; font-weight: bold;">{{ tokenDisplayName(selectedToken) }}</div>
          <div style="font-size: 12px; color:#888; margin: 4px 0 8px;">{{ selectedToken.label }}</div>
          <p style="white-space: pre-wrap; font-size: 13px; color:#555; margin: 0 0 12px;">
            {{ getTokenSource(selectedToken)?.notes || '暂无简介' }}
          </p>
          <div v-for="row in [
            { key: 'hp', label: 'HP' },
            { key: 'atk', label: 'ATK' },
            { key: 'def', label: 'DEF' },
            { key: 'res', label: 'RES' },
            { key: 'spd', label: 'SPD' },
            { key: 'strength', label: '力量' },
            { key: 'agility', label: '敏捷' },
            { key: 'intelligence', label: '智力' },
            { key: 'attack_range', label: '攻击距离' },
            { key: 'move_range', label: '移动格' },
          ]" :key="row.key"
               style="display:flex; justify-content:space-between; align-items:center; padding:5px 0; border-bottom:1px solid #eee; font-size:13px; gap:8px;">
            <span>{{ row.label }}</span>
            <span style="text-align:right;">
              <strong>{{ shownEnemyStat(getTokenSource(selectedToken), row.key) }}</strong>
              <label v-if="isGM" style="display:block; font-size:11px; color:#666; margin-top:2px;">
                <input type="checkbox"
                       :checked="!!getTokenSource(selectedToken)?.reveal?.[row.key]"
                       @change="setEnemyReveal(getTokenSource(selectedToken), row.key, $event.target.checked)" />
                公开
              </label>
            </span>
          </div>
          <div v-if="isGM && canEditHp(selectedToken)" style="margin-top: 10px;">
            <label>当前 HP</label>
            <input
              type="number"
              :value="getTokenSource(selectedToken)?.hp_current ?? getTokenSource(selectedToken)?.hp ?? 0"
              @change="updateTokenHp(selectedToken, $event.target.value)"
              style="width: 80px; margin-left: 6px;"
            />
          </div>
          <h4 style="margin: 14px 0 6px;">固有技能</h4>
          <div v-if="!(getTokenSource(selectedToken)?.skills || []).length" style="color:#999; font-size:13px;">无</div>
          <div v-for="(sk, i) in (getTokenSource(selectedToken)?.skills || [])" :key="'s'+i" style="margin: 8px 0; font-size:13px;">
            <div style="display:flex; justify-content:space-between; align-items:center;">
              <strong>{{ sk.name || ('技能' + (i+1)) }}</strong>
              <label v-if="isGM" style="font-size:11px; color:#666;">
                <input type="checkbox"
                       :checked="!!getTokenSource(selectedToken)?.reveal?.skills?.[i]"
                       @change="setEnemySkillReveal(getTokenSource(selectedToken), 'skills', i, $event.target.checked)" />
                公开效果
              </label>
            </div>
            <div style="color:#555; margin-top:2px;">{{ shownEnemySkillDesc(getTokenSource(selectedToken), 'skills', i) }}</div>
          </div>
          <h4 style="margin: 14px 0 6px;">特殊技能</h4>
          <div v-if="!(getTokenSource(selectedToken)?.boss_skills || []).length" style="color:#999; font-size:13px;">无</div>
          <div v-for="(sk, i) in (getTokenSource(selectedToken)?.boss_skills || [])" :key="'b'+i" style="margin: 8px 0; font-size:13px;">
            <div style="display:flex; justify-content:space-between; align-items:center;">
              <strong>{{ sk.name || ('特殊技' + (i+1)) }}</strong>
              <label v-if="isGM" style="font-size:11px; color:#666;">
                <input type="checkbox"
                       :checked="!!getTokenSource(selectedToken)?.reveal?.boss_skills?.[i]"
                       @change="setEnemySkillReveal(getTokenSource(selectedToken), 'boss_skills', i, $event.target.checked)" />
                公开效果
              </label>
            </div>
            <div style="color:#555; margin-top:2px;">{{ shownEnemySkillDesc(getTokenSource(selectedToken), 'boss_skills', i) }}</div>
          </div>
          <button v-if="isGM" type="button" @click="beginMove(selectedToken)" style="margin-top: 12px; padding: 6px 12px;">
            {{ pendingMoveTokenId === selectedToken.id ? '点空格完成移动' : '移动' }}
          </button>
          <button v-if="isGM" type="button" @click="removeToken(selectedToken)" style="margin-top: 8px; color: #c62828;">从地图移除</button>
        </div>
      </div>
    </div>

    <div style="margin: 16px 0; padding: 12px; border: 1px solid #ccc; border-radius: 8px; background: #fafafa;">
      <strong>骰子</strong>
      <div style="display: flex; flex-wrap: wrap; gap: 8px; margin-top: 8px;">
        <button type="button" @click="rollDice('1d6')">1d6</button>
        <button type="button" @click="rollDice('1d10')">1d10</button>
        <button type="button" @click="rollDice('1d12')">1d12</button>
        <button type="button" @click="rollDice('1d20')">1d20</button>
        <input v-model="customDice" placeholder="2d6+1" style="width: 80px; padding: 4px;" />
        <button type="button" @click="rollDice(customDice)">自定义</button>
      </div>
      <div style="margin-top: 8px; font-size: 13px; max-height: 120px; overflow: auto;">
        <div v-for="d in diceLogs" :key="d.id" style="padding: 2px 0;">
          {{ d.roller_name }} 掷 {{ d.formula }} → <strong>{{ d.result }}</strong>
          <span style="color:#888;">（{{ d.detail }}）</span>
        </div>
      </div>
    </div>
  </div>

  <!-- 角色详情页 -->
<div v-else-if="page === 'character'" style="max-width: 950px; margin: 30px auto; font-family: sans-serif; padding: 0 20px;">
  <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;">
    <h1>扮演角色</h1>
    <div>
      <button @click="openHandbook('character')" style="padding: 8px 14px; background: #5c6bc0; color: white; border: none; border-radius: 4px; cursor: pointer; margin-right: 10px;">
        查看员工手册
      </button>
      <button @click="saveCharacter" :disabled="saving" style="padding: 8px 20px; background: #4CAF50; color: white; border: none; margin-right: 10px; cursor: pointer;">
        {{ saving ? '保存中...' : '保存' }}
      </button>
      <button @click="backToLobby">返回大厅</button>
      <button v-if="isGM" @click="page = 'room'">房间管理（发物品/解锁）</button>
    </div>
  </div>
<template v-if="currentCharacter && ['魔术师','术士','Code Wizard'].includes(currentCharacter.class_name)">
  <h2>魔术收藏</h2>
  <div style="margin-bottom: 12px; font-size: 14px; color: #666;">
    栏位：
    <strong>{{ currentCharacter.learnedMagic?.length || 0 }}</strong>
    /
    {{ magicCollectionCapacity }}
    <button type="button" @click="openMagicTable"
            style="margin-left: 12px; padding: 6px 12px; background: #9c27b0; color: white; border: none; border-radius: 4px; cursor: pointer;">
      打开魔术表
    </button>
  </div>
  <div style="border: 1px solid #e1bee7; border-radius: 8px; padding: 12px; margin-bottom: 24px; background: #faf5ff;">
    <div v-if="!(currentCharacter.learnedMagic?.length)" style="color: #999; font-size: 13px;">尚未学习魔术</div>
    <div v-for="(m, i) in (currentCharacter.learnedMagic || [])" :key="m.name + i"
         style="margin: 8px 0; padding: 10px; background: white; border-radius: 6px;">
      <div style="display: flex; justify-content: space-between;">
        <strong style="color: #6a1b9a;">{{ m.name }}</strong>
        <button type="button" @click="removeMagicSpell(i)"
                style="font-size: 11px; color: #c62828; background: none; border: none; cursor: pointer;">移除</button>
      </div>
      <div style="font-size: 13px; color: #555; margin-top: 6px;">{{ m.desc }}</div>
    </div>
  </div>  
</template>

    <!-- 基础信息 -->
    <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin-bottom: 25px;">
      <div>
        <label>角色名</label><br />
        <input v-model="currentCharacter.name" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>玩家昵称</label><br />
        <input v-model="currentCharacter.player_name" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>职业</label><br />
        <select v-model="currentCharacter.class_name" style="width: 100%; padding: 8px;">
          <option value="未选择">未选择</option>
          <optgroup label="已完成">
            <option value="剑士">剑士</option>
            <option value="赏金猎人">赏金猎人</option>
            <option value="魔术师">魔术师</option>
          </optgroup>
          <optgroup label="近战（待补）">
            <option value="长枪兵">长枪兵</option>
            <option value="格斗家">格斗家</option>
            <option value="处刑者">处刑者</option>
          </optgroup>
          <optgroup label="射手（待补）">
            <option value="射手">射手</option>
            <option value="狙击手">狙击手</option>
            <option value="指挥官">指挥官</option>
          </optgroup>
          <optgroup label="特化（待补）">
            <option value="斥候">斥候</option>
          </optgroup>
          <optgroup label="魔术使（待补）">
            <option value="术士">术士</option>
            <option value="萨满">萨满</option>
            <option value="Code Wizard">Code Wizard</option>
          </optgroup>
        </select>
      </div>
      <div>
  <label>等级</label><br />
  <div style="display: flex; gap: 8px; align-items: center;">
    <input type="number" v-model.number="currentCharacter.level" style="width: 100%; padding: 8px;" />
    <button @click="applyLevelGrowth" style="padding: 8px 12px; background: #4CAF50; color: white; border: none; border-radius: 4px; cursor: pointer; white-space: nowrap;">
      应用本级成长
    </button>
  </div>
</div>
    </div>

    <!-- 职业介绍卡片 -->
<div v-if="currentClassInfo" style="background: #f8f9fa; border: 1px solid #e0e0e0; border-radius: 10px; padding: 20px; margin-bottom: 30px;">
  <div style="display: flex; justify-content: space-between; align-items: center;">
    <h2 style="margin: 0;">{{ currentClassInfo.name }}</h2>
    <span v-if="currentClassInfo.status === '待补' || currentClassInfo.status === '施工中'"
          style="background: #ff9800; color: white; padding: 4px 10px; border-radius: 4px; font-size: 13px;">
      {{ currentClassInfo.status }}
    </span>
    <span v-else-if="currentClassInfo.status === '完'"
          style="background: #4CAF50; color: white; padding: 4px 10px; border-radius: 4px; font-size: 13px;">
      已完成
    </span>
  </div>
  <p style="color: #555; line-height: 1.6; margin-top: 12px;">{{ currentClassInfo.description }}</p>

  <template v-if="currentClassInfo.status === '完'">
    <div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 12px; margin: 15px 0; font-size: 14px;">
      <div><strong>主属性：</strong>{{ currentClassInfo.mainAttribute }}</div>
      <div><strong>可使用Buff：</strong>{{ currentClassInfo.canUseBuff ? '是' : '否' }}</div>
      <div><strong>初始ATK：</strong>{{ currentClassInfo.initialATK }}</div>
      <div><strong>初始SPD：</strong>{{ currentClassInfo.initialSPD }}</div>
      <div><strong>初始RES：</strong>{{ currentClassInfo.initialRES }}</div>
      <div><strong>初始DEF：</strong>{{ currentClassInfo.initialDEF }}</div>
      <div><strong>初始移动格：</strong>{{ currentClassInfo.initialMove }}</div>
      <div><strong>初始PP：</strong>{{ currentClassInfo.initialPP }}</div>
    </div>
    <p style="font-size: 14px;"><strong>属性要求：</strong>{{ currentClassInfo.attributeRequirement }}</p>
    <p style="font-size: 14px;"><strong>可分配属性点：</strong>{{ currentClassInfo.allocatablePoints }}</p>
    <p style="font-size: 14px;"><strong>职业精通：</strong>{{ currentClassInfo.proficiency }}</p>
    <p style="font-size: 14px;"><strong>装备权限：</strong>{{ currentClassInfo.equipment }}</p>

    <!-- 等级奖励表 -->
    <div v-if="currentClassInfo.levelRewards && currentClassInfo.levelRewards.length" style="margin-top: 20px;">
      <strong>等级奖励：</strong>
      <div v-for="r in currentClassInfo.levelRewards" :key="r.level"
           style="margin: 10px 0; padding: 10px; background: white; border-radius: 6px; border-left: 4px solid #2196F3;">
        <strong>LVL {{ r.level }}</strong>
        <div style="white-space: pre-line; font-size: 14px; margin-top: 4px;">{{ r.content }}</div>
        <div style="margin-top: 8px;">
          <button v-if="isPassiveIReward(r)" type="button" @click="openPassivePage('I')"
                  style="padding: 4px 10px; margin-right: 6px; font-size: 12px; background: #5c6bc0; color: white; border: none; border-radius: 4px; cursor: pointer;">选择被动 I</button>
          <button v-if="isPassiveIIReward(r)" type="button" @click="openPassivePage('II')"
                  style="padding: 4px 10px; margin-right: 6px; font-size: 12px; background: #5c6bc0; color: white; border: none; border-radius: 4px; cursor: pointer;">选择被动 II</button>
          <button v-if="isPassiveIIIReward(r)" type="button" @click="openPassivePage('III')"
                  style="padding: 4px 10px; margin-right: 6px; font-size: 12px; background: #5c6bc0; color: white; border: none; border-radius: 4px; cursor: pointer;">选择被动 III</button>
          <button v-if="isHeroReward(r)" type="button" @click="openHeroPage()"
                  style="padding: 4px 10px; font-size: 12px; background: #ff9800; color: white; border: none; border-radius: 4px; cursor: pointer;">选择英雄技能</button>
        </div>
      </div>
    </div>

    <!-- 赏金猎人专属报酬 -->
    <div v-if="currentClassInfo.rewards" style="margin-top: 20px;">
      <strong>赏金猎人的报酬：</strong>
      <div style="margin-top: 10px;">
        <div style="font-weight: bold; color: #4CAF50; margin-bottom: 6px;">永久报酬</div>
        <div v-for="(r, i) in currentClassInfo.rewards.permanent" :key="'p'+i"
             style="margin: 6px 0; padding: 8px; background: white; border-radius: 4px; font-size: 14px;">
          <strong>{{ r.name }}</strong>：{{ r.desc }}
        </div>
        <div style="font-weight: bold; color: #ff9800; margin: 12px 0 6px;">临时报酬（单场战斗）</div>
        <div v-for="(r, i) in currentClassInfo.rewards.temporary" :key="'t'+i"
             style="margin: 6px 0; padding: 8px; background: white; border-radius: 4px; font-size: 14px;">
          <strong>{{ r.name }}</strong>：{{ r.desc }}
        </div>
      </div>
    </div>

    <!-- 架势 -->
    <div v-if="currentClassInfo.stances && currentClassInfo.stances.length" style="margin-top: 15px;">
      <strong>架势：</strong>
      <ul style="margin: 8px 0; padding-left: 20px; font-size: 14px;">
        <li v-for="(s, i) in currentClassInfo.stances" :key="i" style="margin-bottom: 8px;">
          <strong>{{ s.name }}</strong>：{{ s.desc }}
        </li>
      </ul>
    </div>
  </template>

  <div v-else style="margin-top: 15px; padding: 15px; background: #fff3e0; border-radius: 6px; color: #e65100;">
    该职业详细数据尚未补全，敬请期待。
  </div>

  <div v-if="currentClassInfo.subclasses && currentClassInfo.subclasses.length" style="margin-top: 20px;">
    <strong>可转职子职：</strong>
    <div v-for="(sub, i) in currentClassInfo.subclasses" :key="i"
         style="margin: 8px 0; padding: 10px; background: white; border-radius: 6px; border-left: 4px solid #9c27b0;">
      <strong>{{ sub.name }}</strong>
      <div style="font-size: 14px; margin-top: 4px; color: #555; white-space: pre-line;">{{ sub.desc }}</div>
    </div>
  </div>

  <div style="margin-top: 20px; display: flex; gap: 10px; flex-wrap: wrap;">
    <button v-if="currentClassInfo.canUseBuff" @click="openBuffTable"
            style="padding: 8px 16px; background: #ff9800; color: white; border: none; border-radius: 4px; cursor: pointer;">
      查看 Buff 表
    </button>
    <button v-if="isMagicUser" @click="openMagicTable"
        style="padding: 8px 16px; background: #9c27b0; color: white; border: none; border-radius: 4px; cursor: pointer;">
  查看魔术表
</button>
  </div>
</div>

    <!-- 核心属性 -->
<h2>核心属性</h2>

<!-- 三属性 + 成长值 -->
<div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(140px, 1fr)); gap: 15px; margin: 15px 0 10px;">
  <div>
    <label>力量</label><br />
    <input type="number" step="0.1" v-model.number="currentCharacter.strength" style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>力量成长</label><br />
    <input type="number" step="0.1" v-model.number="currentCharacter.strength_growth" style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>智力</label><br />
    <input type="number" step="0.1" v-model.number="currentCharacter.intelligence" style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>智力成长</label><br />
    <input type="number" step="0.1" v-model.number="currentCharacter.intelligence_growth" style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>敏捷</label><br />
    <input type="number" step="0.1" v-model.number="currentCharacter.agility" style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>敏捷成长</label><br />
    <input type="number" step="0.1" v-model.number="currentCharacter.agility_growth" style="width: 100%; padding: 8px;" />
  </div>
</div>

<!-- 派生属性（显示时向下取整） -->
<div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(140px, 1fr)); gap: 15px; margin: 10px 0 20px;">
  <div>
    <label>HP 当前</label><br />
    <input type="number"
           :value="Math.floor(currentCharacter.hp_current || 0)"
           @input="currentCharacter.hp_current = Number($event.target.value)"
           style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>HP 最大</label><br />
    <input type="number"
           :value="Math.floor(currentCharacter.hp_max || 0)"
           @input="currentCharacter.hp_max = Number($event.target.value)"
           style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>ATK</label><br />
    <input type="number"
           :value="Math.floor(currentCharacter.atk || 0)"
           @input="currentCharacter.atk = Number($event.target.value)"
           style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>DEF</label><br />
    <input type="number"
           :value="Math.floor(currentCharacter.def || 0)"
           @input="currentCharacter.def = Number($event.target.value)"
           style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>RES</label><br />
    <input type="number"
           :value="Math.floor(currentCharacter.res || 0)"
           @input="currentCharacter.res = Number($event.target.value)"
           style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>SPD</label><br />
    <input type="number"
           :value="Math.floor(currentCharacter.spd || 0)"
           @input="currentCharacter.spd = Number($event.target.value)"
           style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>移动格</label><br />
    <input type="number" v-model.number="currentCharacter.move_range" style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>PP</label><br />
    <input type="number" v-model.number="currentCharacter.pp" style="width: 100%; padding: 8px;" />
  </div>
  <div>
    <label>主属性加值</label><br />
    <input type="number" :value="mainAttrBonus" disabled style="width: 100%; padding: 8px; background: #f0f0f0;" />
    <div style="font-size: 12px; color: #666;">主属性÷10（向下取整）</div>
  </div>
</div>

<!-- 检定技能栏（可折叠） -->
<div style="margin: 30px 0; border: 1px solid #ddd; border-radius: 8px; overflow: hidden;">
  <div @click="skillsExpanded = !skillsExpanded"
       style="padding: 14px 18px; background: #f5f5f5; cursor: pointer; display: flex; justify-content: space-between; align-items: center; user-select: none;">
    <strong style="font-size: 16px;">检定技能栏</strong>
    <span style="color: #666; font-size: 14px;">
      {{ skillsExpanded ? '▲ 收起' : '▼ 展开' }}
    </span>
  </div>

  <template v-if="currentCharacter && ['魔术师','术士','Code Wizard'].includes(currentCharacter.class_name)">
  <h2>魔术收藏</h2>
  <div style="margin-bottom: 12px; font-size: 14px; color: #666;">
    栏位：
    <strong>{{ currentCharacter.learnedMagic?.length || 0 }}</strong>
    /
    {{ magicCollectionCapacity }}
    <span style="margin-left: 8px; font-size: 12px; color: #888;">
      （默认 2 格；装备魔导书可增加）
    </span>
    <button type="button" @click="openMagicTable"
            style="margin-left: 12px; padding: 6px 12px; background: #9c27b0; color: white; border: none; border-radius: 4px; cursor: pointer;">
      打开魔术表
    </button>
  </div>
  <div style="border: 1px solid #e1bee7; border-radius: 8px; padding: 12px; margin-bottom: 24px; background: #faf5ff;">
    <div v-if="!(currentCharacter.learnedMagic?.length)" style="color: #999; font-size: 13px;">
      尚未学习魔术
    </div>
    <div v-for="(m, i) in (currentCharacter.learnedMagic || [])" :key="m.name + i"
         style="margin: 8px 0; padding: 10px; background: white; border-radius: 6px;">
      <div style="display: flex; justify-content: space-between; align-items: center; gap: 8px;">
        <div>
          <strong style="color: #6a1b9a;">{{ m.name }}</strong>
          <span style="margin-left: 8px; font-size: 12px; color: #888;">{{ m.type }}</span>
          <span v-if="m.volume" style="margin-left: 8px; font-size: 12px; color: #aaa;">{{ m.volume }}</span>
        </div>
        <button type="button" @click="removeMagicSpell(i)"
                style="font-size: 11px; color: #c62828; background: none; border: none; cursor: pointer;">
          移除
        </button>
      </div>
      <div style="font-size: 13px; color: #555; margin-top: 6px;">{{ m.desc }}</div>
    </div>
  </div>
</template>

  <h2>被动技能 / 英雄技能</h2>
<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-bottom: 20px;">
  <div v-for="tier in ['I','II','III']" :key="tier" style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;">
      <span>
  <strong>被动 {{ tier }}</strong>
  <span style="font-size: 12px; color: #888;">
    （{{ currentCharacter.abilitySkills?.[tier]?.length || 0 }} / {{ tier === 'III' ? 1 : 2 }}）
  </span>
</span>
      <button type="button" @click="openPassivePage(tier)" style="font-size: 12px; padding: 4px 8px; cursor: pointer;">去选择</button>
    </div>
    <div v-if="!(currentCharacter.abilitySkills?.[tier]?.length)" style="color: #999; font-size: 13px;">未选择</div>
    <div v-for="(s, i) in (currentCharacter.abilitySkills?.[tier] || [])" :key="s.name"
         style="margin: 6px 0; padding: 8px; background: #fafafa; border-radius: 4px; font-size: 13px;">
      <div style="display: flex; justify-content: space-between;">
        <strong>{{ s.name }}</strong>
        <button type="button" @click="removeAbilitySkill(tier, i)" style="font-size: 11px; color: #c62828; background: none; border: none; cursor: pointer;">移除</button>
      </div>
      <div style="color: #666; margin-top: 4px;">{{ s.desc }}</div>
    </div>
  </div>

  <div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;">
      <span>
  <strong>英雄技能</strong>
  <span style="font-size: 12px; color: #888;">
    （{{ currentCharacter.abilitySkills?.hero?.length || 0 }} / 1）
  </span>
</span>
      <button type="button" @click="openHeroPage()" style="font-size: 12px; padding: 4px 8px; cursor: pointer;">去选择</button>
    </div>
    <div v-if="!(currentCharacter.abilitySkills?.hero?.length)" style="color: #999; font-size: 13px;">未选择</div>
    <div v-for="(s, i) in (currentCharacter.abilitySkills?.hero || [])" :key="s.name"
         style="margin: 6px 0; padding: 8px; background: #fff8e1; border-radius: 4px; font-size: 13px;">
      <div style="display: flex; justify-content: space-between;">
        <strong>{{ s.name }}</strong>
        <button type="button" @click="removeAbilitySkill('hero', i)" style="font-size: 11px; color: #c62828; background: none; border: none; cursor: pointer;">移除</button>
      </div>
      <div style="color: #666; margin-top: 4px;">{{ s.desc }}</div>
    </div>
  </div>
</div>

  <div v-show="skillsExpanded" style="padding: 20px;">
    <!-- 力量相关 -->
    <h3 style="margin: 0 0 12px; color: #c62828; border-bottom: 1px solid #ffcdd2; padding-bottom: 6px;">与力量属性有关的能力</h3>
    <div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); gap: 12px; margin-bottom: 24px;">
      <div :style="{ background: isProficientSkill('athletics') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>运动</label><br />
        <input type="number" v-model.number="currentCharacter.skills.athletics" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          攀爬、游泳、跳跃、破门、摔跤等
          <span v-if="isMainAttrSkill('athletics')" style="color: #c62828;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.athletics || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('toughness') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>坚韧</label><br />
        <input type="number" v-model.number="currentCharacter.skills.toughness" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          身体强韧程度，面对伤害的承受能力
          <span v-if="isMainAttrSkill('toughness')" style="color: #c62828;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.toughness || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('voodoo') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>巫毒</label><br />
        <input type="number" v-model.number="currentCharacter.skills.voodoo" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          对巫毒仪式和造物的知识
          <span v-if="isMainAttrSkill('voodoo')" style="color: #c62828;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.voodoo || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('intimidate') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>威吓</label><br />
        <input type="number" v-model.number="currentCharacter.skills.intimidate" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          威胁、逼问、震慑
          <span v-if="isMainAttrSkill('intimidate')" style="color: #c62828;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.intimidate || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
    </div>

    <!-- 敏捷相关（绿色） -->
    <h3 style="margin: 0 0 12px; color: #2e7d32; border-bottom: 1px solid #c8e6c9; padding-bottom: 6px;">与敏捷属性有关的能力</h3>
    <div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); gap: 12px; margin-bottom: 24px;">
      <div :style="{ background: isProficientSkill('acrobatics') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>特技</label><br />
        <input type="number" v-model.number="currentCharacter.skills.acrobatics" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          平衡、翻滚、穿越困难地形
          <span v-if="isMainAttrSkill('acrobatics')" style="color: #2e7d32;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.acrobatics || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('sleight') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>巧手</label><br />
        <input type="number" v-model.number="currentCharacter.skills.sleight" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          开锁、偷窃、解除陷阱、掉包
          <span v-if="isMainAttrSkill('sleight')" style="color: #2e7d32;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.sleight || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('stealth') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>隐匿</label><br />
        <input type="number" v-model.number="currentCharacter.skills.stealth" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          潜行、躲藏
          <span v-if="isMainAttrSkill('stealth')" style="color: #2e7d32;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.stealth || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('survival') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>求生</label><br />
        <input type="number" v-model.number="currentCharacter.skills.survival" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          追踪、野外生存、找路、预测天气
          <span v-if="isMainAttrSkill('survival')" style="color: #2e7d32;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.survival || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('animal') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>驯兽</label><br />
        <input type="number" v-model.number="currentCharacter.skills.animal" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          与动物互动、安抚、骑乘判断
          <span v-if="isMainAttrSkill('animal')" style="color: #2e7d32;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.animal || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
    </div>

    <!-- 智力相关（蓝色） -->
    <h3 style="margin: 0 0 12px; color: #1565c0; border-bottom: 1px solid #bbdefb; padding-bottom: 6px;">与智力属性有关的能力</h3>
    <div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(240px, 1fr)); gap: 12px;">
      <div :style="{ background: isProficientSkill('insight') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>洞悉</label><br />
        <input type="number" v-model.number="currentCharacter.skills.insight" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          察觉谎言、情绪、意图
          <span v-if="isMainAttrSkill('insight')" style="color: #1565c0;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.insight || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('medicine') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>医疗</label><br />
        <input type="number" v-model.number="currentCharacter.skills.medicine" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          诊断伤势、疾病、稳定濒死
          <span v-if="isMainAttrSkill('medicine')" style="color: #1565c0;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.medicine || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('perception') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>察觉</label><br />
        <input type="number" v-model.number="currentCharacter.skills.perception" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          发现隐藏事物、听到动静、观察细节
          <span v-if="isMainAttrSkill('perception')" style="color: #1565c0;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.perception || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('deception') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>欺瞒</label><br />
        <input type="number" v-model.number="currentCharacter.skills.deception" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          说谎、伪装、欺骗
          <span v-if="isMainAttrSkill('deception')" style="color: #1565c0;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.deception || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('performance') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>表演</label><br />
        <input type="number" v-model.number="currentCharacter.skills.performance" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          演出、吸引注意、伪装成艺人
          <span v-if="isMainAttrSkill('performance')" style="color: #1565c0;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.performance || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('persuasion') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>说服</label><br />
        <input type="number" v-model.number="currentCharacter.skills.persuasion" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          谈判、劝说、外交
          <span v-if="isMainAttrSkill('persuasion')" style="color: #1565c0;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.persuasion || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('investigation') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>调查</label><br />
        <input type="number" v-model.number="currentCharacter.skills.investigation" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          推理
          <span v-if="isMainAttrSkill('investigation')" style="color: #1565c0;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.investigation || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
      <div :style="{ background: isProficientSkill('knowledge') ? '#f5f5f5' : 'transparent', padding: '6px', borderRadius: '4px' }">
        <label>知识</label><br />
        <input type="number" v-model.number="currentCharacter.skills.knowledge" style="width: 100%; padding: 6px;" />
        <div style="font-size: 12px; color: #888; margin-top: 2px;">
          历史、神秘学、仪式、图书馆、科技
          <span v-if="isMainAttrSkill('knowledge')" style="color: #1565c0;">
            （+主属性加值 {{ mainAttrBonus }} → 实际 {{ (currentCharacter.skills.knowledge || 0) + mainAttrBonus }}）
          </span>
        </div>
      </div>
    </div>
  </div>
</div>

  <!-- 装备栏 -->
<h2>装备栏</h2>

<div v-if="currentClassInfo" style="margin-bottom: 12px; padding: 10px 14px; background: #e3f2fd; border-radius: 6px; font-size: 14px; color: #1565c0;">
  <strong>本职业可装备：</strong>
  {{ currentClassInfo.equipment || '暂无详细说明' }}
</div>

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 16px; margin-bottom: 24px;">
<!-- 头盔 -->
<div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
  <label style="font-weight: bold;">头盔</label>
  <div @click="slotExpanded = slotExpanded === 'helmet' ? '' : 'helmet'"
       style="margin-top: 8px; padding: 10px; background: #fafafa; border: 1px dashed #ccc; border-radius: 6px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
    <span v-if="getItemById(currentCharacter.inventory.equipment.helmet)" style="display: flex; align-items: center; gap: 6px;">
      <img
        v-if="getItemIcon(getItemById(currentCharacter.inventory.equipment.helmet))"
        :src="getItemIcon(getItemById(currentCharacter.inventory.equipment.helmet))"
        style="width: 32px; height: 32px; image-rendering: pixelated;"
      />
      <strong>{{ getItemById(currentCharacter.inventory.equipment.helmet).name }}</strong>
    </span>
    <span v-else style="color: #999;">无物品</span>
    <span style="color: #888; font-size: 13px;">{{ slotExpanded === 'helmet' ? '收起' : '选择' }}</span>
  </div>
  <div v-if="slotExpanded === 'helmet'" style="margin-top: 8px; border: 1px solid #e0e0e0; border-radius: 6px; padding: 10px; background: #fff;">
    <div v-if="getOwnedCategories('helmet').length === 0" style="color: #bbb; font-size: 13px;">背包中没有可装备的头盔</div>
    <div v-for="cat in getOwnedCategories('helmet')" :key="cat" style="margin-bottom: 8px;">
      <div style="font-size: 13px; color: #666; margin-bottom: 4px;">{{ cat }}</div>
      <div v-for="entry in getItemsByCategory('helmet', cat)" :key="entry.item_id"
           @click="equipItem('helmet', entry.item_id); slotExpanded = ''"
           style="padding: 6px 8px; margin-bottom: 4px; background: #f5f5f5; border-radius: 4px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
        <span style="display: flex; align-items: center; gap: 6px;">
          <img
            v-if="getItemIcon(entry.item)"
            :src="getItemIcon(entry.item)"
            style="width: 24px; height: 24px; image-rendering: pixelated;"
          />
          {{ entry.item.name }}
        </span>
        <span style="color: #888;">×{{ entry.quantity }}</span>
      </div>
    </div>
    <button v-if="currentCharacter.inventory.equipment.helmet"
            @click="unequipItem('helmet'); slotExpanded = ''"
            style="margin-top: 6px; padding: 6px 12px; background: #f44336; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 13px;">
      卸下
    </button>
  </div>
</div>

<!-- 胸甲 -->
<div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
  <label style="font-weight: bold;">胸甲</label>
  <div @click="slotExpanded = slotExpanded === 'chest' ? '' : 'chest'"
       style="margin-top: 8px; padding: 10px; background: #fafafa; border: 1px dashed #ccc; border-radius: 6px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
    <span v-if="getItemById(currentCharacter.inventory.equipment.chest)" style="display: flex; align-items: center; gap: 6px;">
      <img
        v-if="getItemIcon(getItemById(currentCharacter.inventory.equipment.chest))"
        :src="getItemIcon(getItemById(currentCharacter.inventory.equipment.chest))"
        style="width: 32px; height: 32px; image-rendering: pixelated;"
      />
      <strong>{{ getItemById(currentCharacter.inventory.equipment.chest).name }}</strong>
    </span>
    <span v-else style="color: #999;">无物品</span>
    <span style="color: #888; font-size: 13px;">{{ slotExpanded === 'chest' ? '收起' : '选择' }}</span>
  </div>
  <div v-if="slotExpanded === 'chest'" style="margin-top: 8px; border: 1px solid #e0e0e0; border-radius: 6px; padding: 10px; background: #fff;">
    <div v-if="getOwnedCategories('chest').length === 0" style="color: #bbb; font-size: 13px;">背包中没有可装备的胸甲</div>
    <div v-for="cat in getOwnedCategories('chest')" :key="cat" style="margin-bottom: 8px;">
      <div style="font-size: 13px; color: #666; margin-bottom: 4px;">{{ cat }}</div>
      <div v-for="entry in getItemsByCategory('chest', cat)" :key="entry.item_id"
           @click="equipItem('chest', entry.item_id); slotExpanded = ''"
           style="padding: 6px 8px; margin-bottom: 4px; background: #f5f5f5; border-radius: 4px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
        <span style="display: flex; align-items: center; gap: 6px;">
          <img v-if="getItemIcon(entry.item)" :src="getItemIcon(entry.item)" style="width: 24px; height: 24px; image-rendering: pixelated;" />
          {{ entry.item.name }}
        </span>
        <span style="color: #888;">×{{ entry.quantity }}</span>
      </div>
    </div>
    <button v-if="currentCharacter.inventory.equipment.chest"
            @click="unequipItem('chest'); slotExpanded = ''"
            style="margin-top: 6px; padding: 6px 12px; background: #f44336; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 13px;">
      卸下
    </button>
  </div>
</div>

<!-- 护腿 -->
<div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
  <label style="font-weight: bold;">护腿</label>
  <div @click="slotExpanded = slotExpanded === 'legs' ? '' : 'legs'"
       style="margin-top: 8px; padding: 10px; background: #fafafa; border: 1px dashed #ccc; border-radius: 6px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
    <span v-if="getItemById(currentCharacter.inventory.equipment.legs)" style="display: flex; align-items: center; gap: 6px;">
      <img
        v-if="getItemIcon(getItemById(currentCharacter.inventory.equipment.legs))"
        :src="getItemIcon(getItemById(currentCharacter.inventory.equipment.legs))"
        style="width: 32px; height: 32px; image-rendering: pixelated;"
      />
      <strong>{{ getItemById(currentCharacter.inventory.equipment.legs).name }}</strong>
    </span>
    <span v-else style="color: #999;">无物品</span>
    <span style="color: #888; font-size: 13px;">{{ slotExpanded === 'legs' ? '收起' : '选择' }}</span>
  </div>
  <div v-if="slotExpanded === 'legs'" style="margin-top: 8px; border: 1px solid #e0e0e0; border-radius: 6px; padding: 10px; background: #fff;">
    <div v-if="getOwnedCategories('legs').length === 0" style="color: #bbb; font-size: 13px;">背包中没有可装备的护腿</div>
    <div v-for="cat in getOwnedCategories('legs')" :key="cat" style="margin-bottom: 8px;">
      <div style="font-size: 13px; color: #666; margin-bottom: 4px;">{{ cat }}</div>
      <div v-for="entry in getItemsByCategory('legs', cat)" :key="entry.item_id"
           @click="equipItem('legs', entry.item_id); slotExpanded = ''"
           style="padding: 6px 8px; margin-bottom: 4px; background: #f5f5f5; border-radius: 4px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
        <span style="display: flex; align-items: center; gap: 6px;">
          <img v-if="getItemIcon(entry.item)" :src="getItemIcon(entry.item)" style="width: 24px; height: 24px; image-rendering: pixelated;" />
          {{ entry.item.name }}
        </span>
        <span style="color: #888;">×{{ entry.quantity }}</span>
      </div>
    </div>
    <button v-if="currentCharacter.inventory.equipment.legs"
            @click="unequipItem('legs'); slotExpanded = ''"
            style="margin-top: 6px; padding: 6px 12px; background: #f44336; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 13px;">
      卸下
    </button>
  </div>
</div>

<!-- 主手栏 -->
<div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
  <label style="font-weight: bold;">主手栏</label>
  <div @click="mainHandExpanded = !mainHandExpanded"
       style="margin-top: 8px; padding: 10px; background: #fafafa; border: 1px dashed #ccc; border-radius: 6px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
    <span v-if="getItemById(currentCharacter.inventory.equipment.mainHand)" style="display: flex; align-items: center; gap: 6px;">
      <img
        v-if="getItemIcon(getItemById(currentCharacter.inventory.equipment.mainHand))"
        :src="getItemIcon(getItemById(currentCharacter.inventory.equipment.mainHand))"
        style="width: 32px; height: 32px; image-rendering: pixelated;"
      />
      <span>
        <strong>{{ getItemById(currentCharacter.inventory.equipment.mainHand).name }}</strong>
        <span style="color: #888; margin-left: 6px;">{{ getItemById(currentCharacter.inventory.equipment.mainHand).category }}</span>
      </span>
    </span>
    <span v-else style="color: #999;">无物品</span>
    <span style="color: #888; font-size: 13px;">{{ mainHandExpanded ? '收起' : '选择' }}</span>
  </div>
  <div v-if="mainHandExpanded" style="margin-top: 8px; border: 1px solid #e0e0e0; border-radius: 6px; padding: 10px; background: #fff;">
    <div v-if="getOwnedCategories('mainHand').length === 0" style="color: #bbb; font-size: 13px;">背包中没有可装备的主手物品</div>
    <div v-for="cat in getOwnedCategories('mainHand')" :key="cat" style="margin-bottom: 8px;">
      <div style="font-size: 13px; color: #666; margin-bottom: 4px;">{{ cat }}</div>
      <div v-for="entry in getItemsByCategory('mainHand', cat)" :key="entry.item_id"
           @click="equipItem('mainHand', entry.item_id); mainHandExpanded = false"
           style="padding: 6px 8px; margin-bottom: 4px; background: #f5f5f5; border-radius: 4px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
        <span style="display: flex; align-items: center; gap: 6px;">
          <img v-if="getItemIcon(entry.item)" :src="getItemIcon(entry.item)" style="width: 24px; height: 24px; image-rendering: pixelated;" />
          {{ entry.item.name }}
        </span>
        <span style="color: #888;">×{{ entry.quantity }}</span>
      </div>
    </div>
    <button v-if="currentCharacter.inventory.equipment.mainHand"
            @click="unequipItem('mainHand'); mainHandExpanded = false"
            style="margin-top: 6px; padding: 6px 12px; background: #f44336; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 13px;">
      卸下
    </button>
  </div>
</div>

<!-- 副手栏 -->
<div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
  <label style="font-weight: bold;">副手栏</label>
  <div @click="offHandExpanded = !offHandExpanded"
       style="margin-top: 8px; padding: 10px; background: #fafafa; border: 1px dashed #ccc; border-radius: 6px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
    <span v-if="getItemById(currentCharacter.inventory.equipment.offHand)" style="display: flex; align-items: center; gap: 6px;">
      <img
        v-if="getItemIcon(getItemById(currentCharacter.inventory.equipment.offHand))"
        :src="getItemIcon(getItemById(currentCharacter.inventory.equipment.offHand))"
        style="width: 32px; height: 32px; image-rendering: pixelated;"
      />
      <span>
        <strong>{{ getItemById(currentCharacter.inventory.equipment.offHand).name }}</strong>
        <span style="color: #888; margin-left: 6px;">{{ getItemById(currentCharacter.inventory.equipment.offHand).category }}</span>
      </span>
    </span>
    <span v-else style="color: #999;">无物品</span>
    <span style="color: #888; font-size: 13px;">{{ offHandExpanded ? '收起' : '选择' }}</span>
  </div>
  <div v-if="offHandExpanded" style="margin-top: 8px; border: 1px solid #e0e0e0; border-radius: 6px; padding: 10px; background: #fff;">
    <div v-if="getOwnedCategories('offHand').length === 0" style="color: #bbb; font-size: 13px;">背包中没有可装备的副手物品</div>
    <div v-for="cat in getOwnedCategories('offHand')" :key="cat" style="margin-bottom: 8px;">
      <div style="font-size: 13px; color: #666; margin-bottom: 4px;">{{ cat }}</div>
      <div v-for="entry in getItemsByCategory('offHand', cat)" :key="entry.item_id"
           @click="equipItem('offHand', entry.item_id); offHandExpanded = false"
           style="padding: 6px 8px; margin-bottom: 4px; background: #f5f5f5; border-radius: 4px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
        <span style="display: flex; align-items: center; gap: 6px;">
          <img v-if="getItemIcon(entry.item)" :src="getItemIcon(entry.item)" style="width: 24px; height: 24px; image-rendering: pixelated;" />
          {{ entry.item.name }}
        </span>
        <span style="color: #888;">×{{ entry.quantity }}</span>
      </div>
    </div>
    <button v-if="currentCharacter.inventory.equipment.offHand"
            @click="unequipItem('offHand'); offHandExpanded = false"
            style="margin-top: 6px; padding: 6px 12px; background: #f44336; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 13px;">
      卸下
    </button>
  </div>
</div>

<!-- 护身符 -->
<div v-if="currentClassInfo && currentClassInfo.canUseBuff" style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
  <label style="font-weight: bold;">护身符</label>
  <div @click="slotExpanded = slotExpanded === 'amulet' ? '' : 'amulet'"
       style="margin-top: 8px; padding: 10px; background: #fafafa; border: 1px dashed #ccc; border-radius: 6px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
    <span v-if="getItemById(currentCharacter.inventory.equipment.amulet)" style="display: flex; align-items: center; gap: 6px;">
      <img
        v-if="getItemIcon(getItemById(currentCharacter.inventory.equipment.amulet))"
        :src="getItemIcon(getItemById(currentCharacter.inventory.equipment.amulet))"
        style="width: 32px; height: 32px; image-rendering: pixelated;"
      />
      <strong>{{ getItemById(currentCharacter.inventory.equipment.amulet).name }}</strong>
    </span>
    <span v-else style="color: #999;">无物品</span>
    <span style="color: #888; font-size: 13px;">{{ slotExpanded === 'amulet' ? '收起' : '选择' }}</span>
  </div>
  <div v-if="slotExpanded === 'amulet'" style="margin-top: 8px; border: 1px solid #e0e0e0; border-radius: 6px; padding: 10px; background: #fff;">
    <div v-if="getOwnedCategories('amulet').length === 0" style="color: #bbb; font-size: 13px;">背包中没有可装备的护身符</div>
    <div v-for="cat in getOwnedCategories('amulet')" :key="cat" style="margin-bottom: 8px;">
      <div style="font-size: 13px; color: #666; margin-bottom: 4px;">{{ cat }}</div>
      <div v-for="entry in getItemsByCategory('amulet', cat)" :key="entry.item_id"
           @click="equipItem('amulet', entry.item_id); slotExpanded = ''"
           style="padding: 6px 8px; margin-bottom: 4px; background: #f5f5f5; border-radius: 4px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
        <span style="display: flex; align-items: center; gap: 6px;">
          <img v-if="getItemIcon(entry.item)" :src="getItemIcon(entry.item)" style="width: 24px; height: 24px; image-rendering: pixelated;" />
          {{ entry.item.name }}
        </span>
        <span style="color: #888;">×{{ entry.quantity }}</span>
      </div>
    </div>
    <button v-if="currentCharacter.inventory.equipment.amulet"
            @click="unequipItem('amulet'); slotExpanded = ''"
            style="margin-top: 6px; padding: 6px 12px; background: #f44336; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 13px;">
      卸下
    </button>
  </div>
</div>

<!-- 背包（装备位） -->
<div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
  <label style="font-weight: bold;">背包</label>
  <div @click="slotExpanded = slotExpanded === 'backpack' ? '' : 'backpack'"
       style="margin-top: 8px; padding: 10px; background: #fafafa; border: 1px dashed #ccc; border-radius: 6px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
    <span v-if="getItemById(currentCharacter.inventory.equipment.backpack)" style="display: flex; align-items: center; gap: 6px;">
      <img
        v-if="getItemIcon(getItemById(currentCharacter.inventory.equipment.backpack))"
        :src="getItemIcon(getItemById(currentCharacter.inventory.equipment.backpack))"
        style="width: 32px; height: 32px; image-rendering: pixelated;"
      />
      <strong>{{ getItemById(currentCharacter.inventory.equipment.backpack).name }}</strong>
    </span>
    <span v-else style="color: #999;">无物品</span>
    <span style="color: #888; font-size: 13px;">{{ slotExpanded === 'backpack' ? '收起' : '选择' }}</span>
  </div>
  <div v-if="slotExpanded === 'backpack'" style="margin-top: 8px; border: 1px solid #e0e0e0; border-radius: 6px; padding: 10px; background: #fff;">
    <div v-if="getOwnedCategories('backpack').length === 0" style="color: #bbb; font-size: 13px;">背包中没有可装备的背包</div>
    <div v-for="cat in getOwnedCategories('backpack')" :key="cat" style="margin-bottom: 8px;">
      <div style="font-size: 13px; color: #666; margin-bottom: 4px;">{{ cat }}</div>
      <div v-for="entry in getItemsByCategory('backpack', cat)" :key="entry.item_id"
           @click="equipItem('backpack', entry.item_id); slotExpanded = ''"
           style="padding: 6px 8px; margin-bottom: 4px; background: #f5f5f5; border-radius: 4px; cursor: pointer; display: flex; justify-content: space-between; align-items: center;">
        <span style="display: flex; align-items: center; gap: 6px;">
          <img v-if="getItemIcon(entry.item)" :src="getItemIcon(entry.item)" style="width: 24px; height: 24px; image-rendering: pixelated;" />
          {{ entry.item.name }}
        </span>
        <span style="color: #888;">×{{ entry.quantity }}</span>
      </div>
    </div>
    <button v-if="currentCharacter.inventory.equipment.backpack"
            @click="unequipItem('backpack'); slotExpanded = ''"
            style="margin-top: 6px; padding: 6px 12px; background: #f44336; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 13px;">
      卸下
    </button>
  </div>
</div>
</div>

<!-- 物品栏（背包内） -->
<h2>物品栏（背包内）</h2>
<div style="border: 1px solid #ddd; border-radius: 8px; padding: 15px; margin: 15px 0;">
  <div v-if="!currentCharacter.inventory?.items || currentCharacter.inventory.items.length === 0"
       style="color: #888; margin-bottom: 8px;">
    目前没有物品（需要 GM 发放后才会出现）
  </div>

  <div v-for="(entry, index) in currentCharacter.inventory.items" :key="entry.id || index"
       style="border-bottom: 1px solid #eee; padding: 10px 0;">
    <div @click="backpackOpenId = backpackOpenId === (entry.item_id + '-' + index) ? null : (entry.item_id + '-' + index)"
         style="display: flex; align-items: center; gap: 10px; cursor: pointer;">
      <img
        v-if="getItemById(entry.item_id)?.image_url || getItemIcon(getItemById(entry.item_id))"
        :src="getItemById(entry.item_id)?.image_url || getItemIcon(getItemById(entry.item_id))"
        style="width: 48px; height: 48px; object-fit: contain; image-rendering: pixelated; flex-shrink: 0; background: #fafafa; border-radius: 4px;"
      />
      <div style="flex: 1;">
        <strong>{{ getItemById(entry.item_id)?.name || '未知物品' }}</strong>
        <span style="color: #666; margin-left: 8px;">× {{ entry.quantity }}</span>
        <span v-if="getItemById(entry.item_id)" style="color: #999; margin-left: 8px; font-size: 13px;">
          （{{ getItemById(entry.item_id).category }} · {{ getItemById(entry.item_id).sub_type }}）
        </span>
        <div style="font-size: 12px; color: #aaa; margin-top: 2px;">
          占用 {{ ((getItemById(entry.item_id)?.slots) || 1) * (entry.quantity || 1) }} 格
        </div>
      </div>
      <span style="color: #888; font-size: 13px;">
        {{ backpackOpenId === (entry.item_id + '-' + index) ? '收起' : '详情' }}
      </span>
    </div>

    <div v-if="backpackOpenId === (entry.item_id + '-' + index)"
         style="margin-top: 10px; padding: 12px; background: #fafafa; border-radius: 8px; display: flex; gap: 16px; flex-wrap: wrap;">
      <img
        v-if="getItemById(entry.item_id)?.image_url"
        :src="getItemById(entry.item_id).image_url"
        style="width: 160px; height: 160px; object-fit: contain; background: #fff; border-radius: 6px;"
      />
      <div style="flex: 1; min-width: 220px;">
        <div style="font-size: 14px; color: #555; margin-bottom: 8px; white-space: pre-wrap;">
          {{ getItemById(entry.item_id)?.description || '暂无简介' }}
        </div>
        <div style="font-size: 13px; color: #333; line-height: 1.7;">
          <div v-if="getItemById(entry.item_id)?.armor_type">防具种类：{{ getItemById(entry.item_id).armor_type }}</div>
          <div v-if="Number(getItemById(entry.item_id)?.strength)">力量 +{{ getItemById(entry.item_id).strength }}</div>
          <div v-if="Number(getItemById(entry.item_id)?.agility)">敏捷 +{{ getItemById(entry.item_id).agility }}</div>
          <div v-if="Number(getItemById(entry.item_id)?.intelligence)">智力 +{{ getItemById(entry.item_id).intelligence }}</div>
          <div v-if="Number(getItemById(entry.item_id)?.atk)">ATK +{{ getItemById(entry.item_id).atk }}</div>
          <div v-if="Number(getItemById(entry.item_id)?.hp)">HP +{{ getItemById(entry.item_id).hp }}</div>
          <div v-if="Number(getItemById(entry.item_id)?.def)">DEF +{{ getItemById(entry.item_id).def }}</div>
          <div v-if="Number(getItemById(entry.item_id)?.res)">RES +{{ getItemById(entry.item_id).res }}</div>
          <div v-if="Number(getItemById(entry.item_id)?.spd)">SPD +{{ getItemById(entry.item_id).spd }}</div>
          <div v-if="Number(getItemById(entry.item_id)?.satiety)">恢复饱食度 {{ getItemById(entry.item_id).satiety }}</div>
        </div>
        <div v-if="getItemById(entry.item_id)?.skills?.length" style="margin-top: 8px;">
          <div v-for="(sk, si) in getItemById(entry.item_id).skills" :key="si"
               style="font-size: 13px; margin-top: 4px;">
            <strong>{{ sk.name }}</strong>
            <span style="color: #666;">　{{ sk.desc }}</span>
          </div>
        </div>
        <button type="button" @click.stop="discardItem(index)"
                style="margin-top: 12px; padding: 6px 12px; background: #c62828; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 13px;">
          丢掉
        </button>
      </div>
    </div>
  </div>
</div>

<h2>备注 / 讯息栏</h2>
<textarea v-model="currentCharacter.notes" rows="4" style="width: 100%; padding: 8px; margin-top: 8px;" placeholder="临时状态、任务笔记等..."></textarea>
<h2>解体熔炉事件公告栏</h2>
<div style="border: 1px solid #ffcc80; border-radius: 8px; padding: 12px; margin: 10px 0 24px; background: #fff8e1;">
  <div v-if="announceMessages.length === 0" style="color: #888; font-size: 13px;">暂无公告</div>
  <div v-for="m in announceMessages" :key="m.id" style="padding: 8px 0; border-bottom: 1px solid #ffe0b2;">
    <div style="font-size: 12px; color: #ef6c00;">{{ m.author_name }} · {{ m.created_at?.slice(0, 16)?.replace('T', ' ') }}</div>
    <div style="white-space: pre-wrap; margin-top: 4px;">{{ m.body }}</div>
  </div>
  <div v-if="isGM" style="margin-top: 10px;">
    <textarea v-model="announceDraft" rows="3" style="width: 100%; padding: 8px;" placeholder="GM 发布公告..."></textarea>
    <button type="button" @click="sendRoomMessage('announce')"
            style="margin-top: 8px; padding: 8px 16px; background: #ef6c00; color: white; border: none; border-radius: 4px; cursor: pointer;">
      发布公告
    </button>
  </div>
  <div v-else style="margin-top: 8px; font-size: 12px; color: #888;">仅 GM 可发布，全员可见、实时同步。</div>
</div>

<h2>工作人员沟通群</h2>
<div style="border: 1px solid #bbdefb; border-radius: 8px; padding: 12px; margin: 10px 0 24px; background: #e3f2fd;">
  <div v-if="chatMessages.length === 0" style="color: #888; font-size: 13px;">还没有消息</div>
  <div v-for="m in chatMessages" :key="m.id" style="padding: 8px 0; border-bottom: 1px solid #bbdefb;">
    <div style="font-size: 12px; color: #1565c0;">{{ m.author_name }} · {{ m.created_at?.slice(0, 16)?.replace('T', ' ') }}</div>
    <div style="white-space: pre-wrap; margin-top: 4px;">{{ m.body }}</div>
  </div>
  <textarea v-model="chatDraft" rows="3" style="width: 100%; padding: 8px; margin-top: 10px;" placeholder="输入消息..."></textarea>
  <button type="button" @click="sendRoomMessage('chat')"
          style="margin-top: 8px; padding: 8px 16px; background: #1976d2; color: white; border: none; border-radius: 4px; cursor: pointer;">
    发送
  </button>
</div>
</div>


 <div v-else-if="page === 'magic'" style="max-width: 1000px; margin: 30px auto; font-family: sans-serif; padding: 0 20px;">
  <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;">
    <h1>魔术表</h1>
    <button @click="page = 'character'" style="padding: 8px 16px;">返回角色</button>
  </div>
  <p style="color: #666; margin-bottom: 25px; font-size: 14px;">
    分级说明：一、二册 = 日常　｜　三册 = 专业　｜　四、五册 = 军工　｜　六册 = 神域
  </p>

  <div v-for="(vol, key) in magicList" :key="key" style="margin-bottom: 40px;">
    <h2 style="border-bottom: 2px solid #9c27b0; padding-bottom: 8px; color: #7b1fa2;">
      {{ vol.title }}
    </h2>

    <div v-if="vol.spells && vol.spells.length > 0">
      <div v-for="(m, i) in vol.spells" :key="i"
           style="border: 1px solid #e1bee7; border-radius: 8px; padding: 14px; margin: 12px 0; background: #faf5ff; text-align: center;">
        <div style="font-weight: bold; font-size: 16px; color: #6a1b9a; margin-bottom: 6px;">{{ m.name }}</div>
        <div style="margin-bottom: 8px;">
          <span style="background: #ce93d8; color: #4a148c; padding: 3px 10px; border-radius: 12px; font-size: 12px;">
            {{ m.type }}
          </span>
        </div>
        <div style="font-size: 14px; line-height: 1.6; color: #333; margin-bottom: 12px;">{{ m.desc }}</div>
        <button type="button" @click="addMagicSpell(m, vol.title)"
                style="padding: 8px 18px; background: #9c27b0; color: white; border: none; border-radius: 4px; cursor: pointer;">
          添加
        </button>
      </div>
    </div>

    <div v-else style="padding: 20px; background: #f3e5f5; border-radius: 8px; color: #7b1fa2; text-align: center;">
      本册内容目前空缺，仅保留分类框架。
    </div>
  </div>
</div>

  <!-- Buff表页面 -->
<div v-else-if="page === 'buff'" style="max-width: 1100px; margin: 30px auto; font-family: sans-serif; padding: 0 20px;">
  <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;">
    <h1>Buff 表</h1>
    <button @click="backToCharacter" style="padding: 8px 16px;">返回角色</button>
  </div>

  <div style="display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 20px;">
    <!-- 红 · 伤害增益 -->
    <div>
      <h2 style="color: #f44336; border-bottom: 2px solid #f44336; padding-bottom: 8px;">红 · 伤害增益</h2>
      <div v-for="(b, i) in buffList.red" :key="'r'+i"
           style="border: 1px solid #ffcdd2; border-radius: 8px; padding: 12px; margin-bottom: 10px; background: #fff5f5;">
        <strong style="color: #c62828;">{{ b.name }}</strong>
        <div style="font-size: 13px; margin-top: 6px; line-height: 1.5;">{{ b.desc }}</div>
      </div>
    </div>

    <!-- 绿 · 恢复增益 -->
    <div>
      <h2 style="color: #4CAF50; border-bottom: 2px solid #4CAF50; padding-bottom: 8px;">绿 · 恢复增益</h2>
      <div v-for="(b, i) in buffList.green" :key="'g'+i"
           style="border: 1px solid #c8e6c9; border-radius: 8px; padding: 12px; margin-bottom: 10px; background: #f1f8e9;">
        <strong style="color: #2e7d32;">{{ b.name }}</strong>
        <div style="font-size: 13px; margin-top: 6px; line-height: 1.5;">{{ b.desc }}</div>
      </div>
    </div>

    <!-- 金 · 其他 -->
    <div>
      <h2 style="color: #ff9800; border-bottom: 2px solid #ff9800; padding-bottom: 8px;">金 · 其他</h2>
      <div v-for="(b, i) in buffList.gold" :key="'y'+i"
           style="border: 1px solid #ffe0b2; border-radius: 8px; padding: 12px; margin-bottom: 10px; background: #fff8e1;">
        <strong style="color: #ef6c00;">{{ b.name }}</strong>
        <div style="font-size: 13px; margin-top: 6px; line-height: 1.5;">{{ b.desc }}</div>
      </div>
    </div>
  </div>
</div>
<div v-else-if="page === 'passive'" style="max-width: 900px; margin: 30px auto; padding: 0 20px; font-family: sans-serif;">
  <div style="display: flex; justify-content: space-between; align-items: center;">
    <h1>被动技能</h1>
    <button @click="page = 'character'" style="padding: 8px 16px;">返回角色</button>
  </div>
  <div style="display: flex; gap: 8px; margin: 16px 0;">
    <button v-for="t in ['I','II','III']" :key="t" @click="skillPageTab = t"
            :style="{ padding: '8px 14px', cursor: 'pointer', background: skillPageTab === t ? '#5c6bc0' : '#eee', color: skillPageTab === t ? '#fff' : '#333', border: 'none', borderRadius: '4px' }">
      被动页 {{ t }}
    </button>
  </div>
  <div v-for="skill in passiveSkillList[skillPageTab]" :key="skill.name"
     style="border: 1px solid #e0e0e0; border-radius: 8px; padding: 14px; margin-bottom: 10px; text-align: center;">
  <div style="font-weight: bold; font-size: 16px; margin-bottom: 8px;">{{ skill.name }}</div>
  <div style="font-size: 14px; color: #555; line-height: 1.6; margin-bottom: 12px;">{{ skill.desc }}</div>
  <button @click="addAbilitySkill(skillPageTab, skill)"
          style="padding: 8px 18px; background: #4CAF50; color: white; border: none; border-radius: 4px; cursor: pointer;">
    添加
  </button>
</div>
</div>

<div v-else-if="page === 'hero'" style="max-width: 900px; margin: 30px auto; padding: 0 20px; font-family: sans-serif;">
  <div style="display: flex; justify-content: space-between; align-items: center;">
    <h1>英雄技能</h1>
    <button @click="page = 'character'" style="padding: 8px 16px;">返回角色</button>
  </div>
  <div v-for="skill in heroSkillList" :key="skill.name"
     style="border: 1px solid #ffe0b2; border-radius: 8px; padding: 14px; margin: 12px 0; background: #fff8e1; text-align: center;">
  <div style="font-weight: bold; font-size: 16px; margin-bottom: 8px;">{{ skill.name }}</div>
  <div style="font-size: 14px; color: #555; line-height: 1.6; margin-bottom: 12px;">{{ skill.desc }}</div>
  <button @click="addAbilitySkill('hero', skill)"
          style="padding: 8px 18px; background: #ff9800; color: white; border: none; border-radius: 4px; cursor: pointer;">
    添加
  </button>
</div>
</div>
<!-- 员工手册 -->
<div v-else-if="page === 'handbook'" style="max-width: 900px; margin: 30px auto; font-family: sans-serif; padding: 0 20px 40px;">
  <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;">
    <h1>解体熔炉 · 员工手册</h1>
    <button @click="backFromHandbook" style="padding: 8px 16px;">
      {{ handbookClass ? '返回手册' : '返回' }}
    </button>
  </div>

  <!-- 职业介绍 -->
  <div v-if="handbookClass">
    <h2>{{ handbookClass }}</h2>
    <div v-if="classTemplates[handbookClass]" style="padding: 16px; background: #fafafa; border-radius: 8px; border: 1px solid #eee;">
      <p style="color: #666;">{{ classTemplates[handbookClass].description || '暂无简介' }}</p>
      <p><strong>主属性：</strong>{{ classTemplates[handbookClass].mainAttribute || '—' }}</p>
      <p><strong>可装备：</strong>{{ classTemplates[handbookClass].equipment || '—' }}</p>

      <div v-if="classTemplates[handbookClass].levelRewards && classTemplates[handbookClass].levelRewards.length" style="margin-top: 16px;">
        <strong>等级奖励</strong>
        <div v-for="(r, i) in classTemplates[handbookClass].levelRewards" :key="i"
             style="margin: 10px 0; padding: 10px; background: white; border-radius: 6px; white-space: pre-line;">
          <strong>LVL {{ r.level }}</strong>
          <div>{{ r.content }}</div>
        </div>
      </div>

      <div v-if="classTemplates[handbookClass].subclasses && classTemplates[handbookClass].subclasses.length" style="margin-top: 16px;">
        <strong>子职</strong>
        <div v-for="(sub, i) in classTemplates[handbookClass].subclasses" :key="i"
             style="margin: 8px 0; padding: 10px; background: white; border-left: 4px solid #9c27b0; white-space: pre-line;">
          <strong>{{ sub.name }}</strong>
          <div>{{ sub.desc }}</div>
        </div>
      </div>
    </div>
    <div v-else style="padding: 20px; color: #888;">
      「{{ handbookClass }}」职业详细数据待补。
    </div>
  </div>

  <!-- 目录正文 -->
  <div v-else>
    <h2 style="border-bottom: 2px solid #5c6bc0; padding-bottom: 8px;">Chapter I：工作概论</h2>
    <p>本公司旨在解决影响人类日常生活的有害神秘。工作内容包括：维护公司利益、解决神秘事件、协助警方处理神秘相关案件。</p>
    <p>发现可疑人员请立即联系当地警方。工作准则：<strong>先观察，再行动。</strong></p>

    <h2 style="border-bottom: 2px solid #5c6bc0; padding-bottom: 8px; margin-top: 32px;">Chapter II：工作流程</h2>
    <p style="color: #666; font-size: 14px;">点击职业名可查看详细介绍</p>

    <h3>近战型</h3>
    <p>
      <button type="button" @click="openClassInHandbook('剑士')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">剑士（完）</button>
      — 长剑，可使用 buff。子职：决斗者、重剑士
    </p>
    <p>
      <button type="button" @click="openClassInHandbook('长枪兵')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">长枪兵（补）</button>
      — AoE，可使用 buff
    </p>
    <p>
      <button type="button" @click="openClassInHandbook('格斗家')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">格斗家（补）</button>
      — 近身拳法，可使用 buff
    </p>

    <h3>射手型</h3>
    <p>
      <button type="button" @click="openClassInHandbook('射手')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">射手（施工）</button>
      ·
      <button type="button" @click="openClassInHandbook('狙击手')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">狙击手（补）</button>
      ·
      <button type="button" @click="openClassInHandbook('指挥官')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">指挥官（补）</button>
    </p>

    <h3>特化型</h3>
    <p>
      <button type="button" @click="openClassInHandbook('斥候')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">斥候（补）</button>
      ·
      <button type="button" @click="openClassInHandbook('处刑者')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">处刑者（补）</button>
      ·
      <button type="button" @click="openClassInHandbook('赏金猎人')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">赏金猎人（完）</button>
    </p>

    <h3>魔术使</h3>
    <p>
      <button type="button" @click="openClassInHandbook('魔术师')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">魔术师（完）</button>
      ·
      <button type="button" @click="openClassInHandbook('术士')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">术士（补）</button>
      ·
      <button type="button" @click="openClassInHandbook('萨满')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">萨满（补）</button>
      ·
      <button type="button" @click="openClassInHandbook('Code Wizard')" style="background: none; border: none; color: #5c6bc0; cursor: pointer; text-decoration: underline; padding: 0; font-size: 16px;">Code Wizard（补）</button>
    </p>

    <h3>属性与检定（摘要）</h3>
    <p>力量 1点 = 2 HP　｜　智力 1点 = 1 魔力 + 0.5 RES　｜　敏捷 1点 = 0.1 SPD</p>
    <p>ATK = 默认ATK + 主属性/4（向下取整）+ 武器加值</p>
    <p>检定技能默认 4；主属性相关技能可加 主属性÷10。使用 1d20 判定。</p>

    <h3>装备</h3>
    <p>头盔 / 胸甲 / 护腿 · 主手 / 副手 · 背包 · 护身符（可使用 buff 的职业）</p>
    <p>一回合可：移动、使用一次 buff、使用一次道具、食用一次食物，最后发动攻击。</p>
  </div>
</div>
</template>