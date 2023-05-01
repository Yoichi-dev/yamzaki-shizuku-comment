<template>
  <div id="comment-area">
    <div v-for="(commentItem, idx) in commentList" :key="idx" class="comment-box">
      <img :src="commentItem.avater" :alt="commentItem.name">
      <div v-if="commentItem.stamp">
        <div class="name">
          {{ commentItem.name }}
        </div>
        <img class="stamp" :src="commentItem.comment" :alt="commentItem.name">
      </div>
      <div v-else>
        <div class="name">
          {{ commentItem.name }}
        </div>
        <div class="comment">
          <span>{{ commentItem.comment }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import constants from '~/constants'

export default {
  name: 'GiftPage',
  data() {
    return {
      status: null,
      commentList: []
    }
  },
  mounted() {
    (async () => {
      await axios
        .post(constants.url.showroom_api, {
          category: 'live',
          type: 'gift_list',
          key: constants.room_id,
        })
        .then((res) => {
          const { data } = res
          if ('errors' in data) {
            this.giftList = JSON.parse(localStorage.giftList)
          } else {
            this.giftList = data
            localStorage.giftList = JSON.stringify(data)
          }
        })
        .catch((e) => {
          this.giftList = JSON.parse(localStorage.giftList)
        })
      await axios
        .post(constants.url.showroom_api, {
          category: 'room',
          type: 'status',
          key: constants.room_url_key,
        })
        .then((res) => {
          const { data } = res
          if ('errors' in data) {
            if ('redirect_url' in data.errors[0]) {
              this.premiumLive()
            }
          } else {
            this.status = data
            this.connect()
          }
        })
      await axios
        .post(constants.url.showroom_api, {
          category: 'live',
          type: 'comment_log',
          key: constants.room_id,
        })
        .then((res) => {
          const { data } = res
          for (let i = data.comment_log.length - 1; i >= 0; i--) {
            const commentObj = {
              ac: data.comment_log[i].name,
              av: data.comment_log[i].avatar_id,
              cm: data.comment_log[i].comment,
              created_at: data.comment_log[i].created_at,
              u: data.comment_log[i].user_id,
              ua: data.comment_log[i].ua,
            }
            if (this.commentCountCheck(commentObj)) {
              const editName = this.nameCut(commentObj.ac) + "先生"
              const stamp = constants.key_word.find(item => item.key === commentObj.cm)
              if (stamp) {
                this.commentList.push({
                  name: editName,
                  avater: `https://image.showroom-cdn.com/showroom-prod/image/avatar/${commentObj.av}.png`,
                  comment: require(`~/assets/img/stamp/${stamp.img}.png`),
                  stamp: true
                })
              } else {
                this.commentList.push({
                  name: editName,
                  avater: `https://image.showroom-cdn.com/showroom-prod/image/avatar/${commentObj.av}.png`,
                  comment: commentObj.cm,
                  stamp: false
                })
              }
            }
          }
          setTimeout(() => {
            document.getElementById('comment-area').scrollTop = document.getElementById('comment-area').scrollHeight
          }, 500)
        })
        .catch((e) => {
          // console.log(e)
        })
    })();
  },
  methods: {
    premiumLive() {
      const ping = setInterval(() => {
        axios
          .post(constants.url.showroom_api, {
            category: 'live',
            type: 'onlives',
            key: new Date().getTime(),
          })
          .then((res) => {
            const { data } = res
            const premiumList = []
            for (let i = 0; i < data.onlives.length; i++) {
              if (data.onlives[i].genre_id >= 100 && data.onlives[i].genre_id <= 200) {
                const check = data.onlives[i].lives.find(
                  (e) => e.premium_room_type === 1
                )
                if (check !== undefined) {
                  premiumList.push(check)
                }
              }
            }
            if (premiumList.length !== 0) {
              for (const room of premiumList) {
                if (room.room_id === constants.room_id) {
                  this.status = {
                    broadcast_key: room.bcsvr_key
                  }
                  clearInterval(ping)
                  this.connect()
                  break
                }
              }
            }
          })
      }, 5000)
    },
    connect() {
      const socket = new WebSocket(constants.ws)
      socket.onopen = (e) => {
        socket.send(`SUB\t${this.status.broadcast_key}`)
      }
      const ping = setInterval(() => {
        socket.send('PING\tshowroom')
      }, 60000)
      socket.onerror = (e) => {
        socket.close()
        clearInterval(ping)
        location.reload()
      }
      socket.onmessage = (message) => {
        const { data } = message
        if (data === 'ACK\tshowroom' || data === 'Could not decode a text frame as UTF-8.') {
          return
        }
        if (data === 'ERR') {
          socket.close()
          clearInterval(ping)
          location.reload()
          return
        }
        this.messageCheck(socket, data, ping)
      }
    },
    messageCheck(socket, message, ping) {
      const msgJson = JSON.parse(message.split(`MSG\t${this.status.broadcast_key}`)[1])
      switch (Number(msgJson.t)) {
        case 1:
          this.getComment(msgJson)
          break
        case 2:
          // this.fallGift(msgJson, this.giftUrlCheck(msgJson))
          break
        case 101:
          socket.close()
          clearInterval(ping)
          location.reload()
          break
        case 104:
          socket.close()
          clearInterval(ping)
          location.reload()
          break
        default:
        // console.log(msgJson)
      }
    },
    getComment(msgJson) {
      if (this.commentCountCheck(msgJson)) {
        const editName = this.nameCut(msgJson.ac) + "先生"
        const stamp = constants.key_word.find(item => item.key === msgJson.cm)
        if (stamp) {
          this.commentList.push({
            name: editName,
            avater: `https://image.showroom-cdn.com/showroom-prod/image/avatar/${msgJson.av}.png`,
            comment: require(`~/assets/img/stamp/${stamp.img}.png`),
            stamp: true
          })
        } else {
          this.commentList.push({
            name: editName,
            avater: `https://image.showroom-cdn.com/showroom-prod/image/avatar/${msgJson.av}.png`,
            comment: msgJson.cm,
            stamp: false
          })
        }
        setTimeout(() => {
          document.getElementById('comment-area').scrollTop = document.getElementById('comment-area').scrollHeight
        }, 500)
      }
    },
    nameCut(name) {
      if (name.length <= 10) {
        return name
      }

      let editName = name

      if (editName.includes('@')) {
        editName = editName.substring(0, editName.indexOf('@'))
      }

      if (editName.includes('＠')) {
        editName = editName.substring(0, editName.indexOf('＠'))
      }

      if (editName.includes('◎')) {
        editName = editName.substring(0, editName.indexOf('◎'))
      }

      if (editName.includes('○')) {
        editName = editName.substring(0, editName.indexOf('○'))
      }

      if (editName.includes('〇')) {
        editName = editName.substring(0, editName.indexOf('〇'))
      }

      if (editName.includes('#')) {
        editName = editName.substring(0, editName.indexOf('#'))
      }

      if (editName.includes('＃')) {
        editName = editName.substring(0, editName.indexOf('＃'))
      }

      if (editName.includes('、')) {
        editName = editName.substring(0, editName.indexOf('、'))
      }

      if (editName.includes('【')) {
        editName = editName.substring(0, editName.indexOf('【'))
      }

      if (editName === '') {
        editName = name
      }

      return editName
    },
    commentCountCheck(msgJson) {
      if (msgJson.cm.includes('.')) {
        return true
      }
      const numberFormat = msgJson.cm.replace(/[０-９]/g, (s) => {
        return String.fromCharCode(s.charCodeAt(0) - 0xfee0)
      })
      if (
        !isNaN(numberFormat) &&
        Number(numberFormat) >= 0 &&
        Number(numberFormat) <= 50
      ) {
        return false
      } else {
        return true
      }
    }
  }
}
</script>

<style>
#comment-area {
  max-height: 100vh;
  overflow: scroll;
  color: white;
}

#comment-area::-webkit-scrollbar {
  display: none;
}

.comment-box {
  border-bottom: 2px rgba(255, 255, 255, 1) dashed;
  display: flex;
  margin-top: 10px;
  margin-bottom: 10px;
}

.comment {
  margin-top: 5px;
  font-size: 1.5em;
}

img {
  float: left;
  width: 50px;
  height: 50px;
  margin: 10px;
}

.stamp {
  width: 200px;
  height: 200px;
}
</style>