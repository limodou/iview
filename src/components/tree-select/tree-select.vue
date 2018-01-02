<template>
  <div :class="classes" v-clickoutside="handleClose">
    <div
      :class="selectionCls"
      ref="reference"
      @click="toggleMenu">
      <slot name="input">
        <input type="hidden" :name="name" :value="model">
        <div class="ivu-tag ivu-tag-checked" v-for="(item, index) in selectedMultiple">
            <span class="ivu-tag-text">{{ item.label }}</span>
            <Icon type="ios-close-empty" @click.native.stop="removeTag(index)"></Icon>
        </div>
        <span :class="[prefixCls + '-placeholder']" v-show="showPlaceholder && !filterable">{{ localePlaceholder }}</span>
        <span :class="[prefixCls + '-selected-value']" v-show="!showPlaceholder && !multiple && !filterable">{{ selectedSingle }}</span>
        <input
            :id="elementId"
            type="text"
            v-if="filterable"
            v-model="query"
            :disabled="disabled"
            :class="[prefixCls + '-input']"
            :placeholder="showPlaceholder ? localePlaceholder : ''"
            :style="inputStyle"
            autocomplete="off"
            spellcheck="false"
            @blur="handleBlur"
            @keydown="resetInputState"
            @keydown.delete="handleInputDelete"
            ref="input">
        <Icon type="ios-close" :class="[prefixCls + '-arrow']" v-show="showCloseIcon" @click.native.stop="clearSingleSelect"></Icon>
        <Icon type="arrow-down-b" :class="[prefixCls + '-arrow']" v-if="!remote"></Icon>
      </slot>
    </div>
    <transition :name="transitionName">
        <Drop
          :style="{width: dropWidth}"
            :class="dropdownCls"
            v-show="dropVisible"
            :placement="placement"
            ref="dropdown"
            :data-transfer="transfer"
            v-transfer-dom>
            <ul v-show="notFoundShow" :class="[prefixCls + '-not-found']"><li>{{ localeNotFoundText }}</li></ul>
            <ul v-show="(!notFound && !remote) || (remote && !loading && !notFound)" :class="[prefixCls + '-dropdown-list']">
              <Tree
                ref="selector"
                :data="currentData"
                v-bind="options"
                @on-select-change="handleSelected"
                @on-check-change="handleChecked"
                :load-data="handleRemoteLoadData"
                :multiple="multiple"
                :show-checkbox="multiple"
              ></Tree>
            </ul>
            <ul v-show="loading" :class="[prefixCls + '-loading']">{{ localeLoadingText }}</ul>
        </Drop>
    </transition>
  </div>
</template>

<script>
import Icon from '../icon';
import Drop from '../select/dropdown.vue';
import clickoutside from '../../directives/clickoutside';
import TransferDom from '../../directives/transfer-dom';
import { oneOf, findComponentDownward } from '../../utils/assist';
import Emitter from '../../mixins/emitter';
import Locale from '../../mixins/locale';
import { debounce } from '../select/utils';
import Tree from '../tree';
import { getStyle } from '../../utils/assist';

const prefixCls = 'ivu-select';

export default {
  name: 'TreeSelect',
  mixins: [ Emitter, Locale ],
  components: { Icon, Drop, Tree },
  directives: { clickoutside, TransferDom },

  props: {
    value: {
        type: [String, Number, Array],
        default: ''
    },
    // 使用时，也得设置 value 才行
    label: {
        type: [String, Number, Array],
        default: ''
    },
    multiple: {
        type: Boolean,
        default: false
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
    filterable: {
        type: Boolean,
        default: false
    },
    filterMethod: {
        type: Function
    },
    remote: {
        type: Boolean,
        default: false
    },
    remoteQuery: {
        type: Function
    },
    remoteLoadData: {
        type: Function
    },
    loading: {
        type: Boolean,
        default: false
    },
    loadingText: {
        type: String
    },
    size: {
        validator (value) {
            return oneOf(value, ['small', 'large', 'default']);
        }
    },
    labelInValue: {
        type: Boolean,
        default: false
    },
    notFoundText: {
        type: String
    },
    placement: {
        validator (value) {
            return oneOf(value, ['top', 'bottom']);
        },
        default: 'bottom'
    },
    transfer: {
        type: Boolean,
        default: false
    },
    // Use for AutoComplete
    autoComplete: {
        type: Boolean,
        default: false
    },
    name: {
        type: String
    },
    elementId: {
        type: String
    },

    choices: {
        type: Array,
        default () {
            return []
        }
    },

    //可选项
    options: {
      type: Object
    },

    //先中后关闭，单选可用
    selectClose: {
      type: Boolean,
      default: true
    },

    onlyLeaf: {
      type: Boolean,
      default: true
    }
  },
  data () {
    return {
      prefixCls: prefixCls,
      visible: false,
      selectedSingle: '',    // label
      selectedMultiple: [],
      focusIndex: 0,
      query: '',
      lastQuery: '',
      inputLength: 20,
      notFound: false,
      slotChangeDuration: false,    // if slot change duration and in multiple, set true and after slot change, set false
      model: this.value,
      currentLabel: this.label,
      dropWidth: '',
      oldData: null, // 用于保存原来的数据
      currentData: this.choices // 当前数据
    };
  },
  computed: {
    classes () {
        return [
            `${prefixCls}`,
            {
                [`${prefixCls}-visible`]: this.visible,
                [`${prefixCls}-disabled`]: this.disabled,
                [`${prefixCls}-multiple`]: this.multiple,
                [`${prefixCls}-single`]: !this.multiple,
                [`${prefixCls}-show-clear`]: this.showCloseIcon,
                [`${prefixCls}-${this.size}`]: !!this.size
            }
        ];
    },
    dropdownCls () {
        return {
            [prefixCls + '-dropdown-transfer']: this.transfer,
            [prefixCls + '-multiple']: this.multiple && this.transfer,
            ['ivu-auto-complete']: this.autoComplete,
        };
    },
    selectionCls () {
        return {
            [`${prefixCls}-selection`]: !this.autoComplete
        };
    },
    showPlaceholder () {
        let status = false;

        if ((typeof this.model) === 'string') {
            if (this.model === '') {
                status = true;
            }
        } else if (Array.isArray(this.model)) {
            if (!this.model.length) {
                status = true;
            }
        } else if( this.model === null){
            status = true;
        }

        return status;
    },
    showCloseIcon () {
        return !this.multiple && this.clearable && !this.showPlaceholder;
    },
    inputStyle () {
        let style = {};

        if (this.multiple) {
            if (this.showPlaceholder) {
                style.width = '100%';
            } else {
                style.width = `${this.inputLength}px`;
            }
        }

        return style;
    },
    localePlaceholder () {
        if (this.placeholder === undefined) {
            return this.t('i.select.placeholder');
        } else {
            return this.placeholder;
        }
    },
    localeNotFoundText () {
        if (this.notFoundText === undefined) {
            return this.t('i.select.noMatch');
        } else {
            return this.notFoundText;
        }
    },
    localeLoadingText () {
        if (this.loadingText === undefined) {
            return this.t('i.select.loading');
        } else {
            return this.loadingText;
        }
    },
    transitionName () {
        return this.placement === 'bottom' ? 'slide-up' : 'slide-down';
    },
    dropVisible () {
        let status = true;
        // const options = this.$slots.default || [];
        // if (!this.loading && this.remote && this.query === '' && !options.length) status = false;
        //
        // if (this.autoComplete && !options.length) status = false;

        // return this.visible && status;
        return this.visible
    },
    notFoundShow () {
        // const options = this.$slots.default || [];
        // return (this.notFound && !this.remote) || (this.remote && !this.loading && !options.length);
        return (this.query && this.notFound && !this.remote) || (this.remote && !this.loading && !this.currentData.length > 0)
    }
  },

  methods: {
    toggleMenu () {
        if (this.disabled || this.autoComplete) {
            return false;
        }
        this.visible = !this.visible;
    },
    hideMenu () {
        this.visible = false;
        this.focusIndex = 0;
        this.broadcast('iOption', 'on-select-close');
    },
    removeTag (index) {
        if (this.disabled) {
            return false;
        }

        const tag = this.model[index];
        this.selectedMultiple = this.selectedMultiple.filter(item => item.value !== tag);

        this.model.splice(index, 1);

        if (this.filterable && this.visible) {
            this.$refs.input.focus();
        }

        this.broadcast('Drop', 'on-update-popper');
        this.$nextTick( () => {
            this.deselect(tag)
        })
    },
    handleBlur () {
        // setTimeout(() => {
        //     if (this.autoComplete) return;
        //     const model = this.model;

        //     if (this.multiple) {
        //         this.query = '';
        //     } else {
        //         if (model !== '') {
        //             // 如果删除了搜索词，下拉列表也清空了，所以强制调用一次remoteMethod
        //             if (this.remote && this.query !== this.lastQuery) {
        //                 this.$nextTick(() => {
        //                     this.query = this.lastQuery;
        //                 });
        //             }
        //         } else {
        //             this.query = '';
        //         }
        //     }
        // }, 300);
    },
    resetInputState () {
        this.inputLength = this.$refs.input.value.length * 12 + 20;
    },
    handleInputDelete () {
        if (this.multiple && this.model.length && this.query === '') {
            this.removeTag(this.model.length - 1);
        }
    },
    clearSingleSelect () {
        let model = this.model
        if (this.showCloseIcon) {
            this.model = '';
            this.selectedSingle = ''

            if (this.filterable) {
                this.query = '';
            }

            this.$nextTick( () => {
                this.deselect(model)
            })
        }
    },

    // 开启 transfer 时，点击 Drop 即会关闭，这里不让其关闭
    handleTransferClick () {
      if (this.transfer) this.disableCloseUnderTransfer = true;
    },
    handleClose () {
      // if (this.disableCloseUnderTransfer) {
      //   this.disableCloseUnderTransfer = false;
      //   return false;
      // }
      this.visible = false;
      // this.disableClickOutSide = false;
    },

    // 处理树选中状态
    handleSelected (item) {
      if (!this.multiple) {
        this.selectedSingle = item[0].title
        this.model = item[0].id
        this.query = this.selectedSingle
        if (this.selectClose) {
          this.visible = false
        }
      }
    },

    handleChecked (items) {
      if (this.multiple) {
        for(let row of items) {
          // 非叶子结点，并且当前值中不存在
          if (((this.onlyLeaf && !row.children) || (!this.onlyLeaf)) && this.model.indexOf(row.id) === -1) {
            this.selectedMultiple.push({value:row.id || row.title, label: row.title})
            this.model.push(row.id)
          }
        }
      }
    },

    deselect (value) {
        const _search = (parent) => {
            if (parent.id === value) {
                if (this.multiple) {
                    let changed = {checked: false, nodeKey: parent.nodeKey}
                    this.$refs.selector.handleCheck(changed)
                } else {
                    this.$set(parent, 'selected', false)
                }
                return true
            } else {
                if (parent.children && parent.children.length > 0) {
                    for(let item of parent.children) {
                        if (_search(item)) return true
                    }
                }
            }
        }
        for(let item of this.currentData) {
            if (_search(item)) return true
        }
    },

    // 根据搜索条件，重新构建树，只查找叶子结点
    // keys 表示要比较结点的哪些属性，缺省为 title
    rebuild (keys=['title']) {
        let tree = [] // 新的树
        let stack = [] // 父结点栈
        let query = this.query // 查询串

        // 插入结点, 假设id是主键
        const insertData = (stack, node) => {
            let p = tree
            let found
            for(let item of stack) {
                found = false
                for(let c of p) {
                    if (c.id === item.id) {
                        if (!c.children) c.children = []
                        p = c.children
                        found = true
                        break
                    }
                }
                if (!found) {
                    let n = Object.assign({}, item)
                    p.push(n)
                    n.children = []
                    n.selected = false
                    n.checked = false
                    n.indeterminate = false
                    p = n.children
                }
            }
            let n = Object.assign({}, node)
            p.push(n)
            if (n.hasOwnProperty('children'))
                delete n.children
            n.selected = false
            n.checked = false
            n.indeterminate = false
        }

        const match = (node) => {
            for(let k of keys) {
                if (node[k] && (node[k]+'').indexOf(query) > -1) {
                    return true
                }
            }
        }
        const find = (parent) => {
            if (!parent.children || (parent.children && parent.children.length === 0)) {
                if (
                    (
                      (this.multiple && this.model.indexOf(parent.id) === -1)
                        || (!this.multiple && self.model !== parent.id)
                    )
                    && match(parent)
                ) insertData(stack, parent)
            } else {
                if (!this.onlyLeaf) {
                    if (match(parent)) insertData(stack, parent)
                }
                stack.push(parent)
                for(let c of parent.children) {
                    find(c)
                }
                stack.pop()
            }
        }
        if (this.oldData && this.oldData.length > 0) {
            for(let item of this.oldData)
                find(item)
        }
        return tree
    },

    checkItems () {
        const find = (parent) => {
            if (this.model.indexOf(parent.id) > -1) {
                let changed = {checked: true, nodeKey: parent.nodeKey}
                this.$refs.selector.handleCheck(changed)
            }
            if (parent.children && parent.children.length > 0) {
                for(let c of parent.children) {
                    find(c)
                }
            }
        }
        for(let item of this.currentData)
            find(item)
    },
    loadData (item, callback) {
        if (!this.remoteLoadData) return
        if (item && callback) {
            this.remoteLoadData(item, callback)
        } else {
            const cb = (data) => {
                this.currentData = data
                this.notFound = this.currentData.length === 0
            }
            this.remoteLoadData(item, cb)
        }
    },

    handleRemoteLoadData (item, callback) {
        this.loadData(item, callback)
    }
  },

  watch: {
      value (val) {
          this.model = val;
          if (val === '') this.query = '';
      },
      label (val) {
          this.currentLabel = val;
          this.updateLabel();
      },
      model () {
          this.$emit('input', this.model);
          //this.modelToQuery();
          // if (this.multiple) {
          //     if (this.slotChangeDuration) {
          //         this.slotChangeDuration = false;
          //     } else {
          //         this.updateMultipleSelected();
          //     }
          // } else {
          //     this.updateSingleSelected();
          // }
          // // #957
          // if (!this.visible && this.filterable) {
          //     this.$nextTick(() => {
          //         this.broadcastQuery('');
          //     });
          // }
      },
      visible (val) {
          if (val) {
              // 如果是开始为空时，增加一次远程查询
              if (this.remote && this.currentData.length === 0) {
                  this.loadData()
              }
              if (this.filterable) {
                  if (this.multiple) {
                      this.$refs.input.focus();
                      if (!this.query) {
                        this.checkItems()
                      }
                  } else {
                      if (!this.autoComplete) this.$refs.input.select();
                  }
                  if (this.remote) {
                      const options = this.$slots.default || [];
                      if (this.query !== '') {
                          this.remoteQuery(this.query);
                      }
                  }
              }
              this.broadcast('Drop', 'on-update-popper');
          } else {
              if (this.filterable) {
                  if (!this.autoComplete) this.$refs.input.blur();
                  this.query = ''
              }
              this.broadcast('Drop', 'on-destroy-popper');
          }
      },
      query (val) {
          if (this.remote && this.remoteQuery) {
            if (val) {
                if (!this.oldData) {
                    this.oldData = this.currentData
                }
                const cb = (data) => {
                    this.currentData = data
                    this.notFound = this.currentData.length === 0
                }
                this.remoteQuery(val, cb)
            } else {
                if (this.oldData) {
                    this.currentData = this.oldData
                    this.oldData = null
                }
            }
            this.focusIndex = 0;
          } else {
            // 如果有值，则如果oldData为空，则将currentData保存在上面，然后重建树结构
            if (val) {
                if (!this.oldData) {
                    this.oldData = this.currentData
                }
                this.currentData = this.rebuild()
                this.notFound = this.currentData.length === 0
            } else {
                if (this.oldData) {
                    this.currentData = this.oldData
                    this.oldData = null
                }
            }
          }
        //   this.selectToChangeQuery = false;
          this.broadcast('Drop', 'on-update-popper');
      }
  },

  created () {
    if (this.multiple) {
      if (!this.model) this.model = []
    }
  },

  mounted () {
    this.dropWidth = `${parseInt(getStyle(this.$el, 'width'))}px`
  }
}
</script>
