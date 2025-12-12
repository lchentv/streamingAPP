<template>
  <div :style="{ padding: '20px', fontFamily: 'sans-serif', maxWidth: '1000px' }">
    <h2>Streaming APP</h2>
    <input v-model="reportId" :style="{ width: '300px', margin: '10px' }" placeholder="Enter Report ID" :disabled="loading"/>
    <input v-model="requestToken" :style="{ width: '300px', margin: '10px' }" placeholder="Enter Request Token" :disabled="loading"/>
    <br />

    <input v-model="symbol" :style="{ width: '300px', margin: '10px' }" placeholder="Enter Symbol" :disabled="loading"/>
    <br />

    <select v-model="reportName" :style="{ width: '300px', margin: '10px' }" :disabled="loading">
      <option v-for="report in availableReports" :key="report" :value="report">
        {{ report }}
      </option>
    </select>
    <br />
    <button @click="submitReportRequest" :style="{ margin: '10px' }" :disabled="loading">
      {{ loading ? "Generating Report..." : "Generate Report" }}
    </button>
    
    

    <br />
    <button @click="startStream" :disabled="loading">
      {{ loading ? "Streaming..." : "Start Streaming" }}
    </button>

    <pre :style="{ maxWidth: '1000px', wordWrap: 'normal', whiteSpace: 'pre-wrap' }">
<!-- {{ output }} -->
    <vue-markdown :style="{ maxWidth: '1000px' }" :source="output" />
    </pre>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import VueMarkdown from 'vue-markdown-render'

const output = ref("")
const loading = ref(false)
const reportId = ref("")
const requestToken = ref("")
const symbol = ref("")
const reportName = ref("fundamental_analysis")
const availableReports = ref(["fundamental_analysis", "stock_screening", "news_sentiment", "buy_and_sell"])

async function submitReportRequest() {
  loading.value = true
  if (!requestToken.value) {
    alert("Please enter a request token.")
    return
  }

  if (!symbol.value) {
    alert("Please enter a symbol.")
    return
  }

  if (!reportName.value) {
    alert("Please select a report name.")
    return
  }

  const response = await fetch(
    "https://oqoyupw8ye.execute-api.us-west-2.amazonaws.com/dev/ai/report/request",
    //"http://localhost:3001/ai/report/fundamental-analysis",
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": "Bearer " + requestToken.value,
      },
      body: JSON.stringify(
        {
            "symbol": symbol.value,
            "name": reportName.value
        }
      )
    }
  )

  const data = await response.json()
  console.log("Fundamental Analysis Response:", data)
  reportId.value = data.id || ""

  await startStream()
}

async function startStream() {
  if (!reportId.value) {
    alert("Please enter a report ID.")
    return
  }

  if (!requestToken.value) {
    alert("Please enter a request token.")
    return
  }

  output.value = ""
  loading.value = true

  const response = await fetch(
    "https://oqoyupw8ye.execute-api.us-west-2.amazonaws.com/dev/ai/report/stream",
    //"http://localhost:3001/ai/report/stream",
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": "Bearer " + requestToken.value,
      },
      body: JSON.stringify({
        "id": reportId.value,
      })
    }
  )

  const reader = response.body.getReader()
  const decoder = new TextDecoder()

  try {
    while (true) {
      const { value, done } = await reader.read()
      if (done) break

      // Convert Uint8Array chunk to string
      const chunk = decoder.decode(value, { stream: true })
      console.log("Received chunk:", chunk)

      // handle multiple JSON objects in a single chunk, like {"status":"complete","data":{"type":"status","content":"Report generation complete.","pos":-1,"timestamp":1764710035}}{"status":"generating","data":{"type":"text","content":"Here is the","timestamp":1764710035,"pos":0}}
      const chunks = chunk.split('}{').map((str, index, arr) => {
        if (arr.length === 1) return str
        if (index === 0) return str + '}'
        if (index === arr.length - 1) return '{' + str
        return '{' + str + '}'
      })

      console.log("Split into chunks:", chunks)

      for (const singleChunk of chunks) {
        // chunk is expected to be a JSON string
        const chunkObj = JSON.parse(singleChunk)
        if (chunkObj.status === "complete") {
          console.log("Stream complete")
          output.value += "\n\n"
          continue
        }
        if (chunkObj.status === "waiting") {
          output.value += "\n\n*Waiting for report generation to start...*\n\n"
          continue
        }
        console.log("Chunk data:", chunkObj.data)
        
        output.value += chunkObj.data.content || ""
      }
    }
  } finally {
    loading.value = false
  }
}
</script>

<style>
body { background: #f9f9f9; }
</style>