<template>
  <div class="message-input" v-if="activeRoom">
    <a-popover placement="topLeft" trigger="hover" class="message-popver">
      <template slot="content">
        <genal-emoji @addEmoji="addEmoji"></genal-emoji>
      </template>
      <div class="messagte-tool-icon1">😃</div>
    </a-popover>
    <a-upload :show-upload-list="false" :before-upload="beforeImgUpload"
      ><div class="messagte-tool-icon2"><img src="~@/assets/photo.png" class="message-file-button" alt="" /></div
    ></a-upload>
    <a-input
      autocomplete="off"
      type="text"
      placeholder="say hello..."
      v-model="text"
      ref="input"
      autoFocus
      style="color: #000"
      @pressEnter="throttle(preSendMessage)"
    />
    <img class="message-input-button" @click="throttle(preSendMessage)" src="~@/assets/send.png" alt="" />
  </div>
</template>

<script lang="ts">
import { Component, Vue, Watch } from 'vue-property-decorator';
import GenalEmoji from './GenalEmoji.vue';
import { namespace } from 'vuex-class';
const chatModule = namespace('chat');
const appModule = namespace('app');

@Component({
  components: {
    GenalEmoji,
  },
})
export default class GenalInput extends Vue {
  @appModule.Getter('user') user: User;
  @appModule.Getter('mobile') mobile: boolean;

  @chatModule.State('activeRoom') activeRoom: Group & Friend;
  @chatModule.Getter('socket') socket: SocketIOClient.Socket;
  @chatModule.Getter('dropped') dropped: boolean;
  @chatModule.Getter('groupGather') groupGather: GroupGather;
  @chatModule.Getter('userGather') userGather: FriendGather;

  text: string = '';
  lastTime: number = 0;

  mounted() {
    this.initPaste();
  }

  /**
   * 点击切换房间进入此方法
   */
  @Watch('activeRoom')
  changeActiveRoom() {
    this.$nextTick(() => {
      this.focusInput();
    });
  }

  /**
   * 监听图片粘贴事件
   */
  initPaste() {
    document.addEventListener('paste', (event) => {
      let items = event.clipboardData && event.clipboardData.items;
      let url = window.URL || window.webkitURL;
      let file = null;
      if (items && items.length) {
        // 检索剪切板items
        for (let i = 0; i < items.length; i++) {
          if (items[i].type.indexOf('image') !== -1) {
            file = items[i].getAsFile();
            break;
          }
        }
      }
      if (file) {
        this.throttle(this.handleImgUpload, file);
      }
    });
  }

  /**
   * 消息发送节流
   */
  throttle(fn: Function, file?: File) {
    let nowTime = +new Date();
    if (nowTime - this.lastTime < 200) {
      return this.$message.error('消息发送太频繁！');
    }
    fn(file);
    this.lastTime = nowTime;
  }

  /**
   * 消息发送前校验
   */
  preSendMessage() {
    if (!this.text.trim()) {
      this.$message.error('不能发送空消息!');
      return;
    }
    if (this.text.length > 220) {
      this.$message.error('消息太长!');
      return;
    }
    if (this.activeRoom.groupId) {
      this.sendMessage({ type: 'group', message: this.text, messageType: 'text' });
    } else {
      this.sendMessage({ type: 'friend', message: this.text, messageType: 'text' });
    }
    this.text = '';
  }

  /**
   * 消息发送
   */
  sendMessage(data: SendMessage) {
    console.log('sending');
    if (data.type === 'group') {
      this.socket.emit('groupMessage', {
        userId: this.user.userId,
        groupId: this.activeRoom.groupId,
        content: data.message,
        width: data.width,
        height: data.height,
        messageType: data.messageType,
      });
    } else {
      this.socket.emit('friendMessage', {
        userId: this.user.userId,
        friendId: this.activeRoom.userId,
        content: data.message,
        width: data.width,
        height: data.height,
        messageType: data.messageType,
      });
    }
  }

  /**
   * 添加emoji到input
   */
  addEmoji(emoji: string) {
    this.text += emoji;
    this.focusInput();
  }

  /**
   * focus input框
   */
  focusInput() {
    if (!this.mobile) {
      // @ts-ignore
      this.$refs.input.focus();
    }
  }

  /**
   * 计算图片的比例
   */
  getImageSize(data: ImageSize) {
    let { width, height } = data;
    if (width > 335 || height > 335) {
      if (width > height) {
        height = 335 * (height / width);
        width = 335;
      } else {
        width = 335 * (width / height);
        height = 335;
      }
    }
    return {
      width,
      height,
    };
  }

  /**
   * 图片上传校验
   * @params file
   */
  beforeImgUpload(file: File) {
    this.throttle(this.handleImgUpload, file);
    return false;
  }

  /**
   * 图片消息发送
   * @params file
   */
  async handleImgUpload(imageFile: File) {
    var image_flag = 1;
    const isJpgOrPng =
      imageFile.type === 'image/jpeg' || imageFile.type === 'image/png' || imageFile.type === 'image/jpg' || imageFile.type === 'image/gif';
    if (!isJpgOrPng) {
      image_flag = 0;
    
    }
    const isLt1M = imageFile.size / 1024 / 1024 < 20;
    if (!isLt1M) {
      return this.$message.error('图片必须小于20MB!');
    }
    if (image_flag) {
      let image = new Image();
      let url = window.URL || window.webkitURL; //创建文件流
      image.src = url.createObjectURL(imageFile); //给Image对象传入数据
      image.onload = () => {
        let imageSize: ImageSize = this.getImageSize({ width: image.width, height: image.height }); //拿到图片大小
        this.sendMessage({
          type: this.activeRoom.groupId ? 'group' : 'friend',
          message: imageFile,
          width: imageSize.width,
          height: imageSize.height,
          messageType: 'image',
        });
      };
    } else {
      // console.log('start handle file');
      // let file = new FileReader();

      this.sendMessage({
        type: this.activeRoom.groupId ? 'group' : 'friend',
        message: imageFile,
        width: 10,
        height: 10,
        messageType: 'file*' + imageFile.name,
      });
    }
  }
}
</script>
<style lang="scss" scoped>
.message-input {
  display: flex;
  flex-wrap: nowrap;
  position: absolute;
  width: 100%;
  bottom: 0px;
  input {
    height: 40px;
  }
  .message-input-button {
    width: 30px;
    cursor: pointer;
    position: absolute;
    right: 10px;
    top: 4px;
  }
  .message-file-button {
    width: 21px;
    cursor: pointer;
    position: absolute;
    left: 5px;
    top: 10.5px;
  }
}

//输入框样式
.ant-input {
  padding: 0 50px 0 60px;
}
// 消息工具样式
.messagte-tool-icon1 {
  position: absolute;
  left: 0;
  top: 0;
  width: 50px;
  height: 40px;
  text-align: left;
  line-height: 42px;
  font-size: 16px;
  cursor: pointer;
  z-index: 99;
}
.messagte-tool-icon2 {
  position: absolute;
  left: 25px;
  top: 0;
  width: 25px;
  height: 40px;
  text-align: right;
  line-height: 42px;
  font-size: 16px;
  cursor: pointer;
  z-index: 99;
}
.message-tool-item {
  width: 0px;
  height: 240px;
  cursor: pointer;
  .message-tool-contant {
    width: 50px;
    padding: 5px;
    border-radius: 5px;
    transition: all linear 0.2s;
    .message-tool-item-img {
      width: 40px;
    }
    .message-tool-item-text {
      text-align: center;
      font-size: 10px;
    }
    &:hover {
      background: rgba(135, 206, 235, 0.6);
    }
  }
}
</style>
