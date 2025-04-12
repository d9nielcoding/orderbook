<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref } from "vue";

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
  [key: string]: any;
}

const sellOrders = ref<Order[]>([]);
const buyOrders = ref<Order[]>([]);
const lastPrice = ref<number>(0);
const previousPrice = ref<number>(0);
const hoveredRow = ref<number | null>(null);
let wsOrderBook: WebSocket | null = null;
let wsPrice: WebSocket | null = null;
let lastSeqNum: number | null = null;

const displayedSellOrders = computed<OrderWithTotal[]>(() => {
  return sellOrders.value.slice(0, 8).map((order, index, array) => ({
    ...order,
    total: calculateSellTotal(index, array),
    percentage: (calculateSellTotal(index, array) / getTotalSellSize()) * 100,
  }));
});

const displayedBuyOrders = computed<OrderWithTotal[]>(() => {
  return buyOrders.value.slice(0, 8).map((order, index, array) => ({
    ...order,
    total: calculateBuyTotal(index, array),
    percentage: (calculateBuyTotal(index, array) / getTotalBuySize()) * 100,
  }));
});

const priceDirection = computed<"up" | "down" | "same">(() => {
  if (lastPrice.value > previousPrice.value) return "up";
  if (lastPrice.value < previousPrice.value) return "down";
  return "same";
});

const priceChangeClass = computed(() => ({
  "price-up": priceDirection.value === "up",
  "price-down": priceDirection.value === "down",
  "price-same": priceDirection.value === "same",
}));

function formatNumber(num: number): string {
  return num.toLocaleString("en-US", { maximumFractionDigits: 1 });
}

function calculateSellTotal(index: number, array: Order[]): number {
  return array.slice(0, index + 1).reduce((acc, order) => acc + order.size, 0);
}

function calculateBuyTotal(index: number, array: Order[]): number {
  return array.slice(0, index + 1).reduce((acc, order) => acc + order.size, 0);
}

function getTotalSellSize(): number {
  return sellOrders.value.reduce((acc, order) => acc + order.size, 0);
}

function getTotalBuySize(): number {
  return buyOrders.value.reduce((acc, order) => acc + order.size, 0);
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
  const data = JSON.parse(event.data).data as WebSocketMessage;
  console.log("[OrderBook] handleOrderBookMessage", data);

  if (data.type === "snapshot") {
    // Handle initial snapshot
    lastSeqNum = data.seqNum;
    sellOrders.value = data.asks.map(convertRawOrder);
    buyOrders.value = data.bids.map(convertRawOrder);
  } else if (data.type === "delta") {
    // Handle incremental updates
    if (data.prevSeqNum !== lastSeqNum) {
      // Sequence mismatch, resubscribe
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
  const data = JSON.parse(event.data) as TradeMessage[];
  if (Array.isArray(data) && data.length > 0) {
    previousPrice.value = lastPrice.value;
    lastPrice.value = data[0].price;
  }
}

function updateOrders(data: WebSocketMessage): void {
  // Update asks
  data.asks.forEach((rawAsk) => {
    const newAsk = convertRawOrder(rawAsk);
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
    .sort((a, b) => b.price - a.price)
    .slice(0, 8);
  buyOrders.value = buyOrders.value
    .sort((a, b) => b.price - a.price)
    .slice(0, 8);
}

onMounted(() => {
  connectWebSocket();
});

onUnmounted(() => {
  if (wsOrderBook) wsOrderBook.close();
  if (wsPrice) wsPrice.close();
});
</script>

<template>
  <div
    class="order-book"
    :style="{ backgroundColor: '#131B29', color: '#F0F4F8' }"
  >
    <h2>Order Book</h2>

    <!-- Sell Orders -->
    <div class="order-table">
      <div class="table-header" :style="{ color: '#8698aa' }">
        <div>Price (USD)</div>
        <div>Size</div>
        <div>Total</div>
      </div>
      <div class="sell-orders">
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
          <div class="price" :style="{ color: '#FF5B5A' }">
            {{ formatNumber(order.price) }}
          </div>
          <div class="size">{{ formatNumber(order.size) }}</div>
          <div class="total">{{ formatNumber(order.total) }}</div>
          <div
            class="total-bar"
            :style="{
              width: order.percentage + '%',
              backgroundColor: 'rgba(255, 90, 90, 0.12)',
            }"
          ></div>
        </div>
      </div>
    </div>

    <!-- Last Price -->
    <div class="last-price" :class="priceChangeClass">
      {{ formatNumber(lastPrice) }}
      <img
        src="../assets/IconArrowDown.svg"
        alt="arrow"
        :class="{ up: priceDirection === 'up' }"
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
          <div class="price" :style="{ color: '#00b15d' }">
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
  width: 100%;
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
    Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
}

.order-table {
  width: 100%;
  margin-bottom: 20px;
}

.table-header {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  padding: 10px;
  font-size: 14px;
  text-align: right;
}

.order-row {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  padding: 10px;
  position: relative;
  text-align: right;
  cursor: pointer;
}

.total-bar {
  position: absolute;
  top: 0;
  right: 0;
  height: 100%;
  z-index: 1;
}

.order-row > div {
  position: relative;
  z-index: 2;
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
}

.last-price img {
  width: 12px;
  height: 12px;
  transition: transform 0.2s;
}

.last-price img.up {
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
  animation: flash-green 1s ease-out;
}

.flash-red {
  animation: flash-red 1s ease-out;
}

.size-increase {
  animation: flash-green 1s ease-out;
}

.size-decrease {
  animation: flash-red 1s ease-out;
}
</style>
