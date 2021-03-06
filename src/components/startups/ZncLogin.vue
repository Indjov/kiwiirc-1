<template>
    <startup-layout class="kiwi-welcome-znc" ref="layout">
        <div slot="connection">
            <template v-if="!network || network.state === 'disconnected'">
                <form @submit.prevent="formSubmit" class="u-form kiwi-welcome-znc-form">
                    <h2 v-html="greetingText"></h2>

                    <div class="kiwi-welcome-znc-error" v-if="network && (network.last_error || network.state_error)">We couldn't connect to the server :( <span>{{network.last_error || readableStateError(network.state_error)}}</span></div>

                    <input-text v-if="showUser" class="kiwi-welcome-znc-nick" :label="$t('username')" v-model="username" />
                    <input-text v-if="showPass" class="kiwi-welcome-znc-password" :label="$t('password')" v-model="password" type="password" />
                    <input-text v-if="showNetwork" class="kiwi-welcome-znc-channel" :label="$t('network')" v-model="znc_network" />
                    <button
                        class="u-button u-button-primary u-submit kiwi-welcome-znc-start"
                        type="submit"
                        v-html="buttonText"
                        :disabled="!readyToStart"
                    ></button>
                </form>
            </template>
            <template v-else-if="network.state !== 'connected'">
                <i class="fa fa-spin fa-spinner" style="font-size:2em; margin-top:1em;" aria-hidden="true"></i>
            </template>
        </div>
    </startup-layout>
</template>

<script>

import _ from 'lodash';
import * as Misc from '@/helpers/Misc';
import state from '@/libs/state';
import StartupLayout from './CommonLayout';

export default {
    components: {
        StartupLayout,
    },
    data: function data() {
        return {
            network: null,
            network_extras: null,
            username: '',
            password: '',
            znc_network: '',
            showNetwork: true,
            showPass: true,
            showUser: true,
            show_password_box: false,
        };
    },
    computed: {
        greetingText: function greetingText() {
            let greeting = state.settings.startupOptions.greetingText;
            return typeof greeting === 'string' ?
                greeting :
                this.$t('start_greeting');
        },
        buttonText: function buttonText() {
            let greeting = state.settings.startupOptions.buttonText;
            return typeof greeting === 'string' ?
                greeting :
                this.$t('start_button');
        },
        readyToStart: function readyToStart() {
            return this.username && (this.password || this.showPass === false);
        },
        infoContent: function infoContent() {
            return state.settings.startupOptions.infoContent || '';
        },
    },
    methods: {
        readableStateError(err) {
            return Misc.networkErrorMessage(err);
        },
        formSubmit: function formSubmit() {
            if (this.readyToStart) {
                this.startUp();
            }
        },
        addNetwork: function addNetwork(netName) {
            let options = state.settings.startupOptions;
            let password = this.username;
            if (netName) {
                password += '/' + netName;
            }
            password += ':' + this.password;

            let net = state.addNetwork(netName, this.username, {
                server: _.trim(options.server),
                port: options.port,
                tls: options.tls,
                password: password,
            });
            return net;
        },
        startUp: function startUp() {
            if (this.network) {
                state.removeNetwork(this.network.id);
            }

            let netList = _.compact(this.znc_network.split(','));
            if (netList.length === 0) {
                netList.push('');
            }

            // add our first network and make sure we can connect
            // before trying to add other networks.
            let net = this.network = this.addNetwork(netList.shift());
            this.network_extras = netList;

            let onRegistered = () => {
                state.setActiveBuffer(net.id, net.serverBuffer().name);
                net.ircClient.off('registered', onRegistered);
                net.ircClient.off('close', onClosed);
                this.network_extras.forEach((netName, idx) => {
                    let extraNet = this.addNetwork(_.trim(netName));
                    extraNet.ircClient.connect();
                });
                this.$refs.layout.close();
            };
            let onClosed = () => {
                net.ircClient.off('registered', onRegistered);
                net.ircClient.off('close', onClosed);
            };
            net.ircClient.once('registered', onRegistered);
            net.ircClient.once('close', onClosed);
            net.ircClient.connect();
        },
    },
    created: function created() {
        let options = state.settings.startupOptions;

        this.username = options.username || '';
        this.password = options.password || '';
        this.znc_network = window.location.hash.substr(1) || options.network || '';
        this.showNetwork = typeof options.showNetwork === 'boolean' ?
            options.showNetwork :
            true;
        this.showUser = typeof options.showUser === 'boolean' ?
            options.showUser :
            true;
        this.showPass = typeof options.showPass === 'boolean' ?
            options.showPass :
            true;


        if (options.autoConnect && this.username && this.password) {
            this.startUp();
        }
    },
};
</script>

<style>

.kiwi-welcome-znc h2 {
    margin-bottom: 1.5em;
    font-size: 1.7em;
    text-align: center;
    padding: 0;
    margin: 0.5em 0 1em 0;
}

.kiwi-welcome-znc-error {
    text-align: center;
    margin: 1em 0;
    padding: 0.3em;
}

.kiwi-welcome-znc-error span {
    display: block;
    font-style: italic;
}

.kiwi-welcome-znc-form {
    width: 300px;
    background-color: #fff;
    border-radius: 0.5em;
    padding: 1em;
    border:1px solid #ececec;
}

.kiwi-welcome-znc-form label {
    text-align: left;
    display: inline-block;
    margin-bottom: 1.5em;
}
.kiwi-welcome-znc-form input[type="text"] {
    font-size: 1em;
    margin-top: 5px;
    padding: 0.3em 1em;
    width: 100%;
    box-sizing: border-box;
}

.kiwi-welcome-znc .input-text,
.kiwi-welcome-znc .kiwi-welcome-znc-have-password input {
    margin-bottom: 1.5em;
}
.kiwi-welcome-znc-have-password input:checked {
    margin-bottom: 0;
}

.kiwi-welcome-znc-start {
    font-size: 1.1em;
    cursor: pointer;
}
.kiwi-welcome-znc-start[disabled] {
    cursor: not-allowed;
}

</style>
