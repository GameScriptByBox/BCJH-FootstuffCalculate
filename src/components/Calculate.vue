<template>
    <div class="panel">
        <div class="line"><span>名称</span><el-input v-model="menuTitle" placeholder="菜谱名称" /></div>
        <div class="line"><span>份数</span><el-input v-model="menuAmount" placeholder="一次营业的数量" /></div>
        <div class="line"><span>耗时</span><el-input v-model="timeConsue" placeholder="x小时x分x秒" /></div>
        <div class="line"><span>耗材</span><el-input v-model="consumption" placeholder="鸡蛋*2 面粉*2" /></div>
        <div class="line"><span>数量</span><el-input v-model="menuAmountAll" placeholder="需要制作的总份数" /></div>

        <div class="calculate-info">
            <div class="info">
                <div><span>一份菜的时间(秒)</span>{{ singleTime }}秒</div>
                <div>
                    <span>{{ menuAmountAll || menuAmount || 0 }}份菜的时间(秒)</span>
                    <span>{{ allTimeSeconds }}秒</span>
                    <span>{{ timeFormat1 }}</span>
                </div>
                <div>{{ currentConsuption }}</div>
            </div>

            <!-- 总时间 格式 1h 1m 1s -->
            <div class="line" @click="copy1">{{ timeFormat2 }}</div>
            <!-- 每次开业上菜数量 -->
            <div class="line" @click="copy2">{{ businessNum }}</div>

            <!-- 当前任务的总时间 -->
            <div class="line task-time" @click="copy3">
                <div>
                    <div>{{ currentTaskSumTime }}</div>
                    <div>{{ taskTimeAll }}</div>
                </div>
                <el-button type="primary" @click="currentTaskSumTime = []">清空</el-button>
            </div>

            <!-- 当前任务总耗材 -->
            <div class="line task-foods" @click="copy4">
                <div>{{ foods }}</div>
                <el-button type="primary" @click.stop="foods = {}">清空</el-button>
            </div>
        </div>

        <div class="reset">
            <el-button type="primary" @click="reset">重置</el-button>
        </div>
    </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import useClipboard from 'vue-clipboard3'
import { foodMap } from '@/constant/food.js'

const { toClipboard } = useClipboard()

const menuTitle = ref() //菜谱名称
const menuAmount = ref() //份数
const timeConsue = ref() // 时间消耗
const consumption = ref() // 食材消耗
const menuAmountAll = ref() // 需要制作的总份数

// 实际的份数
const realMenuAmount = computed(() => Number(menuAmountAll.value) || Number(menuAmount.value) || 0)

// 一份菜的食材消耗
const onceFood = computed(() => {
    const tempData = consumption.value
    if (!tempData) return

    const res = []

    const foodList = tempData.split(' ')
    foodList.forEach(item => {
        const food = item.split('*')[0]
        const num = Number(item.split('*')[1])
        res.push({ food, num })
    })

    return res
})

// 当前菜谱的食材消耗
const currentConsuption = computed(() => {
    const tempData = onceFood.value //一份菜的耗材
    const menuNum = realMenuAmount.value //当前菜的数量
    if (!(tempData && menuNum)) return

    const resList = []
    tempData.forEach(item => {
        let t = {}
        t.food = item.food
        t.num = item.num * menuNum
        resList.push(t)
    })

    return resList
})

// 正则获取 时分秒
function handleTimeStr(exp) {
    const data = timeConsue.value.match(exp)
    return data ? Number(data[1]) : 0
}

// 将秒 转为 1h 1m 格式
function formatTime(seconds) {
    let res = ''
    const h = Math.floor(seconds / 3600)
    const m = Math.floor((seconds % 3600) / 60)
    if (h) res += `${h}h `
    if (m) res += `${m}m`
    return res
}

// 一份菜的时间(秒)
const singleTime = computed(() => {
    if (!timeConsue.value) return 0
    if (!menuAmount.value) return 0
    const hours = handleTimeStr(/(\d+)小时/)
    const minutes = handleTimeStr(/(\d+)分/)
    const seconds = handleTimeStr(/(\d+)秒/)
    return Math.round((hours * 3600 + minutes * 60 + seconds) / menuAmount.value)
})

// 该菜谱总份数需要的总时间(秒)
const allTimeSeconds = computed(() => {
    if (!menuAmount.value) return 0
    if (!timeConsue.value) return 0
    const all = menuAmountAll.value || menuAmount.value
    return singleTime.value * all
})

// 格式：1小时1分
const timeFormat1 = computed(() => {
    if (!allTimeSeconds.value) return null
    const timeAll = allTimeSeconds.value
    let res = ''
    const h = Math.floor(timeAll / 3600)
    const m = Math.floor((timeAll % 3600) / 60)
    if (h) res += `${h}小时`
    if (m) res += `${m}分`
    return res
})

// 格式：1h 1m
const timeFormat2 = computed(() => {
    if (!allTimeSeconds.value) return null
    return formatTime(allTimeSeconds.value)
})

// 每次开业上菜数量
const businessNum = computed(() => {
    const once = Number(menuAmount.value)
    const all = Number(menuAmountAll.value)
    if (!menuTitle.value) return null
    if (!once) return null
    if (!all) return `${menuTitle.value}${once}`

    let res = menuTitle.value
    if (all > once) {
        res += once
        const a = Math.floor(all / once)
        const b = all % once
        if (a > 1) res += `*${a}`
        if (b != 0) res += `+${b}`
    } else {
        res += all
    }

    return res
})

// 重置
const reset = () => {
    // 将该菜谱计入当前任务的总时间
    if (allTimeSeconds.value) currentTaskSumTime.value.push(allTimeSeconds.value)
    //   计算耗材
    if (currentConsuption.value) calculateFood(currentConsuption.value)

    // 清空输入
    menuTitle.value = null
    menuAmount.value = null
    consumption.value = null
    timeConsue.value = null
    menuAmountAll.value = null
}

const copy1 = () => {
    if (!timeFormat2.value) return
    toClipboard(timeFormat2.value)
    ElMessage({
        message: '复制成功',
        type: 'success',
    })
}

const copy2 = () => {
    if (!businessNum.value) return
    toClipboard(businessNum.value)
    ElMessage({
        message: '复制成功',
        type: 'success',
    })
}

const copy3 = () => {
    if (!taskTimeAll.value) return
    toClipboard(taskTimeAll.value)
    ElMessage({
        message: '复制成功',
        type: 'success',
    })
}

// 整理耗材顺序
const copy4 = () => {
    if (!foods.value) return
    let res = ''

    for (const [k, v] of Object.entries(foodMap)) {
        res += foods.value[k] ? foods.value[k] + '\t' : '\t'
    }

    toClipboard(res)

    ElMessage({
        message: '复制成功',
        type: 'success',
    })
}

// 计算当前任务的总时间
const currentTaskSumTime = ref([])
const taskTimeAll = computed(() => {
    if (!currentTaskSumTime.value.length) return null
    return formatTime(currentTaskSumTime.value.reduce((a, b) => a + b))
})

// 计算当前任务的总耗材
const foods = ref({})
const calculateFood = data => {
    data.forEach(item => {
        foods.value[item.food] = foods.value[item.food] ? foods.value[item.food] + item.num : item.num
    })
}
</script>

<style lang="scss" scoped>
@import '@/assets/scss/calculate.scss';
</style>
