<template>
  <div>
    <div class="pa-2 my-wrapper">
      <div class="title font-light-regular ma-5">Google</div>

      <div class="wrapper-btn">
        <div id="my-signin2" class="d-flex ma-5"></div>
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
  name: "Google",
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
      provider: "google",
      scope: "openid profile email",
      user: null,
      creds: null,
      timer: null,
      scopeOk: true
    };
  },

  watch: {},

  mounted() {
    this.startRenderBtn();
    // console.log(this.scopeIn); //////////
    if (this.scopeIn) this.scope += " " + this.scopeIn;

    window.goo = this;
  },

  methods: {
    async onSuccess(googleUser) {
      this.user = googleUser;
      this.creds = await this.buildCreds(true);
      this.checkScope();
      this.sendCreds();
      this.setTimer();
      //   console.log("done");
    },

    checkScope() {
      const scopesIn = this.scope.split(" ").map(e => e.trim());
      const scopesOut = this.creds.scope.split(" ").map(e => e.trim());
      let ok = true;
      for (const e of scopesIn) if (!scopesOut.includes(e)) ok = false;
      this.scopeOk = ok;
    },

    onFailure(error) {
      console.log(error);
      this.value = error;
      this.$emit("update", { provider, error });
    },

    async refreshToken() {
      //   console.log("start refreshToken");
      this.creds = await this.buildCreds();
      this.sendCreds();
      this.setTimer();
    },

    sendCreds(creds) {
      this.$emit("update", { provider: this.provider, creds: this.creds });
    },

    setTimer() {
      const maturity = this.creds.expires_at - new Date().getTime();
      const delay = maturity - 5 * 60 * 1000;
      //   const delay = 3 * 1000; /////////////////// DEBUG

      if (this.timer) clearTimeout(this.timer);
      const that = this;
      this.timer = setTimeout(() => that.refreshToken(), delay);
    },

    async buildCreds(first = false) {
      const profile = this.user.getBasicProfile();
      const name = profile.getName();
      const email = profile.getEmail();

      let auth;
      if (first) {
        auth = this.user.getAuthResponse(true);
      } else {
        auth = await this.user.reloadAuthResponse();
      }
      //   console.log(auth);
      const {
        access_token,
        expires_at,
        expires_in,
        first_issued_at,
        scope
      } = auth;
      //   console.log("Google - logged as " + name);
      const creds = {
        name,
        email,
        access_token,
        scope,
        first_issued_at,
        expires_in,
        expires_at
      };
      console.log({ googleInfo: creds });
      return creds;
    },

    renderBtn() {
      window.gapi.signin2.render("my-signin2", {
        scope: this.scope,
        width: 150,
        height: 50,
        longtitle: false,
        theme: "dark",
        onsuccess: this.onSuccess,
        onfailure: this.onFailure
      });
      if (this.timerRender) clearTimeout(this.timerRender);
    },

    startRenderBtn() {
      if (window.gapi) {
        this.renderBtn();
      } else {
        const that = this;
        this.timerRender = setTimeout(() => that.startRenderBtn(), 1000);
      }
    }
  }
};
</script>
  
<style scoped>
</style>    


