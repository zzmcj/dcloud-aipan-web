<template>
  <div class="chat-container">
    <div class="message-list" ref="messageList">
      <div v-for="(message, index) in messages" :key="index" 
           :class="['message-item', message.isUser ? 'user-message' : 'assistant-message']">
        <div class="avatar">
          <img :src="message.isUser ? userAvatar : assistantAvatar" alt="avatar"/>
        </div>
        <div class="message-content">
          <template v-if="message.isLoading">
            <LoadingDots />
          </template>
          <div v-else class="message-text" v-html="message.content"></div>
        </div>
      </div>
    </div>
    
    <div class="input-area">
      <div class="input-wrapper">
        <textarea 
          v-model="inputMessage" 
          placeholder="发消息，请输入..."
          @keyup.enter="sendMessage"
          rows="1"
        ></textarea>
        <div class="send-button" @click="sendMessage" :class="{ 'active': inputMessage.trim() }">
          <svg t="1710400669747" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="4231" width="24" height="24">
            <path d="M931.4 481.3L815.9 396l0.3-0.4-604.7-249c-3.8-1.6-7.9-1.9-11.9-1l-0.3-0.7L135 171.3c-12.3 2.7-20.3 14.8-17.6 27.1 0.9 4.3 3.1 8 6.2 10.9l-0.3 0.7 155.6 130.3L492.3 436l-213.4 95.7-140.9 118c-9.7 8.1-11 22.6-2.8 32.3 3.9 4.6 9.3 7.5 15.2 8.1l0.3 0.7 63.9 26.3c4 1.6 8.1 1.9 12.1 1l0.3 0.7 201.4-83 139.3 57.4c4 1.6 8.1 1.9 12.1 0.9l0.3 0.7 63.9-26.3c12.3-2.7 20.3-14.8 17.6-27.1-0.9-4.3-3.1-8-6.2-10.9l0.3-0.7-102.8-86 389.4-174.6c11.3-5.1 16.3-18.3 11.3-29.6-2.3-4.9-6-8.6-10.8-11z" p-id="4232"></path>
          </svg>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { useLoginUserStore } from "@/store/user";
import { defineComponent, ref, onMounted, nextTick } from 'vue'
import LoadingDots from '@/components/common/LoadingDots.vue'
import { getApiUrl, API_PATHS } from '@/config/api'
import { ElMessage } from 'element-plus'
import { downloadUrlParam, downloadFiles } from '@/api/file'
import JSONBIG from 'json-bigint'

// 创建JSON解析器实例
const jsonParser = JSONBIG({ storeAsString: true })

// 添加处理大数值的函数
function parseJSON(str: string) {
  const bigIntRegex = /([^"]:)(\d{16,})([\D])/g;
  const wrappedStr = str.replace(bigIntRegex, '$1"$2"$3');
  return JSON.parse(wrappedStr);
}

// 添加自定义JSON解析函数
const customJSONParse = (text: string) => {
  return JSON.parse(text, (key, value) => {
    // 检查是否是数字类型且是整数
    if (typeof value === 'number' && Number.isInteger(value)) {
      // 检查是否超过安全整数范围
      if (Math.abs(value) > Number.MAX_SAFE_INTEGER) {
        return value.toString();
      }
    }
    return value;
  });
};

// 扩展 Window 接口
declare global {
  interface Window {
    handleFileDownload: (fileId: string, fileName: string, isFolder: boolean) => Promise<void>;
  }
}

interface FileItem {
  id: string;
  file_id: string | null;
  file_name: string;
  file_type: string | null;
  file_suffix: string | null;
  file_size: number | null;
  gmt_create: string;
  gmt_modified: string;
}

interface StorageInfo {
  used_size: number;
  total_size: number;
  used_percentage: number;
}

interface FileStatistics {
  total_files: number;
  file_types: {
    [key: string]: number;
  };
}

export default defineComponent({
  name: 'Answer',
  components: {
    LoadingDots
  },
  setup() {
    const messages = ref<Array<{isUser: boolean; content: string; isLoading?: boolean}>>([
      {
        isUser: false,
        content: '亲爱的用户,很高兴能为你解答问题、陪你交流 !'
      }
    ])
    const inputMessage = ref('')
    const messageList = ref<HTMLElement | null>(null)
    const userAvatar = require('@/assets/user-avatar.png')
    const assistantAvatar = require('@/assets/assistant-avatar.png')
    const isLoading = ref(false)

    const scrollToBottom = () => {
      setTimeout(() => {
        if (messageList.value) {
          messageList.value.scrollTop = messageList.value.scrollHeight
        }
      }, 100)
    }

    

    // 格式化文件大小
    const formatFileSize = (size: number) => {
      if (!size) return '-';
      const units = ['B', 'KB', 'MB', 'GB', 'TB'];
      let index = 0;
      let fileSize = size;

      while (fileSize >= 1024 && index < units.length - 1) {
        fileSize /= 1024;
        index++;
      }

      return `${fileSize.toFixed(2)} ${units[index]}`;
    };

    // 获取文件图标
    const getFileIcon = (fileType: string) => {
      switch (fileType?.toUpperCase()) {
        case 'IMG':
          return '🖼️';
        case 'DOC':
        case 'DOCX':
          return '📄';
        case 'PDF':
          return '📑';
        case 'VIDEO':
          return '🎥';
        case 'AUDIO':
          return '🎵';
        case '文件夹':
          return '📁';
        default:
          return '📄';
      }
    };

    const sendMessage = async () => {
      if (!inputMessage.value.trim()) return

      const userMessage = inputMessage.value
      messages.value.push({ isUser: true, content: userMessage })
      messages.value.push({ isUser: false, content: '', isLoading: true })
      inputMessage.value = ''
      scrollToBottom()

      try {
        const response = await fetch(getApiUrl(API_PATHS.PAN_QUERY), {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'token': useLoginUserStore().token || ''
          },
          body: JSON.stringify({
            account_id: '0',
            query: userMessage
          })
        })

        if (!response.ok) {
          throw new Error('请求失败: ' + response.status)
        }

        // 获取响应文本并使用json-bigint解析
        const text = await response.text()
        const responseData = jsonParser.parse(text)
        
        console.log('完整的响应数据:', responseData)

        if (responseData.code === 0) {
          // 检查是否是存储信息类型
          if (responseData.data && responseData.data.type === 'storage_info') {
            console.log('存储信息数据:', responseData.data)
            const storageData = responseData.data.data
            
            // 确保数据存在且有效
            if (storageData) {
              const storageInfo = {
                used_size: Number(storageData.used_size) || 0,
                total_size: Number(storageData.total_size) || 10485760,
                used_percentage: Number(storageData.used_percentage) || 0
              }
              
              console.log('处理后的存储信息:', storageInfo)
              
              const usedSize = formatFileSize(storageInfo.used_size)
              const totalSize = formatFileSize(storageInfo.total_size)
              const percentage = (storageInfo.used_percentage).toFixed(2)
              
              const formattedContent = `
                <div class="storage-info">
                  <div class="storage-header">
                    <span class="storage-title">存储空间使用情况</span>
                    <span class="storage-percentage">${percentage}%</span>
                  </div>
                  <div class="storage-progress">
                    <div class="progress-bar" style="width: ${percentage}%"></div>
                  </div>
                  <div class="storage-details">
                    <div class="storage-used">
                      <span class="label">已使用</span>
                      <span class="value">${usedSize}</span>
                    </div>
                    <div class="storage-total">
                      <span class="label">总容量</span>
                      <span class="value">${totalSize}</span>
                    </div>
                  </div>
                </div>
              `
              messages.value[messages.value.length - 1].content = formattedContent
            }
          } else if (responseData.type ==='text' && responseData.data.type === undefined) {
            // 处理文本类型响应
            messages.value[messages.value.length - 1].content = responseData.data.content || ''
          } else if (responseData.data && responseData.data.type === 'file_list') {
            const fileList = responseData.data.data as FileItem[]
            let formattedContent = '<div class="file-list">\n'
            console.log('文件列表数据:', fileList)
            fileList.forEach((file: FileItem) => {
              const isFolder = file.file_size === null;
              const fileType = file.file_type || (isFolder ? '文件夹' : '文件');
              const fileName = file.file_name;
              const fileId = String(file.id);
              console.log('文件数据:', fileId)

              formattedContent += `
                <div class="file-item">
                  <span class="file-icon">${getFileIcon(fileType)}&nbsp&nbsp</span>
                  ${isFolder ? 
                    `<span class="file-name">${fileName}</span>` :
                    `<span class="file-name" style="cursor: pointer; color: #409EFF;" 
                      onclick="handleFileDownload('${String(fileId)}', '${fileName}', false)">${fileName}</span>`
                  }
                </div>
              `
            })
            console.log('文件列表数据:', fileList)
            
            formattedContent += '</div>'
            messages.value[messages.value.length - 1].content = formattedContent

            // 添加文件下载处理函数到window对象
            window.handleFileDownload = async (fileId: string, fileName: string, isFolder: boolean) => {
              try {
                if (isFolder) {
                  ElMessage.warning('文件夹不支持下载');
                  return;
                }

                const fileIds = [fileId];
                const response = await downloadUrlParam({
                  fileIds,
                });

                if (response.data && response.data.success) {
                  console.log('文件下载连接：', response.data.data);
                  const downloadUrls = response.data.data;
                  
                  for (const downloadInfo of downloadUrls) {
                    const response = await fetch(downloadInfo.downloadUrl);
                    const blob = await response.blob();
                    const link = document.createElement('a');
                    const url = URL.createObjectURL(blob);
                    link.href = url;
                    link.download = downloadInfo.fileName;
                    document.body.appendChild(link);
                    link.click();
                    URL.revokeObjectURL(url);
                    document.body.removeChild(link);
                  }
                  
                  ElMessage.success('开始下载文件');
                } else {
                  ElMessage.error(response.data?.msg || '获取下载链接失败');
                }
              } catch (error) {
                console.error('下载文件出错:', error);
                ElMessage.error('下载文件失败');
              }
            };
          } else if (responseData.data && responseData.data.type === 'file_statistics') {
            const stats = responseData.data.data
            const fileTypeMap: { [key: string]: { icon: string; name: string } } = {
              IMG: { icon: '🖼️', name: '图片' },
              DOC: { icon: '📄', name: '文档' },
              VIDEO: { icon: '🎥', name: '视频' },
              AUDIO: { icon: '🎵', name: '音频' },
              PDF: { icon: '📑', name: 'PDF' },
              other: { icon: '📁', name: '其他' }
            }

            let typeStatsHtml = ''
            if (stats && stats.file_types) {
              Object.entries(stats.file_types).forEach(([type, count]) => {
                const typeInfo = fileTypeMap[type] || { icon: '📁', name: type }
                typeStatsHtml += `
                  <div class="file-type-item">
                    <span class="type-icon">${typeInfo.icon}</span>
                    <div class="type-info">
                      <span class="type-name">${typeInfo.name}</span>
                      <span class="type-count">${count} 个文件</span>
                    </div>
                  </div>
                `
              })
            }

            const formattedContent = `
              <div class="file-statistics">
                <div class="stats-header">
                  <div class="total-files">
                    <span class="total-number">${stats.total_files || 0}</span>
                    <span class="total-label">总文件数</span>
                  </div>
                </div>
              </div>
            `
            messages.value[messages.value.length - 1].content = formattedContent
          }else if (responseData.data?.type === 'file_content') {
            const answer = responseData.data.data?.answer || '在你的文件里没有找到相关内容'
            const names = (responseData.data.data?.sources || []).map((item: { file_name?: string }) => item.file_name).filter(Boolean)
            messages.value[messages.value.length - 1].content = names.length? `${answer}\n来源：${names.join('、')}`: answer} 
          else {
            messages.value[messages.value.length - 1].content = '未知的响应类型'
          }
        } else {
          messages.value[messages.value.length - 1].content = responseData.msg || '请求失败'
        }
        
        messages.value[messages.value.length - 1].isLoading = false
      } catch (error) {
        console.error('Error:', error)
        const lastMessageIndex = messages.value.length - 1
        messages.value[lastMessageIndex] = {
          isUser: false,
          content: '抱歉，发生了错误：' + (error instanceof Error ? error.message : '未知错误')
        }
      }
      scrollToBottom()
    }

    return {
      messages,
      inputMessage,
      messageList,
      sendMessage,
      userAvatar,
      assistantAvatar,
      isLoading
    }
  }
})
</script>

<style scoped>
.chat-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  background-color: #ffffff;

}

.message-list {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.message-item {
  display: flex;
  gap: 12px;
  max-width: 80%;
}

.message-item.user-message {
  margin-left: auto;
  flex-direction: row-reverse;
}

.message-item.user-message .message-content {
  background-color: #efefef;
}

.message-item.assistant-message {
  margin-right: auto;
}

.message-item.assistant-message .message-content {
  background-color: #f5f8fc;
}

.avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  overflow: hidden;
  flex-shrink: 0;
}

.avatar img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.message-content {
  padding: 12px 16px;
  border-radius: 8px;
  font-size: 14px;
  line-height: 1.5;
  word-break: break-word;
  min-height: 24px; /* 添加最小高度确保加载动画显示正常 */
}

.input-area {
  padding: 20px;
  background-color: #ffffff;
 
}

.input-wrapper {
  position: relative;
  max-width: 1080px;
  min-height: 100px;
  margin: 0 auto;
  background-color: #ffffff;
  border-radius: 8px;
  border: 2px solid #ededed;
  padding: 12px 50px 12px 16px;
}

.input-wrapper textarea {
  width: 100%;
  border: none;
  background: transparent;
  resize: none;
  outline: none;
  font-size: 14px;
  line-height: 1.5;
  max-height: 120px;
  overflow-y: auto;
}

.send-button {
  position: absolute;
  right: 12px;
  bottom: 8px;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  border-radius: 50%;
  transition: all 0.3s;
}

.send-button svg {
  fill: #999;
  transition: all 0.3s;
}

.send-button.active svg {
  fill: #1890ff;
}

.send-button:hover {
  background-color: rgba(24, 144, 255, 0.1);
}

.send-button.active:hover {
  background-color: rgba(24, 144, 255, 0.2);
}

.file-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 10px;
  background: #ffffff;
  border-radius: 8px;
}

.file-item {
  display: flex;
  align-items: center;
  padding: 8px;
  border-radius: 4px;
  background: #f5f7fa;
  transition: background-color 0.3s;
}

.file-item:hover {
  background: #e6e8eb;
}

.file-icon {
  font-size: 20px;
  margin-right: 12px;
}

.file-name {
  flex: 1;
  margin-right: 16px;
  color: #303133;
}

.file-type {
  width: 80px;
  color: #606266;
  margin-right: 16px;
}

.file-size {
  width: 100px;
  color: #909399;
  text-align: right;
}

.storage-info {
  background: #ffffff;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
}

.storage-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.storage-title {
  font-size: 16px;
  font-weight: 500;
  color: #303133;
}

.storage-percentage {
  font-size: 20px;
  font-weight: 600;
  color: #409EFF;
}

.storage-progress {
  height: 8px;
  background: #E6E8EB;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 16px;
}

.progress-bar {
  height: 100%;
  background: linear-gradient(90deg, #409EFF 0%, #36D1DC 100%);
  border-radius: 4px;
  transition: width 0.3s ease;
}

.storage-details {
  display: flex;
  justify-content: space-between;
}

.storage-used,
.storage-total {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.label {
  font-size: 14px;
  color: #909399;
}

.value {
  font-size: 16px;
  font-weight: 500;
  color: #303133;
}

.file-statistics {
  background: #ffffff;
  border-radius: 12px;
  padding: 24px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
}

.stats-header {
  margin-bottom: 20px;
}

.total-files {
  text-align: center;
}

.total-number {
  font-size: 36px;
  font-weight: 600;
  color: #409EFF;
  display: block;
}

.total-label {
  font-size: 14px;
  color: #909399;
  margin-top: 4px;
}

.type-divider {
  height: 1px;
  background: #EBEEF5;
  margin: 20px 0;
}

.file-types-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px;
}

.file-type-item {
  display: flex;
  align-items: center;
  padding: 12px;
  background: #F5F7FA;
  border-radius: 8px;
  transition: all 0.3s ease;
}

.file-type-item:hover {
  background: #ECF5FF;
  transform: translateY(-2px);
}

.type-icon {
  font-size: 24px;
  margin-right: 12px;
}

.type-info {
  display: flex;
  flex-direction: column;
}

.type-name {
  font-size: 14px;
  font-weight: 500;
  color: #303133;
}

.type-count {
  font-size: 12px;
  color: #909399;
  margin-top: 2px;
}

.no-data {
  text-align: center;
  color: #909399;
  padding: 20px;
  font-size: 14px;
}
</style>
