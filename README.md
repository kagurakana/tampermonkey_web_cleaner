# 描述
使用油猴脚本屏蔽网站的内容，例如弹幕，评论，用户名等。

# 如何使用
首先安装油猴脚本，油猴脚本是一个浏览器插件，chorme和新版edge应该都可以找到，下载插件之后点击添加新脚本：

![QQ截图20210518213625.png](https://cdn.kagurakana.xyz/QQ截图20210518213625.png@webp)

在编辑器中添加下面的代码：
```
// ==UserScript==
// @name         RegExp Cleaner master
// @namespace    http://tampermonkey.net/
// @version      0.1
// @description  Chean whatever you don't want to see with RegExp.***use this script calefully, it may have side effects. ***|使用正则清除网页上任何你不想看到的内容 ***请谨慎使用，可能产生副作用***
//@match           *://*/*
// @grant        none
// ==/UserScript==

let jsonHelper = {
  stringifyJson(json) {
    return JSON.stringify(json, (k, v) => {
      // RegExp -> json
      if (v instanceof RegExp) {
        return v.toString();
      }
      return v;
    });
  },
  parseJson(jsonStr) {
    return JSON.parse(jsonStr, (k, v) => {
      try {
        // JSON -> RegExp
        if (eval(v) instanceof RegExp) {
          return eval(v);
        }
      } catch (e) {
        // nothing
      }

      return v;
    });
  },
};

class _Cleaner {
  constructor() {
    this.mode = 0;
    this.hit = "";
    this.storedConfig = {
      common: {
        url: null,
        pattenRegExps: [], //
      },
      "www.douyu.com": {
        url: "douyu.com",
        pattenRegExps: [], // /^[?？]+$/
      },
      "www.bilibili.com": {
        url: "bilibili.com",
        pattenRegExps: [],
      },
    };
    this.template = `
    <div id="cleaner" class="cleaner-dom">
    <div class="add-rule-btn cleaner-dom" title="添加屏蔽|鼠标中键重置">+</div>
    <div class="add-rule-card cleaner-dom">
      <h1>添加规则</h1>
      <div class="input-group">
        <input
          type="text"
          class="add-input"
          placeholder="RegExp like /foo/"
        />
        <button class="add-btn">添加</button>
      </div>
      <h2>现有规则</h2>
      <ul class="rule-list">
        <li>1d</li>
        <li>2dsfsd</li>
        <li>3sdfs</li>
        <li>4sd</li>
        <li>5a</li>
      </ul>
      <button class="reset-btn">重置</button>
    </div>
  </div>
    `;
    this.styleText = `
    #cleaner * {
    padding: 0;
    margin: 0;
    box-sizing: border-box;
  }
  .add-rule-btn {
    position: fixed;
    cursor: pointer;
    top: 70%;
    right: 0;
    z-index: 10000;
    width: 40px;
    height: 40px;
    line-height: 40px;
    font-size: 35px;
    background-color: #eee;
    color: #fff;
    text-align: center;
  }
  .darken{
    overflow: hidden;
  }
  .darken::before{
    content: "";
    position: fixed;
    left: 0;
    top: 0;
    width: 100vw;
    height: 100vh;
    background-color: rgba(0,0,0,0.8);
    z-index: 1000;
  }
  #cleaner .add-rule-card {
    display: none;
    z-index: 10000;
    background-color: #fff;
    width: 600px;
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    flex-direction: column;
    justify-content: center;
    align-items: center;
    box-shadow: 0 0 50px 5px rgba(0, 0, 0, 0.2);
    border-radius: 8px;
    padding: 10px;
  }
  #cleaner .add-rule-card h1,
  #cleaner .add-rule-card div,
  #cleaner .add-rule-card ul {
    margin: 10px;
    color: rgba(0, 0, 0, 0.85);
  }
  #cleaner .add-rule-card h1 {
    font-size: 24px;
  }
  #cleaner .add-rule-card h2 {
    font-size: 20px;
  }
  #cleaner .add-rule-card .input-group {
    display: flex;
    border-radius: 5px;
    overflow: hidden;
  }
  #cleaner .add-rule-card .input-group input {
    height: 40px;
    font-size: 16px;
    width: 350px;
    padding: 0 5px;
    border: none;
    background-color: #f1f2f4;
    outline: none;
  }
  #cleaner .rule-list li{
    list-style-type: none;
  }
  #cleaner .rule-list li::before{
    content: "◾ ";
  }
  #cleaner .add-btn,
  #cleaner .reset-btn {
    background-color: #fff;
    cursor: pointer;
    border: none;
    outline: none;
    color: #fff;
    font-size: 16px;
    padding: 5px;
    height: 40px;
    width: 60px;
  }
  #cleaner .add-btn {
    background-color: #448aff;
  }
  #cleaner .reset-btn {
    background-color: #ff5252;
    border-radius: 4px;
  }
    `;

    localStorage["_Cleaner_config_"] ||
      (localStorage["_Cleaner_config_"] = jsonHelper.stringifyJson(
        this.storedConfig
      ));
    this.initDom();
    this.clean();
  }
  addKeyWord(keyWordRegExp) {
    keyWordRegExp = eval(keyWordRegExp);
    let tempConfig = jsonHelper.parseJson(localStorage["_Cleaner_config_"]);
    let site = location.hostname;
    if (typeof tempConfig[site] !== "undefined") {
      tempConfig[site].pattenRegExps.push(keyWordRegExp);
    } else {
      tempConfig[site] = {
        url: site,
        pattenRegExps: [keyWordRegExp],
      };
    }
    localStorage["_Cleaner_config_"] = jsonHelper.stringifyJson(tempConfig);
    this.clean();
  }
  refershKeyWord() {
    localStorage["_Cleaner_config_"] = jsonHelper.stringifyJson(
      this.storedConfig
    );
    this.updateRuleList();
  }
  listKeyWord() {
    let configToList = jsonHelper.parseJson(localStorage["_Cleaner_config_"]);
  }
  initDom() {
    // add doms
    let cleanerContainer = document.createElement("div");
    cleanerContainer.classList.add("cleaner-dom");
    cleanerContainer.innerHTML = this.template;
    let cleanerStyle = document.createElement("style");
    cleanerStyle.innerHTML = this.styleText;
    document.body.appendChild(cleanerContainer);
    document.head.appendChild(cleanerStyle);
    this.updateRuleList();
    // add events
    let addRuleBtn = document.querySelector("#cleaner .add-btn");
    let ruleInput = document.querySelector("#cleaner .add-input");
    let miniBtn = document.querySelector("#cleaner .add-rule-btn");
    let resetBtn = document.querySelector("#cleaner .reset-btn");
    let cleanerDom = document.querySelector("#cleaner");
    cleanerDom.addEventListener("click", (e) => {
      e.stopPropagation();
    });
    ruleInput.addEventListener("keypress", (e) => {
      if (e.keyCode === 13) {
        this.addKeyWord(ruleInput.value);
        ruleInput.value = "";
        this.updateRuleList();
      }
    });
    addRuleBtn.addEventListener("click", () => {
      this.addKeyWord(ruleInput.value);
      ruleInput.value = "";
      this.updateRuleList();
    });
    resetBtn.addEventListener("click", () => {
      this.refershKeyWord();
    });
    miniBtn.addEventListener("click", () => {
      this.showDom();
    });
    miniBtn.addEventListener("mousedown", (e) => {
      if (e.button === 1) {
        e.preventDefault();
        localStorage["_Cleaner_config_"] = "";
      }
    });
    document.body.addEventListener("click", () => {
      this.hideDom();
    });
  }
  getRules() {
    let config = jsonHelper.parseJson(localStorage["_Cleaner_config_"]);
    let allPattenExps = config.common.pattenRegExps.slice();
    for (let key in config) {
      if (location.href.includes(config[key].url)) {
        allPattenExps = allPattenExps.concat(config[key].pattenRegExps);
      }
    }
    return allPattenExps;
  }
  // if rule added, update rule list
  updateRuleList() {
    // init ul list
    let ulList = document.querySelector("#cleaner .rule-list");
    ulList.innerHTML = "";
    console.log(this.getRules());
    this.getRules().forEach((rule) => {
      let li = document.createElement("li");
      li.classList.add("cleaner-dom");
      li.innerText = rule;
      ulList.appendChild(li);
    });
  }
  clean() {
    let allPattenExps = this.getRules();
    function mountedClean() {
      let all = document.querySelectorAll("*");
      [].forEach.call(all, (item) => {
        allPattenExps.forEach((exp) => {
          if (shouldClean(exp, item)) {
            item.remove();
            console.log("已屏蔽初始元素：" + item.textContent);
          }
        });
      });
    }
    // when rule added, chean whole dom
    mountedClean();
    // when document ready, clean whole dom
    window.onload = mountedClean;
    //init MutationObserver, when new content added, using mutationObserver watch content
    updatedClean();

    function updatedClean() {
      const callback = function (mutationsList) {
        mutationsList.forEach((mutation) => {
          let addNodes = mutation.addedNodes;
          if (addNodes.length !== 0 && addNodes[0].nodeType === 1) {
            allPattenExps.forEach((exp) => {
              if (shouldCleanDeep(exp, addNodes[0])) {
                addNodes[0].remove();
                console.log("已屏蔽元素：" + addNodes[0].textContent);
              }
            });
          }
        });
      };
      const observer = new MutationObserver(callback);
      const MutationConfig = {
        attributes: false,
        childList: true,
        subtree: true,
      };
      observer.observe(document.body, MutationConfig);
    }

    // when window.ready, clean dom
    function shouldClean(exp, dom) {
      return (
        dom.style.display != "null" &&
        exp.test(dom.textContent) &&
        dom.childNodes[0].nodeType === 3 &&
        !dom.classList.contains("cleaner-dom") &&
        (!dom.offsetParent || dom.offsetParent.className !== "add-rule-card") &&
        !/(STYLE)|(SCRIPT)|(BODY)/.test(dom.nodeName.toUpperCase())
      );
    }
    // while sync content added (AJAX...), chean whole content
    function shouldCleanDeep(exp, dom) {
      return (
        dom.style.display != "null" &&
        exp.test(dom.textContent) &&
        !dom.classList.contains("cleaner-dom") &&
        (!dom.offsetParent || dom.offsetParent.className !== "add-rule-card") &&
        !/(STYLE)|(SCRIPT)|(BODY)/.test(dom.nodeName.toUpperCase())
      );
    }
  }
  showDom() {
    document.body.classList.add("darken");
    let cleanerContainer = document.querySelector("#cleaner .add-rule-card ");
    cleanerContainer.style.display = "flex";
  }
  hideDom() {
    document.body.classList.remove("darken");
    let cleanerContainer = document.querySelector("#cleaner .add-rule-card ");
    cleanerContainer.style.display = "none";
  }
  isVisibleDom() {
    let cleanerContainer = document.querySelector("#cleaner .add-rule-card ");
    let display = cleanerContainer.style.display;
    if (display === "flex" || display === "") {
      return true;
    }
    return false;
  }
}
let _cleaner = new _Cleaner();


```
添加完毕后点击确定，之后新打开一个浏览器标签页，查看油猴脚本是否起作用，如果起作用，页面右下角会出现一个加号按钮：
![QQ截图20210518213859.png](https://cdn.kagurakana.xyz/QQ截图20210518213859.png@webp)
点击加号按钮可以在这个界面中添加正则关键字：

![QQ截图20210518221325.png](https://cdn.kagurakana.xyz/QQ截图20210518221325.png@webp)

例如添加以下代码，就可以屏蔽所有全是1的弹幕：
```
/^1+$/
```
可以在控制台看到屏蔽的东西：

![QQ截图20210518221701.png](https://cdn.kagurakana.xyz/QQ截图20210518221701.png@webp)

# 屏蔽刻不容缓
![lajihua.png](https://cdn.kagurakana.xyz/lajihua.png@webp)
上面是斗鱼的英雄联盟比赛直播间，弹幕上一堆刷问号的，我个人是比较烦弹幕刷问号的，
```
/^(?:\?|？)*(?:\?|？)$/
```
爽到！

![QQ截图20210518222022.png](https://cdn.kagurakana.xyz/QQ截图20210518222022.png@webp)

# 原理
清理脚本会在两个时机进行dom清理：
- 网页完全加载后（window.onload）
- 网页body中dom变动后（mutationObserver）

在网页加载完毕后，脚本会选择所有DOM，将配置项（存储在localStorage中的_Cleaner_config_）当前域名下的正则数组都遍历一遍<black>就硬循环</black>，找到包含屏蔽元素的最小DOM进行删除。

在有新的内容加入时（例如弹幕或其他根据AJAX生成的DOM），借助MutationObserver获取到DOM元素，根据当前域名下的正则检查innerText，如果匹配到就删除。
```
{
      "common": {
        url: null,
        pattenRegExps: [], 
      },
      "www.douyu.com": { // location.hostName
        url: "douyu.com",  // 域名匹配，一般与location.hostName保持一致
        pattenRegExps: [], // 正则
      },
      "www.bilibili.com": {
        url: "bilibili.com",
        pattenRegExps: [],
      },
      // ...
};
```
## 添加关键词
添加关键词有两种方法，一种是直接写在上面油猴脚本的config中，好处是在localStorage清除后也会保留配置信息。坏处是不方便得修改jio本。

另外一种方法就是点加号按钮，在弹出框中添加规则，脚本会将会修改config并存储至localStorage中。缺点是清除localStorage后会失效😔
# 副作用
由于脚本是对**所有DOM**进行关键词检测，所以可能会遇到正则规则不严谨导致脚本将页面内意想不到的DOM进行屏蔽。
ps：你可以修改脚本最顶部的`//@match  *://*/*` 为需要使用脚本的网站。