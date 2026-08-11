<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import { supabase } from './supabase.js'

// 页面状态：home / room / character / magic / buff
const page = ref('home')

// 房间相关
const roomName = ref('')
const gmPassword = ref('')
const joinCode = ref('')
const joinGmPassword = ref('')
const currentSession = ref(null)
const isGM = ref(false)
const message = ref('')

// 角色相关
const characters = ref([])
const newCharacterName = ref('')
const newPlayerName = ref('')
const currentCharacter = ref(null)
const saving = ref(false)

// 物品栏相关
const newItemName = ref('')
const newItemQuantity = ref(1)

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
    equipment: '魔导书、魔杖、轻甲。需同时装备魔导书和魔杖。',
    levelRewards: [
      { level: 1, content: '无法使用buff。特异点锁明：从下列特异点中选择一项（7的战争 / 龙胭 / 厌花之蛇）。' }
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

// 简单魔术表示例
const magicList = [
  { name: '道化机巧，其一', type: '控制', cost: 7, desc: '召唤一个具备30点HP的人型告示牌，嘲讽野兽目标。' },
  { name: '抽丝剥茧', type: '控制&防御', cost: 7, desc: '形成石墙，HP50，可束缚其中单位。' },
  { name: '点火术', type: '伤害', cost: 5, desc: '对最远1格敌人造成14点火属性魔法伤害。' },
  { name: '火焰箭', type: '伤害', cost: 10, desc: '对最远4格敌人造成28点火属性魔法伤害。' },
  { name: '修复术', type: '回复', cost: 17, desc: '瞬间恢复自己或最远2格友军16点HP。' },
  { name: '抽丝剥茧-锐', type: '伤害', cost: '高', desc: '造成80点魔法伤害，高roll可秒杀活体目标。' }
]

// 简单Buff表示例
const buffList = [
  { name: '熔炉的加护', type: '红', cost: 10, desc: '为近战武器施加熔炉的加护，将下两次攻击的伤害类型变为火属性魔法伤害。' },
  { name: '圣母之光', type: '绿', cost: 5, desc: '回复自身15点HP。' },
  { name: '澄澈之心', type: '金', cost: 10, desc: '解除自身30层精神异常。' },
  { name: '暴烈的果实', type: '红', cost: 10, desc: '本场战斗中+20ATK，受到伤害后失效。' },
  { name: '不动之墙', type: '绿', cost: 13, desc: '获得三次DEF+10效果。' },
  { name: '王道宣言', type: '金', cost: 10, desc: '本场战斗中免疫精神异常“堕落”。' }
]

const currentClassInfo = computed(() => {
  if (!currentCharacter.value || !currentCharacter.value.class_name) return null
  return classTemplates[currentCharacter.value.class_name] || null
})

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
      currentSession.value = data.session
      isGM.value = data.isGM
      page.value = 'room'
      loadCharacters()
      startRealtime()
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
    page.value = 'room'
    saveSessionToLocal()
    loadCharacters()
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
        if (currentCharacter.value && payload.new && payload.new.id === currentCharacter.value.id) {
          const updated = { ...payload.new }
          if (!Array.isArray(updated.inventory)) updated.inventory = []
          currentCharacter.value = updated
        }
      }
    )
    .subscribe()
}

async function createCharacter() {
  if (!newCharacterName.value) {
    alert('请输入角色名')
    return
  }
  const { error } = await supabase
    .from('characters')
    .insert({
      session_id: currentSession.value.id,
      name: newCharacterName.value,
      player_name: newPlayerName.value || '未命名玩家',
      class_name: '未选择',
      level: 1,
      inventory: []
    })
  if (error) {
    alert('创建角色失败：' + error.message)
  } else {
    newCharacterName.value = ''
    newPlayerName.value = ''
    loadCharacters()
  }
}

function enterCharacter(char) {
  const charCopy = { ...char }
  if (!Array.isArray(charCopy.inventory)) {
    charCopy.inventory = []
  }
  currentCharacter.value = charCopy
  page.value = 'character'
}

function addItem() {
  if (!newItemName.value) {
    alert('请输入物品名称')
    return
  }
  if (!currentCharacter.value.inventory) {
    currentCharacter.value.inventory = []
  }
  currentCharacter.value.inventory.push({
    id: Date.now(),
    name: newItemName.value,
    quantity: newItemQuantity.value || 1
  })
  newItemName.value = ''
  newItemQuantity.value = 1
}

function removeItem(index) {
  currentCharacter.value.inventory.splice(index, 1)
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
      strength: currentCharacter.value.strength,
      intelligence: currentCharacter.value.intelligence,
      agility: currentCharacter.value.agility,
      hp_current: currentCharacter.value.hp_current,
      hp_max: currentCharacter.value.hp_max,
      atk: currentCharacter.value.atk,
      def: currentCharacter.value.def,
      res: currentCharacter.value.res,
      spd: currentCharacter.value.spd,
      move_range: currentCharacter.value.move_range,
      pp: currentCharacter.value.pp,
      notes: currentCharacter.value.notes,
      inventory: currentCharacter.value.inventory
    })
    .eq('id', currentCharacter.value.id)
  saving.value = false
  if (error) {
    alert('保存失败：' + error.message)
  } else {
    alert('保存成功！')
  }
}

function backToRoom() {
  page.value = 'room'
  currentCharacter.value = null
  loadCharacters()
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

function openBuffTable() {
  page.value = 'buff'
}

function backToCharacter() {
  page.value = 'character'
}

onMounted(() => {
  restoreSessionFromLocal()
})

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
    <h2>角色列表</h2>
    <div v-if="characters.length === 0" style="color: #888; margin: 20px 0;">目前还没有角色，创建一个吧。</div>
    <div v-for="char in characters" :key="char.id" style="border: 1px solid #ddd; padding: 15px; margin-bottom: 10px; border-radius: 8px; display: flex; justify-content: space-between; align-items: center;">
      <div>
        <strong style="font-size: 18px;">{{ char.name }}</strong>
        <span style="color: #666; margin-left: 10px;">（{{ char.player_name }}）</span>
        <div style="font-size: 14px; color: #888; margin-top: 4px;">职业：{{ char.class_name || '未选择' }}　|　等级：{{ char.level || 1 }}</div>
      </div>
      <button @click="enterCharacter(char)" style="padding: 6px 12px; background: #2196F3; color: white; border: none; border-radius: 4px; cursor: pointer;">扮演角色</button>
    </div>
    <hr style="margin: 30px 0;" />
    <h3>创建新角色</h3>
    <div style="display: flex; gap: 10px; margin-top: 10px; flex-wrap: wrap;">
      <input v-model="newCharacterName" placeholder="角色名" style="padding: 8px; flex: 1; min-width: 150px;" />
      <input v-model="newPlayerName" placeholder="玩家昵称（可选）" style="padding: 8px; flex: 1; min-width: 150px;" />
      <button @click="createCharacter" style="padding: 8px 20px; background: #4CAF50; color: white; border: none; cursor: pointer;">创建角色</button>
    </div>
  </div>

  <!-- 角色详情页 -->
  <div v-else-if="page === 'character'" style="max-width: 950px; margin: 30px auto; font-family: sans-serif; padding: 0 20px;">
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;">
      <h1>扮演角色</h1>
      <div>
        <button @click="saveCharacter" :disabled="saving" style="padding: 8px 20px; background: #4CAF50; color: white; border: none; margin-right: 10px; cursor: pointer;">
          {{ saving ? '保存中...' : '保存' }}
        </button>
        <button @click="backToRoom" style="padding: 8px 16px;">返回列表</button>
      </div>
    </div>

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
        <input type="number" v-model.number="currentCharacter.level" style="width: 100%; padding: 8px;" />
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

      <!-- 已完成职业显示详细信息 -->
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

      <!-- 待补职业只显示提示 -->
      <div v-else style="margin-top: 15px; padding: 15px; background: #fff3e0; border-radius: 6px; color: #e65100;">
        该职业详细数据尚未补全，敬请期待。
      </div>

      <!-- 子职显示（所有职业通用） -->
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
        <button v-if="currentCharacter.class_name === '魔术师'" @click="openMagicTable"
                style="padding: 8px 16px; background: #9c27b0; color: white; border: none; border-radius: 4px; cursor: pointer;">
          查看魔术表
        </button>
      </div>
    </div>

    <!-- 核心属性 -->
    <h2>核心属性</h2>
    <div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(140px, 1fr)); gap: 15px; margin: 15px 0 30px;">
      <div><label>力量</label><br /><input type="number" v-model.number="currentCharacter.strength" style="width: 100%; padding: 8px;" /></div>
      <div><label>智力</label><br /><input type="number" v-model.number="currentCharacter.intelligence" style="width: 100%; padding: 8px;" /></div>
      <div><label>敏捷</label><br /><input type="number" v-model.number="currentCharacter.agility" style="width: 100%; padding: 8px;" /></div>
      <div><label>HP 当前</label><br /><input type="number" v-model.number="currentCharacter.hp_current" style="width: 100%; padding: 8px;" /></div>
      <div><label>HP 最大</label><br /><input type="number" v-model.number="currentCharacter.hp_max" style="width: 100%; padding: 8px;" /></div>
      <div><label>ATK</label><br /><input type="number" v-model.number="currentCharacter.atk" style="width: 100%; padding: 8px;" /></div>
      <div><label>DEF</label><br /><input type="number" v-model.number="currentCharacter.def" style="width: 100%; padding: 8px;" /></div>
      <div><label>RES</label><br /><input type="number" v-model.number="currentCharacter.res" style="width: 100%; padding: 8px;" /></div>
      <div><label>SPD</label><br /><input type="number" v-model.number="currentCharacter.spd" style="width: 100%; padding: 8px;" /></div>
      <div><label>移动格</label><br /><input type="number" v-model.number="currentCharacter.move_range" style="width: 100%; padding: 8px;" /></div>
      <div><label>PP</label><br /><input type="number" v-model.number="currentCharacter.pp" style="width: 100%; padding: 8px;" /></div>
    </div>

    <!-- 物品栏 -->
    <h2>物品栏</h2>
    <div style="border: 1px solid #ddd; border-radius: 8px; padding: 15px; margin: 15px 0;">
      <div v-if="!currentCharacter.inventory || currentCharacter.inventory.length === 0" style="color: #888; margin-bottom: 15px;">目前没有物品</div>
      <div v-for="(item, index) in currentCharacter.inventory" :key="item.id" style="display: flex; justify-content: space-between; align-items: center; padding: 8px 0; border-bottom: 1px solid #eee;">
        <div><strong>{{ item.name }}</strong><span style="color: #666; margin-left: 10px;">× {{ item.quantity }}</span></div>
        <button @click="removeItem(index)" style="padding: 4px 10px; background: #f44336; color: white; border: none; border-radius: 4px; cursor: pointer;">删除</button>
      </div>
      <div style="display: flex; gap: 10px; margin-top: 15px; flex-wrap: wrap;">
        <input v-model="newItemName" placeholder="物品名称" style="padding: 8px; flex: 2; min-width: 150px;" />
        <input type="number" v-model.number="newItemQuantity" placeholder="数量" style="padding: 8px; width: 80px;" />
        <button @click="addItem" style="padding: 8px 16px; background: #2196F3; color: white; border: none; cursor: pointer;">添加物品</button>
      </div>
    </div>

    <h2>备注 / 讯息栏</h2>
    <textarea v-model="currentCharacter.notes" rows="4" style="width: 100%; padding: 8px; margin-top: 8px;" placeholder="临时状态、任务笔记等..."></textarea>
  </div>

  <!-- 魔术表页面 -->
  <div v-else-if="page === 'magic'" style="max-width: 900px; margin: 30px auto; font-family: sans-serif; padding: 0 20px;">
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;">
      <h1>魔术表</h1>
      <button @click="backToCharacter" style="padding: 8px 16px;">返回角色</button>
    </div>
    <p style="color: #666; margin-bottom: 20px;">目前仅展示部分常用魔术，完整表后续补充。</p>
    <div v-for="(m, i) in magicList" :key="i" style="border: 1px solid #ddd; border-radius: 8px; padding: 15px; margin-bottom: 12px;">
      <div style="display: flex; justify-content: space-between;">
        <strong style="font-size: 16px;">{{ m.name }}</strong>
        <span style="color: #9c27b0;">消耗：{{ m.cost }}</span>
      </div>
      <div style="font-size: 13px; color: #666; margin: 4px 0;">类型：{{ m.type }}</div>
      <div style="font-size: 14px;">{{ m.desc }}</div>
    </div>
  </div>

  <!-- Buff表页面 -->
  <div v-else-if="page === 'buff'" style="max-width: 900px; margin: 30px auto; font-family: sans-serif; padding: 0 20px;">
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;">
      <h1>Buff 表</h1>
      <button @click="backToCharacter" style="padding: 8px 16px;">返回角色</button>
    </div>
    <p style="color: #666; margin-bottom: 20px;">目前仅展示部分常用Buff，完整表后续补充。</p>
    <div v-for="(b, i) in buffList" :key="i" style="border: 1px solid #ddd; border-radius: 8px; padding: 15px; margin-bottom: 12px;">
      <div style="display: flex; justify-content: space-between;">
        <strong style="font-size: 16px;">{{ b.name }}</strong>
        <span :style="{ color: b.type === '红' ? '#f44336' : b.type === '绿' ? '#4CAF50' : '#ff9800' }">{{ b.type }} · 消耗 {{ b.cost }}</span>
      </div>
      <div style="font-size: 14px; margin-top: 6px;">{{ b.desc }}</div>
    </div>
  </div>
</template>