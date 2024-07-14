<template>
  <form class="mx-0 flex flex-wrap" @submit.prevent="createChannel()">
    <!-- Campos existentes... -->
    
    <div class="w-full">
      <button id="loginBtn" @click.prevent="launchWhatsAppSignup">
        {{ $t('INBOX_MGMT.ADD.WHATSAPP.LOGIN_EMBEDDED_BUTTON') }}
      </button>
    </div>

    <div class="w-full">
      <woot-submit-button
        :loading="uiFlags.isCreating"
        :button-text="$t('INBOX_MGMT.ADD.WHATSAPP.SUBMIT_BUTTON')"
      />
    </div>
  </form>
</template>

<script>
import { mapGetters } from 'vuex';
import alertMixin from 'shared/mixins/alertMixin';
import { required } from 'vuelidate/lib/validators';
import router from '../../../../index';
import { isPhoneE164OrEmpty, isNumber } from 'shared/helpers/Validators';

export default {
  mixins: [alertMixin],
  data() {
    return {
      inboxName: '',
      phoneNumber: '',
      apiKey: '',
      phoneNumberId: '',
      businessAccountId: '',
    };
  },
  computed: {
    ...mapGetters({ uiFlags: 'inboxes/getUIFlags' }),
  },
  validations: {
    inboxName: { required },
    phoneNumber: { required, isPhoneE164OrEmpty },
    apiKey: { required },
    phoneNumberId: { required, isNumber },
    businessAccountId: { required, isNumber },
  },
  methods: {
    async createChannel() {
      this.$v.$touch();
      if (this.$v.$invalid) {
        return;
      }

      try {
        const whatsappChannel = await this.$store.dispatch(
          'inboxes/createChannel',
          {
            name: this.inboxName,
            channel: {
              type: 'whatsapp',
              phone_number: this.phoneNumber,
              provider: 'whatsapp_embedded',
              provider_config: {
                api_key: this.apiKey,
                phone_number_id: this.phoneNumberId,
                business_account_id: this.businessAccountId,
              },
            },
          }
        );

        router.replace({
          name: 'settings_inboxes_add_agents',
          params: {
            page: 'new',
            inbox_id: whatsappChannel.id,
          },
        });
      } catch (error) {
        this.showAlert(
          error.message || this.$t('INBOX_MGMT.ADD.WHATSAPP.API.ERROR_MESSAGE')
        );
      }
    },

    launchWhatsAppSignup() {
      window.fbAsyncInit = function () {
        FB.init({
          appId: '1931336960624054',
          cookie: true,
          xfbml: true,
          version: 'v20.0'
        });
      };

      (function (d, s, id) {
        var js, fjs = d.getElementsByTagName(s)[0];
        if (d.getElementById(id)) return;
        js = d.createElement(s); js.id = id;
        js.src = "https://connect.facebook.net/en_US/sdk.js";
        fjs.parentNode.insertBefore(js, fjs);
      }(document, 'script', 'facebook-jssdk'));

      FB.login(response => {
        if (response.authResponse) {
          const code = response.authResponse.code;
          // Envie o código para seu backend para obter um token de acesso
          this.processLoginResponse(code);
        } else {
          console.log('User cancelled login or did not fully authorize.');
        }
      }, {
        config_id: '490819830133189', // Substitua pelo ID de configuração
        response_type: 'code',
        override_default_response_type: true,
        extras: {
          setup: {
            // Dados preenchidos automaticamente
          }
        }
      });
    },

    async processLoginResponse(code) {
      try {
        // Envie o código para seu backend para obter o token de acesso e detalhes do WhatsApp
        const response = await this.$axios.post('/api/v1/whatsapp_login', { code });
        const { phone_number, api_key, phone_number_id, business_account_id } = response.data;
        this.phoneNumber = phone_number;
        this.apiKey = api_key;
        this.phoneNumberId = phone_number_id;
        this.businessAccountId = business_account_id;
      } catch (error) {
        this.showAlert(error.message || this.$t('INBOX_MGMT.ADD.WHATSAPP.API.ERROR_MESSAGE'));
      }
    }
  },
};
</script>
