<template>
  <div class="section">
    <div class="container">
      <div class="columns has-text-centered">
        <div class="column">Ready to share your code with the world?</div>
        <div class="column">
          <button class="button btn-secondary" @click="createRoom">Create Room</button>
        </div>
      </div>
      <div v-if="rooms.length > 0">
        <h4 class="title is-4 has-text-white has-text-centered">Your Rooms</h4>
        <table class="table">
          <tr v-for="(room, rIndex) in rooms" :key="room.id">
            <td>{{ rIndex + 1 }}</td>
            <td class="room-title">
              <router-link :to="{name: 'room', params: {roomId: room.id}}">{{ room.name }}</router-link>
            </td>
            <td>{{ room.language ? room.language : 'NA' }}</td>
          </tr>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'

export default {
  data() {
    return {
      rooms: []
    }
  },
  computed: {
    ...mapGetters(['currentUser'])
  },
  async created() {
    if (!this.currentUser.verified) {
      this.$router.push({ name: 'verifyuser' })
    } else {
      const response = await this.$http.get('rooms')
      this.rooms = response.data.rooms
    }
  },
  methods: {
    async createRoom() {
      const response = await this.$http.post('rooms')
      this.$router.push({
        name: 'room',
        params: { roomId: response.data.room.id }
      })
    }
  }
}
</script>

<style lang="scss" scoped>
td {
  text-transform: capitalize;
}
td.room-title {
  width: 300px;
  overflow: hidden;
  text-overflow: ellipsis;
}
</style>
