<script setup lang="ts">
import {
  ITable,
  IFieldMeta,
  INumberField,
  IDateTimeField,
  FieldType,
  ITextField,
  IEventCbCtx,
  Selection,
  ITableMeta,
  ToastType,
  IField,
  IFilterAll,
} from "@lark-base-open/js-sdk";
import { bitable } from "@lark-base-open/js-sdk";
import { pinyin } from "pinyin-pro";
import { ref, onMounted, shallowRef, computed, reactive } from "vue";
import { i18n } from "../locales/i18n";
import { WarningFilled, CaretBottom, Select } from "@element-plus/icons-vue";
// import { syntaxReferenceList } from "./Form";
const { t } = i18n.global;

/** 页面加载数据 */
const isTransformed = ref(false);
const sucessCounter = ref(0);
const failureCounter = ref(0);
const stopFlag = ref(false);
const finishFlag = ref(false);
const recordCount = ref(0);
const isLoading = ref(false);
const inputRef = shallowRef();
const my_resetTextValue = shallowRef(true);

/** 页面数据 */
const value = ref();

const regexText = ref("");
const modifierText = ref("");
const activeTable = shallowRef<ITable>();

const tableList = shallowRef<ITable[]>();
const tableNameList = ref<string[]>([]);
const selectedTableFieldList = shallowRef<IFieldMeta[]>([]);
const textValue = ref("");
const optionSet = ref<
  Set<{
    option: string;
    field: string;
  }>
>(new Set());
const optionArray = ref<
  {
    option: string;
    field: string;
  }[]
>([]);
const currentWord = ref("");
const ignoredText = ref("");
const curretnSelection = shallowRef<Selection>();

const usableOptionArray = computed(() => {
  const hardMatchArray = optionArray.value.filter((item) =>
    item.option.includes(currentWord.value)
  );
  const lowerCaseMatchArray = optionArray.value.filter((item) =>
    pinyin(item.option, { toneType: "none" })
      .replace(/\s/g, "")
      .startsWith(currentWord.value)
  );
  const initialMatchArray = optionArray.value.filter((item) => {
    let initial = "";
    pinyin(item.option, { toneType: "none" })
      .split(" ")
      .forEach((item) => {
        initial += item[0];
      });
    return initial.startsWith(currentWord.value);
  });
  const mergedArray = [
    ...new Set([
      ...hardMatchArray,
      ...initialMatchArray,
      ...lowerCaseMatchArray,
    ]),
  ];
  const res = mergedArray.filter((item) => item != undefined && item != null);
  // console.log("mergedArray", res);
  return res;
});
const currentFirstOption = computed(() => {
  if (currentWord.value != "") {
    return usableOptionArray.value[0].option || "";
  }
  return "";
});

const fieldList = shallowRef<IFieldMeta[]>([]);
const activeTableName = ref("");
const activeTableId = ref("");
const errorMsg = ref("");
const replaceText = ref("");
const originField = shallowRef<IFieldMeta>();
const targetField = shallowRef<IFieldMeta>();

const selectedTableName = ref("");
const selectedTable = shallowRef<ITable>();
const selectedFieldList = shallowRef<IFieldMeta[]>([]);
const option = ref("");
const transformPatternList = computed(() => {
  return [
    {
      label: t("mode.test"),
      value: "test",
      desc: t("mode.testDesc"),
    },
    {
      label: t("mode.match"),
      value: "match",
      desc: t("mode.matchDesc"),
    },
    {
      label: t("mode.replace"),
      value: "replace",
      desc: t("mode.replaceDesc"),
    },
    {
      label: t("mode.split"),
      value: "split",
      desc: t("mode.splitDesc"),
    },
  ];
});
const modifierList = computed(() => {
  return [
    {
      label: t("modifier.global"),
      value: "g",
    },
    {
      label: t("modifier.ignoreCase"),
      value: "i",
    },
    {
      label: t("modifier.multiLine"),
      value: "m",
    },
    {
      label: t("modifier.newline"),
      value: "s",
    },
  ];
});

const sampleList = computed(() => {
  return [
    {
      label: t("sample.extractPhoneNumber"),
      value: "extractPhoneNumber",
      regex: "",
    },
    {
      label: t("sample.IdCard"),
      value: "IdCard",
      regex: "",
    },
    {
      label: t("sample.PostalCode"),
      value: "PostalCode",
      regex: "",
    },
    // {
    //   label: t("sample.IPAddress"),
    //   value: "IPAddress",
    //   regex: ""
    // },
  ];
});
const syntaxReferenceList = computed(() => {
  return i18n.global.tm("syntaxReferenceList");
});

/**
 * 提交转换
 */
const handleConfirm = async () => {
  if (
    curretnSelection.value?.tableId &&
    curretnSelection.value.fieldId &&
    curretnSelection.value?.recordId
  ) {
    const table = await bitable.base.getTableById(
      curretnSelection.value.tableId
    );
    const res = await table.setRecord(curretnSelection.value.recordId, {
      fields: {
        [curretnSelection.value.fieldId]: textValue.value,
      },
    });
    if (res && my_resetTextValue.value === true) {
      textValue.value = "";
    }
  } else {
    await bitable.ui.showToast({
      toastType: ToastType.info,
      message: "请选择目标输入框",
    });
  }
};

/**
 * 获取当前选中的表信息
 */
const getCurrentSelection = async () => {
  const selection = await bitable.base.getSelection();
  if (selection.tableId && selection.viewId) {
    activeTable.value = await bitable.base.getTableById(selection.tableId);
    activeTableId.value = activeTable.value.id;
    activeTableName.value = await activeTable.value.getName();
    const view = await activeTable.value.getViewById(selection.viewId);
    fieldList.value = await view.getFieldMetaList();
  }
};

/**
 * 字段变化时触发
 */
const handleFieldChange = () => {
  // console.log("handleFieldChange");
  getCurrentSelection();
  originField.value = undefined;
  targetField.value = undefined;
};

/**
 * 选择变化时触发
 * 仅处理table变化时
 */
const handleSelectionChange = async (e: IEventCbCtx<Selection>) => {
  if (e.data.tableId != activeTableId.value) {
    handleFieldChange();
  }
  curretnSelection.value = await bitable.base.getSelection();
};

/**
 * 获取TableMeta列表
 */
const getTableList = async (): Promise<ITable[]> => {
  return await bitable.base.getTableList();
};

/**
 * 获取选中table的字段列表
 */
const getSelectedTableFieldList = async (): Promise<void> => {
  selectedFieldList.value = []; // 重置
  selectedTable.value = await bitable.base.getTable(selectedTableName.value);
  // selectedTable.value = await bitable.base.getTable(
  //   selectedTableName.value
  // );
  selectedTableFieldList.value = await selectedTable.value.getFieldMetaList();
  // console.log("selectedTableFieldList", selectedTableFieldList.value);
};

const getSelectedFieldRecords = async () => {
  optionSet.value.clear();
  const recordIdList = (await selectedTable.value?.getRecordIdList()) || [];
  for (const fieldMetaInfo of selectedFieldList.value) {
    const field = await selectedTable.value?.getFieldById(fieldMetaInfo.id);
    for (const recordId of recordIdList) {
      const val = await field?.getValue(recordId);
      const fieldName = await field?.getName();
      // const res = val != null ? val[0].text : null;
      const res = val != null ? val.map((item) => item.text).join("") : null;
      if (res) {
        optionSet.value.add({
          option: res,
          field: fieldName || "",
        });
      }
    }
  }
  optionArray.value = Array.from(optionSet.value);
};

const onInputChange = () => {
  // console.log("输入改变");
  currentWord.value = removePrefix(textValue.value, ignoredText.value);
  // console.log("currentWord: ", currentWord.value);
  if (
    findElementWithSubstring(
      new Set(Array.from(optionSet.value.values()).map((item) => item.option)),
      currentWord.value
    )
  ) {
    if (currentWord.value != "") {
      // inputRef.value.blur();
    }
  } else {
    ignoredText.value = textValue.value;
    currentWord.value = "";
  }
};

const handleEnterKey = async (event) => {
  if (event.key === "Enter") {
    if (currentWord.value != "") {
      // console.log("currentWord.value", currentWord.value);
      handleOptionClick(currentFirstOption.value);
      // console.log("textValue", textValue.value);
      setTimeout(() => {
        textValue.value = textValue.value.replace(/\n+$/, "");
      }, 10);
    }
  }
};

const handleOptionClick = (item: string) => {
  textValue.value = removeSuffix(textValue.value, currentWord.value);
  textValue.value += item;
  ignoredText.value = textValue.value;
  currentWord.value = "";
  inputRef.value.focus();
};

function removePrefix(A: string, B: string) {
  const escapedB = B.replace(/\\/g, "\\\\");
  const regex = new RegExp(`^${escapedB}`);
  return A.replace(regex, "");
}

function removeSuffix(A: string, B: string) {
  const escapedB = B.replace(/\\/g, "\\\\");
  const regex = new RegExp(`${escapedB}$`);
  return A.replace(regex, "");
}

function findElementWithSubstring(set: Set<string>, substring: string) {
  const excludeArray = [" ", ";", "\n", "\t", ",", ".", "，", "。", "；"];
  for (const element of set) {
    if (excludeArray.includes(substring)) {
      return;
    }
    if (usableOptionArray.value.length != 0) {
      // console.log(usableOptionArray.value);
      // console.log("element", element);
      return true;
    }
  }
  return null; // 或者返回其他指定的值，表示未找到
}

/**
 * 组件挂载时触发
 */
onMounted(async () => {
  window.addEventListener("keydown", handleEnterKey);
  await getCurrentSelection();
  // 注册选择变化监听器
  const off = bitable.base.onSelectionChange(handleSelectionChange);
  // 注册字段修改监听器
  const offFieldModify = activeTable.value?.onFieldModify(handleFieldChange);
  // 注册字段添加监听器
  const offFieldAdd = activeTable.value?.onFieldAdd(handleFieldChange);
  // 注册字段删除监听器
  const offFieldDelete = activeTable.value?.onFieldDelete(handleFieldChange);
  const table = await bitable.base.getActiveTable();
  tableList.value = await getTableList();
  for (const table of tableList.value) {
    const name = await table.getName();
    tableNameList.value.push(name);
  }
});

type SupportField = ITextField | IDateTimeField | INumberField;
</script>

<template>
  <!-- <div style="margin-bottom: 15px">
    <div>
      <el-alert show-icon type="info" style="border-radius: 6px">
        <template #title>
          {{ t("info.guideDesc") }}
          <a :href="t(`info.url`)" target="_blank">{{ t(`info.guide`) }}</a>
        </template>
      </el-alert>
    </div>
  </div> -->
  <div style="padding-bottom: 10px">
    <el-form label-position="top">
      <!-- 当前数据表 {{ 数据表名称 }} -->
      <el-row
        style="
          display: flex;
          align-items: center;
          padding-bottom: 15px;
          font-size: large;
        "
      >
        {{ t("form.currentTable") }}
        <el-tag
          type="primary"
          size="medium"
          style="margin-left: 10px; border-radius: 10px"
        >
          {{ activeTableName }}
        </el-tag>
      </el-row>
      <!-- 表单 -->
      <div>
        <!-- 关联联想数据表 -->
        <el-form-item :label="t(`form.associateTable`)">
          <el-select
            value-key="name"
            v-model="selectedTableName"
            class="m-2"
            :placeholder="t(`form.associateTable`)"
            :change="getSelectedTableFieldList()"
            style="width: 100%"
          >
            <el-option
              v-for="(item, _) in tableNameList"
              :key="_"
              :label="item"
              :value="item"
            >
              <span style="float: left">{{ item }}</span>
            </el-option>
          </el-select>
        </el-form-item>

        <!-- 关联联想字段 -->
        <el-form-item :label="t(`form.associateTableField`)">
          <el-select
            v-model="selectedFieldList"
            multiple
            collapse-tags
            max-collapse-tags="3"
            collapse-tags-tooltip="true"
            value-key="id"
            class="m-2"
            :placeholder="t(`form.associateTableField`)"
            :change="getSelectedFieldRecords()"
            style="width: 100%"
          >
            <el-option
              v-for="(item, _) in selectedTableFieldList.filter(
                (item) => item.type === FieldType.Text
              )"
              :key="_"
              :label="item.name"
              :value="item"
            />
          </el-select>
        </el-form-item>

        <!-- 要填入的字段 -->
        <el-form-item :label="t(`form.content`)">
          <el-input
            ref="inputRef"
            clearable
            v-model="textValue"
            :autosize="{ minRows: 2, maxRows: 10 }"
            placeholder="填写内容"
            type="textarea"
            style="width: 100%"
            :change="onInputChange()"
          />
        </el-form-item>
        <el-form-item label="联想内容" v-if="optionArray.length != 0">
          <div
            v-if="optionArray.length != 0"
            style="
              border: 1px solid rgb(228, 231, 237);
              border-radius: 6px;
              padding-inline: 4px;
              padding-top: 4px;
              width: 100%;
            "
          >
            <el-scrollbar max-height="200px">
              <div v-for="(item, _) in usableOptionArray">
                <el-card
                  class="option-card"
                  :ref="currentWord && _ == 0 ? `selectableCardRef` : ''"
                  @keydown.enter="handleEnterKey"
                  :class="currentWord && _ == 0 ? 'first' : ''"
                  shadow="hover"
                  @click="handleOptionClick(item.option || ``)"
                  style="margin-bottom: 4px; border-radius: 6px"
                >
                  <el-row style="display: flex; justify-content: space-between">
                    <span>
                      {{ item.option }}
                    </span>
                    <span>
                      <el-tag type="info">
                        <span class="info">
                          {{ item.field }}
                        </span>
                      </el-tag>
                    </span>
                  </el-row>
                </el-card>
              </div>
            </el-scrollbar>
          </div>
        </el-form-item>
      </div>
    </el-form>
    <el-col :span="24"> </el-col>
    <div style="color: rgb(20, 86, 240)">
      <el-button
        :disabled="isLoading"
        @click="handleConfirm"
        color="rgb(20, 86, 240)"
      >
        {{ isLoading ? t("status.transforming") : t("status.confirm") }}
      </el-button>
      <el-tag @click="my_resetTextValue = !my_resetTextValue" class="ml-2 clickable" type="info" style="margin-left: 10px">
        <span class="info" style="display: flex; align-items: center; justify-content: center;"> 
          <el-icon v-if="my_resetTextValue" style="color: rgb(20, 86, 240); font-size: x-large; margin-right: 4px;"><Select /></el-icon>
          填入后清空输入框
         </span>
         </el-tag>
    </div>
  </div>
</template>

<style scoped>
.form :deep(.el-form-item__label) {
  font-size: 16px;
  color: var(--el-text-color-primary);
  margin-bottom: 0;
}

.form :deep(.el-form-item__content) {
  font-size: 16px;
}

.modeSelect :deep(.el-input__inner) {
  font-weight: bold;
}

:deep(.el-form-item__label) {
  font-size: medium;
  font-weight: medium;
}

:deep(.el-card__body) {
  /* padding-left: 0px;
  padding-right: 0px; */
  padding-top: 0px;
  padding-bottom: 0px;
}

:deep(.el-input-group__prepend) {
  padding-top: 0px;
  padding-right: 5px;
  padding-bottom: 0px;
  padding-left: 5px;
}

:deep(.el-input-group__append) {
  padding-top: 0px;
  padding-right: 5px;
  padding-bottom: 0px;
  padding-left: 5px;
}

:deep(.el-input__wrapper) {
  border-radius: 6px;
}

:deep(.el-button) {
  border-radius: 6px;
}

:deep(.el-tag__content) {
  color: rgb(20, 86, 240);
}
.el-tag__content .info {
  color: #73767a;
  font-size: x-small;
}

.el-select-dropdown.is-multiple .el-select-dropdown__item.selected {
  color: rgb(20, 86, 240);
}

.el-select-dropdown.is-multiple .el-select-dropdown__item.selected {
  color: rgb(20, 86, 240);
}

.el-select-dropdown__item.selected {
  color: rgb(20, 86, 240);
}

.card-header {
  display: flex;
  /* margin-left: -10px; */
  /* margin-right: -10px; */
  justify-content: space-between;
  align-items: center;
}

.text {
  font-size: 14px;
}

.item {
  margin-bottom: 18px;
}

.expand-icon :hover {
  cursor: pointer;
}

.clickable :hover {
  cursor: pointer;
}

.clickable {
  border-radius: 8px;
}

.tag {
  margin-right: 5px;
}

.tag-row {
  width: 100%;
  padding-bottom: 15px;
  justify-content: left;
  display: flex;
  flex-wrap: wrap;
  row-gap: 5px;
}
</style>

<style>
.el-popper__arrow {
  display: none !important;
}

.el-popper {
  border-radius: 6px !important;
  margin-top: -8px !important;
}
.option-card :hover {
  cursor: pointer;
  color: rgb(20, 86, 240);
  background-color: rgb(240, 242, 245);
  font-weight: bold;
}

.first {
  color: rgb(20, 86, 240);
  background-color: rgb(240, 242, 245);
  font-weight: bold;
}
.el-switch__label.is-active {
  color: rgb(20, 86, 240);
}
.el-switch.is-checked .el-switch__core {
  background-color: rgb(20, 86, 240);
}
</style>
