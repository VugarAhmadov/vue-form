<template>
    <div id="top-dialog" :class="wrapperClass">
        <div class="dialog alert" :class="visibleClass('alert')" v-html="message" @click="clear"></div>
        <div class="dialog warning" :class="visibleClass('warning')" v-html="message" @click="clear"></div>
        <div class="dialog confirm" :class="visibleClass('confirm')">
            <p v-html="message"></p>
            <nav>
                <span class="small button" v-if="!processing" @click="makeCall">YES</span>
                <span class="small disabled button" v-if="processing"><i class="fas fa-spinner fa-spin fa-fw"></i></span>
                <span class="small success button" @click="clear">NO</span>
            </nav>
        </div>
    </div>
</template>
<script>
    import { AjaxCaller } from "@ssdcode/cms-partials";
    export default {
        name: 'top-dialog',
        mixins: [AjaxCaller],
        props: {
            sessionDialog: {
                type: Object,
                default: () => { return {} }
            }
        },
        data() {
            return {
                id: null,
                type: null,
                message: '',
                url: null,
                data: {},
                methodType: 'get',
                timeout: null,
            }
        },
        computed: {
            isVisible() {
                return this.message !== '';
            },
            wrapperClass() {
                return {
                    'overlay': this.isVisible && this.typeIs('confirm')
                }
            }
        },
        created() {
            window.EventBus.listen('clear-top-dialog', this.clear);
            window.EventBus.listen('top-alert', this.alertEvent);
            window.EventBus.listen('top-warning', this.warningEvent);
            window.EventBus.listen('top-confirm', this.confirmEvent);
        },
        mounted() {
            if (Object.keys(this.sessionDialog).length) {
                setTimeout(() => {
                    window.EventBus.fire(this.sessionDialog.type, {
                        id: 'session-' + this.sessionDialog.type,
                        message: this.sessionDialog.message
                    });
                }, 500);
            }
        },
        methods: {
            typeIs(type) {
                return this.type === type;
            },
            visibleClass(type) {
                return {
                    'active': this.isVisible && this.typeIs(type)
                }
            },
            clear(data = null) {
                this.clearCountDown();
                this.type = null;
                this.message = '';
                this.url = null;
                this.data = {};
                this.methodType = 'get';
                this.stopProcessingAjaxCall();
                window.EventBus.fire(this.id + '-cleared', data || {});
            },
            clearCountDown() {
                if (this.timeout === null) {
                    return;
                }
                clearTimeout(this.timeout);
                this.timeout = null;
            },
            alert(data, type) {
                this.clear();
                this.id = data.id;
                this.type = type;
                this.message = data.message;
                this.countDown();
            },
            alertEvent(data) {
                this.alert(data, 'alert');
            },
            warningEvent(data) {
                this.alert(data, 'warning');
            },
            confirmEvent(data) {
                this.clear();
                this.id = data.id;
                this.type = 'confirm';
                this.message = data.message;
                if (data.url) {
                    this.url = data.url;
                }
                if (data.data) {
                    this.data = data.data;
                }
                if (data.method) {
                    this.methodType = data.method.toLowerCase();
                }
            },
            countDown() {
                this.timeout = setTimeout(() => {
                    this.clear();
                }, 7000);
            },
            makeCall() {
                this.startProcessingAjaxCall();
                if (this.url) {
                    this.makeAjaxRequest(
                        this.success,
                        this.failure,
                        this.data,
                        this.url,
                        this.methodType
                    );
                } else {
                    this.success();
                }
            },
            success(response) {
                window.EventBus.fire(this.id + '-called', {
                    response: response,
                    dialog: this
                });
            },
            failure(error) {
                this.stopProcessingAjaxCall();
                if (error.response) {
                    error = 'Error code: ' + error.response.status + ' (' + error.response.statusText + ')';
                } else {
                    error = error.message;
                }
                this.warningEvent({
                    id: this.id,
                    message: error
                });
            }
        }
    }
</script>