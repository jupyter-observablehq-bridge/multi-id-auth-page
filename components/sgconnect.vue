<template>
  <div>
    <div class="pa-2 my-wrapper">
      <div class="title font-light-regular ma-5">SG Connect</div>

      <div class="wrapper-btn">
        <v-btn id="my-btn" class="pa-1 ma-5" color="black" tile outlined @click="onClick()">
          <img id="logo" src="components/assets/logo-sg.png" />
          <div class="font-weight-normal">{{signed ? 'Signed In':'Sign In'}}</div>
        </v-btn>
      </div>

      <v-text-field outlined label="Scope" class="ma-5" :value="scope"></v-text-field>

      <v-alert
        type="warning"
        v-if="!scopeOk"
      >Granted scopes do not include all requested scopes - Check your client ID</v-alert>

      <div class="body-2 font-italic ml-1 mt-5 mb-0 my-comment">Open console for full credentials</div>
      <json-display class="mt-1" :obj="creds" />
    </div>
  </div>
</template>

<script>
const JsonDisplay = httpVueLoader("./json-display.vue");

module.exports = {
  name: "SGConnect",
  props: {
    scopeIn: {
      required: true,
      validator: p => typeof p === "string" || p === null
    }
  },
  components: {
    JsonDisplay
  },

  data() {
    return {
      provider: "sgconnect",
      client_id: "a17bb668-bd3e-4fc1-9006-19f69e6484f6",
      scope: "openid profile mail",
      creds: null,
      timer: null,
      expires_at: null,
      signed: false,
      status: "ok",
      proxy: true,
      scopeOk: true
    };
  },

  watch: {},

  created() {},

  mounted() {
    if (this.scopeIn) this.scope += " " + this.scopeIn;
    this.setupAuth();

    window.sgc = this;
  },

  methods: {
    async onClick() {
      this.startAuth();
    },

    setupAuth() {
      this.sgwtConnect = window.setupSGWTConnect({
        authorization_endpoint: "https://sso.sgmarkets.com/sgconnect",
        redirect_uri: window.location.href.includes("localhost")
          ? window.location.origin
          : window.location.href,
        client_id: this.client_id,
        scope: this.scope
      });

      this.sgwtConnect.on("authorizationDiscarded", () => {
        console.log("--- EVENT authorizationDiscarded");
      });

      this.sgwtConnect.on("authorizationExpired", () => {
        console.log("--- EVENT authorizationExpired");
      });

      this.sgwtConnect.on("renewAuthorizationError", e => {
        console.log("--- EVENT renewAuthorizationError:" + e);
      });

      this.sgwtConnect.on("renewAuthorizationSuccess", () => {
        console.log("--- EVENT renewAuthorizationSuccess");
      });

      if (this.sgwtConnect.isAuthorized()) {
        this.signed = true;
        this.refresh();
      }
    },

    startAuth() {
      if (!this.sgwtConnect.isAuthorized()) {
        this.sgwtConnect.requestAuthorization();
      } else {
        this.signed = true;
        this.refresh();
      }
    },

    async refresh() {
      this.creds = await this.buildCreds();
      this.checkScope();
      this.sendCreds();
    },

    checkScope() {
      const scopesIn = this.scope.split(" ").map(e => e.trim());
      const scopesOut = this.creds.scope;
      let ok = true;
      for (const e of scopesIn) if (!scopesOut.includes(e)) ok = false;
      this.scopeOk = ok;
    },

    async buildCreds() {
      const token = this.readToken();
      if (!token) {
        this.status = "error";
        return null;
      }
      this.creds = {
        access_token: token,
        token_info: "requesting...",
        user_info: "requesting..."
      };
      await this.updateTokenInfo(token);
      await this.updateUserInfo(token);

      const creds = {
        name: this.userInfo.name,
        email: this.userInfo.sub,
        access_token: this.tokenInfo.access_token,
        scope: this.tokenInfo.scope,
        first_issued_at: this.first_issued_at,
        expires_in: this.expires_in,
        expires_at: this.expires_at
      };
      return creds;
    },

    sendCreds(creds) {
      if (this.creds)
        this.$emit("update", { provider: this.provider, creds: this.creds });
    },

    readToken() {
      const header = this.sgwtConnect.getAuthorizationHeader();
      if (!header || header === "") return;
      const token = header.split(" ")[1];
      console.log(token);
      return token;
    },

    async updateTokenInfo(token) {
      const data = await (this.proxy
        ? this.makeRequestProxy("tokeninfo", token)
        : this.makeRequest("tokeninfo", token));

      if (data.error) {
        this.status = "error";
        this.stopTimer();
        return;
      }
      this.status = "ok";
      this.first_issued_at = new Date().getTime();
      this.expires_in = data.expires_in;
      //   this.expires_in = 2 ////////////////// DEBUG
      this.expires_at = this.first_issued_at + this.expires_in * 1000;

      this.startTimer();
      this.tokenInfo = data;
    },

    async updateUserInfo(token) {
      console.log("start updateUserInfo");
      const data = await (this.proxy
        ? this.makeRequestProxy("userinfo", token)
        : this.makeRequest("userinfo", token));
      console.log({ SGuserInfo: data });

      if (data.error) {
        this.status = "error";
        this.stopTimer();
        return;
      }
      this.status = "ok";
      this.userInfo = data;
    },

    async makeRequest(endpoint, token) {
      const url = "https://sso.sgmarkets.com/sgconnect/oauth2/" + endpoint;
      const headers = {
        authorization: "Bearer " + token
      };
      let data;
      try {
        const res = await fetch(url, { headers });
        data = await res.json();
      } catch (e) {
        if (e.response) {
          data = e.response.data;
        } else {
          data = e;
        }
      }
      return data;
    },

    async makeRequestProxy(endpoint, token) {
      const url =
        "https://europe-west2-cors-proxy-275911.cloudfunctions.net/inspect-token";
      const payload = { token, endpoint };
      console.log(payload);
      let data;
      try {
        const res = await fetch(url, {
          method: "post",
          body: JSON.stringify(payload)
        });
        console.log(res);
        data = await res.json();
        console.log(data);
      } catch (e) {
        console.log("ERROR");
        if (e.response) {
          data = e.response.data;
        } else {
          data = e;
        }
      }
      return data;
    },

    startTimer() {
      this.t0 = new Date();
      const that = this;
      this.stopTimer();
      this.timer = setInterval(() => {
        that.maturity = that.buildMaturity();
        if (that.maturity < 0) {
          that.stopTimer();
          that.refresh();
        }
      }, 1000);
    },

    stopTimer() {
      if (this.timer) clearInterval(this.timer);
    },

    buildMaturity() {
      const buffer = 30;
      const secs = (new Date().getTime() - this.t0.getTime()) / 1000;
      return (this.tokenInfo.expires_in - secs - buffer).toFixed(0);
    }
  }
};
</script>
  
<style scoped>
#my-btn {
  display: flex;
  justify-content: space-around;
  align-items: center;
  border: 1px solid darkgray;
  width: 150px;
  height: 55px;
}

#logo {
  width: 35px;
}
</style>    


