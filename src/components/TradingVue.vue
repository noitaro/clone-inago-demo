<template>
  <trading-vue
    :data="this.$data"
    :width="this.width"
    :height="this.height"
    ref="tradingVue"
  ></trading-vue>
</template>

<script>
import TradingVue from "trading-vue-js";

const name = "localdb";
const version = "1.0";
const description = "Web SQL Database";
const size = 2 * 1024 * 1024;
let db = openDatabase(name, version, description, size);

db.transaction((tx) => {
  tx.executeSql("DROP TABLE IF EXISTS trade;");
  tx.executeSql(
    "CREATE TABLE IF NOT EXISTS trade (id INTEGER, price INTEGER, side TEXT, size INTEGER, timestamp TEXT);"
  );
});

// Create WebSocket connection.
const socket = new WebSocket("wss://stream.bybit.com/realtime");

// Connection opened
socket.addEventListener("open", () => {
  console.log("TradingVue: addEventListener(open)");
  socket.send(JSON.stringify({ op: "subscribe", args: ["trade"] }));
});

// Listen for messages
socket.addEventListener("message", onMessage);

function onMessage(event) {
  console.log("TradingVue: onMessage");
  const obj = JSON.parse(event.data);
  if (obj.topic == "trade.BTCUSD") {
    obj.data.forEach((element) => {
      //   console.log("Message from server ", element);
      db.transaction((tx) =>
        tx.executeSql(
          "INSERT INTO trade (id, price, side, size, timestamp) VALUES (?, ?, ?, ?, ?)",
          [
            element.cross_seq,
            element.price,
            element.side,
            element.size,
            element.timestamp,
          ]
        )
      );
    });
  }
}

export default {
  name: "app",
  components: { TradingVue },
  data() {
    return {
      ohlcv: [],
      width: window.innerWidth,
      height: window.innerHeight,
    };
  },
  mounted() {
    console.log("TradingVue: mounted");
    window.addEventListener("resize", this.onResize);
    this.onResize();
    this.$refs.tradingVue.resetChart();
    setInterval(this.fetchEventsList, 1000);
  },
  methods: {
    onResize() {
      this.width = window.innerWidth;
      this.height = window.innerHeight - 3;
    },
    fetchEventsList() {
      //   console.log("TradingVue: fetchEventsList");
      let ohlcv = [];
      db.transaction((trans) => {
        trans.executeSql(
          "SELECT " +
            "    timestamp " +
            "    , (SELECT price FROM trade WHERE id = min_id) AS open " +
            "    , high " +
            "    , low " +
            "    , (SELECT price FROM trade WHERE id = max_id) AS close " +
            "    , volume " +
            "FROM " +
            "    ( " +
            "        SELECT " +
            "            timestamp " +
            "            , MAX(id) AS max_id " +
            "            , MIN(id) AS min_id " +
            "            , MAX(price) AS high " +
            "            , MIN(price) AS low " +
            "            , ROUND(SUM(size), 2) AS volume " +
            "        FROM " +
            "            trade " +
            "        GROUP BY " +
            "            strftime('%Y%m%d%H%M', timestamp) " +
            "    ); ",
          [],
          (trans, r) => {
            // eslint-disable-line no-unused-vars
            // console.log(r.rows);
            for (let i = 0; i < r.rows.length; i++) {
              const dt = new Date(r.rows.item(i).timestamp);
              const date = new Date(
                dt.getFullYear(),
                dt.getMonth(),
                dt.getDay(),
                dt.getHours(),
                dt.getMinutes()
              );
              ohlcv.push([
                date.getTime(),
                r.rows.item(i).open,
                r.rows.item(i).high,
                r.rows.item(i).low,
                r.rows.item(i).close,
                r.rows.item(i).volume,
              ]);
            }
          }
        );
      });
      
      this.ohlcv = ohlcv;
    },
  },
};
</script>

<style>
</style>