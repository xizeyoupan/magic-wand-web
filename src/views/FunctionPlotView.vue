<template>
  <div class="container mx-auto p-4">
    <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
      <!-- 控制面板 -->
      <div class="bg-white p-6 rounded-lg shadow-md">
        <h2 class="text-xl font-bold mb-4">
          函数参数设置
        </h2>

        <!-- 函数类型选择 -->
        <div class="mb-4">
          <label class="block text-sm font-medium text-gray-700 mb-1">函数类型</label>
          <select
            v-model="functionType"
            class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
          >
            <option value="linear">
              函数
            </option>
            <option value="parametric">
              参数方程
            </option>
            <option value="polar">
              极坐标
            </option>
            <option value="implicit">
              隐函数
            </option>
          </select>
        </div>

        <!-- 函数表达式输入 -->
        <div class="mb-4">
          <label class="block text-sm font-medium text-gray-700 mb-1">
            {{ functionType === 'parametric' ? '参数方程 (x, y)' : '函数表达式' }}
          </label>
          <div v-if="functionType === 'parametric'">
            <input
              v-model="parametricX"
              type="text"
              placeholder="x(t) 表达式，如cos(t)"
              class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none mb-2"
            >
            <input
              v-model="parametricY"
              type="text"
              placeholder="y(t) 表达式，如sin(t)"
              class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
            >
          </div>
          <input
            v-else
            v-model="functionExpression"
            type="text"
            :placeholder="functionExpressionPlaceHolder[functionType]"
            class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
          >
        </div>

        <!-- 定义域输入 -->
        <div class="mb-4">
          <label class="block text-sm font-medium text-gray-700 mb-1">定义域</label>
          <div class="flex space-x-2">
            <input
              v-model="domainMin"
              placeholder="最小值"
              class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
            >
            <input
              v-model="domainMax"
              placeholder="最大值"
              class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
            >
          </div>
        </div>

        <!-- 采样点数输入 -->
        <div class="mb-4">
          <label class="block text-sm font-medium text-gray-700 mb-1">采样点数</label>
          <input
            v-model="samplePoints"
            type="number"
            min="10"
            max="1000"
            class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
          >
        </div>

        <!-- 添加函数按钮 -->
        <button
          class="w-full bg-blue-500 hover:bg-blue-600 text-white font-medium py-2 px-4 rounded-md transition duration-200"
          @click="addFunction"
        >
          添加函数
        </button>
      </div>

      <!-- 图形显示区域 -->
      <div class="md:col-span-2">
        <div
          id="func-plot"
          class="w-full h-96"
        />
      </div>
    </div>

    <!-- 已添加的函数列表 -->
    <div class="mt-6 bg-white p-6 rounded-lg shadow-md">
      <h3 class="text-lg font-semibold mb-4">
        已添加的函数
      </h3>
      <div class="space-y-2">
        <div
          v-for="(func, index) in functions"
          :key="index"
          class="flex justify-between items-center p-3 bg-gray-50 rounded-md"
        >
          <span>{{ func.fnType || 'linear' }}: {{ func.fn || func.r || `x = ${func.x} & y = ${func.y}` }}</span>
          <button
            class="text-red-500 hover:text-red-700"
            @click="removeFunction(index)"
          >
            删除
          </button>
        </div>
      </div>
    </div>
    <!-- 总采样点数和值域映射 -->
    <div class="mt-6 bg-white p-6 rounded-lg shadow-md">
      <h3 class="text-lg font-semibold mb-4">
        波形设置
      </h3>

      <div class="mb-4">
        <label class="block text-sm font-medium text-gray-700 mb-1">通道设置</label>
        <select
          v-model.number="channel"
          class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
        >
          <option value="0">
            仅通道0， GPIO25
          </option>
          <option value="1">
            仅通道1， GPIO26
          </option>
          <option value="2">
            通道0和1， GPIO25和GPIO26
          </option>
        </select>
      </div>

      <!-- 总采样点数 -->
      <div class="mb-4">
        <label class="block text-sm font-medium text-gray-700 mb-1">总采样点数</label>
        <input
          v-model="totalSamplePoints"
          type="number"
          min="10"
          max="10000"
          class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
        >
      </div>

      <div class="mb-4">
        <label class="block text-sm font-medium text-gray-700 mb-1">freq</label>
        <input
          v-model="freq"
          type="number"
          min="10"
          max="10000"
          class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
        >
      </div>

      <div class="mb-4">
        <label class="block text-sm text-gray-600 mb-1">X轴范围</label>
        <div class="flex space-x-2">
          <input
            v-model="xDomainMin"
            placeholder="最小值"
            class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
          >
          <input
            v-model="xDomainMax"
            placeholder="最大值"
            class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
          >
        </div>
      </div>

      <!-- 值域映射 -->
      <div class="mb-4">
        <label class="block text-sm text-gray-600 mb-1">Y轴范围</label>
        <div class="flex space-x-2">
          <input
            v-model="yDomainMin"
            placeholder="最小值"
            class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
          >
          <input
            v-model="yDomainMax"
            placeholder="最大值"
            class="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:outline-none"
          >
        </div>
      </div>

      <!-- 生成数据按钮 -->
      <div class="mb-4">
        <button
          class="w-full bg-green-500 hover:bg-green-600 text-white font-medium py-2 px-4 rounded-md transition duration-200"
          @click="generateData"
        >
          启动
        </button>
      </div>

      <div class="mb-4">
        <button
          class="w-full bg-blue-500 hover:bg-blue-600 text-white font-medium py-2 px-4 rounded-md transition duration-200"
          @click="async () => {
            toast(t('toast.loading'), 'info')
            const result = await wsmgr.sendRequest('stop_dac_dma', )
            toast_on_result(result)
          }"
        >
          停止
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import functionPlot from "function-plot"
import { builtIn, interval } from "function-plot/dist/samplers"
import { onMounted, reactive, ref, toRaw } from 'vue'
import { abs, compile, evaluate, format, parse, parser, range, round } from 'mathjs'
import { scaleLinear as d3ScaleLinear } from 'd3-scale'
import { toast_on_result } from "../util.js"
import { wsmgr } from "../plugins/ws"
import { toast } from '../plugins/toast.js'
import { i18n } from '../i18n.js'
const t = i18n.global.t

const functionExpressionPlaceHolder = {
    linear: "f(x)表达式，例如: x^2 或 sin(x)",
    polar: "r(θ)表达式，例如: 2 * sin(2 theta)",
    implicit: "例如: x * x + y * y - 4",
}

// 响应式数据
const functionType = ref('linear')
const functionExpression = ref('')
const parametricX = ref('')
const parametricY = ref('')
const domainMin = ref('-5')
const domainMax = ref('5')
const xDomainMin = ref('-5')
const xDomainMax = ref('5')
const yDomainMin = ref('-5')
const yDomainMax = ref('5')
const samplePoints = ref(1000)
const totalSamplePoints = ref(1000)
const freq = ref(50)
const functions = ref([])
const channel = ref(0)

const options = {
    target: '#func-plot',
    width: 580,
    height: 400,
    data: functions.value
}
let sampleData = Array.from({ length: 2 }, () => Array.from({ length: totalSamplePoints.value }, () => NaN))
let x_index = 0
const EPS = 0.01
const generateData = async () => {
    x_index = 0
    sampleData = Array.from({ length: 2 }, () => Array.from({ length: totalSamplePoints.value }, () => NaN))
    const height = 256
    const xd = [evaluate(xDomainMin.value), evaluate(xDomainMax.value)]
    const yd = [evaluate(yDomainMin.value), evaluate(yDomainMax.value)]
    const totalYScale = d3ScaleLinear().domain(yd).range([0, height - 1])
    const totalXScale = d3ScaleLinear().domain(xd).range([0, totalSamplePoints.value - 1])
    functions.value.map(func => {
        func = toRaw(func)
        const samplerParams = {
            range: toRaw(func.range),
            nSamples: func.nSamples,
            xAxis: { type: 'linear' },
        }
        const d = { fnType: func.fnType, range: samplerParams.range }
        const width = samplerParams.nSamples

        const xDomain = samplerParams.range
        const yDomain = [evaluate(yDomainMin.value), evaluate(yDomainMax.value)]
        const xScale = d3ScaleLinear().domain(xDomain).range([0, width])
        const yScale = d3ScaleLinear().domain(yDomain).range([-height - 1, height])
        samplerParams.xScale = xScale
        samplerParams.yScale = yScale

        switch (func.fnType) {
        case 'linear':
            d.fn = func.fn
            break
        case 'parametric':
            d.x = func.x
            d.y = func.y
            break
        case 'polar':
            d.r = func.r
            break
        case 'implicit':
            d.fn = func.fn
            break
        default:
            break
        }
        samplerParams.d = d
        console.log(func)
        console.log(samplerParams)

        let data = []
        if (functionType.value === 'implicit') {
            interval(samplerParams).map(i => {
                data = data.concat(i)
            })

            data = data.map(i => {
                i[0] = (i[0].lo + i[0].hi) / 2
                i[1] = (i[1].lo + i[1].hi) / 2
                // console.log(i[0] * i[0] + i[1] * i[1] - 1)
                return i
            })
        } else {
            builtIn(samplerParams).map(i => {
                data = data.concat(i)
            })
        }

        data.sort((a, b) => a[1] - b[1])

        for (let j = 0; j < data.length; j++) {
            const i = data[j]
            const index = Math.round(totalXScale(i[0]))
            // console.log(`index: ${index} x: ${i[0]}, y: ${i[1]}, totalYScale: ${totalYScale(i[1])}`)
            if (isNaN(sampleData[x_index][index])) {
                sampleData[x_index][index] = totalYScale(i[1])
                sampleData[x_index][index] = round(sampleData[x_index][index])

                if (sampleData[x_index][index] > height - 1) {
                    sampleData[x_index][index] = height - 1
                }

                if (sampleData[x_index][index] < 0) {
                    sampleData[x_index][index] = 0
                }
            }
        }

        x_index = 1 - x_index
    })


    if (isNaN(sampleData[0][0])) {
        sampleData[0][0] = 0
    }

    if (isNaN(sampleData[1][0])) {
        sampleData[1][0] = 0
    }

    for (let i = 0; i < 2; i++) {
        for (let j = 1; j < sampleData[0].length; j++) {
            if (isNaN(sampleData[i][j])) {
                sampleData[i][j] = sampleData[i][j - 1]
            }
        }
    }

    console.log(sampleData)
    const wav_data = []

    if (channel.value === 2) {
        for (let i = 0; i < sampleData[0].length; i++) {
            wav_data.push(sampleData[0][i])
            wav_data.push(sampleData[1][i])
        }
    } else {
        for (let i = 0; i < sampleData[0].length; i++) {
            wav_data.push(sampleData[0][i])
        }
    }

    toast(t('toast.loading'), 'info')
    const result = await wsmgr.sendRequest('start_dac_dma', {
        wav_data: wav_data,
        freq: freq.value,
        channel: channel.value,
    })

    toast_on_result(result)
}

// 添加函数
const addFunction = () => {
    const dmin = evaluate(domainMin.value)
    const dmax = evaluate(domainMax.value)
    const func = {
        range: [dmin, dmax],
        nSamples: samplePoints.value,
    }
    switch (functionType.value) {
    case 'linear':
        func.fnType = 'linear'
        func.fn = functionExpression.value
        break
    case 'parametric':
        func.fnType = 'parametric'
        func.x = parametricX.value
        func.y = parametricY.value
        func.graphType = 'polyline'
        break
    case 'polar':
        func.fnType = 'polar'
        func.r = functionExpression.value
        func.graphType = 'polyline'
        break
    case 'implicit':
        func.fnType = 'implicit'
        func.fn = functionExpression.value
    default:
        break
    }

    functions.value.push(func)

    updatePlot()
}

// 删除函数
const removeFunction = (index) => {
    functions.value.splice(index, 1)
    updatePlot()
}

// 更新图表
const updatePlot = () => {
    options.data = toRaw(options.data)
    console.log(options)
    try {
        functionPlot(options)
    } catch (e) {
        console.error(e)
        functions.value.splice(-1, 1)
        functionPlot(options)
    }
}

</script>

<style scoped>
.container {
  max-width: 1200px;
}
</style>