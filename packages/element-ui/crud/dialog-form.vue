<template>
  <component :is="dialogType"
             lock-scroll
             :destroy-on-close="crud.tableOption.dialogDestroy"
             class="avue-dialog"
             :wrapperClosable="crud.tableOption.dialogClickModal"
             :direction="direction"
             v-dialogdrag="vaildData(crud.tableOption.dialogDrag,config.dialogDrag)"
             :class="b('dialog',{'fullscreen':fullscreen})"
             :custom-class="vaildData(crud.tableOption.customClass,config.customClass)"
             :fullscreen="fullscreen?fullscreen:(crud.isMobile?true:crud.tableOption.dialogFullscreen)"
             :modal-append-to-body="false"
             append-to-body
             :top="setPx(dialogTop)"
             :title="dialogTitle"
             :close-on-press-escape="crud.tableOption.dialogEscape"
             :close-on-click-modal="crud.tableOption.dialogClickModal"
             :modal="crud.tableOption.dialogModal"
             :show-close="crud.tableOption.dialogCloseBtn"
             :visible.sync="boxVisible"
             :size="size?size:width"
             :width="setPx(width)"
             @close="closeDialog">
    <div slot="title"
         :class="b('dialog__header')">
      <span class="el-dialog__title">{{dialogTitle}}</span>
      <div :class="b('dialog__menu')">
        <i @click="handleFullScreen"
           class="el-dialog__close el-icon-full-screen"></i>
      </div>
    </div>
    <el-scrollbar :style="styleName">
      <avue-form v-model="tableForm"
                 v-if="boxVisible"
                 ref="tableForm"
                 @submit="handleSubmit"
                 @error="handleError"
                 @reset-change="handleReset"
                 :upload-preview="crud.uploadPreview"
                 :upload-delete="crud.uploadDelete"
                 :upload-before="crud.uploadBefore"
                 :upload-after="crud.uploadAfter"
                 :upload-error="crud.uploadError"
                 :option="formOption">
        <!-- 循环form表单卡槽 -->
        <template slot-scope="scope"
                  v-for="item in crud.columnFormOption"
                  :slot="item.prop">
          <slot :name="item.prop"
                v-if="item.formslot"
                v-bind="Object.assign(scope,{
                  row:item.dynamic?scope.row:tableForm,
                  index:item.dynamic?scope.row.$index:crud.tableIndex,
                })"></slot>
        </template>
        <!-- 循环form表单错误卡槽 -->
        <template slot-scope="scope"
                  v-for="item in crud.columnFormOption"
                  :slot="item.prop+'Error'">
          <slot :name="item.prop+'Error'"
                v-bind="Object.assign(scope,{
                  row:tableForm,
                  index:crud.tableIndex,
                })"
                v-if="item.errorslot"></slot>
        </template>
        <!-- 循环form表单标签卡槽 -->
        <template slot-scope="scope"
                  v-for="item in crud.columnFormOption"
                  :slot="item.prop+'Label'">
          <slot :name="item.prop+'Label'"
                v-bind="Object.assign(scope,{
                  row:tableForm,
                  index:crud.tableIndex,
                })"
                v-if="item.labelslot"></slot>
        </template>
        <template slot="menuForm"
                  slot-scope="scope">
          <slot name="menuForm"
                v-bind="Object.assign(scope,{
                    type:boxType
                  }) "></slot>
        </template>
      </avue-form>
    </el-scrollbar>
  </component>
</template>

<script>
import { filterDefaultParams } from 'utils/util'
import create from "core/create";
import locale from "../../core/common/locale";
import config from "./config";
export default create({
  name: "crud",
  mixins: [locale],
  inject: ["crud"],
  data () {
    return {
      config: config,
      boxType: "",
      fullscreen: false,
      size: null,
      boxVisible: false,
      boxHeight: 0,
      tableForm: {},
      index: -1
    };
  },
  props: {
    value: {
      type: Object,
      default: () => {
        return {};
      }
    }
  },
  watch: {
    boxVisible (val) {
      if (val) {
        this.$nextTick(() => {
          this.initFun()
        })
      }
    },
    value: {
      handler () {
        this.formVal();
      },
      deep: true
    },
    tableForm: {
      handler () {
        this.$emit("input", this.tableForm);
      },
      deep: true
    }
  },
  computed: {
    styleName () {
      return {
        height: this.dialogHeight,
        overflow: 'hidden'
      }
    },
    isView () {
      return this.boxType === 'view'
    },
    isAdd () {
      return this.boxType === 'add'
    },
    isEdit () {
      return this.boxType === 'edit'
    },
    direction () {
      return this.crud.tableOption.dialogDirection
    },
    width () {
      return this.vaildData(this.crud.tableOption.dialogWidth + '', this.crud.isMobile ? '100%' : config.dialogWidth + '')
    },
    dialogType () {
      return this.isDrawer ? 'elDrawer' : 'elDialog'
    },
    dialogTop () {
      return this.crud.tableOption.dialogTop || config.dialogTop
    },
    isDrawer () {
      return this.crud.tableOption.dialogType === 'drawer';
    },
    dialogHeight () {
      if (this.isDrawer) {
        return 'calc(100% - 100px)';
      }
      if (this.crud.tableOption.dialogHeight === config.dialogHeight) {
        return this.setPx(config.clientHeight - 3 * this.dialogTop);
      }
      return this.setPx(this.crud.tableOption.dialogHeight);
    },
    formOption () {
      let option = this.deepClone(this.crud.tableOption);
      option.boxType = this.boxType;
      option.column = this.crud.propOption;
      option.printBtn = false;
      option.mockBtn = false;
      if (this.isView) {
        option.menuBtn = false;
        option.detail = true;
      } else {
        if (!option.menuPosition) option.menuPosition = 'right'
        if (this.isAdd) {
          option.submitBtn = option.saveBtn;
          option.submitText = this.crud.menuIcon('saveBtn');
          option.submitIcon = option.saveBtnIcon || config.saveBtnIcon
        } else if (this.isEdit) {
          option.submitBtn = option.updateBtn;
          option.submitText = this.crud.menuIcon('updateBtn');
          option.submitIcon = option.updateBtnIcon || config.updateBtnIcon
        }
        option.emptyBtn = option.cancelBtn;
        option.emptyIcon = option.cancelBtnIcon || config.cancelBtnIcon;
        option.emptyText = this.crud.menuIcon('cancelBtn')
      }
      //不分组的表单不加载字典
      if (!this.crud.isGroup) {
        option.dicFlag = false;
        option.dicData = this.crud.DIC;
      }
      return option;
    },
    dialogTitle () {
      const key = `${this.boxType}`;
      if (!this.validatenull(this.boxType)) {
        return this.crud.tableOption[key + 'Title'] || this.t(`crud.${key}Title`);
      }
    }
  },
  created () {
    this.crud.updateDic = this.updateDic;
  },
  methods: {
    handleFullScreen () {
      if (this.isDrawer) {
        if (this.validatenull(this.size)) {
          this.size = '100%'
        } else {
          this.size = ''
        }
      } else {
        if (this.fullscreen) {
          this.fullscreen = false;
        } else {
          this.fullscreen = true;
        }
      }

    },
    handleReset () {
      this.closeDialog();
    },
    handleError (error) {
      this.crud.$emit('error', error)
    },
    handleSubmit () {
      if (this.isAdd) {
        this.rowSave();
      } else if (this.isEdit) {
        this.rowUpdate();
      }
    },
    initFun () {
      ['clearValidate', 'validate'].forEach(ele => {
        this.crud[ele] = this.$refs.tableForm[ele]
      })
    },
    formVal () {
      Object.keys(this.value).forEach(ele => {
        this.tableForm[ele] = this.value[ele];
      });
    },
    //清空表单
    resetForm () {
      this.$refs["tableForm"].resetForm();
      this.$emit("input", this.tableForm);
    },
    // 保存
    rowSave () {
      this.$refs["tableForm"].validate(vaild => {
        if (!vaild) return;
        this.crud.$emit(
          "row-save",
          filterDefaultParams(this.tableForm, this.crud.tableOption.translate),
          this.closeDialog,
          this.$refs.tableForm.hide
        );
      });
    },
    // 更新
    rowUpdate () {
      this.$refs["tableForm"].validate(vaild => {
        if (!vaild) return;
        const index = this.crud.tableIndex;
        this.crud.$emit(
          "row-update",
          filterDefaultParams(this.tableForm, this.crud.tableOption.translate),
          this.index,
          this.closeDialog,
          this.$refs.tableForm.hide
        );
      });
    },
    closeDialog (row, index) {
      const callback = () => {
        if (this.isEdit) {
          let obj = this.findObject(this.crud.data, row[this.crud.rowKey], this.crud.rowKey);
          obj = Object.assign(obj, row);
        } else if (this.isAdd) {
          const callback = (list = [], index) => {
            this.validatenull(index) ? list.push(row) : list.splice(index, 0, row);
          }
          if (this.crud.isTree) {
            if (!row.children) row.children = []
            if (this.crud.vaildParent(row)) {
              callback(this.crud.data, index)
            } else {
              let parent = this.findObject(this.crud.data, row.parentId, this.crud.rowKey);
              if (parent === undefined) {
                return callback(this.crud.data, index)
              }
              if (!parent.children) {
                parent.hasChildren = true
                parent.children = []
              }
              callback(parent.children, index)
            }
          } else {
            callback(this.crud.data, index)
          }
        }
      }
      if (row) callback();
      this.crud.tableIndex = -1;
      this.tableForm = {};
      this.hide();


    },

    updateDic (prop, list) {
      const column = this.findObject(this.crud.propOption, prop);
      if (this.validatenull(list) && !this.validatenull(column.dicUrl)) {
        sendDic({
          url: column.dicUrl,
          method: column.dicMethod,
          query: column.dicQuery,
          resKey: (column.props || {}).res,
          formatter: column.dicFormatter
        }).then(list => {
          this.$set(this.crud.DIC, prop, list);
        });
      } else {
        this.$set(this.crud.DIC, prop, list);
      }
    },
    // 隐藏表单
    hide () {
      const callback = () => {
        this.crud.updateDic = this.updateDic;
        this.$nextTick(() => {
          this.boxVisible = false;
        });
      };
      if (typeof this.crud.beforeClose === "function") {
        this.crud.beforeClose(callback, this.boxType);
      } else {
        callback();
      }
    },
    // 显示表单
    show (type, index = -1) {
      this.index = index;
      this.boxType = type;
      const callback = () => {
        this.$nextTick(() => {
          this.boxVisible = true;
          this.crud.updateDic = (prop, list) => {
            this.$refs.tableForm.updateDic(prop, list, (res) => {
              this.updateDic(prop, res)
            })
          }
        });
      };
      if (typeof this.crud.beforeOpen === "function") {
        this.crud.beforeOpen(callback, this.boxType);
      } else {
        callback();
      }
    }
  }
});
</script>
