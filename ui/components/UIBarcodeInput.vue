<template>
    <div class="nrdb-ui-barcode-input barcode">
    <v-text-field
        ref="inputField"
        v-model="value"
        :label="label"
        :placeholder="placeholder"
        :prepend-icon="iconPosition === 'left' && iconInnerPosition === 'outside' ? icon : null"
        :prepend-inner-icon="iconPosition === 'left' && iconInnerPosition === 'inside' ? icon : null"
        :append-icon="iconPosition === 'right' && iconInnerPosition === 'outside' ? icon : null"
        :append-inner-icon="iconPosition === 'right' && iconInnerPosition === 'inside' ? icon : null"
        variant="outlined"
        density="compact"
        hide-details="auto"
        clearable
        @keyup.enter="onEnter"
        @input="onChange"
        @blur="onBlur"
    />
    </div>
</template>

<script>
import { mapState } from 'vuex'

export default {
    name: 'UIBarcodeInput',
    inject: ['$socket', '$dataTracker'],
    props: {
        id: { type: String, required: true },
        props: { type: Object, default: () => ({}) },
        state: { type: Object, default: () => ({}) }
    },
    data () {
        return {
            value: ''
        }
    },
    computed: {
        ...mapState('data', ['messages']),
        label () {
            return this.getProperty('label') || 'Barcode'
        },
        placeholder () {
            return this.getProperty('placeholder') || 'Scan barcode...'
        },
        clearOnSend () {
            return this.getProperty('clearOnSend') !== false // default true
        },
        icon () {
            const iconName = this.getProperty('icon') || 'barcode'
            return iconName ? `mdi-${iconName}` : null
        },
        iconPosition () {
            return this.getProperty('iconPosition') || 'left'
        },
        iconInnerPosition () {
            return this.getProperty('iconInnerPosition') || 'inside'
        },
    },
    created () {
        this.$dataTracker(this.id, this.onInput, this.onLoad, this.onDynamicProperties)
    },
    mounted () {
        // Focus function - ensures the barcode input always stays focused
        this.focusBarcode = () => {
            if (this.$refs.inputField) {
                this.$refs.inputField.focus()
            }
        }

        // Initial focus when component is mounted
        this.$nextTick(() => {
            this.focusBarcode()
        })

        // Global click handler - ANY click on the page triggers refocus
        // This ensures barcode scanner input always reaches this field
        document.addEventListener('click', this.focusBarcode, true)
    },
    beforeUnmount () {
        // Cleanup event listener when component is destroyed
        if (this.focusBarcode) {
            document.removeEventListener('click', this.focusBarcode, true)
        }
    },
    methods: {
        getProperty (property) {
            // Access properties from the node config
            if (this.props && typeof this.props[property] !== 'undefined') {
                return this.props[property]
            }
            return undefined
        },
        onBlur () {
            // Immediately refocus if focus is lost
            // This prevents the barcode scanner from losing its target
            this.$nextTick(() => {
                this.focusBarcode()
            })
        },
        onEnter () {
            if (this.value && this.value.trim() !== '') {
                // Send message to Node-RED
                this.send({
                    payload: this.value.trim()
                })
                
                // Clear input if configured to do so
                if (this.clearOnSend) {
                    this.value = ''
                }
            }
        },
        onChange () {
            // Optional: send on every keystroke if needed
            // For now, we only send on Enter
        },
        send (msg) {
            this.$socket.emit('widget-action', this.id, msg)
        },
        onInput (msg) {
            // Handle incoming messages from Node-RED
            if (msg && typeof msg.payload !== 'undefined') {
                this.value = msg.payload
            }
            
            // Store in vuex
            this.$store.commit('data/bind', {
                widgetId: this.id,
                msg
            })
        },
        onLoad (msg, state) {
            // Load initial value if present
            if (msg && typeof msg.payload !== 'undefined') {
                this.value = msg.payload
            }
        },
        onDynamicProperties (msg) {
            // Handle dynamic property updates
            const updates = msg.ui_update
            if (!updates) {
                return
            }
            
            // Handle property updates
            const dynamicProps = {}
            if (typeof updates.label !== 'undefined') {
                dynamicProps.label = updates.label
            }
            if (typeof updates.placeholder !== 'undefined') {
                dynamicProps.placeholder = updates.placeholder
            }
            if (typeof updates.clearOnSend !== 'undefined') {
                dynamicProps.clearOnSend = updates.clearOnSend
            }
            
            if (Object.keys(dynamicProps).length > 0) {
                this.setDynamicProperties(dynamicProps)
            }
        },
        setDynamicProperties (properties) {
            // Update component properties dynamically
            Object.assign(this.props, properties)
        }
    }
}
</script>

<style scoped>
.nrdb-ui-barcode-input {
    width: 100%;
}
</style>

<style>
/* Global styles for barcode input - not scoped to allow deep targeting of Vuetify components */

/* Barcode input field styling */
.nrdb-ui-barcode-input.barcode :deep(.v-field) {
    border-radius: 9999px;
    box-shadow: 0 0 0 2px rgba(0, 148, 206, 0.25);
    padding-inline: 8px;
}

.nrdb-ui-barcode-input.barcode :deep(input.v-field__input) {
    font-size: 18px;
    letter-spacing: 0.08em;
}

/* Barcode icon styling */
.nrdb-ui-barcode-input.barcode .barcode-icon {
    font-size: 20px;
    color: rgba(0, 148, 206, 0.8);
    margin-right: 4px;
}
</style>