<template>
  <v-app>
    <v-main>
      <v-container>
        <v-row justify="center">
          <v-col cols="4" sm="2">
            <v-select label="アルゴリズム" :items="algorithms" v-model="h_Algorithm"
              @update:model-value="generate"></v-select>
          </v-col>
          <v-col cols="4" sm="2">
            <v-number-input label="マップの幅" :min="1" :max="128" v-model="h_Width" variant="outlined"
              control-variant="stacked" @update:model-value="generate"></v-number-input>
          </v-col>
          <v-col cols="4" sm="2">
            <v-number-input label="マップの高さ" :min="1" :max="128" v-model="h_Height" variant="outlined"
              control-variant="stacked" @update:model-value="generate"></v-number-input>
          </v-col>
          <v-col cols="4" sm="2" v-if="h_Algorithm === algorithms[0]">
            <v-number-input label="壁の破壊率" :min="0" :max="100" v-model="_h_breakWallRate" variant="outlined"
              control-variant="stacked" @update:model-value="generate"></v-number-input>
          </v-col>
          <v-col cols="4" sm="2" v-if="h_Algorithm === algorithms[1]">
            <v-number-input label="部屋の幅の最小値" :min="1" v-model="_h_minWidth" variant="outlined" control-variant="stacked"
              @update:model-value="generate"></v-number-input>
          </v-col>
          <v-col cols="4" sm="2" v-if="h_Algorithm === algorithms[1]">
            <v-number-input label="部屋の高さの最小値" :min="1" v-model="_h_minHeight" variant="outlined"
              control-variant="stacked" @update:model-value="generate"></v-number-input>
          </v-col>
          <v-col cols="4" sm="2" v-if="h_Algorithm === algorithms[1]">
            <v-number-input label="分割マージン" :min="1" v-model="_divideMargin" variant="outlined" control-variant="stacked"
              @update:model-value="generate"></v-number-input>
          </v-col>
          <!-- <v-col cols="4" sm="2" v-if="h_Algorithm === algorithms[1]">
            <v-number-input label="部屋数の最大値" :min="1" v-model="_h_maxRoomNum" variant="outlined"
              control-variant="stacked" @update:model-value="generate"></v-number-input>
          </v-col> -->

          <v-col cols="2" sm="2">
            <v-btn @click="generate" width="120" height="56" color="green-accent-4">再生成</v-btn>
          </v-col>
          <v-btn icon href="https://github.com/Eniwder/MazeSimulator" target="_blank" density="compact" variant="text"
            location="bottom right" position="absolute" class="githubicon">
            <v-icon>mdi-github</v-icon>
          </v-btn>
        </v-row>
        <v-row justify="center" class="canvasRow">
          <v-col cols="12" class="canvasCol">
            <canvas ref="canvasRef" width="660" height="660"></canvas>
          </v-col>
        </v-row>
      </v-container>
    </v-main>

  </v-app>
</template>

<script setup>
import { computed, nextTick, onMounted, reactive, ref, watch } from 'vue';


const _h_minWidth = ref(6);
const _h_minHeight = ref(6);
const _h_maxRoomNum = { value: 1000 }; //ref(10);
const algorithms = ['穴掘り法', '区域分割法'];
const h_Algorithm = ref('区域分割法');
const h_Width = ref(33);
const h_Height = ref(33);
const _divideMargin = ref(8);
const _h_breakWallRate = ref(0);
let scale = 5;

const canvasRef = ref();
let ctx = null;
let currentMap;

window.addEventListener('resize', debounce(onResize, 100));

onMounted(() => {
  onResize();
  // Canvas描画
  ctx = canvasRef.value.getContext("2d");
  generate();
});

class MyMap {
  constructor(width, height) {
    this._width = width;
    this._height = height;
    this._ground = [];
    this.Wall = 0;
    this.Route = 1;
    this._divideArea = [];
    this._divideLine = [];
    this._divideRoom = [];

    // マップの初期化
    for (let i = 0; i < this._height; ++i) {
      this._ground[i] = [];
      for (let j = 0; j < this._width; ++j) {
        this._ground[i][j] = this.Wall;
      }
    }
  }

  // 区間分割
  InsertDivideArea(left, top, right, bottom, id) {
    this._divideArea.push([left, top, right, bottom, id]);
  }

  // 分割する線
  InsertDivideLine(sx, sy, tx, ty, dir) {
    this._divideLine.push([sx, sy, tx, ty, dir]);
  }

  // 部屋を作成
  InsertDivideRoom(left, top, right, bottom) {
    this._divideRoom.push([left, top, right, bottom]);
  }

  // 領域外かどうかの判定
  IsOutOfRange(x, y) {
    return (x < 0 || x >= this._width || y < 0 || y >= this._height);
  }

  // 要素の取得
  Get(x, y) {
    if (this.IsOutOfRange(x, y)) return -1;
    return this._ground[y][x];
  }

  // 要素の書き換え
  Set(x, y, v) {
    if (this.IsOutOfRange(x, y)) return;
    this._ground[y][x] = v;
  }

  // 範囲内を全て書き換える
  AllSet(left, top, right, bottom, v) {
    for (let i = top; i <= bottom; ++i) {
      for (let j = left; j <= right; ++j) {
        this.Set(j, i, v);
      }
    }
  }

  // マップを表示（Canvas描画）
  Show(ctx, scale) {
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
    for (let i = 0; i < this._height; ++i) {
      for (let j = 0; j < this._width; ++j) {
        if (this._ground[i][j] === this.Wall) {
          ctx.fillStyle = "#00C853";
        } else {
          ctx.fillStyle = "black";
        }
        // ctx.fillRect(j * scale, i * scale, scale, scale); // 黒の時はグリッドなくてもいいかも
        ctx.fillRect(j * scale, i * scale, scale - 1, scale - 1);
      }
    }
  }
}

// 区間を分割する
function AreaDivide(myMap) {

  let lowerWidth = 0;
  let upperWidth = myMap._width - 1;
  let lowerHeight = 0;
  let upperHeight = myMap._height - 1;
  let roomId = 0;

  let isVertical = Math.random() < 0.5;
  const h_minWidth = _h_minWidth.value;
  const h_minHeight = _h_minHeight.value;
  const h_maxRoomNum = _h_maxRoomNum.value;
  const divideMargin = _divideMargin.value;

  while (true) {
    // 分割できない条件
    if (upperWidth - lowerWidth < h_minWidth + divideMargin) break;
    if (upperHeight - lowerHeight < h_minHeight + divideMargin) break;
    if (roomId >= h_maxRoomNum) break;

    let dividePoint;

    if (isVertical) {
      dividePoint = GetRandomInt(lowerWidth + 2 + h_minWidth, upperWidth - h_minWidth - 2);
      if ((upperWidth + lowerWidth) / 2 < dividePoint) {
        myMap.InsertDivideArea(dividePoint + 2, lowerHeight + 1, upperWidth - 1, upperHeight - 1, roomId);
        upperWidth = dividePoint - 1;
      } else {
        myMap.InsertDivideArea(lowerWidth + 1, lowerHeight + 1, dividePoint - 2, upperHeight - 1, roomId);
        lowerWidth = dividePoint + 1;
      }
      myMap.InsertDivideLine(dividePoint, lowerHeight + 1, dividePoint, upperHeight - 1, 0);
      isVertical = false;
    } else {
      dividePoint = GetRandomInt(lowerHeight + h_minHeight + 2, upperHeight - h_minHeight - 2);
      if ((upperHeight + lowerHeight) / 2 < dividePoint) {
        myMap.InsertDivideArea(lowerWidth + 1, dividePoint + 2, upperWidth - 1, upperHeight - 1, roomId);
        upperHeight = dividePoint - 1;
      } else {
        myMap.InsertDivideArea(lowerWidth + 1, lowerHeight + 1, upperWidth - 1, dividePoint - 2, roomId);
        lowerHeight = dividePoint + 1;
      }
      myMap.InsertDivideLine(lowerWidth + 1, dividePoint, upperWidth - 1, dividePoint, 1);
      isVertical = true;
    }
    ++roomId;
  }
}

// ランダムな整数を取得
function GetRandomInt(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}

// 穴掘りアルゴリズム
function Dig(myMap, x, y) {
  myMap.Set(x, y, myMap.Route);

  const moveX = [0, 0, -1, 1];
  const moveY = [1, -1, 0, 0];
  let act = [0, 1, 2, 3];

  // 行動順をランダムにシャッフル
  act = act.sort(() => 0.5 - Math.random());

  for (let a of act) {
    const dx = moveX[a];
    const dy = moveY[a];

    if (myMap.Get(x + dx * 2, y + dy * 2) === myMap.Wall) {
      myMap.Set(x + dx, y + dy, myMap.Route);
      Dig(myMap, x + dx * 2, y + dy * 2);
    }
  }
}

// 穴掘り開始
function DigStart(width, height, ctx, scale) {
  const myMap = new MyMap(width, height);
  let startX = GetRandomInt(0, width - 2);
  let startY = GetRandomInt(0, height - 2);
  if (startY % 2 === 0) startY++;
  if (startX % 2 === 0) startX++;

  Dig(myMap, startX, startY);
  randomDig(myMap, width, height);
  myMap.Show(ctx, scale);
  return myMap;
}

function randomDig(myMap, width, height) {
  const walls = [...Array(height).keys()].flatMap(y =>
    [...Array(width).keys()].filter(x => myMap.Get(x, y) === myMap.Wall).map(x => [x, y])
  );
  const breakNum = parseInt(walls.length * (_h_breakWallRate.value / 100.0));
  let rest = [];
  shuffle(walls).slice(0, breakNum).forEach(([x, y]) => {
    if (myMap.Get(x - 1, y) === myMap.Route || myMap.Get(x + 1, y) === myMap.Route ||
      myMap.Get(x, y - 1) === myMap.Route || myMap.Get(x, y + 1) === myMap.Route) {
      myMap.Set(x, y, myMap.Route);
    } else {
      rest.push([x, y]);
    }
  });
  let restLen = rest.length;
  while (true) {
    // よくないけどめんどい
    rest = rest.filter(([x, y]) => {
      if (myMap.Get(x - 1, y) === myMap.Route || myMap.Get(x + 1, y) === myMap.Route ||
        myMap.Get(x, y - 1) === myMap.Route || myMap.Get(x, y + 1) === myMap.Route) {
        myMap.Set(x, y, myMap.Route);
        return false;
      }
      return true;
    });
    if (rest.length === restLen) break;
    restLen = rest.length;
  }
  function shuffle(array) {
    let n = array.length;
    for (let i = n - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [array[i], array[j]] = [array[j], array[i]];  // 要素の入れ替え
    }
    return array;
  }
}

// 分割法開始
function DivideStart(width, height, ctx, scale) {
  const myMap = new MyMap(width, height);
  const h_minWidth = _h_minWidth.value;
  const h_minHeight = _h_minHeight.value;
  const h_maxRoomNum = _h_maxRoomNum.value;

  // マップを区画に分ける
  AreaDivide(myMap);
  // 区画に収まる部屋を用意
  CreateRoom(myMap);
  // 通路を伸ばす
  CreateRoute(myMap);

  for (let a of myMap._divideRoom) {
    myMap.AllSet(a[0], a[1], a[2], a[3], myMap.Route);
  }

  // 部屋を作成する
  function CreateRoom(myMap) {
    for (let v of myMap._divideArea) {
      const [left, top, right, bottom] = v;
      const roomLeft = GetRandomInt(left, right - h_minWidth + 1);
      const roomRight = GetRandomInt(roomLeft + h_minWidth - 1, right);
      const roomTop = GetRandomInt(top, bottom - h_minHeight + 1);
      const roomBottom = GetRandomInt(roomTop + h_minHeight - 1, bottom);

      myMap.InsertDivideRoom(roomLeft, roomTop, roomRight, roomBottom);
    }
  }

  // 部屋から通路を伸ばす
  function CreateRoute(myMap) {
    let cnt = 0;
    const size = myMap._divideLine.length - 1;

    for (let v of myMap._divideLine) {
      if (cnt >= size) break;

      const [sx, sy, tx, ty, dir] = v;
      const roomBefore = myMap._divideRoom[cnt];
      const roomAfter = myMap._divideRoom[cnt + 1];

      const [leftBefore, topBefore, rightBefore, bottomBefore] = roomBefore;
      const [leftAfter, topAfter, rightAfter, bottomAfter] = roomAfter;

      if (dir === 0) {
        let routeBefore = GetRandomInt(topBefore + 1, bottomBefore - 1);
        let routeAfter = GetRandomInt(topAfter + 1, bottomAfter - 1);
        if (leftBefore < leftAfter) {
          myMap.AllSet(rightBefore, routeBefore, sx, routeBefore, myMap.Route);
          myMap.AllSet(sx, routeAfter, leftAfter, routeAfter, myMap.Route);
        } else {
          myMap.AllSet(rightAfter, routeAfter, sx, routeAfter, myMap.Route);
          myMap.AllSet(sx, routeBefore, leftBefore, routeBefore, myMap.Route);
        }
        if (routeBefore > routeAfter) [routeBefore, routeAfter] = [routeAfter, routeBefore];
        myMap.AllSet(sx, routeBefore, sx, routeAfter, myMap.Route);

      } else {
        let routeBefore = GetRandomInt(leftBefore + 1, rightBefore - 1);
        let routeAfter = GetRandomInt(leftAfter + 1, rightAfter - 1);

        if (topBefore < topAfter) {
          myMap.AllSet(routeBefore, bottomBefore, routeBefore, sy, myMap.Route);
          myMap.AllSet(routeAfter, sy, routeAfter, topAfter, myMap.Route);
        } else {
          myMap.AllSet(routeAfter, bottomAfter, routeAfter, sy, myMap.Route);
          myMap.AllSet(routeBefore, sy, routeBefore, topBefore, myMap.Route);
        }
        if (routeBefore > routeAfter) [routeBefore, routeAfter] = [routeAfter, routeBefore];
        myMap.AllSet(routeBefore, sy, routeAfter, sy, myMap.Route);
      }
      ++cnt;
    }
  }

  myMap.Show(ctx, scale);
  return myMap;
}

function generate() {
  const base = canvasRef.value.parentNode.parentNode;
  currentMap = null;
  onResize();
  if (h_Algorithm.value === algorithms[0]) {
    currentMap = DigStart(h_Width.value, h_Height.value, ctx, scale);
  } else {
    currentMap = DivideStart(h_Width.value, h_Height.value, ctx, scale);
  }
}

function onResize() {
  const base = canvasRef.value.parentNode.parentNode;
  scale = Math.max(2, Math.floor(Math.min((base.clientWidth / h_Width.value), (base.clientHeight / h_Height.value))));
  canvasRef.value.setAttribute("width", scale * (h_Width.value + 1));
  canvasRef.value.setAttribute("height", scale * (h_Height.value + 1));

  if (currentMap && ctx) currentMap.Show(ctx, scale);
}

function debounce(func, timeout) {
  let timer;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => {
      func.apply(this, args);
    }, timeout);
  };
}

</script>

<style>
.githubicon {
  margin: 24px;
}

.canvasRow {
  height: min(64vh, 80vw);
  width: 80vw;
}

.canvasCol {
  flex: 0 0 !important;
}
</style>
