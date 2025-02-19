<template>
  <div :style="`margin-top:${this.marginTop}`">
    <div class="log-filter">
      <template>
        <a-row type="flex">
          <a-col :span="18">
            <a-space>
              <slot name="before"></slot>
              <a-tooltip title="关键词高亮,支持正则(正则可能影响性能请酌情使用)">
                <a-input addonBefore="正则:/" placeholder="关键词高亮,支持正则" v-model="temp.searchValue" :style="`width: ${searchWidth}`" @pressEnter="onSearch">
                  <a-select slot="addonAfter" :value="'/' + this.regModifier" style="width: 80px">
                    <div @mousedown="(e) => e.preventDefault()" slot="dropdownRender" style="width: 300px; padding: 10px; cursor: pointer; background-color: #fff; border-radius: 5px">
                      <a-checkbox-group @change="regModifierChange" :value="regModifiers" :options="modifiers"> </a-checkbox-group>
                    </div>
                  </a-select>
                </a-input>
              </a-tooltip>

              <a-tooltip title="为避免显示内容太多而造成浏览器卡顿,限制只显示最后多少行日志。修改后需要回车才能生效">
                <a-input addonBefore="行数" :min="1" style="width: 200px" :disabled="!this.temp.logScroll" v-model="temp.tempLogShowLine" placeholder="显示行数" @pressEnter="onSearch">
                  <template slot="addonAfter">
                    <a-icon type="swap" @click="onSearch" />
                  </template>
                </a-input>
              </a-tooltip>
            </a-space>
          </a-col>
          <a-col v-if="this.extendBar" :span="6" style="text-align: right">
            <a-space>
              <a-tooltip title="有新内容后是否自动滚动到底部">
                <a-switch v-model="temp.logScroll" checked-children="自动滚动" un-checked-children="不滚动" />
              </a-tooltip>
              <a-tooltip title="清空当前缓冲区内容">
                <a-button type="link" style="padding: 0" @click="clearLogCache" icon="delete"><span style="margin-left: 2px">清空</span></a-button>
              </a-tooltip>
              <a-popover title="正则语法参考">
                <template slot="content">
                  <ul>
                    <li><b>.</b> - 除换行符以外的所有字符。</li>
                    <li><b>^</b> - 字符串开头。</li>
                    <li><b>$</b> - 字符串结尾。</li>
                    <li><b>\d,\w,\s</b> - 匹配数字、字符、空格。</li>
                    <li><b>\D,\W,\S</b> - 匹配非数字、非字符、非空格。</li>
                    <li><b>[abc]</b> - 匹配 a、b 或 c 中的一个字母。</li>
                    <li><b>[a-z]</b> - 匹配 a 到 z 中的一个字母。</li>
                    <li><b>[^abc]</b> - 匹配除了 a、b 或 c 中的其他字母。</li>
                    <li><b>aa|bb</b> - 匹配 aa 或 bb。</li>
                    <li><b>[^%&',;=?$\x22]+</b> - 含有^%&',;=?$\"等字符。</li>
                    <li><b>[\u4E00-\u9FA5A-Za-z0-9_]</b> - 中文、英文、数字包括下划线。</li>
                    <li><b>[\u4e00-\u9fa5]</b> - 汉字。</li>
                  </ul>
                </template>
                <a-button type="link" style="padding: 0" icon="unordered-list"><span style="margin-left: 2px">语法参考</span></a-button>
              </a-popover>
            </a-space>
          </a-col>
        </a-row>
      </template>
    </div>
    <pre class="log-view" :id="`${this.id}`" :style="`height:${this.height}`">{{ defText }}</pre>
  </div>
</template>

<script>
export default {
  name: "LogView",
  components: {
    // VNodes: {
    //   functional: true,
    //   render: (h, ctx) => ctx.props.vnodes,
    // },
  },
  computed: {
    regModifier() {
      return this.regModifiers.join("");
    },
  },
  props: {
    seg: {
      String,
      default: "</br>",
    },
    height: {
      String,
      default: "50vh",
    },
    marginTop: {
      String,
      default: "0px",
    },
    searchWidth: {
      String,
      default: "360px",
    },
    extendBar: {
      Boolean,
      default: true,
    },
  },
  data() {
    return {
      regReplaceText: "<b style='color:red'>$1</b>",
      regRemove: /<b[^>]*>([^>]*)<\/b[^>]*>/gi,
      searchReg: null,
      regModifiers: ["i", "g"],
      modifiers: [
        {
          label: "ignore - 不区分大小写",
          value: "i",
        },
        {
          label: "global - 全局匹配",
          value: "g",
        },
        {
          label: "multi line - 多行匹配",
          value: "m",
        },
        {
          label: "特殊字符圆点 . 中包含换行符 \n",
          value: "s",
        },
      ],
      // 日志内容
      logContextArray: [],
      temp: {
        logScroll: true,
        logShowLine: 500,
        tempLogShowLine: 500,
        searchValue: "",
      },
      defLogShowLine: 500,
      defId: "logScrollArea",
      defText: "loading context...",
      id: "",
    };
  },
  mounted() {
    this.id = this.defId + new Date().getTime();
    // let html = "<b><b style='color:OrangeRed;'>222</b></b>";
    // console.log(html.replace(this.regRemove, "$1").replace(this.regRemove, "$1"));
  },
  methods: {
    regModifierChange(checkedValues) {
      if (!checkedValues.length) {
        this.$notification.warn({
          message: "修饰符至少选择一个",
        });
        return;
      }
      this.regModifiers = checkedValues;
    },

    //
    appendLine(data) {
      if (!data) {
        return;
      }
      const dataArray = Array.isArray(data) ? data : [data];
      dataArray.forEach((item) => {
        item = item.replace(/[<>&]/g, function (match) {
          //  pos, originalText
          switch (match) {
            case "<":
              return "&lt;";
            case ">":
              return "&gt;";
            case "&":
              return "&amp;";
            case '"':
              return "&quot;";
          }
        });
        this.logContextArray.push(this.lineFormat(item));
      });

      //console.log(this.temp.logScroll);
      if (this.temp.logScroll) {
        this.logContextArray = this.logContextArray.slice(-this.temp.logShowLine);
      }

      // 自动滚动到底部
      this.$nextTick(() => {
        this.toHtml();
        const projectConsole = document.getElementById(this.id);
        if (!projectConsole) {
          return;
        }
        // console.log("111");
        if (this.temp.logScroll) {
          projectConsole.scrollTop = projectConsole.scrollHeight;
        }
      });
    },
    // 搜索
    onSearch() {
      //
      let logShowLineTemp = parseInt(this.temp.tempLogShowLine);
      logShowLineTemp = isNaN(logShowLineTemp) ? this.defLogShowLine : logShowLineTemp;
      logShowLineTemp = logShowLineTemp > 0 ? logShowLineTemp : 1;
      this.temp.logShowLine = logShowLineTemp;
      this.temp.tempLogShowLine = logShowLineTemp;
      //
      // console.log(this.regModifier);
      this.searchReg = this.temp.searchValue ? new RegExp("(" + this.temp.searchValue + ")", this.regModifier) : null;
      // console.log(this.searchReg);
      this.logContextArray = this.logContextArray.map((item) => this.lineFormat(item));
      this.$nextTick(() => {
        this.toHtml();
      });
    },
    toHtml() {
      const projectConsole = document.getElementById(this.id);
      if (!projectConsole) {
        return;
      }
      projectConsole.innerHTML = this.logContextArray.join(this.seg) || this.defText;
      return projectConsole;
    },
    lineFormat(item) {
      // console.log(item.match(this.regRemove), item.replace(this.regRemove, "$1"));
      item = item.replace(this.regRemove, "$1").replace(this.regRemove, "$1");
      if (this.searchReg) {
        item = item.replace(this.searchReg, this.regReplaceText);
      }
      if (item.match(/error|Exception/i)) {
        item = "<b style='color:OrangeRed'>" + item + "</b>";
      } else if (item.match(/WARNING|failed/i)) {
        item = "<b style='color:orange'>" + item + "</b>";
      }
      // item = "<li>" + item + "</li>";
      // console.log(item);
      return item;
    },
    clearLogCache() {
      this.logContextArray = [];
      this.$nextTick(() => {
        const projectConsole = document.getElementById(this.id);
        projectConsole.innerHTML = this.defText;
      });
    },
  },
};
</script>

<style scoped>
.log-view {
  padding: 5px;
  color: #fff;
  font-size: 14px;
  background-color: black;
  width: 100%;
  height: calc(100vh - 120px);
  overflow-y: auto;
  border: 1px solid #e2e2e2;
  border-radius: 5px 5px;
}
.log-filter {
  /* margin-top: -22px; */
  /* margin-bottom: 10px; */
  padding: 0 10px;
  padding-top: 0;
  padding-bottom: 10px;
  line-height: 0;
}
</style>
