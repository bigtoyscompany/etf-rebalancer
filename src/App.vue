<script setup lang="ts">
import { ref } from 'vue';

const cached_data = JSON.parse(localStorage.getItem('etfs') || 'null') as {
  etfs: any;
  balance_mode: string;
  amount: number;
  amount_remaining: number;
};

const etfs = ref((cached_data?.etfs || []) as {
  name: string;
  price: number;
  current_pos: number;
  target_pos: number;
  pos_diff: number;
  current_percent: number;
  after_percent: number;
  expect_percent: number;
  isAvg: boolean;
}[]);
const balance_mode = ref(cached_data?.balance_mode);
const amount = ref(cached_data?.amount);
const amount_remaining = ref(cached_data?.amount_remaining);

function compute() {
  let totalValue = 0;
  let totalAvgCount = 0;
  let totalAvgPercent = 100;
  etfs.value.forEach((item) => {
    totalValue += item.price * item.current_pos;
    if (item.isAvg) {
      totalAvgCount += 1;
    } else {
      totalAvgPercent -= item.expect_percent;
    }
  });

  let finalValue = (balance_mode.value === 're' ? totalValue + +amount.value : +amount.value);

  etfs.value.forEach((item) => {
    item.current_percent = Number.parseFloat((item.price * item.current_pos / totalValue * 100).toFixed(2));
    if (!balance_mode.value || amount.value === 0) {
      return;
    }
    if (item.isAvg) {
      item.target_pos = Math.floor((finalValue * totalAvgPercent / 100 / totalAvgCount / item.price));
    } else {
      item.target_pos = Math.floor((finalValue * item.expect_percent / 100 / item.price));
    }
    if (balance_mode.value === 're') {
      item.pos_diff = item.target_pos - item.current_pos;
    } else {
      item.pos_diff = item.target_pos;
      item.target_pos += +item.current_pos;
    }
    item.after_percent = Number.parseFloat((item.price * item.target_pos / (totalValue + +amount.value) * 100).toFixed(2));
  });

  const total_ops_value = etfs.value.reduce((acc, item) => acc + item.pos_diff * item.price, 0);
  amount_remaining.value = +amount.value > 0 ? +amount.value - total_ops_value : -total_ops_value;

  localStorage.setItem('etfs', JSON.stringify({
    etfs: etfs.value,
    amount: amount.value,
    amount_remaining: amount_remaining.value,
    balance_mode: balance_mode.value
  }));
}

function remove(index: number) {
  etfs.value.splice(index, 1);
  compute();
}

function set_mode(mode: string) {
  balance_mode.value = mode;
  compute();
}

function save() {
  amount.value = 0;
  etfs.value.forEach((item) => {
    item.current_pos = item.target_pos;
    item.target_pos = 0;
    item.pos_diff = 0;
    item.after_percent = 0;
  });
  compute();
}
</script>

<template>
  <div>
    <h1>ETF 再平衡计算器</h1>
    <button @click="etfs.push({} as any)">新增 ETF</button>&nbsp;
    <input @input="compute" type="number" v-model="amount" placeholder="本次操作金额">&nbsp;
    <button @click="set_mode('re')" :disabled="balance_mode == 're'">再平衡</button>&nbsp;
    <button @click="set_mode('eq')" :disabled="balance_mode == 'eq'">平均分配</button>&nbsp;
    <button @click="save">记录操作</button>&nbsp;
    <table>
      <thead>
        <tr>
          <th>名称</th>
          <th>价格</th>
          <th>当前持仓</th>
          <th>目标持仓</th>
          <th>本次操作</th>
          <th>当前占比</th>
          <th>操作后占比</th>
          <th>期望占比</th>
          <th>剩余平均</th>
          <th>操作</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(item, index) in etfs">
          <td><input type="text" v-model="item.name"></td>
          <td><input type="number" @input="compute" v-model="item.price"></td>
          <td><input type="number" @input="compute" v-model="item.current_pos"></td>
          <td>{{ item.target_pos }}</td>
          <td><span v-if="item.pos_diff > 0">+</span>{{ item.pos_diff }}</td>
          <td>{{ item.current_percent }}%</td>
          <td>{{ item.after_percent }}%</td>
          <td>
            <input v-if="!item.isAvg" type="number" @input="compute" v-model="item.expect_percent">
            <input v-else="!item.isAvg" type="text" disabled value="--">
            %
          </td>
          <td><input type="checkbox" @change="compute" v-model="item.isAvg"></td>
          <td>
            <button @click="remove(index)">删除</button>
          </td>
        </tr>
      </tbody>
    </table>
    <div v-if="amount >= 0">本次买入剩余：{{ amount_remaining }}</div>
    <div v-else>本次卖出所得：{{ amount_remaining }}</div>
    <br>
    <div><i>*金额为负数时为卖出</i></div>
    <div><i>*「剩余平均」的意思是扣去特别指定百分比的 ETF，剩余的百分比的平均分配给选中平均的 ETF</i></div>
    <p>Powered by <a href="https://lifelonglearn.ing">naiba</a></p>
  </div>
</template>

<style scoped></style>
