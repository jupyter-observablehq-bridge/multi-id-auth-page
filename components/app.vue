<template>
  <div>
    <v-app-bar app color="blue accent-3" dark>
      <v-spacer></v-spacer>
      <v-toolbar-title>Multi ID Provider Auth Page</v-toolbar-title>
      <v-spacer></v-spacer>
      <a href="https://github.com/oscar6echo/TBD" target="_blank">
        <v-icon>mdi-github-circle</v-icon>
      </a>
    </v-app-bar>

    <v-content>
      <v-container fluid>
        <v-row class="d-flex justify-center mt-5 mb-2">
          <div
            class="text-center font-italic font-weight-normal"
          >Do not close this page for the parent window to get updated when tokens refresh</div>
        </v-row>

        <v-row class="d-flex justify-center mt-2 mb-5">
          <table>
            <tr>
              <td class="text-left font-weight-light tleft">Parent Window</td>
              <td class="text-left font-weight-bold tright">{{parentWin}}</td>
            </tr>
            <tr>
              <td class="text-left font-weight-light tleft">Last Update Sent</td>
              <td id="time" class="text-left font-weight-bold tright">{{lastMsgTime}}</td>
            </tr>
          </table>
        </v-row>

        <div>
          <v-row no-gutters>
            <v-col cols="1"></v-col>
            <v-col cols="5">
              <google class="mr-1" :scope-in="scope.google" @update="updateCreds" />
            </v-col>
            <v-col cols="5">
              <sgconnect :scope-in="scope.sgconnect" @update="updateCreds" />>
            </v-col>
          </v-row>
        </div>
      </v-container>
    </v-content>
  </div>
</template>

<script>
const Google = httpVueLoader("./google.vue");
const Sgconnect = httpVueLoader("./sgconnect.vue");

module.exports = {
  name: "App",
  components: {
    Google,
    Sgconnect
  },
  data() {
    return {
      parentWin: "None",
      params: {},
      creds: {},
      lastMsgTime: null,
      scope: {}
    };
  },
  computed: {},
  mounted() {
    this.readParams();
    if (window.opener) this.setListener();
  },
  methods: {
    setListener() {
      window.opener.postMessage("url", "*");

      const listener = window.addEventListener("message", evt => {
        // console.log(evt);
        const { protocol, host } = window.location;
        if (event.origin === protocol + "//" + host) return;
        // console.log("candidate event");
        // console.log(evt.data);
        if (evt.data && evt.data.url) this.parentWin = evt.data.url;
        removeEventListener("message", listener);
      });
    },

    updateCreds(data) {
      console.log("-- updateCreds");
      //   console.log(data);
      this.creds[data.provider] = data.creds;
      this.sendCreds();
    },

    sendCreds() {
      console.log("-- sendCreds");
      console.log(this.creds);
      this.lastMsgTime = new Date().toLocaleTimeString("fr-FR");
      if (!window.opener){
          console.log('******** WARNING')
          console.log('THIS PAGE MUST BE OPENED FROM ANOTHER ONE')
          console.log('SEE DOCUMENTATION http://TBD')
          return
      }
      window.opener.postMessage({ updateCreds: this.creds }, "*");
    },

    readParams() {
      const urlParams = new URLSearchParams(window.location.search);
      this.scope = {
        google: urlParams.get("google-scope"),
        sgconnect: urlParams.get("sgconnect-scope")
      };
    }
  }
};
</script>

<style scoped>
.table {
  table-layout: fixed;
}

.tleft {
  width: 150px;
}
.tright {
  width: 200px;
  max-width: 550px;
  word-wrap: break-word;
}
</style>

<style >
.my-wrapper {
  display: flex;
  flex-direction: column;
  width: 100%;
  border: 1px solid lightgrey;
  height: 750px;
}

.wrapper-btn {
  height: 200px;
}
</style>