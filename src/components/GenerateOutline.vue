<template>
  <div class="content">
      <h1>🤖 AI智能生成PPT演示文稿</h1>
      <div class="desc">生成大纲 ---&gt; 挑选模板 --&gt; 实时生成PPT</div>
      <div v-if="genStatus == 0" class="input_div">
        <select v-model="selectType">
          <option value="subject">根据主题</option>
          <option value="text">根据内容</option>
          <option value="file">根据文件</option>
        </select>
        <div v-if="selectType == 'subject'">
          <input v-model="subject" placeholder="请输入PPT主题" maxlength="20" />
        </div>
        <div v-else-if="selectType == 'text'">
          <textarea v-model="text" placeholder="请输入内容" rows="5" cols="50" maxlength="6000"></textarea>
        </div>
        <div v-else-if="selectType == 'file'">
          <input id="input_file" type="file" placeholder="请选择文件" accept=".doc, .docx, .xls, .xlsx, .pdf, .ppt, .pptx, .txt" />
        </div>
        <button @click="generateOutline">生成大纲</button>
      </div>
      <div v-if="genStatus == 1" v-html="outlineHtml" class="outline"></div>
      <div v-if="genStatus == 2">
        <button @click="nextStep">下一步：选择模板</button>
        <OutlineEdit :outlineTree="outlineTree" @update="updateOutline" class="outline_edit" />
      </div>
  </div>
</template>

<script>
import { marked } from 'marked'
import { SSE } from '../utils/sse.js'
import OutlineEdit from '@/components/OutlineEdit'
export default {
  name: 'GenerateOutline',
  components: { OutlineEdit },
  props: {
    token: String
  },
  data() {
    return {
      selectType: 'subject', // subject 根据主题；text 根据内容；file 根据文件
      subject: '',
      text: '',
      dataUrl: null,
      genStatus: 0, // 生成状态: 0未开始 1生成中 2已完成
      outline: '',
      outlineTree: null,
      outlineHtml: ''
    }
  },
  created() {
    marked.setOptions({
      renderer: new marked.Renderer(),
      gfm: true,
      async: false,
      breaks: false,
      pedantic: false,
      silent: true
    })
  },
  methods: {
    parseFileData(formData) {
      let url = 'https://docmee.cn/api/ppt/parseFileData'
      let xhr = new XMLHttpRequest()
      xhr.open('POST', url, false)
      xhr.setRequestHeader('token', this.token)
      xhr.send(formData)
      if (xhr.readyState === 4) {
        if (xhr.status === 200) {
            let resp = JSON.parse(xhr.responseText)
            if (resp.code != 0) {
                alert('解析文件异常：' + resp.message)
                return null
            }
            return resp.data.dataUrl
        } else {
          alert('解析文件网络异常, httpStatus: ' + xhr.status)
          return null
        }
      }
    },
    generateOutline() {
      if (this.genStatus != 0) {
        return
      }
      this.genStatus = 1
      this.outlineHtml = '<h3>正在生成中，请稍后....</h3>'
      const inputData = {}
      if (this.selectType == 'subject') {
        // 根据主题
        if (!this.subject) {
          alert('请输入主题')
          this.genStatus = 0
          return
        }
        inputData.subject = this.subject
      } else if (this.selectType == 'text') {
        // 根据内容
        if (!this.text) {
          alert('请输入内容')
          this.genStatus = 0
          return
        }
        const formData = new FormData()
        formData.append('content', this.text)
        inputData.dataUrl = this.parseFileData(formData)
      } else if (this.selectType == 'file') {
        // 根据文件
        const file = document.getElementById('input_file')?.files[0]
        if (!file) {
          alert('请选择文件')
          this.genStatus = 0
          return
        }
        const formData = new FormData()
        formData.append('file', file)
        inputData.dataUrl = this.parseFileData(formData)
      }
      if (!inputData.subject && !inputData.dataUrl) {
        this.genStatus = 0
        return
      }
      this.genStatus = 1
      this.dataUrl = inputData.dataUrl
      const url = 'https://docmee.cn/api/ppt/generateOutline'
      var source = new SSE(url, {
          method: 'POST',
          headers: {
              'token': this.token,
              'Cache-Control': 'no-cache',
              'Content-Type': 'application/json'
          },
          payload: JSON.stringify(inputData),
      })
      source.onmessage = (data) => {
        let json = JSON.parse(data.data)
        if (json.status == -1) {
          alert('生成大纲异常：' + json.error)
          this.genStatus = 0
          return
        }
        if (json.status == 4 && json.result) {
          this.outlineTree = json.result
        } else {
          this.outline += json.text
          this.outlineHtml = marked.parse(this.outline.replace('```markdown', '').replace(/```/g, ''))
          window.scrollTo({ behavior: 'smooth', top: document.body.scrollHeight })
        }
      }
      source.onend = (data) => {
        if (data.data.startsWith('{') && data.data.endsWith('}')) {
          let json = JSON.parse(data.data)
          if (json.code != 0) {
            alert('生成大纲异常：' + json.message)
            this.genStatus = 0
            return
          }
        }
        this.genStatus = 2
        window.scrollTo(0, 0)
      }
      source.onerror = (err) => {
        console.error('生成大纲异常', err)
        alert('生成大纲异常')
        this.genStatus = 0
      }
      source.stream()
    },
    updateOutline(md) {
      this.outline = md + ''
    },
    nextStep() {
      this.$emit('nextStep', { outline: this.outline, dataUrl: this.dataUrl })
    }
  }
}
</script>

<style scoped>
.content {
  padding-top: 4em;
  text-align: center;
}
.input_div {
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
}
select, button {
  margin: 0 .8em;
}
input[type=file] {
  border: 1px dashed #ccc;
}
.desc {
  font-size: 1em;
  margin-top: 1em;
  color: #999;
  margin-bottom: 4em;
}
.outline {
  text-align: left;
  margin: 0 calc(20vw);
}
.outline_edit {
  width: 32rem;
  margin: 1rem auto;
}
</style>
