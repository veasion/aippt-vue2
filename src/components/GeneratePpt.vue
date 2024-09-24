<template>
    <div class="content">
      <div v-if="gening" class="desc">
        <span>{{ descMsg }}</span>
        <span style="margin-left: 5px">{{  descTime }}秒</span>
      </div>
      <div class="left_div">
        <div class="left_div_child">
          <div class="left_div_child_child">
            <div v-for="(page, index) in pages" :key="index" class="left_div_item" @click="drawPptx(index)">
              <div class="left_div_item_index">{{ index + 1 }}</div>
              <canvas :ref="'canvas_' + index" width="288" height="162" :class="currentIdx == index ? 'left_div_item_img item_select' : 'left_div_item_img'" />
            </div>
            <div v-if="gening && currentIdx > 0">
              <div class="left_div_item">
                <div class="left_div_item_index">{{ currentIdx + 2 }}</div>
                <div class="left_div_item_img img_gening">生成中...</div>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="right_div">
        <a v-if="!gening && pptId" class="download" @click="downloadPptx">下载</a>
        <div style="margin-left: 200px;">
          <svg ref="svg" class="right_canvas"></svg>
        </div>
      </div>
    </div>
</template>

<script>
import pako from 'pako'
import base64js from 'base64-js'
import { SSE } from '../utils/sse.js'
import { Ppt2Svg } from '../utils/ppt2svg.js'
import { Ppt2Canvas } from '../utils/ppt2canvas.js'

var pptxObj = null
var painter = null

export default {
  name: 'GeneratePpt',
  components: {},
  props: {
    token: String,
    templateId: String,
    outline: String,
    dataUrl: String
  },
  data() {
    return {
      gening: false,
      descTime: 0,
      descMsg: '正在生成中，请稍后...',
      pptId: null,
      pages: [],
      currentIdx: 0
    }
  },
  created() {
  },
  mounted() {
    // svg
    painter = new Ppt2Svg(this.$refs.svg)
    painter.setMode('edit')

    var mTimer = 0
    window.addEventListener('resize', () => {
      mTimer && clearTimeout(mTimer)
      mTimer = setTimeout(() => {
        this.resetSize()
      }, 50)
    })

    this.resetSize()

    let _pptId = new URLSearchParams(window.location.search).get('pptId')
    if (_pptId) {
      this.loadById(_pptId)
    } else {
      this.generatePptx(this.templateId, this.outline, this.dataUrl)
    }
  },
  methods: {
    generatePptx(templateId, outline, dataUrl) {
      var count = 0
      var timer = setInterval(() => {
          count = count + 1
          this.descTime = count
      }, 1000)
      this.gening = true
      const url = 'https://docmee.cn/api/ppt/generateContent'
      var source = new SSE(url, {
          method: 'POST',
          headers: {
            'token': this.token,
            'Cache-Control': 'no-cache',
            'Content-Type': 'application/json',
          },
          payload: JSON.stringify({ asyncGenPptx: true, templateId, outlineMarkdown: outline, dataUrl }),
      })
      source.onmessage = (data) => {
          let json = JSON.parse(data.data)
          if (json.status == -1) {
            alert('生成PPT异常：' + json.error)
            return
          }
          if (json.pptId) {
            this.descMsg = `正在生成中，进度 ${json.current}/${json.total}，请稍后...`
            this.asyncGenPptxInfo(json.pptId)
          }
      }
      source.onend = (data) => {
          if (data.data.startsWith('{') && data.data.endsWith('}')) {
            let json = JSON.parse(data.data)
            if (json.code != 0) {
              alert('生成PPT异常：' + json.message)
              return
            }
          }
          clearInterval(timer)
          this.gening = false
          this.descMsg = '正在生成中，请稍后...'
          setTimeout(() => {
            this.drawPptxList(0, false)
          }, 200)
      }
      source.onerror = (err) => {
          clearInterval(timer)
          console.error('生成内容异常', err)
          alert('生成内容异常')
      }
      source.stream()
    },
    asyncGenPptxInfo(id) {
      this.pptId = id
      let url = `https://docmee.cn/api/ppt/asyncPptInfo?pptId=${id}`
      let xhr = new XMLHttpRequest()
      xhr.open('GET', url, true)
      xhr.setRequestHeader('token', this.token)
      xhr.send()
      xhr.onload = () => {
        if (xhr.status === 200) {
          let resp = JSON.parse(xhr.responseText)
          let gzipBase64 = resp.data.pptxProperty
          let gzip = base64js.toByteArray(gzipBase64)
          let json = pako.ungzip(gzip, { to: 'string' })
          let _pptxObj = JSON.parse(json)
          pptxObj = _pptxObj
          this.drawPptxList(resp.data.current - 1, true)
        }
      }
      xhr.onerror = function (e) {
          console.error(e)
      }
    },
    drawPptxList(_idx, asyncGen) {
      let idx = _idx || 0
      this.currentIdx = idx || 0
      if (_idx == null || asyncGen) {
        let _pages = []
        for (let i = 0; i < pptxObj.pages.length; i++) {
          if (asyncGen && i > idx) {
              break
          }
          _pages.push(pptxObj.pages[i])
        }
        this.pages = _pages
        this.$nextTick(async () => {
          if (asyncGen) {
            this.getCanvas(idx).scrollIntoView(true)
          }
          for (let i = 0; i < _pages.length; i++) {
            let imgCanvas = this.getCanvas(i)
            if (!imgCanvas) {
                continue
            }
            try {
                let _ppt2Canvas = new Ppt2Canvas(imgCanvas)
                await _ppt2Canvas.drawPptx(pptxObj, i)
            } catch(e) {
                console.log('渲染第' + (i + 1) + '页封面异常', e)
            }
          }
        })
      } else {
        this.pages = pptxObj.pages || []
        this.$nextTick(async () => {
          try {
            let imgCanvas = this.getCanvas(idx)
            let _ppt2Canvas = new Ppt2Canvas(imgCanvas)
            await _ppt2Canvas.drawPptx(pptxObj, idx)
          } catch(e) {
            console.log('渲染第' + (idx + 1) + '页封面异常', e)
          }
        })
      }

      this.drawPptx(idx)
    },
    drawPptx(idx) {
      this.currentIdx = idx
      painter.drawPptx(pptxObj, idx)
      if (idx == 0) {
        this.$nextTick(async () => {
          this.getCanvas(idx).scrollIntoView(true)
        })
      }
    },
    downloadPptx() {
      let url = 'https://docmee.cn/api/ppt/downloadPptx'
      let xhr = new XMLHttpRequest()
      xhr.open('POST', url, true)
      xhr.setRequestHeader('token', this.token)
      xhr.setRequestHeader('Content-Type', 'application/json')
      xhr.send(JSON.stringify({ id: this.pptId }))
      xhr.onload = () => {
        if (xhr.status === 200) {
          let resp = JSON.parse(xhr.responseText)
          let fileUrl = resp.data.fileUrl
          let a = document.createElement('a')
          a.href = fileUrl
          a.download = (resp.data.subject || 'download') + '.pptx'
          a.click()
        }
      }
      xhr.onerror = function (e) {
          console.error(e)
      }
    },
    loadById(id) {
      this.gening = false
      this.pptId = id
      let url = 'https://docmee.cn/api/ppt/loadPptx?id=' + id
      let xhr = new XMLHttpRequest()
      xhr.open('GET', url, true)
      xhr.setRequestHeader('token', this.token)
      xhr.send()
      xhr.onload = () => {
        if (xhr.status === 200) {
          let resp = JSON.parse(xhr.responseText)
          if (resp.code != 0) {
              alert(resp.message)
              return
          }
          let pptInfo = resp.data.pptInfo
          let gzipBase64 = pptInfo.pptxProperty
          let gzip = base64js.toByteArray(gzipBase64)
          let json = pako.ungzip(gzip, { to: 'string' })
          pptInfo.pptxProperty = JSON.parse(json)
          pptxObj = pptInfo.pptxProperty
          this.drawPptxList()
        }
      }
      xhr.onerror = function (e) {
          console.error(e)
      }
    },
    resetSize() {
      let width = Math.max(Math.min(document.body.clientWidth - 400, 1600), 480)
      painter.resetSize(width, width * 0.5625)
    },
    getCanvas(idx) {
      return this.$refs['canvas_' + idx][0]
    }
  }
}
</script>

<style scoped>
::-webkit-scrollbar {
  width: 5px;
  height: 5px;
  background-color: #fff;
}
::-webkit-scrollbar-thumb {
  background-color: #c1c1c1;
}
body {
  overflow: hidden;
}
.content {
  padding-left: 1em;
}
.desc {
  text-align: center;
  padding-top: 1.2em;
}
.left_div {
  align-items: center;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
  height: 96vh;
  justify-content: center;
  position: absolute;
  width: 158px;
}
.left_div_child {
  background: #fff;
  border: 1px solid #fff;
  border-radius: 6px;
  box-shadow: 0 4px 10px 0 #0003;
  margin-left: 12px;
  margin-top: 28px;
  max-height: calc(100vh - 130px);
  width: 200px;
}
.left_div_child_child {
  max-height: calc(100vh - 130px);
  overflow-x: hidden;
  overflow-y: auto;
  padding: 0 8px 0 2px;
}
.left_div_item {
  display: flex;
  cursor: pointer;
  margin: 10px 0;
}
.left_div_item_index {
  color: #8d90a5;
  flex-shrink: 0;
  padding-right: 6px;
  padding-top: 30px;
  text-align: right;
  width: 23px;
}
.left_div_item_img {
  height: 81px;
  width: 144px;
  border: 1px solid #ccc;
}
.img_gening {
  text-align: center;
  line-height: 81px;
  color: #666;
  cursor: default;
}
.right_div {
  padding-top: calc(15vh);
}
.right_canvas {
  margin: 0px auto;
  display: block;
  border: 1px solid #666;
}
.item_select {
  border: 2px solid #491ff8;
}
.draw_el_class {
  position: fixed;
}
.draw_el_class:hover {
  cursor: text;
  border: 1.5px solid red;
}
.download {
  float: right;
  cursor: pointer;
  margin-right: 1.5rem;
}
</style>
