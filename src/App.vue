<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import { supabase } from './supabase.js'

// 页面状态：home / room / character / magic / buff
const page = ref('home')

// 检定相关
const skillsExpanded = ref(false)

// 所有物品库
const itemCatalog = ref([])          
const loadingItems = ref(false)

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

const testItemId = ref('')
const testItemQty = ref(1)

function addTestItem() {
  if (!testItemId.value) {
    alert('请先选择物品')
    return
  }
  if (!currentCharacter.value.inventory.items) {
    currentCharacter.value.inventory.items = []
  }
  // 如果已经有这个物品，就增加数量
  const existing = currentCharacter.value.inventory.items.find(i => i.item_id === testItemId.value)
  if (existing) {
    existing.quantity += (testItemQty.value || 1)
  } else {
    currentCharacter.value.inventory.items.push({
      id: Date.now(),
      item_id: testItemId.value,
      quantity: testItemQty.value || 1
    })
  }
  testItemId.value = ''
  testItemQty.value = 1
}

const mainHandExpanded = ref('')   // 当前展开的主手大类
const offHandExpanded = ref('')    // 当前展开的副手大类

// 主手可用的大类
const mainHandCategories = ['刀剑', '长枪', '枪械', '斧子', '魔导用具']

// 副手可用的大类
const offHandCategories = ['刀剑', '枪械', '斧子', '魔导用具', '盾牌']

// 获取背包中属于某个大类且可用于指定栏位的物品
function getItemsByCategory(slot, category) {
  if (!currentCharacter.value?.inventory?.items) return []
  return currentCharacter.value.inventory.items
    .map(entry => {
      const item = getItemById(entry.item_id)
      return item ? { ...entry, item } : null
    })
    .filter(x => x && x.item.category === category && (
      x.item.slot === slot ||
      (slot === 'offHand' && ['刀剑', '枪械', '斧子', '魔导用具', '盾牌'].includes(x.item.category))
    ))
}

function toggleMainCategory(cat) {
  mainHandExpanded.value = mainHandExpanded.value === cat ? '' : cat
}

function toggleOffCategory(cat) {
  offHandExpanded.value = offHandExpanded.value === cat ? '' : cat
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

  // 处理 inventory 结构
  if (!updated.inventory || Array.isArray(updated.inventory)) {
    const oldItems = Array.isArray(updated.inventory) ? updated.inventory : []
    updated.inventory = {
      items: oldItems,
      equipment: {
        helmet: '', chest: '', legs: '',
        mainHand: '', offHand: '', amulet: '', backpack: ''
      }
    }
  } else {
    if (!updated.inventory.items) updated.inventory.items = []
    if (!updated.inventory.equipment) {
      updated.inventory.equipment = {
        helmet: '', chest: '', legs: '',
        mainHand: '', offHand: '', amulet: '', backpack: ''
      }
    }
  }

  // 处理 skills 结构
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

function enterCharacter(char) {
  const charCopy = { ...char }

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

  currentCharacter.value = charCopy
  page.value = 'character'
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

// 装备物品到指定栏位
function equipItem(slot, itemId) {
  if (!currentCharacter.value?.inventory?.equipment) return
  currentCharacter.value.inventory.equipment[slot] = itemId || null
}

// 卸下装备
function unequipItem(slot) {
  if (!currentCharacter.value?.inventory?.equipment) return
  currentCharacter.value.inventory.equipment[slot] = null
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

<<!-- 检定技能栏（可折叠） -->
<div style="margin: 30px 0; border: 1px solid #ddd; border-radius: 8px; overflow: hidden;">
  <div @click="skillsExpanded = !skillsExpanded"
       style="padding: 14px 18px; background: #f5f5f5; cursor: pointer; display: flex; justify-content: space-between; align-items: center; user-select: none;">
    <strong style="font-size: 16px;">检定技能栏</strong>
    <span style="color: #666; font-size: 14px;">
      {{ skillsExpanded ? '▲ 收起' : '▼ 展开' }}
    </span>
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
    <div style="margin-top: 8px;">
      <select
        :value="currentCharacter.inventory.equipment.helmet || ''"
        @change="equipItem('helmet', $event.target.value || null)"
        style="width: 100%; padding: 8px;"
      >
        <option value="">— 未装备 —</option>
        <option
          v-for="entry in getEquippableItems('helmet')"
          :key="entry.item_id"
          :value="entry.item_id"
        >
          {{ entry.item.name }} ×{{ entry.quantity }}
        </option>
      </select>
      <div v-if="getItemById(currentCharacter.inventory.equipment.helmet)" style="margin-top: 6px; font-size: 13px; color: #555;">
        {{ getItemById(currentCharacter.inventory.equipment.helmet).name }}
      </div>
    </div>
  </div>

  <!-- 胸甲 -->
  <div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
    <label style="font-weight: bold;">胸甲</label>
    <div style="margin-top: 8px;">
      <select
        :value="currentCharacter.inventory.equipment.chest || ''"
        @change="equipItem('chest', $event.target.value || null)"
        style="width: 100%; padding: 8px;"
      >
        <option value="">— 未装备 —</option>
        <option
          v-for="entry in getEquippableItems('chest')"
          :key="entry.item_id"
          :value="entry.item_id"
        >
          {{ entry.item.name }} ×{{ entry.quantity }}
        </option>
      </select>
      <div v-if="getItemById(currentCharacter.inventory.equipment.chest)" style="margin-top: 6px; font-size: 13px; color: #555;">
        {{ getItemById(currentCharacter.inventory.equipment.chest).name }}
      </div>
    </div>
  </div>

  <!-- 护腿 -->
  <div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
    <label style="font-weight: bold;">护腿</label>
    <div style="margin-top: 8px;">
      <select
        :value="currentCharacter.inventory.equipment.legs || ''"
        @change="equipItem('legs', $event.target.value || null)"
        style="width: 100%; padding: 8px;"
      >
        <option value="">— 未装备 —</option>
        <option
          v-for="entry in getEquippableItems('legs')"
          :key="entry.item_id"
          :value="entry.item_id"
        >
          {{ entry.item.name }} ×{{ entry.quantity }}
        </option>
      </select>
      <div v-if="getItemById(currentCharacter.inventory.equipment.legs)" style="margin-top: 6px; font-size: 13px; color: #555;">
        {{ getItemById(currentCharacter.inventory.equipment.legs).name }}
      </div>
    </div>
  </div>

  <!-- 主手栏 -->
<div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
  <label style="font-weight: bold;">主手栏</label>
  
  <!-- 当前已装备 -->
  <div v-if="getItemById(currentCharacter.inventory.equipment.mainHand)" 
       style="margin: 8px 0; padding: 8px; background: #f3e5f5; border-radius: 6px; display: flex; justify-content: space-between; align-items: center;">
    <span>
      <strong>{{ getItemById(currentCharacter.inventory.equipment.mainHand).name }}</strong>
      <span style="color: #888; margin-left: 6px;">{{ getItemById(currentCharacter.inventory.equipment.mainHand).category }}</span>
    </span>
    <button @click="unequipItem('mainHand')" style="padding: 4px 10px; font-size: 12px; background: #f44336; color: white; border: none; border-radius: 4px; cursor: pointer;">
      卸下
    </button>
  </div>
  <div v-else style="margin: 8px 0; color: #999; font-size: 13px;">未装备</div>

  <!-- 大类折叠选择 -->
  <div style="margin-top: 10px;">
    <div v-for="cat in mainHandCategories" :key="cat" style="margin-bottom: 6px;">
      <div @click="toggleMainCategory(cat)"
           style="padding: 8px 10px; background: #f5f5f5; border-radius: 4px; cursor: pointer; display: flex; justify-content: space-between; user-select: none;">
        <span>{{ cat }}</span>
        <span style="color: #888;">{{ mainHandExpanded === cat ? '▲' : '▼' }}</span>
      </div>
      <div v-show="mainHandExpanded === cat" style="padding: 8px 10px; background: #fafafa; border: 1px solid #eee; border-top: none; border-radius: 0 0 4px 4px;">
        <div v-if="getItemsByCategory('mainHand', cat).length === 0" style="color: #bbb; font-size: 13px;">
          背包中没有此类物品
        </div>
        <div v-for="entry in getItemsByCategory('mainHand', cat)" :key="entry.item_id"
             @click="equipItem('mainHand', entry.item_id); mainHandExpanded = ''"
             style="padding: 6px 0; cursor: pointer; border-bottom: 1px solid #f0f0f0; display: flex; justify-content: space-between;">
          <span>{{ entry.item.name }}</span>
          <span style="color: #888;">×{{ entry.quantity }}</span>
        </div>
      </div>
    </div>
  </div>
</div>

<!-- 副手栏 -->
<div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
  <label style="font-weight: bold;">副手栏</label>
  
  <!-- 当前已装备 -->
  <div v-if="getItemById(currentCharacter.inventory.equipment.offHand)" 
       style="margin: 8px 0; padding: 8px; background: #e3f2fd; border-radius: 6px; display: flex; justify-content: space-between; align-items: center;">
    <span>
      <strong>{{ getItemById(currentCharacter.inventory.equipment.offHand).name }}</strong>
      <span style="color: #888; margin-left: 6px;">{{ getItemById(currentCharacter.inventory.equipment.offHand).category }}</span>
    </span>
    <button @click="unequipItem('offHand')" style="padding: 4px 10px; font-size: 12px; background: #f44336; color: white; border: none; border-radius: 4px; cursor: pointer;">
      卸下
    </button>
  </div>
  <div v-else style="margin: 8px 0; color: #999; font-size: 13px;">未装备</div>

  <!-- 大类折叠选择 -->
  <div style="margin-top: 10px;">
    <div v-for="cat in offHandCategories" :key="cat" style="margin-bottom: 6px;">
      <div @click="toggleOffCategory(cat)"
           style="padding: 8px 10px; background: #f5f5f5; border-radius: 4px; cursor: pointer; display: flex; justify-content: space-between; user-select: none;">
        <span>{{ cat }}</span>
        <span style="color: #888;">{{ offHandExpanded === cat ? '▲' : '▼' }}</span>
      </div>
      <div v-show="offHandExpanded === cat" style="padding: 8px 10px; background: #fafafa; border: 1px solid #eee; border-top: none; border-radius: 0 0 4px 4px;">
        <div v-if="getItemsByCategory('offHand', cat).length === 0" style="color: #bbb; font-size: 13px;">
          背包中没有此类物品
        </div>
        <div v-for="entry in getItemsByCategory('offHand', cat)" :key="entry.item_id"
             @click="equipItem('offHand', entry.item_id); offHandExpanded = ''"
             style="padding: 6px 0; cursor: pointer; border-bottom: 1px solid #f0f0f0; display: flex; justify-content: space-between;">
          <span>{{ entry.item.name }}</span>
          <span style="color: #888;">×{{ entry.quantity }}</span>
        </div>
      </div>
    </div>
  </div>
</div>

  <!-- 护身符（仅可使用 buff 的职业） -->
  <div v-if="currentClassInfo && currentClassInfo.canUseBuff" style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
    <label style="font-weight: bold;">护身符</label>
    <div style="margin-top: 8px;">
      <select
        :value="currentCharacter.inventory.equipment.amulet || ''"
        @change="equipItem('amulet', $event.target.value || null)"
        style="width: 100%; padding: 8px;"
      >
        <option value="">— 未装备 —</option>
        <option
          v-for="entry in getEquippableItems('amulet')"
          :key="entry.item_id"
          :value="entry.item_id"
        >
          {{ entry.item.name }} ×{{ entry.quantity }}
        </option>
      </select>
      <div v-if="getItemById(currentCharacter.inventory.equipment.amulet)" style="margin-top: 6px; font-size: 13px; color: #555;">
        {{ getItemById(currentCharacter.inventory.equipment.amulet).name }}
      </div>
    </div>
  </div>

  <!-- 背包（装备位） -->
  <div style="border: 1px solid #ddd; border-radius: 8px; padding: 12px;">
    <label style="font-weight: bold;">背包</label>
    <div style="margin-top: 8px;">
      <select
        :value="currentCharacter.inventory.equipment.backpack || ''"
        @change="equipItem('backpack', $event.target.value || null)"
        style="width: 100%; padding: 8px;"
      >
        <option value="">— 未装备 —</option>
        <option
          v-for="entry in getEquippableItems('backpack')"
          :key="entry.item_id"
          :value="entry.item_id"
        >
          {{ entry.item.name }} ×{{ entry.quantity }}
        </option>
      </select>
      <div v-if="getItemById(currentCharacter.inventory.equipment.backpack)" style="margin-top: 6px; font-size: 13px; color: #555;">
        {{ getItemById(currentCharacter.inventory.equipment.backpack).name }}
      </div>
    </div>
  </div>
</div>

<!-- 物品栏（背包内） -->
<h2>物品栏（背包内）</h2>
<div style="border: 1px solid #ddd; border-radius: 8px; padding: 15px; margin: 15px 0;">
  <div v-if="!currentCharacter.inventory?.items || currentCharacter.inventory.items.length === 0" style="color: #888; margin-bottom: 15px;">
    目前没有物品（需要 GM 发放后才会出现）
  </div>

  <div v-for="(entry, index) in currentCharacter.inventory.items" :key="entry.id || index"
       style="display: flex; justify-content: space-between; align-items: center; padding: 10px 0; border-bottom: 1px solid #eee;">
    <div>
      <strong>{{ getItemById(entry.item_id)?.name || '未知物品' }}</strong>
      <span style="color: #666; margin-left: 8px;">× {{ entry.quantity }}</span>
      <span v-if="getItemById(entry.item_id)" style="color: #999; margin-left: 8px; font-size: 13px;">
        （{{ getItemById(entry.item_id).category }} · {{ getItemById(entry.item_id).sub_type }}）
      </span>
    </div>
    <div style="font-size: 13px; color: #888;">
      {{ getItemById(entry.item_id)?.description || '' }}
    </div>
  </div>

  <!-- 临时测试：手动添加物品到背包（方便你现在测试，以后会改成只有 GM 能发） -->
  <div style="margin-top: 20px; padding-top: 15px; border-top: 1px dashed #ccc;">
    <div style="font-size: 13px; color: #666; margin-bottom: 8px;">【临时测试】添加物品到背包（正式版会改成 GM 发放）：</div>
    <div style="display: flex; gap: 10px; flex-wrap: wrap;">
      <select v-model="testItemId" style="padding: 8px; min-width: 180px;">
        <option value="">选择物品</option>
        <option v-for="item in itemCatalog" :key="item.id" :value="item.id">
          {{ item.name }}（{{ item.category }}）
        </option>
      </select>
      <input type="number" v-model.number="testItemQty" min="1" style="padding: 8px; width: 70px;" placeholder="数量" />
      <button @click="addTestItem" style="padding: 8px 16px; background: #2196F3; color: white; border: none; border-radius: 4px; cursor: pointer;">
        添加到背包
      </button>
    </div>
  </div>
</div>

    <h2>备注 / 讯息栏</h2>
    <textarea v-model="currentCharacter.notes" rows="4" style="width: 100%; padding: 8px; margin-top: 8px;" placeholder="临时状态、任务笔记等..."></textarea>
  </div>

  <!-- 魔术表页面 -->
<div v-else-if="page === 'magic'" style="max-width: 1000px; margin: 30px auto; font-family: sans-serif; padding: 0 20px;">
  <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;">
    <h1>魔术表</h1>
    <button @click="backToCharacter" style="padding: 8px 16px;">返回角色</button>
  </div>

  <p style="color: #666; margin-bottom: 25px; font-size: 14px;">
    分级说明：一、二册 = 日常　｜　三册 = 专业　｜　四、五册 = 军工　｜　六册 = 神域
  </p>

  <!-- 循环显示每一册 -->
  <div v-for="(vol, key) in magicList" :key="key" style="margin-bottom: 40px;">
    <h2 style="border-bottom: 2px solid #9c27b0; padding-bottom: 8px; color: #7b1fa2;">
      {{ vol.title }}
    </h2>

    <!-- 有魔术内容时显示 -->
    <div v-if="vol.spells && vol.spells.length > 0">
      <div v-for="(m, i) in vol.spells" :key="i"
           style="border: 1px solid #e1bee7; border-radius: 8px; padding: 14px; margin: 12px 0; background: #faf5ff;">
        <div style="display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 8px;">
          <strong style="font-size: 16px; color: #6a1b9a;">{{ m.name }}</strong>
          <span style="background: #ce93d8; color: #4a148c; padding: 3px 10px; border-radius: 12px; font-size: 12px;">
            {{ m.type }}
          </span>
        </div>
        <div style="font-size: 14px; margin-top: 8px; line-height: 1.6; color: #333;">
          {{ m.desc }}
        </div>
      </div>
    </div>

    <!-- 空缺内容时显示提示 -->
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
</template>