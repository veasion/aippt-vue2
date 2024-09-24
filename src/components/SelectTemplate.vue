<template>
    <div class="content">
      <div>---- 选择模板 ----</div>
      <div class="but_div">
        <button @click="nextStep">下一步: 生成PPT</button>
      </div>
      <div class="template_div">
        <div v-for="template in templates" :key="template.id" :class="template.id == templateId ? 'template select' : 'template'" @click="selectTemplate(template)">
          <img :src="template.coverUrl + '?token=' + token" />
        </div>
        <div class="template_but">
          <button @click="loadTemplates">换一批</button>
        </div>
        <div v-if="templates.length == 0">模板加载中...</div>
      </div>
    </div>
</template>

<script>
const background = document.body.style.background
export default {
  name: 'SelectTemplate',
  components: {},
  props: {
    token: String
  },
  data() {
    return {
      templates: [],
      templateId: ''
    }
  },
  mounted() {
    this.loadTemplates()
  },
  methods: {
    async loadTemplates() {
      const url = 'https://docmee.cn/api/ppt/randomTemplates'
      const resp = await (await fetch(url, {
        method: 'POST',
        headers: {
          'token': this.token,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ page: 1, size: 28, filters: { type: 1 } })
      })).json()
      if (resp.code != 0) {
        alert('获取模板异常：' + resp.message)
        return
      }
      this.templates = resp.data || []
      this.selectTemplate(resp.data[0])
    },
    selectTemplate(template) {
      this.templateId = template.id
      const src = template.coverUrl + '?token=' + this.token
      this.calcSubjectColor(src).then((color) => {
        document.body.style.background = color
      })
    },
    async calcSubjectColor(src) {
      let img = new Image()
      await new Promise(resolve => {
          let eqOrigin = src.startsWith('data:') || src.startsWith(document.location.origin) || (src.startsWith('//') && (document.location.protocol + src).startsWith(document.location.origin))
          if (!eqOrigin) {
            img.crossOrigin = 'anonymous'
          }
          img.src = src
          img.onload = function() {
            resolve(img)
          }
          img.onerror = function (e) {
            resolve(null)
          }
      })
      let canvas = document.createElement('canvas')
      let ctx = canvas.getContext('2d')
      canvas.width = img.width
      canvas.height = img.height
      ctx.drawImage(img, 0, 0)
      let imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
      let data = imageData.data
      let map = {}
      for (let i = 0; i < data.length; i += 4) {
          let r = data[i]
          let g = data[i + 1]
          let b = data[i + 2]
          let rgb = r + g + b
          if (rgb <= 50 || rgb >= 660) {
              // 忽略黑白
              continue
          }
          let color = `${r},${g},${b}`
          map[color] = (map[color] || 0) + 1
      }
      let valueMap = {}
      for (let k in map) {
          let v = map[k]
          let ks = valueMap[v]
          if (ks) {
              ks.push(k)
          } else {
              valueMap[v] = [k]
          }
      }
      let colors = []
      let values = Object.values(map).sort()
      for (let i = values.length - 1; i >= 0; i--) {
          let ks = valueMap[values[i]]
          for (let j = 0; j < ks.length; j++) {
              colors.push(ks[j])
              if (colors.length >= 3) {
                  break
              }
          }
          if (colors.length >= 3) {
              break
          }
      }
      // return `rgb(${colors[0]})`
      return `linear-gradient(to bottom right, rgb(${colors[0]}), rgb(${colors[1]}), rgb(${colors[2]}))`
    },
    nextStep() {
      if (!this.templateId) {
        alert('请选择模板')
        return
      }
      document.body.style.background = background
      this.$emit('nextStep', this.templateId)
    }
  }
}
</script>

<style scoped>
.content {
  padding-top: 4em;
  text-align: center;
}
.but_div {
  margin-top: 0.8em;
}
.template_div {
  width: 80%;
  max-width: 1050px;
  margin: auto;
  display: flex;
  flex-wrap: wrap;
  flex-direction: row;
  align-items: center;
  text-align: center;
}
.template {
  margin: 8px;
  width: 240px;
  height: 136px;
  cursor: pointer;
  overflow: hidden;
  border-radius: 12px;
  background-color: #ecedf4;
  border: 2px solid transparent;
}
.template img {
  width: 100%;
  height: 100%;
  opacity: 1;
  object-fit: cover;
  transition: all .6s ease;
}
.template img:hover {
  transform: scale(1.2);
  transition-duration: .3s;
}
.template_but {
  width: 100%;
  padding: 10px;
  text-align: center;
}
.select {
  border: 2px solid #491ff8;
}
</style>
