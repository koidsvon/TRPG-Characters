<script setup>
import { ref } from 'vue'
import { supabase } from './supabase.js'

// 页面状态：home / room / character
const page = ref('home')

// 房间相关
const roomName = ref('')
const gmPassword = ref('')
const joinCode = ref('')
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

// 生成房间码
function generateCode() {
  const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'
  let code = ''
  for (let i = 0; i < 6; i++) {
    code += chars.charAt(Math.floor(Math.random() * chars.length))
  }
  return code
}

// 创建房间
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
    loadCharacters()
  }
}

// 加入房间
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
  } else {
    currentSession.value = data
    isGM.value = false
    page.value = 'room'
    loadCharacters()
  }
}

// 加载角色列表
async function loadCharacters() {
  if (!currentSession.value) return

  const { data, error } = await supabase
    .from('characters')
    .select('*')
    .eq('session_id', currentSession.value.id)
    .order('name', { ascending: true })

  if (error) {
    console.error(error)
    alert('加载角色失败：' + error.message)
  } else {
    characters.value = data || []
  }
}

// 创建新角色
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

// 进入某个角色
function enterCharacter(char) {
  // 确保 inventory 是数组
  const charCopy = { ...char }
  if (!Array.isArray(charCopy.inventory)) {
    charCopy.inventory = []
  }
  currentCharacter.value = charCopy
  page.value = 'character'
}

// 添加物品
function addItem() {
  if (!newItemName.value) {
    alert('请输入物品名称')
    return
  }

  if (!currentCharacter.value.inventory) {
    currentCharacter.value.inventory = []
  }

  currentCharacter.value.inventory.push({
    id: Date.now(),          // 简单用时间戳当临时id
    name: newItemName.value,
    quantity: newItemQuantity.value || 1
  })

  newItemName.value = ''
  newItemQuantity.value = 1
}

// 删除物品
function removeItem(index) {
  currentCharacter.value.inventory.splice(index, 1)
}

// 保存角色（包含物品栏）
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

// 返回角色列表
function backToRoom() {
  page.value = 'room'
  currentCharacter.value = null
  loadCharacters()
}

// 返回首页
function backToHome() {
  page.value = 'home'
  currentSession.value = null
  characters.value = []
  currentCharacter.value = null
  message.value = ''
}
</script>

<template>
  <!-- 首页 -->
  <div v-if="page === 'home'" style="max-width: 500px; margin: 50px auto; font-family: sans-serif;">
    <h1>跑团角色工具</h1>
    <p style="color: #666;">私用多人在线角色卡</p>

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
    <button @click="createRoom" style="padding: 10px 20px; background: #4CAF50; color: white; border: none; cursor: pointer;">
      创建房间
    </button>

    <hr style="margin: 40px 0;" />

    <h2>加入房间</h2>
    <div style="margin-bottom: 15px;">
      <label>房间码：</label><br />
      <input v-model="joinCode" placeholder="输入6位房间码" style="width: 100%; padding: 8px; margin-top: 5px;" />
    </div>
    <button @click="joinRoom" style="padding: 10px 20px; background: #2196F3; color: white; border: none; cursor: pointer;">
      加入房间
    </button>

    <p style="margin-top: 30px; color: #c00;">{{ message }}</p>
  </div>

  <!-- 角色列表页 -->
  <div v-else-if="page === 'room'" style="max-width: 800px; margin: 30px auto; font-family: sans-serif; padding: 0 20px;">
    <div style="display: flex; justify-content: space-between; align-items: center;">
      <div>
        <h1>{{ currentSession?.name }}</h1>
        <p>房间码：<strong>{{ currentSession?.code }}</strong>　|　{{ isGM ? '你是 GM' : '你是玩家' }}</p>
      </div>
      <button @click="backToHome" style="padding: 8px 16px;">返回首页</button>
    </div>

    <hr style="margin: 20px 0;" />

    <h2>角色列表</h2>

    <div v-if="characters.length === 0" style="color: #888; margin: 20px 0;">
      目前还没有角色，创建一个吧。
    </div>

    <div v-for="char in characters" :key="char.id" 
         style="border: 1px solid #ddd; padding: 15px; margin-bottom: 10px; border-radius: 8px; display: flex; justify-content: space-between; align-items: center;">
      <div>
        <strong style="font-size: 18px;">{{ char.name }}</strong>
        <span style="color: #666; margin-left: 10px;">（{{ char.player_name }}）</span>
        <div style="font-size: 14px; color: #888; margin-top: 4px;">
          职业：{{ char.class_name || '未选择' }}　|　等级：{{ char.level || 1 }}
        </div>
      </div>
      <button @click="enterCharacter(char)" style="padding: 6px 12px; background: #2196F3; color: white; border: none; border-radius: 4px; cursor: pointer;">
        进入角色
      </button>
    </div>

    <hr style="margin: 30px 0;" />

    <h3>创建新角色</h3>
    <div style="display: flex; gap: 10px; margin-top: 10px; flex-wrap: wrap;">
      <input v-model="newCharacterName" placeholder="角色名" style="padding: 8px; flex: 1; min-width: 150px;" />
      <input v-model="newPlayerName" placeholder="玩家昵称（可选）" style="padding: 8px; flex: 1; min-width: 150px;" />
      <button @click="createCharacter" style="padding: 8px 20px; background: #4CAF50; color: white; border: none; cursor: pointer;">
        创建角色
      </button>
    </div>
  </div>

  <!-- 角色详情/编辑页 -->
  <div v-else-if="page === 'character'" style="max-width: 900px; margin: 30px auto; font-family: sans-serif; padding: 0 20px;">
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;">
      <h1>编辑角色</h1>
      <div>
        <button @click="saveCharacter" :disabled="saving" style="padding: 8px 20px; background: #4CAF50; color: white; border: none; margin-right: 10px; cursor: pointer;">
          {{ saving ? '保存中...' : '保存' }}
        </button>
        <button @click="backToRoom" style="padding: 8px 16px;">返回列表</button>
      </div>
    </div>

    <!-- 基础信息 -->
    <div style="display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin-bottom: 30px;">
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
        <input v-model="currentCharacter.class_name" style="width: 100%; padding: 8px;" placeholder="剑士 / 赏金猎人 / 魔术师..." />
      </div>
      <div>
        <label>等级</label><br />
        <input type="number" v-model.number="currentCharacter.level" style="width: 100%; padding: 8px;" />
      </div>
    </div>

    <h2>核心属性</h2>
    <div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(140px, 1fr)); gap: 15px; margin: 15px 0 30px;">
      <div>
        <label>力量</label><br />
        <input type="number" v-model.number="currentCharacter.strength" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>智力</label><br />
        <input type="number" v-model.number="currentCharacter.intelligence" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>敏捷</label><br />
        <input type="number" v-model.number="currentCharacter.agility" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>HP 当前</label><br />
        <input type="number" v-model.number="currentCharacter.hp_current" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>HP 最大</label><br />
        <input type="number" v-model.number="currentCharacter.hp_max" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>ATK</label><br />
        <input type="number" v-model.number="currentCharacter.atk" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>DEF</label><br />
        <input type="number" v-model.number="currentCharacter.def" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>RES</label><br />
        <input type="number" v-model.number="currentCharacter.res" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>SPD</label><br />
        <input type="number" v-model.number="currentCharacter.spd" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>移动格</label><br />
        <input type="number" v-model.number="currentCharacter.move_range" style="width: 100%; padding: 8px;" />
      </div>
      <div>
        <label>PP</label><br />
        <input type="number" v-model.number="currentCharacter.pp" style="width: 100%; padding: 8px;" />
      </div>
    </div>

    <!-- 物品栏 -->
    <h2>物品栏</h2>
    <div style="border: 1px solid #ddd; border-radius: 8px; padding: 15px; margin: 15px 0;">
      <div v-if="!currentCharacter.inventory || currentCharacter.inventory.length === 0" style="color: #888; margin-bottom: 15px;">
        目前没有物品
      </div>

      <div v-for="(item, index) in currentCharacter.inventory" :key="item.id"
           style="display: flex; justify-content: space-between; align-items: center; padding: 8px 0; border-bottom: 1px solid #eee;">
        <div>
          <strong>{{ item.name }}</strong>
          <span style="color: #666; margin-left: 10px;">× {{ item.quantity }}</span>
        </div>
        <button @click="removeItem(index)" style="padding: 4px 10px; background: #f44336; color: white; border: none; border-radius: 4px; cursor: pointer;">
          删除
        </button>
      </div>

      <div style="display: flex; gap: 10px; margin-top: 15px; flex-wrap: wrap;">
        <input v-model="newItemName" placeholder="物品名称" style="padding: 8px; flex: 2; min-width: 150px;" />
        <input type="number" v-model.number="newItemQuantity" placeholder="数量" style="padding: 8px; width: 80px;" />
        <button @click="addItem" style="padding: 8px 16px; background: #2196F3; color: white; border: none; cursor: pointer;">
          添加物品
        </button>
      </div>
    </div>

    <h2>备注 / 讯息栏</h2>
    <textarea v-model="currentCharacter.notes" rows="4" style="width: 100%; padding: 8px; margin-top: 8px;" placeholder="可以写一些临时状态、任务笔记等..."></textarea>

    <p style="margin-top: 30px; color: #888; font-size: 14px;">
      提示：修改属性或物品后，记得点右上角「保存」。
    </p>
  </div>
</template>