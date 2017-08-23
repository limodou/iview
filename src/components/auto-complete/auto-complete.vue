<template>
    <i-select
        ref="select"
        class="ivu-auto-complete"
        :label="label"
        :disabled="disabled"
        :clearable="clearable"
        :placeholder="placeholder"
        :size="size"
        filterable
        remote
        auto-complete
        :remote-method="remoteMethod"
        @on-change="handleChange"
        :transfer="transfer">
        <slot name="input">
            <i-input
                ref="input"
                slot="input"
                v-model="currentValue"
                :placeholder="placeholder"
                :disabled="disabled"
                :size="size"
                :icon="closeIcon"
                @on-click="handleClear"
                @on-focus="handleFocus"
                @on-blur="handleBlur"></i-input>
        </slot>
        <slot>
            <i-option v-for="item in filteredData" :value="item" :key="item">{{ item }}</i-option>
        </slot>
    </i-select>
</template>
<script>
    import iSelect from '../select/select.vue';
    import iOption from '../select/option.vue';
    import iInput from '../input/input.vue';
    import { oneOf } from '../../utils/assist';

    export default {
        name: 'AutoComplete',
        components: { iSelect, iOption, iInput },
        props: {
            value: {
                type: [String, Number],
                default: ''
            },
            label: {
                type: [String, Number],
                default: ''
            },
            data: {
                type: Array,
                default: () => []
            },
            disabled: {
                type: Boolean,
                default: false
            },
            clearable: {
                type: Boolean,
                default: false
            },
            placeholder: {
                type: String
            },
            size: {
                validator (value) {
                    return oneOf(value, ['small', 'large', 'default']);
                }
            },
            filterMethod: {
                type: [Function, Boolean],
                default: false
            },
            transfer: {
                type: Boolean,
                default: false
            }
        },
        data () {
            return {
                currentValue: this.value
            };
        },
        computed: {
            closeIcon () {
                return this.clearable && this.currentValue ? 'ios-close' : '';
            },
            filteredData () {
                if (this.filterMethod) {
                    return this.data.filter(item => this.filterMethod(this.currentValue, item));
                } else {
                    return this.data;
                }
            }
        },
        watch: {
            value (val) {
                this.currentValue = val;
            },
            currentValue (val) {
                this.$refs.select.query = val;
                this.$emit('input', val);
                this.$emit('on-change', val);
            }
        },
        methods: {
            remoteMethod (query) {
                this.$emit('on-search', query);
            },
            handleChange (val) {
                this.currentValue = val;
                this.$refs.select.model = val;
                this.$refs.input.blur();
                this.$emit('on-select', val);
            },
            handleFocus () {
                this.$refs.select.visible = true;
            },
            handleBlur () {
                this.$refs.select.visible = false;
            },
            handleClear () {
                this.currentValue = '';
                this.$refs.select.model = '';
            }
        }
    };
</script>