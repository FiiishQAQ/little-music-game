<script setup>
import {onMounted, reactive, ref} from "vue";
import {thePiano} from "../assets/the piano.js";
import {NOTE_TYPE} from "../assets/NOTE_TYPE.js";

const VISUAL_DELAY = 0;
const MAX_CHECK = 300;
const CHECK_RESULT = {
  PERFECT: 'perfect',
  GOOD: 'good',
  OK: 'ok',
  BAD: 'bad',
  MISS: 'miss',
}

let NOTE_ANIMATION_DUR = ref(2000);

let score = reactive({sum: 0, cnt: 0});
let play = ref(false);
let end = ref(false);
let mousePos = reactive({x: "", y: ""})
let clickPrompt = reactive({
  * [Symbol.iterator]() {
    let keys = Reflect.ownKeys(this);
    for (let i = 0; i < keys.length; i++) {
      if (keys[i] !== Symbol.iterator) {
        yield this[keys[i]];
      }
    }
  }
});
let notePrompt = reactive({
  * [Symbol.iterator]() {
    let keys = Reflect.ownKeys(this);
    for (let i = 0; i < keys.length; i++) {
      if (keys[i] !== Symbol.iterator) {
        yield this[keys[i]];
      }
    }
  }
});

let checkList = [];

function wrapSheet(sheet) {
  sheet[Symbol.iterator] = function* () {
    let speed = 60 * 1000 / sheet.tempo;
    for (let item of sheet.notes) {
      yield {
        noteDuration: speed * (4 / item[0]),
        type: item[1],
      };
    }
  }
  return sheet;
}

let sheet = wrapSheet(thePiano);

function setMousePos(x, y) {
  mousePos.x = `${x}px`;
  mousePos.y = `${y}px`;
}

function addClickPrompt(status = "") {
  let id = Symbol(Date.now());
  clickPrompt[id] = {
    id,
    status
  }
  setTimeout(() => {
    delete clickPrompt[id]
  }, 300);
}

function addNotePrompt(type) {
  let id = Symbol(Date.now());
  notePrompt[id] = {
    id,
    type
  }

  let remover = () => {
    if(notePrompt[id]) delete notePrompt[id]
  };
  setTimeout(remover, NOTE_ANIMATION_DUR.value);

  return remover;
}

async function readAndInitNote() {
  for (let note of sheet) {
    let toBestTimeDur = NOTE_ANIMATION_DUR.value

    if (note.type !== NOTE_TYPE.NONE) {
      console.log(note)
      let expBestTime = performance.now() + toBestTimeDur;

      checkList.push(
          {
            expBestTime,
            timer: setTimeout(() => {
              checkList.shift();
              score.cnt++;
              addClickPrompt(CHECK_RESULT.MISS);
              // console.log(CHECK_RESULT.MISS);
            }, toBestTimeDur + MAX_CHECK),
            remover: addNotePrompt(),
          });

    }
    await new Promise(resolve => setTimeout(() => resolve(), note.noteDuration));
  }
  setTimeout(()=>end.value = true,5000);
}

let sum1 = 0;
let cnt1 = 0;
function checkNote() {
  let time = performance.now();
  if (!checkList.length || checkList[0].expBestTime - time > MAX_CHECK) {
    addClickPrompt();
    return;
  }
  let note = checkList.shift();
  clearTimeout(note.timer)
  let dur = Math.abs(note.expBestTime - time);
  sum1 += dur;
  cnt1++;
  console.log("diff dur: ", dur.toFixed(2), (sum1/cnt1).toFixed(2))
  let thisScore = 100;
  if (dur <= 60) {
    addClickPrompt(CHECK_RESULT.PERFECT);
    // console.log(CHECK_RESULT.PERFECT);
  } else if (dur <= 90) {
    thisScore = 80;
    addClickPrompt(CHECK_RESULT.GOOD);
    // console.log(CHECK_RESULT.GOOD);
  } else if (dur <= 150) {
    thisScore = 60;
    addClickPrompt(CHECK_RESULT.OK);
    // console.log(CHECK_RESULT.OK);
  } else if (dur <= MAX_CHECK) {
    thisScore = 40;
    addClickPrompt(CHECK_RESULT.BAD);
    // console.log(CHECK_RESULT.BAD);
  }
  note.remover();

  score.sum += thisScore;
  score.cnt++;
}


function startGame() {
  let audio = document.querySelector("#audio");
  setTimeout(()=> {
    audio.play();
  }, NOTE_ANIMATION_DUR.value + VISUAL_DELAY);
  play.value = true;
  readAndInitNote();
}

function initInteractionEvent(downEvent, upEvent, onDown, onPress, onRelease, additionalCheck = (e) => true) {
  let waiting = null;
  let pressing = false;
  document.documentElement.addEventListener(downEvent, e => {
    if (pressing) return;
    if (!additionalCheck(e)) return;
    onDown && onDown();
    waiting = setTimeout(() => {
      onPress && onPress();
      pressing = true;
      waiting = null;
    }, 200)
  });
  document.documentElement.addEventListener(upEvent, e => {
    if (!additionalCheck(e)) return;
    if (pressing) {
      onRelease && onRelease();
      pressing = false;
    }
    if (waiting) {
      clearTimeout(waiting);
      waiting = null;
    }
  })
}

function initMouseEvent(onDown, onPress, onRelease) {
  document.documentElement.addEventListener("mousemove", e => setMousePos(e.clientX, e.clientY));

  initInteractionEvent('mousedown', 'mouseup', onDown, onPress, onRelease);
}

onMounted(() => {
  let userHit = () => {
    if (play.value) checkNote(performance.now());
    else startGame();
  };
  setMousePos(document.documentElement.clientWidth / 2, document.documentElement.clientHeight / 2);
  initMouseEvent(userHit, null, addClickPrompt);
  initInteractionEvent('keydown', 'keyup', userHit, null, addClickPrompt, e => e.code === "Space");
})
</script>

<template>
  <div class="mouseEffect">
    <Transition name="startBtn">
      <div v-if="end || !play" class="clickPrompt startBtn">
        <span v-if="!play">开始</span>
        <span style="cursor: none" v-if="end">{{(score.sum/score.cnt).toFixed(2)}}%</span>
      </div>
    </Transition>
    <div v-if="!play" class="clickPrompt startBtnAnimation"/>
    <div class="clickPrompt clickAnimation" v-for="item in clickPrompt" :class="item.status" :key="item.id"/>
    <div class="notePrompt noteAnimation" :style="{animationDuration: `${NOTE_ANIMATION_DUR}ms`}"
         v-for="item in notePrompt" :class="item.status" :key="item.id"/>
    <!--    <div class="notePrompt noteAnimation"/>-->
    <audio id="audio" src="/src/assets/Cheetah%20Mobile%20Games%20-%20The%20Piano.mp3" />
  </div>
</template>

<style scoped>
.mouseEffect {
  cursor: none;
  mix-blend-mode: difference;
  width: 100px;
  height: 100px;
  border-radius: 50%;
  border: 4px solid white;
  position: absolute;
  left: v-bind(mousePos.x);
  top: v-bind(mousePos.y);
  display: flex;
  justify-content: center;
  align-items: center;
  transform: translate(-50%, -50%);
}

.clickPrompt {
  width: 100%;
  height: 100%;
  background: white;
  border-radius: 50%;
  position: absolute;
}

.startBtn {
  background: none;
  height: 0;
  width: 0;
  border: 50px solid white;
  user-select: none;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  font-size: 24px;
  font-weight: lighter;
  transition-duration: 500ms;
  transition-timing-function: linear;
  position: relative;
}

.startBtn span {
  position: absolute;
  display: block;
  width: 100px;
  text-align: center;
  color: #222;
}

.startBtn-enter-from,
.startBtn-leave-to {
  height: 100%;
  width: 100%;
  border: 0 solid white;
}

.startBtnAnimation {
  width: 100px;
  height: 100px;
  pointer-events: none;
  background: #ffffff88;
  animation: 2s click infinite;
  transition-duration: 100ms;
}

.clickAnimation {
  animation: 300ms click forwards;
}

.notePrompt {
  width: 100%;
  height: 100%;
  border: 3px solid #ffffffcc;
  /*background: radial-gradient(transparent 30%, #ffffff33 100%);*/
  border-radius: 50%;
  position: absolute;
}

.noteAnimation {
  animation: cubic-bezier(0.210, 0.305, 0.950, 1.000) note forwards;
}

.miss {
  background: rgb(246, 90, 90);
}

.bad {
  background: #40DC84BD;
}

.ok {
  background: #70AAFFFF;
}

.good {
  background: #ff8e4b;
}

.perfect {
  background: rgb(255, 195, 69);
}

@keyframes click {
  from {
    transform: scale(0);
    opacity: 1;
  }
  20% {
    opacity: 1;
  }
  80% {
    transform: scale(2);
  }
  to {
    opacity: 0;
    transform: scale(2);
  }
}

@keyframes note {
  from {
    width: 1600%;
    height: 1600%;
    /*transform: scale(16);*/
    opacity: 0;
  }
  50% {
    opacity: 1;
  }
}
</style>
