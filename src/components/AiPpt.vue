<template>
  <div>
    <!-- 生成大纲 -->
    <GenerateOutline v-if="step == 1" :token="token" @nextStep="handleOutline" />
    <!-- 选择模板 -->
    <SelectTemplate v-if="step == 2" :token="token" @nextStep="selectTemplate" />
    <!-- 生成PPT -->
    <GeneratePpt v-if="step == 3" :token="token" :templateId="templateId" :outline="outline" :dataUrl="dataUrl" />
  </div>
</template>

<script>
import GenerateOutline from '@/components/GenerateOutline'
import SelectTemplate from '@/components/SelectTemplate'
import GeneratePpt from '@/components/GeneratePpt'
export default {
  name: 'AiPpt',
  components: { GenerateOutline, SelectTemplate, GeneratePpt },
  props: {},
  data() {
    return {
      step: 1,
      outline: '',
      dataUrl: null,
      templateId: '',
      token: null
    }
  },
  created() {
    this.createApiToken()
  },
  methods: {
    async createApiToken() {
      // 文多多AiPPT
      // 官网 https://docmee.cn
      // 开放平台 https://docmee.cn/open-platform/api#接口鉴权

      // api key
      const apiKey = ''
      // 用户ID（数据隔离）
      const uid = 'test'

      if (!apiKey) {
        alert('请在代码中设置apiKey')
        return
      }
      const url = 'https://docmee.cn/api/user/createApiToken'
      const resp = await (await fetch(url, {
        method: 'POST',
        headers: {
          'Api-Key': apiKey,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          uid,
          limit: null
        })
      })).json()
      if (resp.code != 0) {
        alert('创建接口token异常：' + resp.message)
        return
      }
      this.token = resp.data.token
    },
    handleOutline(params) {
      this.step++
      this.outline = params.outline
      this.dataUrl = params.dataUrl
    },
    selectTemplate(id) {
      this.step++
      this.templateId = id
    }
  }
}
</script>
