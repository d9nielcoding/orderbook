<script setup lang="ts">
import IconArrowDown from "@/assets/icon/IconArrowDown.vue";
import { computed, onMounted, onUnmounted, ref, watch } from "vue";

interface Order {
  price: number;
  size: number;
  isNew?: boolean;
  sizeIncreased?: boolean;
  sizeDecreased?: boolean;
}

interface OrderWithTotal extends Order {
  total: number;
  percentage: number;
}

type RawOrder = [number, number];

function convertRawOrder(rawOrder: RawOrder): Order {
  const [price, size] = rawOrder;
  return {
    price,
    size,
    isNew: false,
    sizeIncreased: false,
    sizeDecreased: false,
  };
}

interface WebSocketMessage {
  type: "snapshot" | "delta";
  symbol: string;
  timestamp: number;
  seqNum: number;
  prevSeqNum?: number;
  asks: RawOrder[];
  bids: RawOrder[];
}

interface TradeMessage {
  price: number;
  side: "BUY" | "SELL";
  size: number;
  symbol: string;
  timestamp: number;
  tradeId: number;
}

const sellOrders = ref<Order[]>([]);
const buyOrders = ref<Order[]>([]);
const lastPrice = ref<number>(0);
const previousPrice = ref<number>(0);
const hoveredRow = ref<number | null>(null);
let wsOrderBook: WebSocket | null = null;
let wsPrice: WebSocket | null = null;
let lastSeqNum: number | null = null;

const quoteTotalSellSize = computed(() => {
  return sellOrders.value
    .filter((order) => order.size > 0)
    .reduce((acc, order) => acc + Number(order.size), 0);
});

const quoteTotalBuySize = computed(() => {
  return buyOrders.value
    .filter((order) => order.size > 0)
    .reduce((acc, order) => acc + Number(order.size), 0);
});

const displayedSellOrders = computed<OrderWithTotal[]>(() => {
  return sellOrders.value
    .filter((order) => order.size > 0)
    .sort((a, b) => a.price - b.price) // Sort from lowest to highest
    .slice(0, 8)
    .map((order, index, array) => {
      const accumulativeTotal = calculateSellTotal(index, array);
      return {
        ...order,
        total: accumulativeTotal,
        percentage: (accumulativeTotal / quoteTotalSellSize.value) * 100,
      };
    });
});

const displayedBuyOrders = computed<OrderWithTotal[]>(() => {
  return buyOrders.value
    .filter((order) => order.size > 0)
    .sort((a, b) => b.price - a.price) // Sort from highest to lowest
    .slice(0, 8)
    .map((order, index, array) => {
      const accumulativeTotal = calculateBuyTotal(index, array);
      return {
        ...order,
        total: accumulativeTotal,
        percentage: (accumulativeTotal / quoteTotalBuySize.value) * 100,
      };
    });
});

const priceDirection = computed<"up" | "down" | "same">(() => {
  if (lastPrice.value > previousPrice.value) return "up";
  if (lastPrice.value < previousPrice.value) return "down";
  return "same";
});

function formatNumber(num: number): string {
  return num.toLocaleString("en-US", { maximumFractionDigits: 1 });
}

function calculateSellTotal(index: number, array: Order[]): number {
  return array.slice(index).reduce((acc, order) => acc + Number(order.size), 0);
}

function calculateBuyTotal(index: number, array: Order[]): number {
  return array
    .slice(0, index + 1)
    .reduce((acc, order) => acc + Number(order.size), 0);
}

function connectWebSocket(): void {
  // OrderBook WebSocket
  wsOrderBook = new WebSocket("wss://ws.btse.com/ws/oss/futures");
  wsOrderBook.onopen = () => {
    if (wsOrderBook) {
      wsOrderBook.send(
        JSON.stringify({
          op: "subscribe",
          args: ["update:BTCPFC"],
        })
      );
    }
  };
  wsOrderBook.onmessage = handleOrderBookMessage;

  // Price WebSocket
  wsPrice = new WebSocket("wss://ws.btse.com/ws/futures");
  wsPrice.onopen = () => {
    if (wsPrice) {
      wsPrice.send(
        JSON.stringify({
          op: "subscribe",
          args: ["tradeHistoryApi:BTCPFC"],
        })
      );
    }
  };
  wsPrice.onmessage = handlePriceMessage;
}

function handleOrderBookMessage(event: MessageEvent): void {
  const rawMessage = JSON.parse(event.data);
  if (rawMessage.data === undefined) {
    return;
  }
  const data = rawMessage.data as WebSocketMessage;

  if (data.type === "snapshot") {
    lastSeqNum = data.seqNum;
    sellOrders.value = data.asks.map(convertRawOrder);
    buyOrders.value = data.bids.map(convertRawOrder);
  } else if (data.type === "delta") {
    if (data.prevSeqNum !== lastSeqNum) {
      if (wsOrderBook) {
        wsOrderBook.send(
          JSON.stringify({
            op: "unsubscribe",
            args: ["update:BTCPFC"],
          })
        );
        wsOrderBook.send(
          JSON.stringify({
            op: "subscribe",
            args: ["update:BTCPFC"],
          })
        );
      }
      return;
    }

    lastSeqNum = data.seqNum;
    updateOrders(data);
  }
}

function handlePriceMessage(event: MessageEvent): void {
  const rawMessage = JSON.parse(event.data);
  if (rawMessage.data === undefined) {
    return;
  }
  const data = rawMessage.data as TradeMessage[];

  if (Array.isArray(data) && data.length > 0) {
    previousPrice.value = lastPrice.value;
    lastPrice.value = data[0].price;
  }
}

function updateOrders(data: WebSocketMessage): void {
  // Update asks
  data.asks.forEach((rawAsk) => {
    const newAsk = convertRawOrder(rawAsk);
    if (newAsk.size === 0) {
      // Remove order if size is 0
      const index = sellOrders.value.findIndex(
        (ask) => ask.price === newAsk.price
      );
      if (index !== -1) {
        sellOrders.value.splice(index, 1);
      }
      return;
    }
    const index = sellOrders.value.findIndex(
      (ask) => ask.price === newAsk.price
    );
    if (index === -1) {
      newAsk.isNew = true;
      setTimeout(() => {
        newAsk.isNew = false;
      }, 1000);
      sellOrders.value.push(newAsk);
    } else {
      const oldSize = sellOrders.value[index].size;
      sellOrders.value[index] = {
        ...newAsk,
        sizeIncreased: newAsk.size > oldSize,
        sizeDecreased: newAsk.size < oldSize,
      };
      setTimeout(() => {
        if (sellOrders.value[index]) {
          sellOrders.value[index].sizeIncreased = false;
          sellOrders.value[index].sizeDecreased = false;
        }
      }, 1000);
    }
  });

  // Update bids
  data.bids.forEach((rawBid) => {
    const newBid = convertRawOrder(rawBid);
    if (newBid.size === 0) {
      // Remove order if size is 0
      const index = buyOrders.value.findIndex(
        (bid) => bid.price === newBid.price
      );
      if (index !== -1) {
        buyOrders.value.splice(index, 1);
      }
      return;
    }
    const index = buyOrders.value.findIndex(
      (bid) => bid.price === newBid.price
    );
    if (index === -1) {
      newBid.isNew = true;
      setTimeout(() => {
        newBid.isNew = false;
      }, 1000);
      buyOrders.value.push(newBid);
    } else {
      const oldSize = buyOrders.value[index].size;
      buyOrders.value[index] = {
        ...newBid,
        sizeIncreased: newBid.size > oldSize,
        sizeDecreased: newBid.size < oldSize,
      };
      setTimeout(() => {
        if (buyOrders.value[index]) {
          buyOrders.value[index].sizeIncreased = false;
          buyOrders.value[index].sizeDecreased = false;
        }
      }, 1000);
    }
  });

  // Sort and trim orders
  sellOrders.value = sellOrders.value
    .filter((order) => order.size > 0)
    .sort((a, b) => b.price - a.price)
    .slice(0, 8);
  buyOrders.value = buyOrders.value
    .filter((order) => order.size > 0)
    .sort((a, b) => b.price - a.price)
    .slice(0, 8);
}

const updateTitle = () => {
  if (lastPrice.value) {
    document.title = `BTCPFC ${
      priceDirection.value === "up"
        ? "↑"
        : priceDirection.value === "down"
        ? "↓"
        : ""
    } ${lastPrice.value} | Order Book`;
  } else {
    document.title = "Order Book";
  }
};

// Watch for price changes
watch(lastPrice, () => {
  updateTitle();
});

// Initial title update
onMounted(() => {
  connectWebSocket();
  updateTitle();
});

onUnmounted(() => {
  if (wsOrderBook) wsOrderBook.close();
  if (wsPrice) wsPrice.close();
});
</script>

<template>
  <div class="order-book">
    <h2>Order Book</h2>

    <!-- Sell Orders -->
    <div class="order-table">
      <div class="table-header" :style="{ color: '#8698aa' }">
        <div>Price (USDT)</div>
        <div>Size (BTC)</div>
        <div>Total (BTC)</div>
      </div>
      <div class="order-row-wrapper">
        <div
          v-for="order in displayedSellOrders"
          :key="order.price"
          class="order-row"
          :class="{
            'flash-red': order.isNew,
            'size-increase': order.sizeIncreased,
            'size-decrease': order.sizeDecreased,
          }"
          @mouseover="hoveredRow = order.price"
          @mouseleave="hoveredRow = null"
          :style="{
            backgroundColor:
              hoveredRow === order.price ? '#1E3059' : 'transparent',
          }"
        >
          <div
            class="price"
            :style="{ color: '#FF5B5A', paddingRight: '10px' }"
          >
            {{ formatNumber(order.price) }}
          </div>
          <div class="size">{{ formatNumber(order.size) }}</div>
          <div class="total">{{ formatNumber(order.total) }}</div>
          <div
            class="total-bar"
            :style="{
              width: order.percentage + '%',
              backgroundColor: 'rgba(255, 90, 90, 0.12)',
              opacity: 0.5,
            }"
          ></div>
        </div>
      </div>
    </div>

    <!-- Last Price -->
    <div class="last-price" :class="priceDirection">
      {{ formatNumber(lastPrice) }}
      <IconArrowDown
        v-show="priceDirection !== 'same'"
        :class="priceDirection"
        :size="15"
      />
    </div>

    <!-- Buy Orders -->
    <div class="order-table">
      <div class="buy-orders">
        <div
          v-for="order in displayedBuyOrders"
          :key="order.price"
          class="order-row"
          :class="{
            'flash-green': order.isNew,
            'size-increase': order.sizeIncreased,
            'size-decrease': order.sizeDecreased,
          }"
          @mouseover="hoveredRow = order.price"
          @mouseleave="hoveredRow = null"
          :style="{
            backgroundColor:
              hoveredRow === order.price ? '#1E3059' : 'transparent',
          }"
        >
          <div
            class="price"
            :style="{ color: '#00b15d', paddingRight: '10px' }"
          >
            {{ formatNumber(order.price) }}
          </div>
          <div class="size">{{ formatNumber(order.size) }}</div>
          <div class="total">{{ formatNumber(order.total) }}</div>
          <div
            class="total-bar"
            :style="{
              width: order.percentage + '%',
              backgroundColor: 'rgba(16, 186, 104, 0.12)',
            }"
          ></div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.order-book {
  background-color: #131b29;
  color: #f0f4f8;
  padding: 20px;
  border-radius: 8px;
  font-family: "Lato", sans-serif;
}

h2 {
  font-size: 18px;
  font-weight: 700;
  color: #fff;
  margin-left: 10px;
}

.order-table {
  width: 100%;
  margin-bottom: 20px;
}

.table-header {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  padding: 5px 10px;
  font-size: 14px;
  text-align: right;
}

.order-row {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  /* padding: 1px 0 2px 10px; */
  margin: 1px 0 2px 10px;
  text-align: right;
  cursor: pointer;
  position: relative;
}

.order-row .total-bar {
  position: absolute;
  top: 0;
  right: 0;
  height: 100%;
  z-index: 0;
  transition: width 0.3s ease-out;
  pointer-events: none;
}

.order-row > div {
  position: relative;
  z-index: 1;
}

.order-row-wrapper {
  position: relative;
}

.last-price {
  text-align: center;
  padding: 10px;
  margin: 10px 0;
  font-size: 18px;
  font-weight: bold;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  transition: color 0.2s;
}

.last-price.up {
  color: #00b15d;
}

.last-price.down {
  color: #ff5b5a;
}

.last-price img {
  width: 12px;
  height: 12px;
  transition: transform 0.2s;
}

.last-price svg.up {
  transform: rotate(180deg);
}

@keyframes flash-green {
  from {
    background-color: rgba(0, 177, 93, 0.5);
  }
  to {
    background-color: transparent;
  }
}

@keyframes flash-red {
  from {
    background-color: rgba(255, 91, 90, 0.5);
  }
  to {
    background-color: transparent;
  }
}

.flash-green {
  animation: flash-green 0.5s ease-out;
}

.flash-red {
  animation: flash-red 0.5s ease-out;
}

.size-increase .size {
  animation: flash-green 0.5s ease-out;
}

.size-decrease .size {
  animation: flash-red 0.5s ease-out;
}
</style>
