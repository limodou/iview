<template>
  <div :class="classes" v-clickoutside:exactElement="handleClose">
    <div
      :class="selectionCls"
      class="exactElement"
      ref="reference"
      @click="toggleMenu">
      <slot name="input">
        <input type="hidden" :name="name" :value="model">
        <div class="ivu-tag ivu-tag-checked" v-for="(item, index) in selectedMultiple">
            <span class="ivu-tag-text">{{ item.label }}</span>
            <Icon type="ios-close" @click.native.stop="removeTag(index)"></Icon>
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
        <Icon type="ios-close-circle" :class="[prefixCls + '-arrow']" v-show="showCloseIcon" @click.native.stop="clearSingleSelect"></Icon>
        <Icon type="ios-arrow-down" :class="[prefixCls + '-arrow']" v-if="!remote"></Icon>
      </slot>
    </div>
        <Drop
            :class="dropdownCls"
            class="ivu-tree-select-dropdown exactElement"
            v-show="dropVisible"
            :placement="placement"
            ref="dropdown"
            :data-transfer="transfer"
            v-transfer-dom
            @on-update-popper="handleUpdatePopper"
            @click="handleTransferClick">
            <ul v-show="notFoundShow" :class="[prefixCls + '-not-found']"><li>{{ localeNotFoundText }}</li></ul>
            <ul v-show="(!notFound && !remote) || (remote && !loading && !notFound)" :class="[prefixCls + '-dropdown-list']">
              <Tree
                ref="selector"
                :data="treedata"
                v-bind="options"
                @on-select-change="handleSelected"
                @on-check-change="handleChecked"
                :load-data="handleRemoteLoadData"
                :multiple="multiple"
                :show-checkbox="multiple"
                check-directly
                :check-strictly="checkStrictly"
              ></Tree>
            </ul>
            <ul v-show="loading" :class="[prefixCls + '-loading']">{{ localeLoadingText }}</ul>
        </Drop>
  </div>
</template>

<script>
import Icon from '../icon';
import Drop from '../select/dropdown.vue';
import clickoutside from '../../directives/clickoutside';
import TransferDom from '../../directives/transfer-dom';
import { oneOf, findComponentDownward, deepCompare } from '../../utils/assist';
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
        default: ''
    },
    // // 使用时，也得设置 value 才行
    // label: {
    //     type: [String, Number, Array],
    //     default: ''
    // },
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
    // 是否保留
    keepFilter: {
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
        default: true
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
    childrenKey: {
        type: String,
        default: 'children'
    },

    choices: {
        type: Array,
        default () {
            return []
        }
    },

    checkStrictly: {
        type: Boolean,
        default: false
    },

    //可选项
    options: {
      type: Object
    },

    //选中后关闭，单选可用
    selectClose: {
      type: Boolean,
      default: true
    },

    // 是否只允许选叶子结点
    onlyLeaf: {
      type: Boolean,
      default: true
    },

    // 是否在只允许选叶子结点时，禁用父结点
    parentDisabledWhenOnlyLeaf: {
        type: Boolean,
        default: true
    }
  },
  data () {
    let model = ''
    if (this.multiple) {
        if (!this.value) model = []
    }
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
      model: model,
      currentLabel: '',
      currentNode: {}, // 单选时选中结点
      oldData: null, // 用于保存原来的数据
      currentData: this.choices // 当前数据
    }
  },
  computed: {
    classes () {
        return [
            `${prefixCls}`,
            'ivu-tree-select',
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
    treedata () {
        const f = (parent) => {
            for (let c of parent) {
                // 处理 disabled
                let status_disabled = Boolean(this.onlyLeaf && this.parentDisabledWhenOnlyLeaf && ((c.children && c.children.length) || ('loading' in c && !c.loadding)))
                c.disabled = c.disabled || status_disabled
                if (c.children && c.children.length) {
                    f(c.children)
                }
            }
        }
        f(this.currentData)
        return this.currentData
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

        if (!this.model || JSON.stringify(this.model) === '{}' || Array.isArray(this.model) && this.model.length === 0) {
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
        // if (!this.visible) {
        //     this.query = ''
        // }
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

        let model = this.model.slice()
        model.splice(index, 1)
        this.model = model

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
    reset () {
        let model = this.multiple ? [] : this.labelInValue ? {} : '';
        this.currentData = []
        this.oldData = null
        this.selectedSingle = ''
        this.selectedMultiple = []
        this.model = model
    },
    clearSingleSelect () {
        let model = this.model
        if (this.showCloseIcon) {
            this.model = ''
            this.selectedSingle = ''
            this.currentNode = {}

            if (this.filterable) {
                this.query = '';
            }
            this.notFound = false

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
      if (this.disableCloseUnderTransfer) {
        this.disableCloseUnderTransfer = false;
        return ;
      }
      this.visible = false;
      this.disableClickOutSide = false;
    },

    // 处理树选中状态
    handleSelected (item) {
      if (!this.multiple) {
        if (item.length === 0) {
            this.selectedSingle = ''
            this.model = ''
            this.query = this.selectedSingle
            this.currentNode = {}
        } else {
            this.selectedSingle = item[0].title
            this.model = item[0].id
            this.query = this.selectedSingle
            this.currentNode = item[0]
        }
        if (this.selectClose) {
          this.visible = false
        }
      }
    },

    // 将 node 的值推送到 model 中，用于多选情况
    // model 是 id 的数组，不是对象
    pushData (model, selectedMultiple, node) {
        let d = Object.assign({}, node, {value:node.id || node.title, label: node.title})
        // 清除子结点
        if (d[this.childrenKey]) {
            delete d[this.childrenKey]
        }
        selectedMultiple.push(d)
        model.push(node.id)
    },

    handleChecked (items, node, deleted) {
        if (this.multiple) {
            let selectedMultiple = [...this.selectedMultiple]
            let model = [...this.model]
            if (deleted.length > 0){
                for(let i=selectedMultiple.length-1; i>-1; i--) {
                    let item = selectedMultiple[i]
                    if (deleted.indexOf(item.value) > -1) {
                        selectedMultiple.splice(i, 1)
                        model.splice(i, 1)
                    }
                }
            }
            for(let item of items) {
                if (model.indexOf(item.id) === -1) {
                    this.pushData(model, selectedMultiple, item);
                }
            }
            this.selectedMultiple = selectedMultiple;
            this.model = model;
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
                if (parent[this.childrenKey] && parent[this.childrenKey].length > 0) {
                    for(let item of parent[this.childrenKey]) {
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
                        if (!c[this.childrenKey]) c[this.childrenKey] = []
                        p = c[this.childrenKey]
                        found = true
                        break
                    }
                }
                if (!found) {
                    let n = Object.assign({}, item)
                    p.push(n)
                    n[this.childrenKey] = []
                    n.selected = false
                    n.checked = false
                    n.indeterminate = false
                    n.expand = true
                    p = n[this.childrenKey]
                }
            }
            let n = Object.assign({}, node)
            p.push(n)
            if (n.hasOwnProperty(this.childrenKey))
                delete n[this.childrenKey]
            n.selected = false
            n.checked = false
            n.indeterminate = false
            n.expand = true
        }

        const match = (node) => {
            for(let k of keys) {
                if (node[k] && (node[k]+'').indexOf(query) > -1) {
                    return true
                }
            }
        }
        const find = (parent) => {
            if (!parent[this.childrenKey] || (parent[this.childrenKey] && parent[this.childrenKey].length === 0)) {
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
                for(let c of parent[this.childrenKey]) {
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
            if (parent[this.childrenKey] && parent[this.childrenKey].length > 0) {
                for(let c of parent[this.childrenKey]) {
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
                this.doQuery(this.query)
            }
            this.remoteLoadData(item, cb)
        }
    },

    isSelected (item) {
        if (this.multiple) {
            for (let c of this.selectedMultiple) {
                if (item.id === c.value) {
                    return true
                }
            }
        } else {
            if (item.title === this.selectedSingle) {
                return true
            }
        }
    },

    handleRemoteLoadData (item, callback) {
        this.loadData(item, callback)
    },

    // 当下拉窗打开时，设置节点选中状态
    handleUpdatePopper () {
        const walk = (data) => {
            for(let item of data) {
                //检查是否选中状态
                let flag = this.isSelected(item)
                if (this.multiple) {
                    this.$set(item, 'checked', flag)
                } else {
                    this.$set(item, 'selected', flag)
                }
                if (item[this.childrenKey] && item[this.childrenKey].length > 0) {
                    walk(item[this.childrenKey])
                }
            }
        }
        walk(this.currentData)
    },
    doQuery (val) {
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
        // 如果有值且可以搜索，则如果oldData为空，则将currentData保存在上面，然后重建树结构
        if (val && this.filterable) {
            if (!this.oldData) {
                this.oldData = this.currentData
            }
            this.currentData = this.rebuild()
        } else {
            if (this.oldData) {
                this.currentData = this.oldData
                this.oldData = null
            }
        }
        this.notFound = this.currentData.length === 0
        }
    //   this.selectToChangeQuery = false;
        this.broadcast('Drop', 'on-update-popper');        
    }
  },

  watch: {
      // value支持 {label, value} 形式
      value: {
          immediate: true,
          handler (val) {
            if (this.multiple) {
                if (!val) val = []
            }
            if ((typeof val === 'object' && this.model === val.value) || 
                (typeof val !== 'object' && this.model===val)) return
            // if (deepCompare(val, this.model)) return
            let model = this.model
            if (this.labelInValue) {
                if (this.multiple) {
                    this.model = val.map(x=>x.value)
                    this.currentLabel = val.slice()
                } else {
                    this.model = val.value
                    this.currentLabel = this.model ? val.label : ''
                }
            } else {
                this.model = val
            }
            if (this.multiple && this.model.length === 0) {
                this.query = ''
                this.selectedMultiple = []
                this.$nextTick( () => {
                    for(const item of model) {
                        this.deselect(item)
                    }
                })
            }
            if (!this.multiple) {
                if (!this.model) {
                    this.query = ''
                    this.selectedSingle = ''
                    this.$nextTick( () => {
                        this.deselect(model)
                    })
                } else {
                    this.query = this.currentLabel
                }
            } 
          },
          deep: true
      },
    //   label: {
    //       immediate: true,
    //       handler (val) {
    //         this.currentLabel = val;
    //         if (val) {
    //             if (this.multiple && this.value) {
    //                 this.selectedMultiple = val // 需要 value 和 label 都有值,或label 是 [{label: value}, ...]的形式
    //             } else {
    //                 this.selectedSingle = val
    //             }
    //         }
    //       }
    //   },
      currentLabel: {
        immediate: true,
        handler (val) {
          if (val) {
            if (this.multiple && this.value) {
              this.selectedMultiple = val // 需要 value 和 label 都有值,或label 是 [{label: value}, ...]的形式
            } else {
              this.selectedSingle = val
            }
          }
        },
        deep: true
      },
      model:{
        handler (v) {
            if (this.labelInValue) {
                if (!this.multiple) {
                    // 有值返回对象，无值空对象
                    if (this.model) {
                        let r = Object.assign({}, this.currentNode, {value: this.model, label: this.selectedSingle})
                        delete r.children
                        this.$emit('input', r)
                    } else {
                        this.$emit('input', {})
                    }
                } else {
                    this.$emit('input', this.selectedMultiple.slice())
                }
            } else {
                if (!this.multiple) {
                    this.$emit('input', this.model)
                } else {
                    this.$emit('input', this.model.slice())
                }
            }
        },
        deep: true
      },
      choices: {
        handler (val) {
          this.currentData = val
        },
        deep: true
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
                    //   if (!this.query) {
                    //     this.checkItems()
                    //   }
                  } else {
                      if (!this.autoComplete) this.$refs.input.select();
                  }
                  if (this.remote) {
                      const options = this.$slots.default || [];
                      if (this.query !== '' && this.remoteQuery) {
                          this.remoteQuery(this.query);
                      }
                  }
              }
              this.broadcast('Drop', 'on-update-popper');
              // if (this.multiple)
              if (!this.keepFilter && this.oldData) {
                  this.currentData = this.oldData
              }
          } else {
              if (this.filterable) {
                  if (!this.autoComplete) this.$refs.input.blur();
                  if (this.multiple){
                    this.query = ''
                  } else {
                    this.query = this.selectedSingle
                  }
              }
              this.broadcast('Drop', 'on-destroy-popper');
              this.$emit('close')
          }
      },
      query (val) {
          this.doQuery(val)
      }
  }
}
</script>
