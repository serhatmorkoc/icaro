<template>
  <div class="ui segment form">
    <div v-if="!authorized && !dedaloError" class="ui active centered inline text loader">
      {{
      $t("social.auth_progress") }}...
    </div>
    <div v-if="authorized" class="ui icon positive message">
      <i class="check icon"></i>
      <div class="content">
        <div class="header">{{ $t("social.auth_success") }}</div>
        <p>{{ $t("social.auth_success_sub") }}...</p>
      </div>
    </div>
    <div
      v-if="authorized && $parent.hotspot.preferences.marketing_0_reason_country == 'true' && userId != 0"
    >
      <h3>{{ $t("login.additional_info") }}</h3>
      <div class="field">
        <label>{{ $t("login.country") }}</label>
        <div class="ui big left icon input">
          <select v-model="additionalCountry">
            <option :value="c.code" v-for="c in countries" v-bind:key="c.code">{{c.name}}</option>
          </select>
        </div>
      </div>
      <div class="field">
        <label>{{ $t("login.reason") }}</label>
        <div class="ui big left icon input">
          <select v-model="additionalReason">
            <option value="business">{{$t("login.business")}}</option>
            <option value="family">{{$t("login.family")}}</option>
            <option value="other">{{$t("login.other")}}</option>
          </select>
        </div>
      </div>
    </div>
    <div v-if="dedaloError" class="ui icon negative message">
      <i class="remove icon"></i>
      <div class="content">
        <div class="header">{{ $t("social.auth_error") }}</div>
        <p>{{ $t("social.auth_error_sub") }}</p>
      </div>
    </div>
    <div
      :class="$parent.hotspot.preferences.marketing_0_reason_country == 'true' ? 'adjust-top-big' : ''"
      v-if="authorized"
    >
      <h3>{{ $t("login.disclaimer_marketing") }}</h3>
      <div class="inline field">
        <textarea readonly class="text-center" v-model="hotspot.disclaimers.marketing_use"></textarea>
      </div>
      <div class="ui inline">
        <input id="conditions" v-model="conditions" type="checkbox" class="ui checkbox field">
        <label for="conditions">{{ $t("login.disclaimer_privacy_accept") }}</label>
      </div>
      <div v-if="$parent.hotspot.preferences.marketing_1_enabled == 'true'" class="ui inline">
        <input id="surveys" v-model="surveys" type="checkbox" class="ui checkbox field">
        <label for="surveys">{{ $t("login.disclaimer_survey_accept") }}</label>
      </div>
      <button v-on:click="navigate()" class="ui big button green">{{ $t("login.navigate") }}</button>
    </div>
  </div>
</template>

<script>
import AuthMixin from "./../../mixins/auth";
import { setTimeout } from "timers";
export default {
  name: "LinkedInPage",
  mixins: [AuthMixin],
  data: function() {
    var authorized = false;
    var dedaloError = false;

    // extract code and state
    var params = this.extractParams();

    if (params.code && params.state) {
      // extract wings preferences
      this.getPreferences(
        this.parseState(params.state),
        function(success) {
          this.$parent.hotspot.name = success.body.hotspot_name;
          this.$parent.hotspot.disclaimers = success.body.disclaimers;
          this.$parent.hotspot.preferences = success.body.preferences;
          this.$root.$options.hotspot.disclaimers = success.body.disclaimers;
          this.$root.$options.hotspot.preferences = success.body.preferences;
          this.$root.$options.hotspot.integrations = success.body.integrations;
          this.hotspot.disclaimers = success.body.disclaimers;
          $("body").css(
            "background-color",
            success.body.preferences.captive_7_background || "#2a87be"
          );
        },
        function(error) {
          this.authorized = false;
          console.error(error);
        }
      );

      // make request to wax
      var url = this.createWaxURL(
        params.code,
        this.parseState(params.state),
        "social/linkedin",
        false,
        this.$route.query.integration_done ? this.$route.query.user : ""
      );

      // get user id
      this.$http.get(url).then(
        function(responseAuth) {
          this.userId = responseAuth.body.user_db_id;

          // check integrations
          if (
            this.$root.$options.hotspot.integrations &&
            this.$root.$options.hotspot.integrations[0] &&
            this.$root.$options.hotspot.integrations[0]
              .post_auth_redirect_url &&
            this.$root.$options.hotspot.integrations[0].post_auth_redirect_url
              .length > 0 &&
            !this.$route.query.integration_done
          ) {
            // go to post_auth_redirect_url
            var redirectUrl = this.$root.$options.hotspot.integrations[0]
              .post_auth_redirect_url;

            var appendParams = this.parseState(params.state);

            var query =
              "?digest=" +
              appendParams.digest +
              "&uuid=" +
              appendParams.uuid +
              "&sessionid=" +
              appendParams.sessionid +
              "&uamip=" +
              appendParams.uamip +
              "&uamport=" +
              appendParams.uamport +
              "&user=" +
              this.userId +
              "&nasid=" +
              appendParams.nasid;

            var search = window.location.search.includes("?")
              ? window.location.search.replace("?", "&")
              : "";

            var pathname = window.location.pathname;

            window.location.replace(redirectUrl + pathname + query + search);
          } else {
            var context = this;
            setTimeout(function() {
              context.authorized = false;
              context.dedaloError = false;
              // exec dedalo login
              context.doDedaloLogin(
                {
                  id: responseAuth.body.user_id,
                  password: responseAuth.password || ""
                },
                function(responseDedalo) {
                  if (responseDedalo.body.clientState == 1) {
                    context.authorized = true;
                    context.dedaloError = false;
                  } else {
                    context.authorized = false;
                    context.dedaloError = true;
                  }
                },
                function(error) {
                  context.authorized = false;
                  context.dedaloError = true;
                  console.error(error);
                }
              );
            }, 1000);
          }
        },
        function(error) {
          this.authorized = false;
          this.dedaloError = true;
          console.error(error);
        }
      );
    } else {
      // get social login url
      params.li_client_id = this.$root.$options.hotspot.socials.linkedin_client_id;
      var url = this.getSocialLoginURL(params, "linkedin");

      // open social login url
      window.location.replace(url);
    }
    return {
      authorized: authorized,
      dedaloError: dedaloError,
      hotspot: {
        disclaimers: this.$root.$options.hotspot.disclaimers,
        preferences: this.$root.$options.hotspot.preferences
      },
      userId: 0,
      conditions: false,
      surveys: false,
      countries: require("./../../i18n/countries.json"),
      additionalCountry: "-",
      additionalReason: "-"
    };
  },
  methods: {
    navigate() {
      if (this.conditions && this.surveys) {
        this.accept();
      } else {
        this.deleteInfo();
      }
    },
    deleteInfo: function() {
      var context = this;
      // extract code and state
      var params = this.extractParams();

      // delete marketing
      if (!this.conditions) {
        this.deleteMarketingInfo(
          this.userId,
          this.parseState(params.state),
          function(success) {
            context.accept();
          },
          function(error) {
            console.error(error);
            if (error.status == 404) {
              context.accept();
            }
          }
        );
      }

      if (!this.surveys) {
        this.deleteSurveyInfo(
          this.userId,
          this.parseState(params.state),
          function(success) {
            context.accept();
          },
          function(error) {
            console.error(error);
            if (error.status == 404) {
              context.accept();
            }
          }
        );
      }
    },
    accept: function() {
      var context = this;

      // extract code and state
      var params = this.extractParams();

      if (
        this.$parent.hotspot.preferences.marketing_0_reason_country == "true"
      ) {
        this.addAdditionalInfo(
          this.userId,
          this.parseState(params.state),
          {
            reason: this.additionalReason,
            country: this.additionalCountry
          },
          function(success) {
            // open redir url
            window.location.replace(
              context.$root.$options.hotspot.preferences.captive_1_redir
            );
          },
          function(error) {
            console.error(error);
            if (error.status == 404) {
              // open redir url
              window.location.replace(
                context.$root.$options.hotspot.preferences.captive_1_redir
              );
            }
          }
        );
      } else {
        // open redir url
        window.location.replace(
          context.$root.$options.hotspot.preferences.captive_1_redir
        );
      }
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1,
h2 {
  font-weight: normal;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}

.text-center {
  text-align: center;
}

textarea {
  min-height: 150px !important;
}

.adjust-top {
  margin-top: 10px !important;
}
.adjust-top-big {
  margin-top: 20px !important;
}
.adjust-checkbox {
  display: block !important;
}
.adjust-button {
  margin-top: 0px !important;
}
</style>