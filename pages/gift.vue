<template>
  <div id="gift">
  </div>
</template>

<script>
import { gsap } from 'gsap'
import axios from 'axios'
import constants from '~/constants'

export default {
  name: 'GiftPage',
  data() {
    return {
      status: null,
      giftList: null,
      width: 1850
    }
  },
  mounted() {
    this.width = window.innerWidth - 70;
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
      switch (msgJson.t) {
        case '1':
          // console.log('コメント')
          break
        case '2':
          this.fallGift(msgJson, this.giftUrlCheck(msgJson))
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
    giftUrlCheck(msgJson) {
      if (msgJson.gt === 2) {
        if (msgJson.g > 10000 && msgJson.g <= 10070) {
          return this.giftList.enquete.find(item => item.gift_id === msgJson.g).image
        } else if (msgJson.g === 1601) {
          return this.giftList.normal.find(item => item.gift_id === msgJson.g).image
        } else if (
          msgJson.g === 1 ||
          msgJson.g === 1001 ||
          msgJson.g === 1002 ||
          msgJson.g === 1003 ||
          msgJson.g === 2 ||
          msgJson.g === 1501 ||
          msgJson.g === 1502 ||
          msgJson.g === 1503 ||
          msgJson.g === 1504 ||
          msgJson.g === 1505
        ) {
          return require(`@/assets/img/gummies${Math.floor(this.getRandomNum(0, 2))}.png`)
        } else {
          return this.giftList.normal.find(item => item.gift_id === msgJson.g).image
        }
      } else {
        return this.giftList.normal.find(item => item.gift_id === msgJson.g).image
      }
    },
    fallGift(msgJson, imgSrc) {
      const elmId = Math.random().toString(32).substring(2)
      for (let i = 0; i < msgJson.n; i++) {
        const id = `gift_${msgJson.u}_${msgJson.g}_${i}_${elmId}`
        const giftImgElement = document.createElement('img')
        giftImgElement.src = imgSrc
        giftImgElement.style.width = '70px'
        giftImgElement.setAttribute('id', id)
        giftImgElement.style.position = 'absolute'
        giftImgElement.style.top = '-100px'
        giftImgElement.style.left = this.getRandomNum(10, this.width - 70) + 'px'
        document.getElementById('gift').append(giftImgElement)
        this.gsaoExe(id)
      }
    },
    gsaoExe(elementId) {
      gsap.to(`#${elementId}`, {
        duration: this.getRandomNum(6, 9),
        rotation: this.getRandomNum(90, 720),
        y: 1200,
        onComplete: () => {
          if (document.getElementById(elementId) !== null) {
            document.getElementById(elementId).remove()
          }
        },
      })
      setTimeout(() => {
        if (document.getElementById(elementId) !== null) {
          document.getElementById(elementId).remove()
        }
      }, 10000)
    },
    getRandomNum(min, max) {
      min = Math.ceil(min)
      max = Math.floor(max)
      return Math.random() * (max - min + 1) + min
    },
  }
}
</script>

<style>
body {
  margin: 0 !important;
}
</style>