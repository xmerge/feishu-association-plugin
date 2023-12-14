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
import { ref, onMounted, shallowRef, computed, reactive } from "vue";
import { i18n } from "../locales/i18n";
import { WarningFilled, CaretBottom } from "@element-plus/icons-vue";
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
const resetTextValue = ref(true);

/** 页面数据 */
const value = ref();

const regexText = ref("");
const modifierText = ref("");
const activeTable = shallowRef<ITable>();

const tableList = shallowRef<ITable[]>();
const tableNameList = ref<string[]>([]);
const selectedTableFieldList = shallowRef<IFieldMeta[]>([]);
const textValue = ref("");
const optionSet = ref<Set<string>>(new Set());
const optionArray = ref<string[]>([]);
const currentWord = ref("");
const ignoredText = ref("");
const curretnSelection = shallowRef<Selection>();
const usableOptionArray = computed(() => {
  return optionArray.value.filter((item) => item.includes(currentWord.value));
});
const currentFirstOption = computed(() => {
  if (currentWord.value != "") {
    return usableOptionArray.value[0];
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
 * 处理单位变化的函数
 */
const handleUnitChange = (): void => {
  // originField.value = undefined; // 重置原字段的值为undefined
  // targetField.value = undefined; // 重置目标字段的值为undefined
};

/**
 * 重置页面信息
 */
const handleReset = (): void => {
  errorMsg.value = "";
  isTransformed.value = true;
  isLoading.value = true;
  stopFlag.value = false;
  finishFlag.value = false;
  sucessCounter.value = 0;
  recordCount.value = 0;
  failureCounter.value = 0;
};

/**
 * 提取字段值
 * @param selectedField 源字段
 * @param recordId 记录Id
 */
const extractValueByRecordId = async (
  selectedField: SupportField,
  recordId: string
): Promise<string | null> => {
  const originFieldVar = await selectedField.getValue(recordId);
  if (originFieldVar === null) {
    return null;
  }
  let res: string | null = null;
  // 本身是 IDateTimeField || INumberField 类型
  if (originField.value?.type === FieldType.Number) {
    res = originFieldVar.toString();
  } else if (originField.value?.type === FieldType.Text) {
    // 本身是 ITextField 类型
    if (Array.isArray(originFieldVar) && originFieldVar[0].type == "text") {
      if (modifierText.value.includes("m")) {
        res = originFieldVar.map((object) => object.text).join("");
      } else {
        res = originFieldVar[0].text;
      }
      console.log("res ", res);
    }
  }
  return res;
};

/**
 * 设置字段值
 * @param targetSelectField 目标字段
 * @param recordId 记录Id
 * @param targetVal 目标值
 */
const setValueByRecordId = async (
  targetSelectField: ITextField,
  recordId: string,
  targetVal: string
): Promise<void> => {
  let res: Promise<void>;
  if (targetField.value?.type === FieldType.Text) {
    res = (targetSelectField as ITextField)
      .setValue(recordId, targetVal.toString())
      .then(() => {
        sucessCounter.value++;
      })
      .catch(() => {
        failureCounter.value++;
      });
  } else {
    return;
  }

  return res;
};

// /**
//  * 验证表单
//  */
// const fomrValidate = () => {
//   // 源字段和目标字段都为空
//   if (!(originField.value && targetField.value)) {
//     return t("message.emptyField");
//   }
//   // 转换模式为空
//   if (!selectedTable.value.value.length) {
//     return t("message.emptyMode");
//   }
//   // 正则表达式文本为空
//   if (!regexText.value.length) {
//     return t("message.emptyRegex");
//   }
//   // 正则表达式无效
//   if (!isRegexValid(regexText.value)) {
//     return t("message.wrongRegex");
//   }
//   // 验证通过
//   return null;
// };

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
    if (res && resetTextValue.value) {
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
 * 转换模版
 * @param mode 模式
 */
const handleClickSample = (mode: string) => {
  const regexes = {
    number: "[0-9]+",
    alpha: "[a-z]+",
    phoneNumber: "^1[0-9]{10}$",
    chinese: `[\\u4e00-\\u9fff]+`,
    extractPhoneNumber: "1[0-9]{10}",
    IdCard: `^[1-9]\\d{5}(18|19|20)\\d{2}((0[1-9])|(1[0-2]))(([0-2][1-9])|10|20|30|31)\\d{3}[0-9Xx]$`,
    PostalCode: `^[1-9]\\d{5}$`,
    IPAddress: `^((2[0-4]\\d|25[0-5]|[01]?\\d\\d?)\\.){3}(2[0-4]\\d|25[0-5]|[01]?\\d\\d?)$`,
  };
  regexText.value = regexes[mode];
};

/**
 *
 * @param originFieldText 源字段文本
 * @param regexExpression 正则表达式
 */
// const regexTranform = (
//   originFieldText: string,
//   regexExpression: string,
//   replaceText: string
// ) => {
//   let regex: RegExp;
//   if (modifierText.value.length) {
//     regex = new RegExp(regexExpression, modifierText.value);
//   } else {
//     regex = new RegExp(regexExpression);
//   }
//   if (selectedTable.value.value == "test") {
//     console.log(regex);
//     return regex.test(originFieldText) ? t("res.true") : t("res.false");
//   }
//   if (selectedTable.value.value == "match") {
//     const matches = originFieldText.match(regex);
//     console.log(matches);
//     return matches ? matches.join(" ") : "";
//   }
//   if (selectedTable.value.value == "replace") {
//     return originFieldText.replace(regex, replaceText);
//   }
//   if (selectedTable.value.value == "split") {
//     return originFieldText.split(regex).join(" ");
//   }
// };

/**
 * 停止转换
 */
const handleStop = () => {
  if (finishFlag.value) {
    return;
  }
  isLoading.value = false;
  stopFlag.value = true;
};

/**
 * 验证用户输入的字符串是否是合法的正则表达式
 * @param pattern 用户输入的字符串
 */
const isRegexValid = (pattern: string) => {
  try {
    new RegExp(pattern);
    return true;
  } catch (error) {
    return false;
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
  console.log("handleFieldChange");
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
      const res = val != null ? val[0].text : null;
      if (res) {
        optionSet.value.add(res);
      }
    }
  }
  optionArray.value = Array.from(optionSet.value);
};

const onInputChange = () => {
  console.log("输入改变");
  currentWord.value = removePrefix(textValue.value, ignoredText.value);
  console.log(currentWord.value);
  if (findElementWithSubstring(optionSet.value, currentWord.value)) {
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
      console.log("currentWord.value", currentWord.value);
      handleOptionClick(currentFirstOption.value.toString());
      console.log("textValue", textValue.value);
      setTimeout(() => {
        textValue.value = textValue.value.replace(/\n+$/, "");
      }, 10);
    }
  }
};

const handleOptionClick = (item: string) => {
  textValue.value += removePrefix(item, currentWord.value);
  ignoredText.value = textValue.value;
  currentWord.value = "";
  inputRef.value.focus();
};

function removePrefix(A: string, B: string) {
  const regex = new RegExp(`^${B}`);
  return A.replace(regex, "");
}

function findElementWithSubstring(set: Set<string>, substring: string) {
  const excludeArray = [" ", ";", "\n", "\t", ",", ".", "，", "。", "；"];
  for (const element of set) {
    if (excludeArray.includes(substring)) {
      return;
    }
    if (element.includes(substring)) {
      return element;
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
              <div
                v-for="(item, _) in optionArray.filter((item) =>
                  item.includes(currentWord)
                )"
              >
                <el-card
                  class="option-card"
                  :ref="currentWord && _ == 0 ? `selectableCardRef` : ''"
                  @keydown.enter="handleEnterKey"
                  :class="currentWord && _ == 0 ? 'first' : ''"
                  shadow="hover"
                  @click="handleOptionClick(item)"
                  style="margin-bottom: 4px; border-radius: 6px"
                >
                  {{ item }}
                </el-card>
              </div>
            </el-scrollbar>
          </div>
        </el-form-item>
      </div>
    </el-form>
    <el-col :span="24"> </el-col>
    <div style="color: rgb(20, 86, 240);">
      <el-button
        :disabled="isLoading"
        @click="handleConfirm"
        color="rgb(20, 86, 240)"
      >
        {{ isLoading ? t("status.transforming") : t("status.confirm") }}
      </el-button>
      <el-switch
        v-model="resetTextValue"
        active-text="填入后清空输入框"
        style="margin-left: 10px"
      />
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
