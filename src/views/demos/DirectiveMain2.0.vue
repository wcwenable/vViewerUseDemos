<template>
  <div>
    <el-row>
      <el-col :span="12">
        <el-upload :disabled="isNeedHideUpload" :headers="{ Authorization: authorization}" drag action="" :show-file-list="false" multiple :http-request="handleUploadManually" :before-upload="beforeImageUpload">
          <div class="el-upload__tip" slot="tip"></div>
          <div slot="trigger" :class="triggerClass">
            <img src="@/assets/ocr/upload@2x.png">
            <div class="el-upload__text uploadDesc">您可以拖拽上传也可以选择<em>本地文件</em><br>支持JPG/PNG格式，大小4M以内</div>
          </div>
        </el-upload>
        <div v-if="uploadErrList.length > 0" class="uploadErr">
          <el-alert title="以下文件上传失败" type="error" @close="handleErrAlertClose" effect="dark">
            <ul>
              <li class="errli" v-for="(err, index) in uploadErrList" :key="index">{{err}}</li>
            </ul>
          </el-alert>
        </div>
        <recognize-situation-list :uploadFiles="uploadFiles" @onToggleFullPreview="handleToggleFullPreview" ref="recognizeSituationList"></recognize-situation-list>
      </el-col>
      <el-col :span="12">
        <div class="grid-content bg-purple-light">4324234</div>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import { compress } from '@/views/common/utils'
import RecognizeSituationList from '@/views/demos/RecognizeSituationList'
import localforage from 'localforage'
export default {
  components: {
    RecognizeSituationList
  },
  data () {
    return {
      // 单次最多允许同时上传的文件数量
      MAXFILESNUM: 20,
      RECOGNING_TO_DELETE: 3,
      isNeedHideUpload: false,
      canUploadMore: true,
      authorization: null,
      uploadErrList: [],
      currentOcrTemplateId: '1234556',
      uploadFiles: []
    }
  },
  created () {
    this.$http.post('/api/authcenter/user/login', {
      username: 'fetest',
      password: '79043ac2819b1864fa2030f686c26f46'
    }, {
      emulateJSON: true
    }).then(res => {
      this.authorization = res.data.data.token
      // console.log('login>res515', res, res.data.data.token)
    })
    // 清除已上传的图片缓存
    this.clearUploadedFiles()
    // this.removeCachedUploadFiles()
  },
  mounted () {},
  computed: {
    triggerClass () {
      return {
        'triggerClass': this.isNeedHideUpload
      }
    }
  },
  methods: {
    handleErrAlertClose () {
      this.uploadErrList = []
    },
    beforeImageUpload (file) { // 上传文件前的钩子对文件进行校验
      const isValidSize = file.size <= 1 * 1024 * 1024
      const isValidType = ['image/png', 'image/jpeg'].includes(file.type)
      let errDesc = `文件【${file.name}】:`

      !isValidSize && (errDesc = `${errDesc}超过了4M！`)

      !isValidType && (errDesc = `${errDesc}不受支持的图片格式！`)

      const isValid = isValidSize && isValidType

      !isValid && this.uploadErrList.push(errDesc)

      return isValid
    },
    async handleUploadManually (file) {
      const actualFile = file.file
      const isInCache = this.uploadFiles.some(file => file.name === actualFile.name)
      if (isInCache) {
        this.uploadErrList.unshift(`文件【${actualFile.name}】已在上传队列中，请勿重复上传！`)
      } else {
        const countCachedFiles = this.uploadFiles.length
        const customFile = {
          ocrRecognizeRecordId: `orrId${countCachedFiles}`,
          ocrTemplateId: this.currentOcrTemplateId,
          picUrl: require('@/assets/ocr/uploadBackground.png'),
          uploadStatus: 0,
          isUploadWaiting: true,
          name: actualFile.name,
          type: actualFile.type, // (0:还未手动上传，1：正在上传，2：已上传，待识别，3：正在识别，此时需要从缓存中删除)
          file: actualFile
        }
        this.uploadFiles.push(customFile)
        this.setCachedUploadFiles(this.uploadFiles)
        // 开始上传图片
        this.submitUploadFileRequest(customFile)
      }
      // console.log('handleUploadManually (file515', file, this.uploadFiles, actualFile.name)
    },
    handleToggleFullPreview (newVal) {
      this.isNeedHideUpload = newVal
    },
    async getCacheddUploadFiles () {
      return localforage.getItem('uploadFiles').then(value => {
        // console.log('getCacheddUploadFiles>value515', value)
        return value || []
      }).catch(err => {
        console.log('getCacheddUploadFiles>err515', err)
        return []
      })
    },
    submitUploadFileRequest (file) {
      const fileToUpload = this.uploadFiles.find(f => f.name === file.name)
      if (fileToUpload) {
        fileToUpload.uploadStatus = 1
        fileToUpload.isUploadWaiting = true
        this.handleCompressFile(fileToUpload)
        // this.$http.post('/api/common/v1/uploadFile', formData).then(res => {
        //   fileToUpload.picUrl = res.data.data
        //   fileToUpload.uploadStatus = 2
        //   fileToUpload.isUploadWaiting = true
        //   fileToUpload.bizStatus = 1
        //   fileToUpload.recognizeStatus = 1
        //   // console.log('v1/uploadFile>res515', res)
        // })
      }
    },
    // 图片预览
    handleCompressFile (file) {
      const isLt10M = file.file.size / 1024 / 1024 < 1.5
      if (!isLt10M) {
        this.$message.error('上传图片大小不能超过 10M!')
        return false
      } else {
        // this.dialogImageUrl = URL.createObjectURL(file.raw)
        compress(file.file, val => {
          ((context, file, val) => {
            const formData = new FormData()
            formData.append('file', val)
            formData.append('type', file.type)
            const uploadFile = context.uploadFiles.find(f => f.name === file.name)
            context.$http.post('/api/common/v1/uploadFile', formData).then(res => {
              uploadFile.picUrl = res.data.data
              uploadFile.uploadStatus = 2
              uploadFile.isUploadWaiting = true
              uploadFile.bizStatus = 1
              uploadFile.recognizeStatus = 1
            })
          })(this, file, val)
        })
      }
    },
    removeCachedUploadFiles () {
      localforage.removeItem('uploadFiles').then(() => {
        // Run this code once the key has been removed.
        this.uploadFiles = []
        // console.log('uploadFiles is cleared!')
      }).catch(function (err) {
        // This code runs if there were any errors
        console.log('removeItem(uploadFiles515', err)
      })
    },
    setCachedUploadFiles () {
      // Unlike localStorage, you can store non-strings.
      return localforage.setItem('uploadFiles', this.uploadFiles).then(value => {
        // console.log('setCachedUploadFiles>vaule515', value)
      }).catch(function (err) {
        console.log('setCachedUploadFiles>err', err)
      })
    },
    async clearUploadedFiles () {
      let currentCachedFiles = await this.getCacheddUploadFiles()
      this.uploadFiles = currentCachedFiles
      // console.log('clearUploadedFiles>currentCachedFiles515', currentCachedFiles)
      const isRecognizingToDelete = currentCachedFiles.some(file => file.uploadStatus === this.RECOGNING_TO_DELETE)
      if (isRecognizingToDelete) {
        this.uploadFiles = currentCachedFiles = currentCachedFiles.filter(file => file.uploadStatus !== this.RECOGNING_TO_DELETE)
        this.setCachedUploadFiles(currentCachedFiles)
      }
      this.uploadFiles.forEach(file => {
        this.submitUploadFileRequest(file)
      })
    }
  }
}
</script>

<style lang="scss">
  .triggerClass {
    cursor: not-allowed;
  }
  .uploadDesc {
    margin-top: 20px;
  }
  .el-upload-dragger {
    padding: 20px 0;
    height: auto;
    width: 500px;
  }
  .uploadErr {
    margin: 10px;
    align-items: center;
  }
  .errli {
    text-align: left;
  }
</style>
