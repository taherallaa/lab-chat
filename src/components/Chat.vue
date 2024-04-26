<template>
  <div class="chat">
    <div class="left-panel">
      <user v-for="user in users" :key="user.userID" :user="user" :selected="selectedUser === user"
        @select="onSelectUser(user)" />
    </div>
    <message-panel v-if="selectedUser" :user="selectedUser" @input="onMessage" class="right-panel" />
  </div>
</template>

<script>
import socket from "../socket";
import User from "./User";
import MessagePanel from "./MessagePanel";

export default {
  name: "Chat",
  components: { User, MessagePanel },
  data() {
    return {
      selectedUser: null,
      users: [],
    };
  },
  methods: {
    onMessage(content) {
      if (this.selectedUser) {
        socket.emit("private message", {
          content,
          to: this.selectedUser.userID,
        });
        this.selectedUser.messages.push({
          content,
          fromSelf: true,
        });
      }
    },
    onSelectUser(user) {
      this.selectedUser = user;
      user.hasNewMessages = false;
    },
  },
  created() {
    socket.on("connect", () => {
      this.users.forEach((user) => {
        if (user.self) {
          user.connected = true;
        }
      });
    });

    socket.on("disconnect", () => {
      this.users.forEach((user) => {
        if (user.self) {
          user.connected = false;
        }
      });
    });

    const initReactiveProperties = (user) => {
      user.connected = true;
      user.messages = [];
      user.hasNewMessages = false;
    };

    socket.on("users", (users) => {
      users.forEach((user) => {
        user.self = user.userID === socket.id;
        initReactiveProperties(user);
      });
      // put the current user first, and sort by username
      this.users = users.sort((a, b) => {
        if (a.self) return -1;
        if (b.self) return 1;
        if (a.username < b.username) return -1;
        return a.username > b.username ? 1 : 0;
      });
    });

    socket.on("user connected", (user) => {
      initReactiveProperties(user);
      this.users.push(user);
    });

    socket.on("user disconnected", (id) => {
      for (let i = 0; i < this.users.length; i++) {
        const user = this.users[i];
        if (user.userID === id) {
          user.connected = false;
          break;
        }
      }
    });

    socket.on("private message", ({ content, from }) => {
      for (let i = 0; i < this.users.length; i++) {
        const user = this.users[i];
        if (user.userID === from) {
          user.messages.push({
            content,
            fromSelf: false,
          });
          if (user !== this.selectedUser) {
            user.hasNewMessages = true;
          }
          break;
        }
      }
    });
  },
  destroyed() {
    socket.off("connect");
    socket.off("disconnect");
    socket.off("users");
    socket.off("user connected");
    socket.off("user disconnected");
    socket.off("private message");
  },
  mounted() {
    socket.on("disconnect", () => {
      const errorBox = document.createElement("span");
      errorBox.textContent = "Server is down, Will return soon";
      errorBox.classList.add("errorBox");
      errorBox.style.backgroundColor = "red";
      errorBox.style.color = "white";
      errorBox.style.position = 'absolute';
      errorBox.style.top = '28px';
      errorBox.style.right = '20px';
      errorBox.style.zIndex = '9999';
      errorBox.style.padding = '7px 10px'
      errorBox.style.borderRadius = '4px';
      errorBox.style.width = 'fit-content';
      document.body.append(errorBox);
      setTimeout(function () {
        errorBox.remove();
      }, 5000);
    })
  }
};
</script>

<style scoped>
.chat {
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: row;
  align-items: flex-start;
  justify-content: space-between;
  position: relative;
}

.slide-out {
  animation: slideOut 1s forwards;
}

@keyframes slideOut {
  0% {
    opacity: 1;
    transform: translateX(0);
  }

  100% {
    opacity: 0;
    transform: translateX(100%);
  }
}

.left-panel {
  width: 260px;
  height: 95%;
  overflow-x: hidden;
  background-color: #3f0e40;
  background-color: #192837;
  color: white;
  margin: auto 5px;
  border-radius: 8px;
  border: 2px solid black;
}

.right-panel {
  width: 100%;
  height: 95%;
  margin: auto 5px;
  border-radius: 8px;
  border: 2px solid black;
  background-color: #d0dee8;
}
</style>
