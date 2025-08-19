<script setup>
const serverURL = "/ssng/";
let tid = 0;
let packetId = 0;
let active_packet_id = "";
let dataLogArray = [];
// let analyzedData = "";

import { ref } from "vue";

const ipServer = ref("");
const ipData = ref("224.0.23.0");
const el = ref({
  deojData: "0x013001",
  esvData: "0x62",
  epcData: "0x80",
  edtData: "0x30",
});
const freeData = ref("10,81,00,0A,05,FF,01,01,30,01,62,01,80,00");
const ipDataStyle = ref({ color: "black" });
const deojDataStyle = ref({ color: "black" });
const esvDataStyle = ref({ color: "black" });
const epcDataStyle = ref({ color: "black" });
const edtDataStyle = ref({ color: "black" });
const freeDataStyle = ref({ color: "black" });
const rbInputData = ref("el");
const rbOrder = ref("normalOrder");
const filters = ref(["showGet", "showInf", "showGetres", "showSNA"]);
const packet_list = ref([]);
const packetDetail = ref("placeholder");

const buttonClickSearch = () => {
  fButtonClickSearch();
};
const buttonClickSend = () => {
  fButtonClickSend(ipData.value, el.value, freeData.value);
};
const updateRbOrder = () => {
  displayLog();
};
const updateFilters = () => {
  displayLog();
};
const clearLog = () => {
  packetDetail.value = "";
  fClearLog();
};
const saveLog = () => {
  fSaveLog();
};
// パケット一覧からパケット行がクリックされたときの処理 (パケット詳細を表示)
const showPacketDetail = packetMonitorShowPacketDetail.bind(this);
console.log("bp1");
// パケット一覧で矢印キーが押されたときの処理
const upDownList = packetMonitorUpDownList.bind(this);
console.log("bp2");
/*
var vm = new Vue({
    el: '#app',
    data: {
        ipServer: "",
        ipData: "224.0.23.0",
        el: {
            deojData: "0x013001",
            esvData: "0x62",
            epcData: "0x80",
            edtData: "0x30"
        },
        freeData: "10,81,00,0A,05,FF,01,01,30,01,62,01,80,00",
        ipDataStyle: {color: 'black'},
        deojDataStyle: {color: 'black'},
        esvDataStyle: {color: 'black'},
        epcDataStyle: {color: 'black'},
        edtDataStyle: {color: 'black'},
        freeDataStyle: {color: 'black'},
        rbInputData: "el",
        rbOrder: "normalOrder",
        filters: ["showGet", "showInf", "showGetres", "showSNA"],
        packet_list: [],
        packetDetail: ""
    },
    methods: {
        buttonClickSearch: function () {
            buttonClickSearch();
        },
        buttonClickSend: function () {
            buttonClickSend(ipData, el, freeData);
        },
        updateRbOrder: function () {
            displayLog();
        },
        updateFilters: function () {
            displayLog();
        },
        clearLog: function () {
            clearLog();
        },
        saveLog: function () {
            saveLog();
        },
		// パケット一覧からパケット行がクリックされたときの処理 (パケット詳細を表示)
		showPacketDetail: packetMonitorShowPacketDetail.bind(this),
		// パケット一覧で矢印キーが押されたときの処理
		upDownList: packetMonitorUpDownList.bind(this)
    }
});

*/

// Show server IP address
let request = new XMLHttpRequest();
request.addEventListener("load", reqListener);
request.open("GET", "ssng/ipv4");
request.send();
function reqListener() {
  ipServer.value = this.responseText;
}

// connect websocket
console.log("ws://" + document.location.host);
// let ws = new WebSocket('ws://' + document.location.host);
const ws = new WebSocket("ws://localhost:8080");
ws.onopen = function (event) {
  console.log("connected");
};

ws.onmessage = function (event) {
  console.log("server_to_client", event.data);
  const obj = JSON.parse(event.data);
  if (obj.ip != ipServer.value) {
    const packet_id = "packet-" + packetId++;
    const pkt = {
      id: packet_id,
      timeStamp: timeStamp(),
      direction: "R",
      ip: obj.ip,
      data: obj.uint8Array,
    };
    dataLogArray.push(pkt);
    displayLog();
  }
};

function displayLog() {
  let log = [];
  for (let dataLog of dataLogArray) {
    const esv = dataLog.data[10];
    const pkt = {
      id: dataLog.id,
      timeStamp: dataLog.timeStamp,
      direction: dataLog.direction,
      address: dataLog.ip,
      hex: elFormat(dataLog.data),
    };
    if (dataLog.direction == "T" || filterEsv(esv)) {
      log.push(pkt);
    }
  }
  if (rbOrder.value == "reverseOrder") {
    log.reverse();
  }
  packet_list.value = log;
  // clear packet selection
  if (active_packet_id) {
    // $("#" + active_packet_id).removeClass("active");
    document.querySelector("#" + active_packet_id).classList.remove("active");
    active_packet_id = "";
  }
  packetDetail.value = "";
  return;

  function filterEsv(esv) {
    if (!filters.value.includes("showGet") && esv == 0x62) {
      return false;
    }
    if (!filters.value.includes("showInf") && esv == 0x73) {
      return false;
    }
    if (!filters.value.includes("showGetres") && (esv == 0x72 || esv == 0x71)) {
      return false;
    }
    if (
      !filters.value.includes("showSNA") &&
      (esv == 0x50 || esv == 0x51 || esv == 0x52 || esv == 0x53 || esv == 0x5e)
    ) {
      return false;
    }
    return true;
  }
}

function timeStamp() {
  const date = new Date();
  let hour = date.getHours().toString();
  let minute = date.getMinutes().toString();
  let second = date.getSeconds().toString();
  hour = hour.length == 1 ? "0" + hour : hour;
  minute = minute.length == 1 ? "0" + minute : minute;
  second = second.length == 1 ? "0" + second : second;
  return hour + ":" + minute + ":" + second;
}

function analyzeData(uint8Array) {
  console.log("analyzeData1");
  // uint8Array: [UInt8]
  let analyzedData = "";
  let epcArray = [];
  const esv = uint8Array[10];
  const epc = uint8Array[12];
  const edt = uint8Array.slice(14);

  // Decode PropertyMap
  if (shouldDecodePropertyMap()) {
    if (edt.length < 17) {
      // PropertyMapがEPCの列挙の場合
      for (let i = 1; i < edt.length; i++) {
        epcArray.push(toStringHex(edt[i], 1));
      }
    } else {
      // PropertyMapがbitmapの場合
      for (let i = 1; i < 17; i++) {
        for (let j = 0; j < 8; j++) {
          if ((edt[i] & (1 << j)) !== 0) {
            let epc = 0x80 + 0x10 * j + i - 1;
            epcArray.push(toStringHex(epc, 1));
          }
        }
      }
    }
    epcArray.sort();
    analyzedData = "EPC:";
    for (let data of epcArray) {
      analyzedData += " " + data;
    }
  } else {
    // return null;
    return "null";
  }
  console.log("analyzeData2:", "analyzedData=", analyzedData);
  return analyzedData; // analyzedData: string
  function shouldDecodePropertyMap() {
    return esv == 0x72 && (epc == 0x9d || epc == 0x9e || epc == 0x9f);
  }
}

function elFormat(uint8Array) {
  let elString = "";
  for (let value of uint8Array) {
    elString += toStringHex(value, 1);
  }
  elString = strIns(elString, 4, " ");
  elString = strIns(elString, 9, " ");
  elString = strIns(elString, 16, " ");
  elString = strIns(elString, 23, " ");
  elString = strIns(elString, 26, " ");
  elString = strIns(elString, 29, " ");
  elString = strIns(elString, 32, " ");
  elString = strIns(elString, 35, " ");
  return elString;
}

// 数値(number)を16進数表記の文字列に変換する
// 数値のbyte数は(bytes)
// example: toStringHex(10, 1) => "0A"
// example: toStringHex(10, 2) => "000A"
function toStringHex(number, bytes) {
  let str = number.toString(16).toUpperCase();
  while (str.length < 2 * bytes) {
    str = "0" + str;
  }
  return str;
}

// stringに文字列を挿入
function strIns(str, idx, val) {
  // str:string（元の文字列）, idx:number（挿入する位置）, val:string（挿入する文字列）
  var res = str.slice(0, idx) + val + str.slice(idx);
  return res;
}

// Check input value of text field
// argument: inputType:string, enum("ip", "deoj", "esv", "epc", "edt", "free")
// get text data from text input field of "inputType"
// return value: boolean
function checkInputValue(inputType, inputValue) {
  console.log(
    "function checkInputValue:",
    "inputType ",
    inputType,
    "inputValue ",
    inputValue
  );
  let regex;
  switch (inputType) {
    case "ip":
      regex =
        /^(([1-9]?[0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([1-9]?[0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])$/;
      break;
    case "deoj":
      regex = /^(0x)?(\d|[a-f]|[A-F]){6}$/;
      break;
    case "esv":
    case "epc":
      regex = /^(0x)?(\d|[a-f]|[A-F]){2}$/;
      break;
    case "edt":
      regex = /^((0x)?((\d|[a-f]|[A-F]){2}){1,})?$/;
      break;
    case "free":
      regex = /^((\d|[a-f]|[A-F]){2},\s*){1,}(\d|[a-f]|[A-F]){2}\s*$/;
      break;
    default:
  }
  if (regex.test(inputValue)) {
    return true;
  } else {
    return false;
  }
}

function fButtonClickSend(ipData, el, freeData) {
  console.log("ipData ", ipData, "el ", el, "freeData", freeData);
  if (!checkInputValue("ip", ipData)) {
    ipDataStyle.value.color = "red";
    window.alert("Check IP address");
    return false;
  } else {
    ipDataStyle.value.color = "black";
  }
  let uint8Array = [];
  // let binaryString = "";
  uint8Array =
    rbInputData.value == "el"
      ? createUint8ArrayFromElData(el)
      : createUint8ArrayFromFreeData(freeData);

  if (uint8Array !== false) {
    const message = { ip: ipData, uint8Array: uint8Array };
    const request = new XMLHttpRequest();
    request.open("PUT", serverURL + "send");
    request.setRequestHeader("Content-type", "application/json");
    request.send(JSON.stringify(message));

    // push "Sent Data" to LOG
    const packet_id = "packet-" + packetId++;
    const pkt = {
      id: packet_id,
      timeStamp: timeStamp(),
      direction: "T",
      ip: ipData,
      data: uint8Array,
    };
    dataLogArray.push(pkt);
    displayLog();
  }
}

function createUint8ArrayFromElData(el) {
  console.log("createUint8ArrayFromElData1")
  if (!checkInputValue("deoj", el.deojData)) {
    deojDataStyle.value.color = "red";
    window.alert("Check DEOJ");
    return false;
  } else {
    deojDataStyle.value.color = "black";
  }
  if (!checkInputValue("esv", el.esvData)) {
    esvDataStyle.value.color = "red";
    window.alert("Check ESV");
    return false;
  } else {
    esvDataStyle.value.color = "black";
  }
  if (!checkInputValue("epc", el.epcData)) {
    epcDataStyle.value.color = "red";
    window.alert("Check EPC");
    return false;
  } else {
    epcDataStyle.value.color = "black";
  }
  if (!checkInputValue("edt", el.edtData)) {
    edtDataStyle.value.color = "red";
    window.alert("Check EDT");
    return false;
  } else {
    edtDataStyle.value.color = "black";
  }
  let uint8Array = [0x10, 0x81]; // EHD
  tid = tid == 0xffff ? 0 : tid + 1;
  uint8Array.push(Math.floor(tid / 16), tid % 16); // TID
  uint8Array.push(0x05, 0xff, 0x01); // SEOJ
  for (let data of hex2Array(el.deojData)) {
    // DEOJ
    uint8Array.push(data);
  }
  uint8Array.push(parseInt(el.esvData, 16)); // ESV
  uint8Array.push(0x01); // OPC
  uint8Array.push(parseInt(el.epcData, 16)); // EPC
  const esv = parseInt(el.esvData, 16);
  if (
    esv == 0x62 ||
    esv == 0x63 ||
    esv == 0x71 ||
    esv == 0x7a ||
    esv == 0x7e ||
    esv == 0x50 ||
    esv == 0x51 ||
    esv == 0x52 ||
    esv == 0x53 ||
    esv == 0x5e
  ) {
    uint8Array.push(0x00); // PDC
  } else {
    // EPC= 0x60:SetI, 0x61:SetC, 0x6E:SetGet, 0x72:Get_Res, 0x73:INF, 0x74:INFC,
    const edtArray = hex2Array(el.edtData);
    uint8Array.push(edtArray.length); // PDC
    for (let data of hex2Array(el.edtData)) {
      // EDT
      uint8Array.push(data);
    }
  }
  return uint8Array;
}

function hex2Array(hex) {
  // hex: string of this format 0xXXXX or XXXX
  if (hex.slice(0, 2) != "0x") {
    hex = "0x" + hex;
  }
  let array = [];
  const bytes = (hex.length - 2) / 2;
  for (let i = 0; i < bytes; i++) {
    array.push(parseInt(hex.slice((i + 1) * 2, (i + 1) * 2 + 2), 16));
  }
  return array; // array: array of byte data
}

function createUint8ArrayFromFreeData(freeData) {
  if (!checkInputValue("free", freeData)) {
    // console.log("freeData.valueStyle.color: ", freeData.valueStyle.color);
    freeData.valueStyle.color = "red";
    window.alert("Check Free data");
    return false;
  } else {
    freeData.valueStyle.color = "black";
  }
  let uint8Array = [];
  let arrayFromFreeData = freeData.split(",");
  for (let value of arrayFromFreeData) {
    uint8Array.push(parseInt(value.trim(), 16));
  }
  return uint8Array;
}

function fButtonClickSearch() {
  console.log("fButtonClickSearch");
  const ipData = "224.0.23.0";
  const el = {
    deojData: "0x0EF001",
    esvData: "0x62",
    epcData: "0xD6",
    edtData: "",
  };
  const freeData = "10,81,00,04,05,FF,01,0E,F0,01,62,01,D6,00";
  fButtonClickSend(ipData, el, freeData);
}

function fSaveLog() {
  let log = "";
  for (let dataLog of dataLogArray) {
    log =
      log +
      dataLog.timeStamp +
      "," +
      dataLog.direction +
      "," +
      dataLog.ip +
      "," +
      elFormat(dataLog.data) +
      "\n";
  }
  const message = { log: log };
  const request = new XMLHttpRequest();
  request.open("POST", serverURL + "saveLog");
  request.setRequestHeader("Content-type", "application/json");
  request.send(JSON.stringify(message));
}

function fClearLog() {
  packetId = 0;
  dataLogArray.length = 0;
  packet_list.value = [];
  packetDetail.value = "";
  console.log("function fClearLog");
}

function packetMonitorShowPacketDetail(event) {
  console.log("packetMonitorShowPacketDetail1");
  if (active_packet_id) {
    // $("#" + active_packet_id).removeClass("active");
    document.querySelector("#" + active_packet_id).classList.remove("active");
    active_packet_id = "";
  }
  let t = event.target;
  console.log("t.id: ", t.id);
  // $("#" + t.id).addClass("active");
  document.querySelector("#" + t.id).classList.add("active");
  active_packet_id = t.id;

  // 現在選択中のパケット ID
  let id_parts = active_packet_id.split("-");
  let pno = parseInt(id_parts[1], 10);

  // packetの解析結果の表示
  console.log("packetMonitorShowPacketDetail2");
  packetDetail.value = analyzeData(dataLogArray[pno].data);
  console.log("packetMonitorShowPacketDetail3", " packetDetail.value=", packetDetail.value);
}

function packetMonitorUpDownList(event) {
  console.log("packetMonitorUpDownList")
  event.preventDefault();
  event.stopPropagation();
  // 選択中のパケット行がなければ終了
  if (!active_packet_id) {
    return;
  }
  // 現在選択中のパケット ID
  let id_parts = active_packet_id.split("-");
  let pno = parseInt(id_parts[1], 10);

  let c = event.keyCode;
  let k = event.key;
  if (c === 38 || k === "ArrowUp") {
    // 上矢印キー
    if (rbOrder.value == "normalOrder") {
      if (pno-- < 0) {
        pno = 0;
      }
    } else {
      if (pno++ >= dataLogArray.length) {
        pno = dataLogArray.length - 1;
      }
    }
  } else if (c === 40 || k === "ArrowDown") {
    // 下矢印キー
    if (rbOrder.value == "normalOrder") {
      if (pno++ >= dataLogArray.length) {
        pno = dataLogArray.length - 1;
      }
    } else {
      if (pno-- < 0) {
        pno = 0;
      }
    }
  } else {
    return;
  }
  // 遷移したパケット行にフォーカスする
  // $("#packet-" + pno).focus();
  document.querySelector("#packet-" + pno).focus();
}
</script>

<template>
  <!--
<div class="container" id="app">
-->
  <div class="container">
    <div class="card mb-3">
      <!-- ECHONET Lite Packets -->
      <!-- ECHONET Lite Packets header-->
      <div class="card-header py-1">
        <div class="row">
          <div class="col-auto h5 mt-2">ECHONET Lite Packets</div>
          <!--                 <div class="col-auto mt-2" id="ipv4" ></div> -->
          <div class="col-auto mt-2">{{ ipServer }}</div>
          <div class="col"></div>
          <div class="col-auto">
            <div class="input-group">
              <div class="onput-group-prepend">
                <span class="input-group-text">IP</span>
              </div>
              <input
                type="text"
                class="form-control"
                v-model="ipData"
                v-bind:style="ipDataStyle"
              />
            </div>
          </div>
          <div class="col-auto mt-1">
            <button
              type="button"
              class="btn btn-secondary me-2 btn-sm"
              v-on:click="buttonClickSend"
            >
              SEND
            </button>
            <button
              type="button"
              class="btn btn-secondary btn-sm"
              v-on:click="buttonClickSearch"
            >
              SEARCH
            </button>
          </div>
        </div>
      </div>
      <!-- ECHONET Lite Packets body-->
      <div class="card-body pt-2 pb-2">
        <form>
          <div class="form-check form-check-inline mb-2">
            <input
              type="radio"
              class="form-check-input"
              id="rb_el"
              v-model="rbInputData"
              value="el"
            />
            <label for="rb_el" class="form-check-label"
              >ECHONET Lite Data</label
            >
          </div>
          <div class="row align-items-center">
            <div class="col-3">
              <div class="input-group mb-2">
                <span class="input-group-text">DEOJ</span>
                <input
                  type="text"
                  class="form-control"
                  v-model="el.deojData"
                  v-bind:style="deojDataStyle"
                />
              </div>
            </div>
            <div class="col-2">
              <div class="input-group mb-2">
                <span class="input-group-text">ESV</span>
                <input
                  type="text"
                  class="form-control"
                  v-model="el.esvData"
                  v-bind:style="esvDataStyle"
                />
              </div>
            </div>
            <div class="col-2">
              <div class="input-group mb-2">
                <span class="input-group-text">EPC</span>
                <input
                  type="text"
                  class="form-control"
                  v-model="el.epcData"
                  v-bind:style="epcDataStyle"
                />
              </div>
            </div>
            <div class="col-auto">
              <div class="input-group mb-2">
                <span class="input-group-text">EDT</span>
                <input
                  type="text"
                  class="form-control"
                  v-model="el.edtData"
                  v-bind:style="edtDataStyle"
                />
              </div>
            </div>
          </div>
          <div class="form-check form-check-inline my-2">
            <input
              type="radio"
              class="form-check-input"
              id="rb_free"
              v-model="rbInputData"
              value="free"
            />
            <label for="rb_free" class="form-check-label">Free Data</label>
          </div>
          <div>
            <input
              type="text"
              class="form-control"
              v-model="freeData"
              v-bind:style="freeDataStyle"
            />
          </div>
        </form>
      </div>
    </div>

    <!-- Packets Monitor -->
    <!-- Packets Monitor header-->
    <div class="card">
      <div class="card-header py-1">
        <div class="row">
          <div class="col-auto h5 mt-2">Packets Monitor</div>
          <div class="col"></div>

          <div class="col-auto pl-0">
            <div class="input-group border ml-2">
              <span class="input-group-text">Order</span>
              <!-- radio button "Normal" -->
              <div class="form-check form-check-inline mt-2">
                <input
                  type="radio"
                  class="form-check-input"
                  id="normalOrder"
                  v-model="rbOrder"
                  value="normalOrder"
                  v-on:change="updateRbOrder"
                />
                <label for="normalOrder" class="form-check-label">Normal</label>
              </div>

              <div class="form-check form-check-inline mt-2">
                <input
                  type="radio"
                  class="form-check-input"
                  id="reverseOrder"
                  v-model="rbOrder"
                  value="reverseOrder"
                  v-on:change="updateRbOrder"
                />
                <label for="reverseOrder" class="form-check-label"
                  >Reverse</label
                >
              </div>
            </div>
          </div>

          <div class="col-auto pl-0">
            <div class="input-group border">
              <div class="input-group-text">Filter</div>
              <div class="form-check form-check-inline mt-2">
                <input
                  type="checkbox"
                  class="form-check-input"
                  id="showGet"
                  value="showGet"
                  v-model="filters"
                  v-on:change="updateFilters"
                />
                <label class="form-check-label" for="showGet">GET</label>
              </div>
              <div class="form-check form-check-inline mt-2">
                <input
                  type="checkbox"
                  class="form-check-input"
                  id="showInf"
                  value="showInf"
                  v-model="filters"
                  v-on:change="updateFilters"
                />
                <label class="form-check-label" for="showInf">INF</label>
              </div>
              <div class="form-check form-check-inline mt-2">
                <input
                  type="checkbox"
                  class="form-check-input"
                  id="showGetres"
                  value="showGetres"
                  v-model="filters"
                  v-on:change="updateFilters"
                />
                <label class="form-check-label" for="showGetres">GET_RES</label>
              </div>
              <div class="form-check form-check-inline mt-2">
                <input
                  type="checkbox"
                  class="form-check-input"
                  id="showSNA"
                  value="showSNA"
                  v-model="filters"
                  v-on:change="updateFilters"
                />
                <label class="form-check-label" for="showSNA">SNA</label>
              </div>
            </div>
          </div>

          <div class="col-auto mt-1 pl-0">
            <button
              type="button"
              class="btn btn-secondary me-2 btn-sm"
              v-on:click="clearLog"
            >
              CLEAR
            </button>
            <button
              type="button"
              class="btn btn-secondary btn-sm"
              v-on:click="saveLog"
            >
              SAVE
            </button>
          </div>
        </div>
      </div>

      <!-- パケットモニター body -->
      <div class="card-body" id="packet-monitor-body">
        <div id="packet-list-wrapper">
          <ul
            class="list-group"
            id="packet-list"
            v-on:keyup.stop
            v-on:keydown.stop
          >
            <li class="list-group-item" id="packet-monitor-header" tabindex="0">
              <span class="col1">HH MM SS</span>
              <span class="col2">T/R</span>
              <span class="col3">IP</span>
              <span class="col4">DATA</span>
            </li>
            <li
              v-for="packet in packet_list"
              class="list-group-item"
              v-bind:id="packet.id"
              tabindex="0"
              v-on:focus="showPacketDetail"
              v-on:keydown="upDownList"
            >
              <span class="col1">{{ packet.timeStamp }}</span>
              <span class="col2">{{ packet.direction }}</span>
              <span class="col3">{{ packet.address }}</span>
              <span class="col4">{{ packet.hex }}</span>
            </li>
          </ul>
        </div>
        <div id="packet-detail-wrapper">
          <!-- <packet-detail>{{ packetDetail }}</packet-detail> -->
          <span>{{ packetDetail }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
body {
  padding-top: 1rem;
}
.card-deck {
  margin-bottom: 20px;
}
.card-header {
  padding: 0.1em 1em;
}
#packet-monitor-body {
  margin: 0;
  padding: 0;
}
#packet-list-wrapper {
  height: 30em;
  overflow-y: scroll;
}
#packet-list li {
  font-family: Consolas, "Courier New", Courier, Monaco, monospace;
  font-size: 90%;
  padding: 0.2em 1em;
  cursor: pointer;
}
#packet-list li span {
  display: inline-block;
}
#packet-list li span.col1 {
  width: 5em;
}
#packet-list li span.col2 {
  width: 2em;
}
#packet-list li span.col3 {
  width: 9em;
}
#packet-detail-wrapper {
  padding: 0.5em 1em;
  border-top: 1px solid #cccccc;
}
</style>
