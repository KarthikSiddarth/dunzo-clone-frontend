<template>
  <div class="components-after-login">
    <user-menu v-if="showMenu" v-on:hideMenu="hideMenu"/>
    <a href="" v-on:click.prevent="renderMenu" class="menu__hamburger-icon">&#9776;</a>
    <router-view
    class="component"
    v-on:coords="getCoords"
    v-bind:pickUpLocation="pickUpLocation"
    v-bind:dropLocation="dropLocation"
    v-bind:socket="socket"/>
  </div>
</template>

<script>
import userMenu from '../../components/menu/userMenu.vue'

export default {
  components: {
    userMenu
  },
  data() {
    return {
      pickUpLocation: null, 
      dropLocation: null,
      showMenu: false,
      socket: '',
      urlMapsAPI: 'https://maps.googleapis.com/maps/api/js?key=AIzaSyCyrqmHGMWIf_a_EmXRsFi_3KWTr2koaBU&libraries=places',
      urlSocketio: 'https://dunzoclone.now.sh/socket.io/socket.io.js',
      intervalId: null,
      showSideBar: false,
    }
  },
  methods: {
    hideMenu () {
      this.showMenu = false
    },
    getCoords (data) {
      console.log('got coordinates')
      if (data[0] === 'provideDropAddress') {
        this.dropLocation = data[1]
      } else if (data[0] === 'providePickUpAddress') {
        this.pickUpLocation = data[1]
      }
    },
    renderMenu () {
      this.showMenu = true
    },
    async getPublicVapidKey () {
      let url = 'https://dunzoclone.now.sh/publicVapidKey'
      let publicVapidKey = (await (await fetch(url)).json())
      return publicVapidKey
    },
    async initiateServiceWorker () {
      try {
        let registeredSW = await navigator.serviceWorker.getRegistration()
        if (!registeredSW) {
          let registeredSW = await navigator.serviceWorker.register('userSW.js', { scope: '/' })
        }
        let subscriptionObj = await registeredSW.pushManager.getSubscription()
        let isNewSubscription = !subscriptionObj
        if (!subscriptionObj) {
          subscriptionObj = await registeredSW.pushManager.subscribe({
            userVisibleOnly: true,
            applicationServerKey: await this.getPublicVapidKey()
          })
        }
        return isNewSubscription ? subscriptionObj : false
      } catch (e) {
        console.log(e)
      }
    },
    async subscribePushNotification (subscriptionObj) {
      await fetch('https://dunzoclone.now.sh/subscribe', {
        mode: 'cors',
        method: 'post',
        body: JSON.stringify(subscriptionObj),
        headers: {
          'content-type': 'application/json',
          'authorization': document.cookie.split(';').map(e=>e.trim()).filter(e=>e.startsWith('access_token='))[0].substring(13)
        }
      })
    },
    loadScript (url) {
      let script = document.createElement('script')
      script.setAttribute('src', url)
      script.setAttribute('async', true)
      script.setAttribute('defer', true)
      document.head.appendChild(script)
    },
    checkForSocket () {
      if (io) {
        this.socket = io('https://dunzoclone.now.sh/')
        clearInterval(this.intervalId)
      }
    }
  },
  async mounted() {
    this.loadScript(this.urlMapsAPI)
    this.loadScript(this.urlSocketio)
    this.intervalId = setInterval(this.checkForSocket, 3000)
    if (navigator.serviceWorker) {
      let subscriptionObj = await this.initiateServiceWorker()
      if (subscriptionObj) {
        this.subscribePushNotification(subscriptionObj)
      }
    }
  }
}
</script>

<style>
.components-after-login {
  height: inherit;
  width: inherit;
}

.menu__hamburger-icon {
  height: 5%;
  display: block;
  font-size: 3em;
  padding-left: 0.5em;
  text-decoration: none;
  color: white;
  outline: 0;
}

.component {
  height: 95%;
}
</style>
