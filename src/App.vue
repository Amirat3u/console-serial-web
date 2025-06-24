<template>
  <div id="app">
    <h1>Console Série Web — ESP8266</h1>

    <div class="config">
      <label>Port :
        <select v-model="selectedPort">
          <option v-for="port in ports" :key="port" :value="port">{{ port }}</option>
        </select>
      </label>
      <label>Baud :
        <select v-model="selectedBaud">
          <option v-for="baud in bauds" :key="baud" :value="baud">{{ baud }}</option>
        </select>
      </label>
      <button @click="connectPort">Connecter</button>
      <span class="status" :class="isConnected ? 'ok' : 'nok'">
        {{ isConnected ? '✅ Connecté' : '❌ Déconnecté' }}
      </span>
    </div>

    <div v-if="isConnected" class="console" ref="consoleContainer">
      <div v-for="(line, index) in messages" :key="index" :class="[line.type, getMessageClass(line.text)]">
        <span v-if="showTime">{{ line.time }} </span>{{ line.text }}
      </div>
    </div>

    <div v-if="isConnected">
      <label>
        <input type="checkbox" v-model="showTime" />
        Afficher l'heure
      </label>

      <div class="actions">
        <button @click="clearConsole">Effacer</button>
        <button @click="copyConsole">Copier</button>
        <button @click="exportConsole">Exporter</button>
        <button @click="sendStatus">Status</button>
      </div>

      <div class="input-line">
        <input v-model="command" @keyup.enter="sendCommand" placeholder="Entrez une commande..." />
        <button @click="sendCommand">Envoyer</button>
      </div>
    </div>
  </div>
</template>

<script>
import { io } from "socket.io-client";
import axios from "axios";

export default {
  data() {
    return {
      socket: null,
      messages: [],
      command: "",
      showTime: false,
      ports: [],
      selectedPort: "",
      bauds: [9600, 115200, 57600, 19200],
      selectedBaud: 115200,
      isConnected: false
    };
  },
  mounted() {
    this.fetchPorts();
    this.socket = io("http://localhost:3000");

    this.socket.on("serial-data", (msg) => {
      this.addMessage(msg, "in");
    });

    this.socket.on("connect", () => {
      console.log("✅ WebSocket connecté");
    });

    this.socket.on("disconnect", () => {
      this.isConnected = false;
      this.addMessage("Déconnecté du serveur", "error");
    });

    this.socket.on("connect_error", (err) => {
      this.isConnected = false;
      this.addMessage("Erreur WebSocket : " + err.message, "error");
    });
  },
  methods: {
    fetchPorts() {
      axios.get("http://localhost:3000/ports")
        .then(response => {
          this.ports = response.data;
          if (this.ports.length > 0) this.selectedPort = this.ports[0];
        })
        .catch(() => this.addMessage("Impossible de récupérer les ports", "error"));
    },
    connectPort() {
      if (!this.selectedPort || !this.selectedBaud) return;
      axios.post("http://localhost:3000/connect", {
        port: this.selectedPort,
        baud: this.selectedBaud
      })
        .then(() => {
          this.isConnected = true;
          this.addMessage(`Connecté à ${this.selectedPort} (${this.selectedBaud} bauds)`, "out");
        })
        .catch(() => {
          this.isConnected = false;
          this.addMessage("Erreur de connexion au port série", "error");
        });
    },
    currentTime() {
      return new Date().toLocaleTimeString();
    },
    addMessage(text, type = "in") {
      this.messages.push({ text, type, time: this.currentTime() });
      this.$nextTick(() => {
        const el = this.$refs.consoleContainer;
        if (el) el.scrollTop = el.scrollHeight;
      });
    },
    getMessageClass(text) {
      if (/temp/i.test(text)) {
        const match = text.match(/([0-9]+\.[0-9]+|[0-9]+) ?°C/i);
        if (match) {
          const temp = parseFloat(match[1]);
          if (temp >= 35) return "temp-high";
          if (temp >= 30) return "temp-medium";
          return "temp-normal";
        }
      }
      return "";
    },
    sendCommand() {
      if (this.command.trim() !== "") {
        this.socket.emit("send-message", this.command);
        this.addMessage("> " + this.command, "out");
        this.command = "";
      }
    },
    sendStatus() {
      this.command = "STATUS";
      this.sendCommand();
    },
    clearConsole() {
      this.messages = [];
    },
    copyConsole() {
      const text = this.messages.map(m => m.text).join("\n");
      navigator.clipboard.writeText(text);
      alert("Console copiée !");
    },
    exportConsole() {
      const blob = new Blob([this.messages.map(m => m.text).join("\n")], { type: "text/plain" });
      const link = document.createElement("a");
      link.href = URL.createObjectURL(blob);
      link.download = "console.txt";
      link.click();
    }
  }
};
</script>

<style>
#app {
  font-family: sans-serif;
  padding: 2rem;
  max-width: 700px;
  margin: auto;
}
.config, .input-line, .actions {
  margin: 1rem 0;
}
.status.ok {
  color: green;
  margin-left: 1rem;
}
.status.nok {
  color: red;
  margin-left: 1rem;
}
.console {
  background: #1e1e1e;
  color: #eee;
  height: 250px;
  overflow-y: auto;
  padding: 1rem;
  margin: 1rem 0;
  border: 1px solid #ccc;
  border-radius: 5px;
  font-family: monospace;
}
.in { color: #80ff80; }
.out { color: #80c0ff; }
.error { color: #ff8080; }
.temp-normal { color: #80ff80; }
.temp-medium { color: orange; }
.temp-high { color: red; }
input[type="text"], select {
  padding: 0.5rem;
  margin-right: 0.5rem;
}
button {
  padding: 0.5rem 1rem;
  margin-right: 0.5rem;
  cursor: pointer;
}
</style>
