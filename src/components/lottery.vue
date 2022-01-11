<!--
 * @Author: your name
 * @Date: 2021-12-27 18:22:02
 * @LastEditTime: 2022-01-10 10:53:26
 * @LastEditors: Please set LastEditors
 * @Description: 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
 * @FilePath: \lottery\src\components\HelloWorld.vue
-->
<template>
  <div class="bg_page">
    <div>
      <img class="grade_img" :src="gradeImg" />
      <div>
        <!-- <div v-if="award">抽中{{ award }}</div> -->
        <img :src="awardImg" />
      </div>
      <div>
        <div v-for="(item, index) in workerName.value" :key="index">
          {{ item.name }}
        </div>
      </div>
    </div>
    <div>
      <img v-show="isStop" src="../assets/start.png" @click="startLottery" />
      <img v-show="!isStop" src="../assets/stop.png" @click="startLottery" />
    </div>
    <div class="modal" v-show="showModal"></div>
    <div class="modal_bg" v-show="showModal" @click="showModal = false">
      <div>
        <div>
          <div v-for="(item, index) in workerName.value" :key="index">
            {{ item.name }}
          </div>
        </div>

        <img :src="awardImg" />
        <div>{{ awardList.value[i]?.name || "" }}</div>
      </div>
    </div>
    <!-- <div>
      {{ awardList.value[i]?.grade }}等奖品：{{ awardList.value[i]?.name }}
    </div> -->
    <!-- <div v-for="(item, index) in workerName.value" :key="index">
      姓名：{{ item.name }};员工号：{{ item.uid }}
    </div> -->

    <div></div>

    <!-- <button @click="startLottery">{{ buttonName }}</button> -->
    <span style="position: absolute; right: 0; top: 50%" @click="downloadExcel">
      下载excel
    </span>
    <span
      style="position: absolute; right: 0; top: calc(40px + 50%)"
      @click="cleanAward"
      v-if="i <= 0 || awardList.value[awardList.value.length - 1]?.num !== 0"
    >
      清空奖池
    </span>
  </div>
</template>

<script>
import { defineComponent, reactive, ref } from "vue";
import moment from "moment";
import Excel from "exceljs/dist/exceljs";

let db = null;
export default defineComponent({
  setup() {
    let showModal = ref(false);
    let awardList = reactive({
      value: [],
    });
    let gradeLists = reactive({
      value: { 3: "一", 5: "二", 10: "三", 15: "四" },
    });
    let workerList = reactive({
      value: [],
    });
    let workerName = reactive({ value: [] });
    let award = ref("");
    let workerId = ref([]);
    let i = ref(17);
    let interval = reactive({ value: null });

    let isStop = ref(true);
    let lotteryList = reactive({
      value: [],
    });
    let gradeList = ref(["一", "二", "三", "四", "五"]);
    let getData = ({ tableName }) => {
      lotteryList.value = [];
      if (tableName === "Awards") {
        return new Promise((resolve, reject) => {
          let search = db
            .transaction([tableName], "readwrite")
            .objectStore(tableName);
          let request = search.openCursor();
          request.onerror = function () {
            console.log("失败");
            reject();
          };
          request.onsuccess = function (e) {
            var cursor = e.target.result;
            console.log("成功");
            if (cursor) {
              //console.log(cursor.value, 8888);
              lotteryList.value.push(cursor.value);
              cursor.continue();
            } else {
              if (lotteryList[i.value].num == 0) {
                i.value = i.value + 1;
              }
              resolve();
            }
          };
        });
      }

      // console.log(999);
    };
    function cleanAward() {
      if (!confirm("是否清空")) {
        return;
      }
      let store = db.transaction(["Users"], "readwrite").objectStore("Users");
      store.clear();
      history.go(0);
    }
    async function downloadExcel() {
      let a = [];
      try {
        const request = db
          .transaction(["Users"], "readwrite")
          .objectStore("Users")
          .openCursor();
        request.onsuccess = function (e) {
          let result = e.target.result;

          if (result) {
            a.push(result.value);
            result.continue();
          } else {
            var jsonData = a;
            //列标题，逗号隔开，每一个逗号就是隔开一个单元格
            let str = `id,工号,姓名,等级,奖项,时间\n`;
            //增加\t为了不让表格显示科学计数法或者其他格式
            for (let i = 0; i < jsonData.length; i++) {
              for (let item in jsonData[i]) {
                str += `${jsonData[i][item] + "\t"},`;
              }
              str += "\n";
            }
            //encodeURIComponent解决中文乱码
            let uri =
              "data:text/xlsx;charset=utf-8,\ufeff" + encodeURIComponent(str);
            //通过创建a标签实现
            var link = document.createElement("a");
            link.href = uri;
            //对下载的文件命名
            link.download = "2022年会中奖名单.xls";
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
          }
        };
      } catch (error) {
        console.log(error);
      }
    }

    function writeDataBase({ tableName, id }) {
      //奖池减一，人数获一
      //获奖人不再参与下次抽奖？
      let stores = db
        .transaction([tableName], "readwrite")
        .objectStore(tableName);
      let request = stores.put({ id });
      request.onsuccess = function (e) {
        console.log(e);
      };
    }
    function startLottery() {
      // workerList.value = workerList.value.filter((item) => item.num === 0);
      //let length = [...workerList.value].length;
      //console.log(88800, aR.value.get(1));
      for (let i = 1; i < this.workerList.value.length; i++) {
        const random = Math.floor(Math.random() * (i + 1));
        [this.workerList.value[i], this.workerList.value[random]] = [
          this.workerList.value[random],
          this.workerList.value[i],
        ];
      }
      if (this.i < 0) {
        return;
      }
      let stopLottery = () => {
        if (interval.value) {
          //写操作
          //员工id员工num
          //奖品id奖品num
          //是否进行下一等级抽奖

          clearInterval(interval.value);
          let saveData = (ind) => {
            try {
              let bR = null;
              if (!bR) {
                bR = db
                  .transaction(["Users"], "readwrite")
                  .objectStore("Users");
              }

              bR.add({
                id: workerName.value[ind].id,
                uid: workerName.value[ind].uid,
                name: workerName.value[ind].name,
                grade: awardList.value[i.value].grade,
                award: awardList.value[i.value].name,
                time: moment().format("YYYY-MM-DD HH:mm:ss"),
                img: awardList.value[i.value].img,
              });
            } catch (error) {
              console.log(888, error);
            }
          };
          if (i.value == 17) {
            saveData(0);
            saveData(1);
          } else if (i.value == 16) {
            saveData(0);
            saveData(1);
            saveData(2);
          } else {
            saveData(0);
          }
          workerList.value = workerList.value.filter((item) => {
            if (i.value == this.awardList.value.length - 1) {
              return (
                workerName.value[0].id !== item.id &&
                workerName.value[1].id !== item.id
              );
            } else if (i.value == 16) {
              return (
                workerName.value[0].id !== item.id &&
                workerName.value[1].id !== item.id &&
                workerName.value[2].id !== item.id
              );
            } else {
              return workerName.value[0].id !== item.id;
            }
          });

          i.value >= 0 && (award.value = awardList.value[i.value].name);

          //console.log(22, workerList.value);
          //let a = [...workerList.value];

          // console.log(22, workerList.value);
          //writeDataBase()

          interval.value = null;
        }
      };
      isStop.value = !isStop.value;
      if (i.value == 0) {
        if (interval.value) {
          stopLottery();
          clearInterval(interval.value);
        }

        return;
      }
      //   awardList.forEach((item) => {
      //     if(){}
      //   });

      let index = 0;
      // awardList.forEach(item=>{

      // })
      console.log(888888, isStop.value, i.value);
      if (!isStop.value) {
        if (awardList.value[awardList.value.length - 1]?.num == 0) {
          if (i.value > 0) {
            i.value = i.value - 1;
          } else {
            i.value = 0;
          }
        }
        if (awardList.value[i.value].num > 0) {
          awardList.value[i.value].num = 0;
        }

        award.value = "";
        // if (i.value < 0) {
        // }
        //读操作
        //员工表
        //await getData({ tableName: "Awards" });
        interval.value = setInterval(() => {
          if (index < workerList.value.length) {
            if (i.value == this.awardList.value.length - 1) {
              workerName.value = [
                {
                  name: workerList.value[index].name,
                  id: workerList.value[index].id,
                  uid: workerList.value[index].uid,
                },
                {
                  name: workerList.value[
                    index < workerList.value.length - 1 ? index + 1 : 0
                  ].name,
                  id: workerList.value[
                    index < workerList.value.length - 1 ? index + 1 : 0
                  ].id,
                  uid: workerList.value[
                    index < workerList.value.length - 1 ? index + 1 : 0
                  ].uid,
                },
              ];
            } else if (i.value == this.awardList.value.length - 2) {
              workerName.value = [
                {
                  name: workerList.value[index].name,
                  id: workerList.value[index].id,
                  uid: workerList.value[index].uid,
                },
                {
                  name: workerList.value[
                    index < workerList.value.length - 1 ? index + 1 : 0
                  ].name,
                  id: workerList.value[
                    index < workerList.value.length - 1 ? index + 1 : 0
                  ].id,
                  uid: workerList.value[
                    index < workerList.value.length - 1 ? index + 1 : 0
                  ].uid,
                },
                {
                  name: workerList.value[
                    index < workerList.value.length - 2 ? index + 2 : 0
                  ].name,
                  id: workerList.value[
                    index < workerList.value.length - 2 ? index + 2 : 0
                  ].id,
                  uid: workerList.value[
                    index < workerList.value.length - 2 ? index + 2 : 0
                  ].uid,
                },
              ];
            } else {
              workerName.value = [
                {
                  name: workerList.value[index].name,
                  id: workerList.value[index].id,
                  uid: workerList.value[index].uid,
                },
              ];

              //workerId.value = [workerList.value[index].id];
            }

            index++;
          } else {
            index = 0;
          }
        }, 100);
        //   timeout = setTimeout(() => {

        //     console.log(89000);
        //   }, Infinity);
      }
      if (isStop.value) {
        //写入
        stopLottery();
      }
    }
    const request = indexedDB.open("user", 10);
    request.onerror = function (e) {
      // 错误处理
      console.log(" 打开数据库报错", e);
    };

    request.onupgradeneeded = function (e) {
      console.log(99999999999999);
      e.target.result && (db = e.target.result);
      //db = e.target.result;
      if (!db.objectStoreNames.contains("Users")) {
        var a = db.createObjectStore("Users", {
          keyPath: "id",
        });
        a.createIndex("name", "name", {
          unique: false,
        });
      }

      //   if (!db.objectStoreNames.contains("Users")) {

      //   }
      if (!db.objectStoreNames.contains("Awards")) {
        var d = db.createObjectStore("Awards", {
          keyPath: "id",
        });
        d.createIndex("name", "name", {
          unique: false,
        });
      }
    };
    request.onsuccess = function (e) {
      e.target.result && (db = e.target.result);
      // var obj1 = { hello: "world" };
      //var blob = new Blob([JSON.stringify(obj1, null, 2)], {
      //  type: "application/json",
      // });

      let aR = db.transaction(["Awards"], "readwrite").objectStore("Awards");
      [
        { name: "iphone12", num: 1 },
        { name: "小米电视", num: 3 },
        { name: "不沾锅", num: 5 },
        { name: "电吹风", num: 8 },
        { name: "日记本", num: 12 },
      ].forEach((element, index) => {
        aR.add({
          id: index + 1,
          name: element.name,
          image: "XXX",
          num: element.num,
        });
      });

      //   aR.addEventListener("success", (e) => {
      //     console.log(12333, e);
      //   });
    };
    //查询
    let img = "";
    return {
      awardList,
      lotteryList,
      startLottery,
      gradeLists,
      isStop,
      workerList,
      workerName,
      award,
      i,
      interval,
      workerId,
      gradeList,
      getData,
      writeDataBase,
      downloadExcel,
      cleanAward,
      showModal,
      img,
    };

    // var a = window.openDatabase("mydb", "1.0", "Test DB", 2 * 1024 * 1024);
    // console.log(9900, a);
  },
  watch: {
    isStop(val) {
      this.showModal = val;
    },
  },
  computed: {
    gradeImg() {
      if (
        this.awardList.value.length > 0 &&
        this.awardList.value[this.i].grade
      ) {
        switch (this.awardList.value[this.i].grade) {
          case "五等奖":
            return require("../assets/award5.png");
          case "四等奖":
            return require("../assets/award4.png");
          case "三等奖":
            return require("../assets/award3.png");
          case "二等奖":
            return require("../assets/award2.png");
          case "一等奖":
            return require("../assets/award1.png");
        }
      }
      return "";
    },
    buttonName() {
      if (this.i == 0) {
        return !this.isStop ? "停止抽一等奖 " : "本次抽奖已结束";
      } else {
        return this.isStop
          ? `开始抽${
              this.awardList.value[this.i]?.nextGrade ||
              this.awardList.value[this.i]?.grade
            }等奖`
          : "停止抽奖";
      }
    },
    awardImg() {
      // console.log(123456, this.awardList.value[this.i]?.img);
      if (this.awardList.value.length > 0 && this.awardList.value[this.i].img) {
        // if (this.i < 1) {
        //   this.i = 1;
        // }
        return require("../assets/" +
          this.awardList.value[this.i < 0 ? 0 : this.i].img +
          ".jpg");
      }
      return "";
    },
  },
  mounted() {
    // console.log(workbook);

    // const d = await workbook.xlsx.load(require("../assets/name_list.xlsx"));
    //console.log(8899, d);
    //workbook.xlsx.readFile("../assets/name_list.xlsx");
    // const response = await fetch("./name_list.xlsx");
    // const buffer = await response.arrayBuffer();
    // const options = {
    //   sharedStrings: "emit",
    //   hyperlinks: "emit",
    //   worksheets: "emit",
    // };
    let workerList = [];
    const workbook = new Excel.Workbook();
    const setWorkerList = (aList) => {
      this.workerList.value = aList;
    };

    const setAwardList = (aList) => {
      this.awardList.value = aList;
    };
    let gradeLists = this.gradeLists.value;
    // const workbook = ExcelJS.stream.xlsx.WorkbookReader(
    //   "./namelist.xlsx",
    //   options
    // );
    //workbook.read();
    fetch("./namelist.xlsx", {
      responseType: "arrayBuffer",
    })
      .then((res) => {
        return res.arrayBuffer();
      })
      .then(async (buffer) => {
        workbook.xlsx.load(buffer).then(async (d) => {
          const worksheet = d.getWorksheet(1);
          //const a = await worksheet.parse();
          worksheet.eachRow(function (row, rowNumber) {
            if (rowNumber > 1) {
              workerList.push({
                id: rowNumber,
                uid: row.values[2],
                name: row.values[3],
                num: 0,
              });
            }

            // console.log(
            //   "Row " + rowNumber + " = " + JSON.stringify(row.values)
            // );
          });
        });
        setWorkerList(workerList);
      });
    let awardList = [];
    fetch("./award_list.xlsx", {
      responseType: "arrayBuffer",
    })
      .then((res) => {
        return res.arrayBuffer();
      })
      .then(async (buffer) => {
        workbook.xlsx.load(buffer).then(async (d) => {
          const worksheet = d.getWorksheet(1);
          //const a = await worksheet.parse();
          worksheet.eachRow(function (row, rowNumber) {
            console.log(123, rowNumber, row.values[5]);
            if (rowNumber > 1) {
              if (rowNumber == 21) {
                gradeLists[rowNumber]
                  ? awardList.push({
                      name: row.values[3],
                      num: row.values[4],
                      grade: row.values[2],
                      nextGrade: gradeLists[rowNumber],
                      img: row.values[5] + "",
                    })
                  : awardList.push({
                      name: row.values[3],
                      num: row.values[4],
                      grade: row.values[2],
                      img: row.values[5],
                    });
              }
              if (rowNumber == 20) {
                gradeLists[rowNumber]
                  ? awardList.push({
                      name: row.values[3],
                      num: row.values[4],
                      grade: row.values[2],
                      nextGrade: gradeLists[rowNumber],
                      img: row.values[5] + "",
                    })
                  : awardList.push({
                      name: row.values[3],
                      num: row.values[4],
                      grade: row.values[2],
                      img: row.values[5],
                    });
              }
              if (rowNumber <= 17) {
                gradeLists[rowNumber]
                  ? awardList.push({
                      name: row.values[3],
                      num: row.values[4],
                      grade: row.values[2],
                      nextGrade: gradeLists[rowNumber],
                      img: row.values[5] + "",
                    })
                  : awardList.push({
                      name: row.values[3],
                      num: row.values[4],
                      grade: row.values[2],
                      img: row.values[5],
                    });
              }
            }
            // awardList.push({
            //   id: rowNumber,
            //   uid: row.values[2],
            //   name: row.values[3],
            //   num: 0,
            // });
            // console.log(
            //   "Row " + rowNumber + " = " + JSON.stringify(row.values)
            // );
          });
          setAwardList(awardList);
        });
        setAwardList(workerList);
      });
  },
});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="less" scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
.modal_bg {
  background-image: url("../assets/lotterry_modal.png");
  background-repeat: no-repeat;
  min-height: 48.3vw;
  position: fixed;
  z-index: 1000;
  width: 42.3vw;
  background-size: 100% auto;
  background-position-y: 5vw;
  left: 50%;
  top: 50%;
  cursor: pointer;
  transform: translate(-50%, -50%);
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
  align-items: center;

  & > div {
    color: #ffda43;
    font-size: 145%;
    margin-bottom: 11.8vw;
    & > :nth-child(1) {
      display: flex;
      flex-wrap: wrap-reverse;
      justify-content: center;
      margin-bottom: 20px;
      & > div {
        margin-right: 10px;
      }
    }
    & > :last-child {
      color: white;
      margin-top: 2px;
      font-size: 95%;
    }
  }
  & img {
    width: 11.5vw;

    line-height: 0;
  }
}
.modal {
  position: fixed;
  top: 0;
  z-index: 500;
  width: 100vw;
  height: 100vh;
  background-color: rgba(0, 0, 0, 0.8);
}
.bg_page {
  top: 0;
  margin: 0;
  padding: 0;
  background-image: url("../assets/BG.png");
  background-size: 100% 100%;
  height: 100vh;
  width: 100vw;
  overflow: hidden;
  display: block;
  clear: both;
  position: relative;
  & > :nth-child(1) {
    .grade_img {
      margin-bottom: -2.1vh;
    }
    position: absolute;
    left: 50%;
    transform: translateX(-50%);
    bottom: 20vh;
    & > :nth-child(2) {
      width: 33.3vh;
      height: 33.3vh;
      margin: 0 auto;
      background-image: url("../assets/award_area.png");
      background-size: 100% 100%;
      display: flex;
      align-items: center;
      justify-content: center;
      & > img {
        width: 90%;
        height: 90%;
      }
    }

    & > :nth-child(3) {
      background-image: url("../assets/name_BG.png");

      margin-bottom: 1.5vw;
      margin-top: 0.8vw;
      height: 5vh;
      line-height: 5vh;
      color: #ffda43;
      display: flex;
      flex-direction: row;
      justify-content: center;
      align-items: center;
      font-size: 140%;
      background-size: auto 100%;
      & > :nth-child(even) {
        margin-right: 10px;
        margin-left: 10px;
      }
    }
    & > :nth-child(4) {
      & > img {
        width: 40vw;
        display: block;
        line-height: 0;
      }
    }
  }
  & > :nth-child(2) {
    display: flex;
    width: 40vw;
    position: absolute;
    bottom: 0;
    z-index: 20;
    left: 50%;
    transform: translateX(-50%);
    & > img {
      width: 100%;
      cursor: pointer;
    }
  }
}
</style>
