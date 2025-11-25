<template>
  <div :style="{ padding: '20px', fontFamily: 'sans-serif', maxWidth: '1000px' }">
    <h2>Streaming APP</h2>

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

async function startStream() {
  output.value = ""
  loading.value = true

  const response = await fetch(
    "https://xxsqiggyz3pk2s6sdh3odffy3m0yoxaq.lambda-url.us-west-2.on.aws/ai/report/stream",
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": "Bearer XXXXXX"
      },
      body: JSON.stringify({
        "id": "report-001",
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
      // chunk is expected to be a JSON string
      const chuckObj = JSON.parse(chunk)
      
      output.value += chuckObj.data.content || ""
    }
  } finally {
    loading.value = false
  }
}
</script>

<style>
body { background: #f9f9f9; }
</style>