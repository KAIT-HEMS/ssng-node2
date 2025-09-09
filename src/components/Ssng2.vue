<script setup>
const ServerURL = "/ssng/"; // <string>
const MultiCastAddress = "224.0.23.0"; // <string> MultiCast Address for ECHONET Lite
let tid = 0; // <INT> ECHONET Lite の TIDの初期値
let active_packet_id = ""; // <string> Packet Monitor のアクティブなpacket id
let dataLogs = [];  // <array[obj]> 送受信したデータを全て保存しておく配列
// <array[{id:<INT>, timeStamp:<string>, direction:<string> enum:["T", "R"], address:<string>, data:<object>}]

import { ref } from "vue";

const ipServer = ref("");
const ipData = ref(MultiCastAddress);
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
const rbOrder = ref("normal"); //<string> enum["normal", "reverse"]
const rbFilter = ref("all"); //<string> enum["all", "self", "self+INF"]
const packets = ref([]); // <array[obj]> ログ表示用の配列
// <array[{id:<INT>, timeStamp:<string>, direction:<string> enum:["T", "R"], address:<string>, data:<object>, hex:<string>}]
const packetDetail = ref(""); // <string>

// SEARCH ボタンクリック時の処理 method
const buttonClickSearch = () => {
  const ip = MultiCastAddress;
  const el = {
    deojData: "0x0EF001",
    esvData: "0x62",
    epcData: "0xD6",
    edtData: "",
  };
  const data = createUint8ArrayFromElData(el);
  sendData(ip, data);
  pushDataToLog("T", ip, data);
  displayLog();
};

// SEND ボタンクリック時の処理 method
const buttonClickSend = () => {
  ipDataStyle.value.color = "black";
  if (!checkInputValue("ip", ipData.value)) {
    ipDataStyle.value.color = "red";
    window.alert("Check IP address");
    return;
  }

  const uint8Array =
    rbInputData.value == "el"
      ? createUint8ArrayFromElData(el.value)
      : createUint8ArrayFromFreeData(freeData.value);

  if (uint8Array.length != 0) {
    sendData(ipData.value, uint8Array);
    pushDataToLog("T", ipData.value, uint8Array);
    displayLog();
  }
};

// CLEAR ボタンクリック時の処理 method
const clearLog = () => {
  packetDetail.value = "";
  dataLogs.length = 0;
  packets.value = [];
  packetDetail.value = "";
};

// SAVE ボタンクリック時の処理 method
const saveLog = () => {
  let saveLogs = "";
  for (let dataLog of dataLogs) {
   saveLogs =
     saveLogs +
      dataLog.timeStamp +
      "," +
      dataLog.direction +
      "," +
      dataLog.ip +
      "," +
      elFormat(dataLog.data) +
      "\n";
  }
  const message = { log: saveLogs };
  const request = new XMLHttpRequest();
  request.open("POST", ServerURL + "saveLog");
  request.setRequestHeader("Content-type", "application/json");
  request.send(JSON.stringify(message));
};

// IP クリック時の処理 method
const clickIP = () => {
  ipData.value = MultiCastAddress;
};

// パケット一覧からパケット行がクリックされたときの処理 method (パケット詳細を表示)
const onFocus = packetMonitorShowPacketDetail.bind(this);
// パケット一覧で上下矢印キーが押されたときの処理 method
const onKeydown = packetMonitorUpDownList.bind(this);

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
const ws = new WebSocket("ws://localhost:8080");
ws.onopen = function (event) {
  console.log("connected");
};

// websocketで受信したデータを dataLogs に保存し、ログ表示を更新する
ws.onmessage = function (event) {
  console.log("ws:server_to_client", event.data);
  const obj = JSON.parse(event.data);
  if (obj.ip != ipServer.value) { // loop back を排除
    pushDataToLog(obj.direction, obj.ip, obj.uint8Array);
    displayLog();
  }
};

// input: none
// output: none
// function: Packets Monitorのログ表示を更新する
// global変数dataLogsからログデータを取り出し、Filterの条件に一致するPacketを表示用配列displayLogsにpushする
// reverseボタンが選択されている場合は配列displayLogsの要素の順序を逆転する
// displayLogsの角要素のidを更新（0,1,2,...)
// displayLogsの中身をログ表示用property packetsに渡す
function displayLog() {
  let displayLogs = [];
  for (let dataLog of dataLogs) {
    const esv = dataLog.data[10];
    const pkt = {
      id: 0,
      timeStamp: dataLog.timeStamp,
      direction: dataLog.direction,
      address: dataLog.ip,
      data: dataLog.data, // 詳細表示のための元データ
      hex: elFormat(dataLog.data), // 表示用の文字列
    };
    // radio button "self"または"self+INF" が選択されている場合の処理
    // "Self": TはGet:62, SetI:60, SetC:61, INF:73、 RはGet_Res:72, Set_Res:71, SNA:50, 51, 52
    // "Self+INF": "self"の条件 + RでINF:73 を追加
    if (filterLog(rbFilter.value, dataLog.direction, esv)) {
      displayLogs.push(pkt);
    }
  }

  // radio button "reverse" が選択されている場合の処理
  if (rbOrder.value == "reverse") {
    displayLogs.reverse();
  }

  // id を修正 0 から連続した番号を付加する
  for (let i = 0; i < displayLogs.length; i++) {
    displayLogs[i].id = "packet-" + i;
  }

  // ログ表示用プロパティにログデータを渡す
  packets.value = displayLogs;

  // clear packet selection
  if (active_packet_id) {
    const element = document.querySelector("#" + active_packet_id);
    if (element !== null) {
      element.classList.remove("active");
    }
    active_packet_id = "";
  }

  // packet detailをクリア
  packetDetail.value = "";
  return;

  // input rb: <string> enum:["all", "self", "self+INF"]
  // input: direction: <string> enum:["T", "R"]
  // input: esv: <UINT8>
  // output: boolean
  // function: radio button "Self", "Self+INF" が選択された場合 esv でフィルタをかける
  //   "Self": SSNGが送信した全てのパケットと、それに対応する受信パケット
  //   "Self+INF": "Self" の内容と、外部デバイスのINFの受信
  function filterLog(rb, direction, esv) {
    const esvForT = [0x62, 0x60, 0x61, 0x73];
    const esvForRSelf = [0x72, 0x71, 0x50, 0x51, 0x52];
    const esvForRSelfINF = [0x72, 0x71, 0x50, 0x51, 0x52, 0x73];
    switch(rb) {
      case "all":
        return true;
      case "self":
        if (direction == "T" && esvForT.includes(esv)) {
          return true;
        }
        if (direction == "R" && esvForRSelf.includes(esv)) {
          return true;
        }
        return false;
      case "self+INF":
        if (direction == "T" && esvForT.includes(esv)) {
          return true;
        }
        if (direction == "R" && esvForRSelfINF.includes(esv)) {
          return true;
        }
        return false;
      default:
        return false;
    }
  }
}

// input: none
// output: <string> ログ用のtime code
// function: ログ用のtime codeを返す
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

// input: <array[UINT8]>
// output: <string> property map
// function EPC:9D, 9E, 9F のレスポンス(property map)を解析する
function analyzeData(uint8Array) {
  let analyzedData = "";
  let epcArray = [];
  const epcsForPropertyMap = [0x9d, 0x9e, 0x9f];
  const esv = uint8Array[10];
  const epc = uint8Array[12];
  const edt = uint8Array.slice(14);

  // Decode PropertyMap
  if (esv== 0x72 && epcsForPropertyMap.includes(epc)) {
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
    return " ";
  }
  // console.log("analyzeData2:", "analyzedData=", analyzedData);
  return analyzedData;
}

// input uin8array: <array[UINT8]>
// output <string>
// function: ECHONET Liteのバイナリデータを表示用にスペースを挿入する
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

// input number: <INT8>
// input bytes: <INT8>
// output <string>
// numberを16進数表記の文字列に変換する。bytesでbyte数を指定する。
// ex. toStringHex(10, 1) => "0A", toStringHex(10, 2) => "000A"
function toStringHex(number, bytes) {
  let str = number.toString(16).toUpperCase();
  while (str.length < 2 * bytes) {
    str = "0" + str;
  }
  return str;
}

// input str: <string> オリジナルの文字列
// input idx: <INT> 挿入する位置
// input val: <string> 挿入する文字列
// output: <string>
// function: オリジナルの文字列strに、idxで指定した位置に文字列valを挿入する
function strIns(str, idx, val) {
  var res = str.slice(0, idx) + val + str.slice(idx);
  return res;
}

// input inputType: <string> enum:["ip", "deoj", "esv", "epc", "edt", "free"]
// input inputValue: <string>
// output: boolean
// function: inputValueの値のデータフォーマットをinputTypeに応じてチェックする
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

// input ip: <string> ex. "224.0.23.0"
// input data: <array[UINT8]>
// function: Server APIを利用してUDP送信を実行
function sendData(ip, data){
  if (data.length != 0) {
    const message = { ip: ip, uint8Array: data };
    const request = new XMLHttpRequest();
    request.open("PUT", ServerURL + "send");
    request.setRequestHeader("Content-type", "application/json");
    request.send(JSON.stringify(message));
  }
}

// input direction: <string> "enum":["T", "R"]
// input ip: <string> ex. "224.0.23.0"
// input data: <array[UINT8]>
// function: 送受信データをglobal変数のdataLogsにPUSHする
function pushDataToLog(direction, ip, data){
  const pkt = {
    id: 0,
    timeStamp: timeStamp(),
    direction: direction,
    ip: ip,
    data: data,
  };
  dataLogs.push(pkt);
}

// input el: <object> ex. {deojData: "0x013001", esvData: "0x62", epcData: "0x80", edtData: "0x30"}
// output array[UINT8] （elのフォーマットがおかしい場合は empty array [] を返す）
// function: 送信用のECHONET Liteの各要素のデータフォーマットを確認後、送信用に[UINT8]に変換する
function createUint8ArrayFromElData(el) {
  let uint8Array = [];
  deojDataStyle.value.color = "black";
  esvDataStyle.value.color = "black";
  epcDataStyle.value.color = "black";
  edtDataStyle.value.color = "black";

  if (!checkInputValue("deoj", el.deojData)) {
    deojDataStyle.value.color = "red";
    window.alert("Check DEOJ");
    return uint8Array;
  } 
  if (!checkInputValue("esv", el.esvData)) {
    esvDataStyle.value.color = "red";
    window.alert("Check ESV");
    return uint8Array;
  }
  if (!checkInputValue("epc", el.epcData)) {
    epcDataStyle.value.color = "red";
    window.alert("Check EPC");
    return uint8Array;
  }
  if (!checkInputValue("edt", el.edtData)) {
    edtDataStyle.value.color = "red";
    window.alert("Check EDT");
    return uint8Array;
  }

  uint8Array.push(0x10, 0x81); // EHD
  tid = tid == 0xffff ? 0 : tid + 1;  // increment TID
  uint8Array.push(Math.floor(tid / 16), tid % 16); // TID
  uint8Array.push(0x05, 0xff, 0x01); // SEOJ
  for (let data of hex2Array(el.deojData)) {
    uint8Array.push(data); // DEOJ
  }
  const esv = parseInt(el.esvData, 16); 
  uint8Array.push(esv); // ESV
  uint8Array.push(0x01); // OPC
  uint8Array.push(parseInt(el.epcData, 16)); // EPC
  // const esv = parseInt(el.esvData, 16);
  const esvs = [0x62,0x63,0x71,0x7a,0x7e,0x50,0x51,0x52,0x53,0x5e];
  if (esvs.includes(esv)) {
    uint8Array.push(0x00); // PDC = 0
  } else {
    // EPC= 0x60:SetI, 0x61:SetC, 0x6E:SetGet, 0x72:Get_Res, 0x73:INF, 0x74:INFC,
    const edtArray = hex2Array(el.edtData);
    uint8Array.push(edtArray.length); // PDC
    for (let data of hex2Array(el.edtData)) {
      uint8Array.push(data); // EDT
    }
  }
  return uint8Array;
}

// input hex:<string> HEX data、先頭に'0x'がついていてもいなくてもOK ex. 0x12...FF or 12...FF
// output <array[UINT8]> array of byte data
// function: 文字列表現のHEX dataをBYTEデータの配列に変換する
function hex2Array(hex) {
  if (hex.slice(0, 2) != "0x") {
    hex = "0x" + hex;
  }
  let array = [];
  const bytes = (hex.length - 2) / 2;
  for (let i = 0; i < bytes; i++) {
    array.push(parseInt(hex.slice((i + 1) * 2, (i + 1) * 2 + 2), 16));
  }
  return array;
}

// input freeData: <string> ex. "10,81,00,0A,05,FF,01,01,30,01,62,01,9E,00"
// (データの間に space が入ってもOK)
// output array[UINT8] （freeDataのフォーマットがおかしい場合は empty array [] を返す）
// function: 入力文字列を[UINT8]に変換する
function createUint8ArrayFromFreeData(freeData) {
  let uint8Array = [];
  if (!checkInputValue("free", freeData)) {
    freeDataStyle.value.color = "red";
    window.alert("Check Free data");
    return uint8Array;
  } else {
    freeDataStyle.value.color = "black";
  }
  const arrayFromFreeData = freeData.split(",");
  for (let value of arrayFromFreeData) {
    uint8Array.push(parseInt(value.trim(), 16));
  }
  return uint8Array;
}

// function: Packets Monitor 画面のある行が選択された場合の処理
function packetMonitorShowPacketDetail(event) {
  // 行のハイライトを削除: remove "active"
  if (active_packet_id) {
    const element = document.querySelector("#" + active_packet_id);
    if (element !== null) {
      element.classList.remove("active");
    }
  }

  // 選択された行をハイライト: add "active"
  active_packet_id = event.target.id;
  const element = document.querySelector("#" + active_packet_id);
  if (element !== null) {
    element.classList.add("active");
  }

  // 現在選択中のパケットIDを抽出
  const id_parts = active_packet_id.split("-");
  const pno = parseInt(id_parts[1], 10);

  // packetの解析結果の表示
  packetDetail.value = analyzeData(packets.value[pno].data);
}

// input: none
// output: none
// function: 上下の矢印キーが押された時の処理
// ハイライト行がある場合は、ハイライトを上下させる
function packetMonitorUpDownList(event) {
  event.preventDefault();
  event.stopPropagation();

  // 選択中のパケット行がなければ終了
  if (!active_packet_id) {
    return;
  }

  // 現在選択中のパケット行番号を抽出
  let id_parts = active_packet_id.split("-");
  let pno = parseInt(id_parts[1], 10);

  // パケット行番号を修正
  let c = event.keyCode;
  let k = event.key;
  if (c === 38 || k === "ArrowUp") { // 上矢印キー
    if (--pno < 0) {
      pno = 0;
    }
  } else if (c === 40 || k === "ArrowDown") { // 下矢印キー
    ++pno
  } else {
    return;
  }

  // 遷移したパケット行にフォーカスする。elementがnullの場合は下へ行き過ぎなので１行戻る。
  const element = document.querySelector("#packet-" + pno);
  if (element === null) {
    --pno;
  } else {
    element.focus();
  }
}
</script>

<template>
  <div class="container">
    <div class="card mb-3">
      <!-- ECHONET Lite Packets -->
      <!-- ECHONET Lite Packets header-->
      <div class="card-header py-1">
        <div class="row">
          <div class="col-auto h5 mt-2">ECHONET Lite Packets</div>
          <div class="col-auto mt-2">{{ ipServer }}</div>
          <div class="col"></div>
          <div class="col-auto">
            <div class="input-group">
              <div class="onput-group-prepend">
                <span class="input-group-text" v-on:click="clickIP">IP</span>
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

          <div class="col-auto">
            <div class="input-group border">
              <span class="input-group-text">Order</span>
                <!-- radio button "Normal" -->
                <div class="form-check form-check-inline mt-2 ms-2">
                  <input
                    type="radio"
                    class="form-check-input"
                    id="normal"
                    v-model="rbOrder"
                    value="normal"
                    v-on:change="displayLog"
                    checked
                  />
                  <label for="normal" class="form-check-label">Normal</label>
                </div>
                <!-- radio button "Reverse" -->
                <div class="form-check form-check-inline mt-2">
                  <input
                    type="radio"
                    class="form-check-input"
                    id="reverse"
                    v-model="rbOrder"
                    value="reverse"
                    v-on:change="displayLog"
                  />
                  <label for="reverse" class="form-check-label"
                    >Reverse</label
                  >
                </div>
            </div>
          </div>

          <div class="col-auto">
            <div class="input-group border">
              <span class="input-group-text">Filter</span>
              <!-- radio button "All" -->
              <div class="form-check form-check-inline mt-2 ms-2">
                <input
                  type="radio"
                  class="form-check-input"
                  id="filterAll"
                  v-model="rbFilter"
                  value="all"
                  v-on:change="displayLog"
                  checked
                />
                <label for="filterAll" class="form-check-label">All</label>
              </div>

              <!-- radio button "Self" -->
              <div class="form-check form-check-inline mt-2">
                <input
                  type="radio"
                  class="form-check-input"
                  id="filterSelf"
                  v-model="rbFilter"
                  value="self"
                  v-on:change="displayLog"
                />
                <label for="filterSelf" class="form-check-label">Self</label>
              </div>

              <!-- radio button "Self+INF" -->
              <div class="form-check form-check-inline mt-2">
                <input
                  type="radio"
                  class="form-check-input"
                  id="filterSelfPlusINF"
                  v-model="rbFilter"
                  value="self+INF"
                  v-on:change="displayLog"
                />
                <label for="filterSelfINF" class="form-check-label">Self+INF</label>
              </div>

            </div>
          </div>
          <div class="col-auto mt-1">
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
            <!-- Line-1 Label "HH MM SS T/R IP DATA" -->
            <li class="list-group-item" id="packet-monitor-header" tabindex="0">
              <span class="col1">HH MM SS</span>
              <span class="col2">T/R</span>
              <span class="col3">IP</span>
              <span class="col4">DATA</span>
            </li>
            <li
              v-for="packet in packets"
              class="list-group-item"
              v-bind:key="packet.id"
              v-bind:id="packet.id"
              tabindex="0"
              v-on:focus="onFocus"
              v-on:keydown="onKeydown"
            >
              <span class="col1">{{ packet.timeStamp }}</span>
              <span class="col2">{{ packet.direction }}</span>
              <span class="col3">{{ packet.address }}</span>
              <span class="col4">{{ packet.hex }}</span>
            </li>
          </ul>
        </div>
        <div id="packet-detail-wrapper">
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
