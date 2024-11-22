<template>
  <div class="container">
    <van-config-provider theme="dark">
      <van-popup v-model:show="showCurrencyPicker" position="bottom">
        <van-picker
          :columns="currencies"
          show-toolbar
          title="选择货币类型"
          @confirm="onCurrencyConfirm"
        />
      </van-popup>
      <van-space direction="vertical" :size="30">
        <!-- <van-cell-group> -->
        <!-- 选择货币类型 -->
        <van-field
          v-model="form.symbol"
          label="货币类型"
          readonly
          clickable
          placeholder="请选择货币类型"
          @click="showCurrencyPicker = true"
          class="field"
        />

        <!-- 选择做多或做空 -->
        <van-radio-group v-model="form.direction" direction="horizontal" class="radio-group">
          <van-radio name="Long">买涨</van-radio>
          <van-radio name="Short">买跌</van-radio>
        </van-radio-group>

        <!-- 设置保证金 -->
        <van-space :size="30">
          <span>委托金额</span>
          <van-stepper v-model="form.margin" integer min="0" max="1000" />
        </van-space>

        <!-- 设置杠杆 -->
        <van-space :size="30">
          <span>杠杆倍数</span>
          <van-stepper v-model="form.leverage" integer min="0" max="100" />
        </van-space>

        <!-- 止损比例 -->
        <van-space :size="30">
          <span>止损比例</span>
          <van-stepper v-model="form.stop_loss_percent" step="0.01" min="0.01" max="0.5" />
        </van-space>

        <!-- 提交按钮 -->
        <div class="button-container">
          <van-button round type="primary" block @click="handleSubmit"> 提交 </van-button>
        </div>
      </van-space>

      <div>
        <!-- Tab切换 -->
        <van-tabs v-model:active="activeTab" @change="handleTabChange">
          <van-tab title="当前数据">
            <div class="excel-list">
              <!-- 表头 -->
              <div class="header">
                <span v-for="col in columns" :key="col.key" class="header-cell">{{
                  col.label
                }}</span>
              </div>
              <!-- 内容 -->
              <van-cell-group>
                <div class="row" v-for="(row, index) in rows" :key="index">
                  <span v-for="col in columns" :key="col.key" class="cell">
                    <!-- 如果是操作列，显示按钮 -->
                    <template v-if="col.key === 'action'">
                      <van-button type="danger" size="small" @click="handleClose(row)"
                        >平仓</van-button
                      >
                    </template>
                    <!-- 如果不是操作列，显示内容 -->
                    <template v-else>
                      {{ getValue(row, col.key) }}
                    </template>
                  </span>
                </div>
              </van-cell-group>
            </div>
          </van-tab>

          <van-tab title="历史记录">
            <div class="excel-list">
              <!-- 表头 -->
              <div class="header">
                <span v-for="col in historyColumns" :key="col.key" class="header-cell">{{
                  col.label
                }}</span>
              </div>
              <!-- 内容 -->
              <van-cell-group>
                <div class="row" v-for="(row, index) in historyRows" :key="index">
                  <span v-for="col in historyColumns" :key="col.key" class="cell">
                    <!-- 如果是操作列，历史记录不需要操作按钮 -->
                    <template v-if="col.key === 'profit_loss'">
                      <span v-html="calculateProfitLoss(row)"></span>
                    </template>
                    <template v-else-if="col.key === 'created_at'">
                      {{ formatDate(row.created_at) }}
                    </template>

                    <!-- 如果不是操作列，显示内容 -->
                    <template v-else>
                      {{ getValue(row, col.key) }}
                    </template>
                  </span>
                </div>
              </van-cell-group>
            </div>
          </van-tab>
        </van-tabs>
      </div>
    </van-config-provider>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { showToast } from 'vant'
import type { PickerOption } from 'vant'
import axios from 'axios'

// 定义表单数据接口
interface Form {
  symbol: string
  direction: 'Long' | 'Short'
  margin: number
  leverage: number
  stop_loss_percent: number
}

type RowData = {
  id: number
  symbol: string
  margin: number
  direction: string
  entry_price: number
  quantity: number
}

type TradeHistory = {
  id: number // 交易唯一标识
  symbol: string // 交易对名称，例如 BTCUSDT
  entry_price: string // 开仓价格，使用字符串表示
  close_price: string // 平仓价格，使用字符串表示
  direction: 'Long' | 'Short' // 交易方向，仅限 Long 或 Short
  quantity: string // 交易数量，使用字符串表示
  leverage: string // 杠杆倍数，使用字符串表示
  created_at: number // 创建时间的时间戳
}

// 可选货币类型，转换为 PickerOption 数组
const baseCurrencies = [
  'FIL',
  'ADA',
  'CRV',
  'DOGE',
  'DOT',
  'HBAR',
  'OM',
  'XLM',
  'XRP',
  'SUI',
  'WIF',
  'RENDER',
  'NEIRO',
  'PNUT',
  'ACT',
  'LTC',
  'TRX',
  'BNB',
  'WLD',
]

const currencies = ref<PickerOption[]>(
  baseCurrencies.map((currency) => ({
    text: `${currency}/USDT`,
    value: `${currency}/USDT`,
  })),
)

const columns = [
  { key: 'symbol', label: '币种' },
  { key: 'margin', label: '保证金' },
  { key: 'direction', label: '方向' },
  { key: 'entry_price', label: '开仓价格' },
  { key: 'quantity', label: '数量' },
  { key: 'action', label: '操作' },
]

const historyColumns = [
  { key: 'symbol', label: '币种' },
  { key: 'entry_price', label: '开仓价格' },
  { key: 'close_price', label: '平仓价格' },
  { key: 'direction', label: '方向' },
  { key: 'quantity', label: '数量' },
  { key: 'leverage', label: '杠杆' },
  { key: 'profit_loss', label: '盈亏' },
  { key: 'created_at', label: '时间' },
]

const activeTab = ref(0)

onMounted(() => {
  handleTabChange(activeTab.value) // 初始化时触发一次逻辑
})

const handleTabChange = async (index: number): Promise<void> => {
  activeTab.value = index
  try {
    if (index === 0) {
      // 调用获取当前交易数据的接口
      const response = await axios.get('http://127.0.0.1:5000/trade/get_trade')
      // 假设需要将返回数据赋值给 currentRows
      // eslint-disable-next-line @typescript-eslint/no-explicit-any
      rows.value = response.data.map((trade: any) => ({
        id: trade.id,
        symbol: trade.symbol,
        margin: parseFloat(
          (
            (parseFloat(trade.entry_price) * parseFloat(trade.quantity)) /
            parseFloat(trade.leverage)
          ).toFixed(2),
        ),

        direction: trade.direction === 'Long' ? 'long' : 'short', // 转换方向
        entry_price: parseFloat(trade.entry_price),
        quantity: parseFloat(trade.quantity),
      }))
    } else if (index === 1) {
      // 调用获取历史交易数据的接口
      const response = await axios.get('http://127.0.0.1:5000/trade/get_all_history_trades')
      // 假设需要将返回数据赋值给 historyRows
      historyRows.value = response.data
    }
  } catch (error) {
    console.error(`获取交易数据失败 (Tab索引: ${index}):`, error)
  }
}

const rows = ref<RowData[]>([])

const historyRows = ref<TradeHistory[]>([])

// 定义类型安全的 getValue 函数
const getValue = <T extends Record<string, unknown>>(row: T, key: string): string | number => {
  const value = row[key]
  if (typeof value === 'string' || typeof value === 'number') {
    return value
  }
  throw new Error(`Invalid value type for key: ${key}`)
}

const calculateProfitLoss = (trade: TradeHistory): string => {
  const entryPrice = parseFloat(trade.entry_price)
  const closePrice = parseFloat(trade.close_price)
  const quantity = parseFloat(trade.quantity)
  const leverage = parseFloat(trade.leverage)

  // 根据方向计算盈亏
  let profitLoss: number
  if (trade.direction === 'Long') {
    // 多头：平仓价 - 开仓价
    profitLoss = (closePrice - entryPrice) * quantity
  } else if (trade.direction === 'Short') {
    // 空头：开仓价 - 平仓价
    profitLoss = (entryPrice - closePrice) * quantity
  } else {
    throw new Error("Invalid trade direction: must be 'Long' or 'Short'")
  }

  // 计算盈亏百分比
  const profitLossPercentage = ((profitLoss / (entryPrice * quantity)) * leverage * 100).toFixed(2)

  // 格式化盈亏结果
  const profitLossText = `${profitLoss.toFixed(2)} (${profitLossPercentage}%)`

  // 设置颜色：盈为绿色，亏为红色
  const color = profitLoss >= 0 ? 'green' : 'red'

  // 返回带颜色的字符串（如 HTML 格式）
  return `<span style="color: ${color};">${profitLossText}</span>`
}

const formatDate = (timestamp: number): string => {
  const date = new Date(timestamp * 1000) // 假设时间戳是秒级
  const year = date.getFullYear()
  const month = String(date.getMonth() + 1).padStart(2, '0')
  const day = String(date.getDate()).padStart(2, '0')
  const hours = String(date.getHours()).padStart(2, '0')
  const minutes = String(date.getMinutes()).padStart(2, '0')
  const seconds = String(date.getSeconds()).padStart(2, '0')

  return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`
}

// 控制选择器显示
const showCurrencyPicker = ref<boolean>(false)

// 表单数据
const form = ref<Form>({
  symbol: '',
  direction: 'Long', // 默认方向为做多
  margin: 1,
  leverage: 10,
  stop_loss_percent: 0.05,
})

// 选择货币确认处理
const onCurrencyConfirm = (value: PickerOption) => {
  const sv = value.selectedValues[0]
  form.value.symbol = sv // 获取选中的货币类型
  showCurrencyPicker.value = false
}

// 表单提交处理
const handleSubmit = async (): Promise<void> => {
  // 校验输入内容
  if (!form.value.symbol) {
    showToast('请选择货币类型')
    return
  }

  // 准备提交数据
  const postData = {
    symbol: form.value.symbol.replace('/', '').toLowerCase(),
    margin: form.value.margin,
    leverage: form.value.leverage,
    direction: form.value.direction,
    stop_loss_percent: form.value.stop_loss_percent,
  }

  try {
    // 发送 POST 请求
    const response = await axios.post('http://127.0.0.1:5000/trade/create_trade', postData)
    const item = response.data
    rows.value.push({
      id: item.id, // usize 转为 number
      symbol: item.symbol || '', // String 转为 string
      direction: item.direction || '', // TradeDirection 转为 string (后端需确认)
      margin: item.margin || 0, // f64 转为 number
      quantity: item.quantity || '0', // String 转为 string
      entry_price: item.entry_price || 0, // f64 转为 number
    })
    showToast('提交成功')
  } catch (error) {
    console.error('提交失败：', error)
    showToast('提交失败，请重试')
  }

  showToast('提交成功')
}
// 平仓操作
const handleClose = async (row: RowData) => {
  try {
    // 发送 POST 请求
    const postData = {
      id: row.id,
      symbol: row.symbol,
    }
    await axios.post('http://127.0.0.1:5000/trade/close_trade', postData)

    rows.value = rows.value.filter((item) => item.id !== row.id)

    showToast(`平仓操作触发，ID：${row.id}`)
  } catch (error) {
    console.error('提交失败：', error)
    showToast('提交失败，请重试')
  }
}
</script>

<style scoped>
.excel-list {
  width: 100%;
}

.header {
  display: flex;
  /* background-color: #f5f5f5; */
  font-weight: bold;
  padding: 8px;
}

.header-cell {
  flex: 1;
  text-align: center;
}

.row {
  display: flex;
  padding: 8px;
  border-bottom: 1px solid #ddd;
}

.cell {
  flex: 1;
  text-align: center;
}
</style>
