<template>
  <div class="confirm">
    <h1>{{status}}</h1>
    <h2>{{bleStatus}}</h2>
  </div>
</template>

<script>
export default {
  data () {
    return {
      status: '',
      USER_SERVICE_UUID: process.env.VUE_APP_USER_SERVICE_UUID,
      LED_CHARACTERISTIC_UUID: 'E9062E71-9E62-4BC6-B0D3-35CDCD9B027B', /* トライアルは固定値 */
      bleConnect: false,
      bleStatus: 'デバイス未接続',
      characteristic: '',
      code: ''
    }
  },
  methods: {
    confirmAction: function () {
      if (!this.$route.query.transactionId){
        throw new Error("Transaction Id not found.")
      }

      // Retrieve the reservation from database.
      var reservation = JSON.parse(sessionStorage.getItem(this.$route.query.transactionId))
      if (!reservation){
          throw new Error("Reservation not found.")
      }

      var params = new URLSearchParams()
      params.set('type', 'confirm')
      params.set('transactionId', this.$route.query.transactionId)
      params.set('reservations', JSON.stringify(reservation))

      axios.post(process.env.VUE_APP_LINE_PAY_BASE_URL + '/.netlify/functions/pay', params)
        .then(response => {
          sessionStorage.clear()
          this.status = '決済完了しました！'
      
          this.code = '1234'
          const ch_array = this.code.split("");
          for(let i = 0; i < 16; i = i + 1) {
            ch_array[i] = (new TextEncoder('ascii')).encode(ch_array[i]);
          }

          this.characteristic.writeValue(new Uint8Array(ch_array)
          ).catch(error => {
            this.bleStatus = error.message
          })

        })
    },
    // BLEが接続できる状態になるまでリトライ
    liffCheckAvailablityAndDo: async function (callbackIfAvailable) {
      try {
        const isAvailable = await liff.bluetooth.getAvailability();
        if (isAvailable) {
          callbackIfAvailable()
        } else {
          // リトライ
          this.bleStatus = `Bluetoothをオンにしてください。`
          setTimeout(() => this.liffCheckAvailablityAndDo(callbackIfAvailable), 10000)
        }
      } catch (error) {
        this.bleStatus = `Bluetoothをオンにしてください。`
      }
    },
    // サービスとキャラクタリスティックにアクセス
    liffRequestDevice: async function () {
      const device = await liff.bluetooth.requestDevice()
      await device.gatt.connect()
      const service = await device.gatt.getPrimaryService(this.USER_SERVICE_UUID)
      service.getCharacteristic(this.LED_CHARACTERISTIC_UUID).then(characteristic => {
        this.characteristic = characteristic
        this.bleConnect = true
        this.bleStatus = `デバイスに接続しました！`
        this.confirmAction()
      }).catch(error => {
        this.bleConnect = true
        this.bleStatus = `デバイス接続に失敗=${error.message}`
      })
    },
    initializeLiff: async function(){
      await liff.initPlugins(['bluetooth']);
      this.liffCheckAvailablityAndDo(() => this.liffRequestDevice())
    }
  },
  mounted: function () {
    liff.init(
        () => this.initializeLiff()
    )
  }
}
</script>