<template>
    <div class="ui segment form">
        <h3>{{ $t("login.disclaimer_use") }}</h3>
        <div class="inline field">
            <textarea readonly class="text-center" v-model="hotspot.disclaimers.terms_of_use"></textarea>
        </div>
        <button v-on:click="decline()" class="ui big button red">{{ $t("login.decline") }}</button>
        <button v-on:click="accept()" class="ui big button green">{{ $t("login.accept") }}</button>
    </div>
</template>

<script>
    import AuthMixin from './../mixins/auth';
    export default {
        name: 'LoginPage',
        mixins: [AuthMixin],
        data: function() {
            return {
                hotspot: {
                    disclaimers: this.$root.$options.hotspot.disclaimers
                },
                loginDest: this.$root.$options.session['loginDest'],
            }
        },
        methods: {
            decline: function() {
                this.$router.push({
                    path: '/'
                })
            },
            accept: function() {
                this.$router.push({
                    path: '/login'
                })
            }
        }
    }
</script>

<style scoped>
    .text-center {
        text-align: center;
    }

    textarea {
        min-height: 300px !important;
    }
</style>