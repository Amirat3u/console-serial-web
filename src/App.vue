<template>
  <div id="app">
    <h1>Console Série Web</h1>

    <textarea v-model="receivedData" rows="10" cols="50" readonly></textarea>

    <div>
      <input v-model="message" placeholder="Entrez une commande..." />
      <button @click="sendMessage">Envoyer</button>
    </div>
  </div>
</template>

<script>
import { io } from "socket.io-client";

export default {
  data() {
    return {
      socket: null,
      message: '',
      receivedData: ''
    };
  },
  mounted() {
    // Connexion au backend WebSocket
    this.socket = io('http://localhost:3000');

    // Réception des messages simulés
    this.socket.on('serial-data', (msg) => {
      this.receivedData += '\n' + msg;
    });
  },
  methods: {
    sendMessage() {
      if (this.message.trim() !== '') {
        this.socket.emit('send-message', this.message);
        this.receivedData += '\n> ' + this.message;
        this.message = '';
      }
    }
  }
};
</script>

<style>
#app {
  font-family: Arial, sans-serif;
  padding: 2rem;
}
textarea {
  width: 100%;
  margin-top: 1rem;
}
input {
  margin-top: 1rem;
  width: 70%;
  padding: 0.5rem;
}
button {
  padding: 0.5rem 1rem;
  margin-left: 1rem;
}
</style>
