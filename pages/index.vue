<template>
  <div>
    <div>
      <h2>
        lasso (click and drag, or just click!) to create a group of time cells
        and then add a description
      </h2>
      <div class="landscape:hidden">
        <p>green cells have a logged entry</p>
        <p>purple cells will be updated with your message</p>
        <p>red cells will be cleared</p>
      </div>
    </div>
    <!-- <pre>{{ ui }}</pre> -->

    <div
      class="flex flex-col-reverse sm:flex-row"
      @touchmove="touchmove($event)"
    >
      <div class="flex-1" id="cells">
        <div class="portrait:hidden">
          <p>green cells have a logged entry</p>
          <p>purple cells will be updated with your message</p>
          <p>red cells will be cleared</p>
        </div>
        <div
          v-for="(time, i) in config.times"
          :key="i"
          @mousedown="mousedown($event, i)"
          @mouseover="mouseover($event, i)"
          :data-idx="i"
          class="mt-1 mb-1 ml-10 mr-10 sm:m-1 p-1 basis-full border border-black rounded-lg border-solid"
          :style="getRGB(i, getMessage(i))"
          :class="{
            highlight: ui.selection.includes(i),
            disableHighlight: mouseDown,
            danger: getMessage(i) && ui.selection.includes(i),
          }"
        >
          {{ time }} ({{ i }}) - {{ getMessage(i) }}
        </div>
      </div>
      <div class="flex-1">
        <div class="flex flex-wrap relative p-4 sm:fixed">
          <div class="basis-full mb-2.5">
            <div class="mb-0.5">
              <div>
                <button @click="clearSelections">clear selections</button>
                clear cells that are purple and red
              </div>
              <div>
                <button @click="clearCache">clear cache</button> clear local
                storage
              </div>
              <div>
                <button @click="reset">reset</button> reset everything to
                default values
              </div>
              <div>
                <button @click="undoLast">undo</button> undo the last change
              </div>
              <div>current selection (debug): {{ ui.selection }}</div>
              <select
                class="w-full"
                v-model="ui.dropdown"
                @change="changeMessage($event.target.value)"
                ref="messageDropdown"
              >
                <option></option>
                <option
                  v-for="(item, i) in Object.keys(summary)"
                  :value="item"
                  :key="i"
                >
                  {{ item }}
                </option>
              </select>
            </div>

            <div>
              <textarea
                placeholder="your message goes here"
                rows="5"
                class="w-full"
                v-model="ui.message"
              >
              </textarea>
            </div>
            <div><button @click="save">save</button></div>
          </div>

          <div class="basis-1/4 font-bold">
            {{ totalTime.toFixed(2) }} hours
          </div>
          <div class="basis-3/4 font-bold">Total Time</div>

          <template v-for="item in Object.keys(summary)">
            <div class="basis-1/4">
              {{ summary[item].sum.toFixed(2) }} hours
            </div>
            <div class="basis-3/4">
              <button
                style="height: 15px; font-size: 8px; vertical-align: middle"
                @click="copyText(item)"
              >
                copy
              </button>
              {{ item }}
            </div>
          </template>
        </div>
      </div>
    </div>
    <div id="overlay" ref="overlay" :style="overlayStyle"></div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import { times } from './times'
type MessageRecord = { idx: number; message: string }

type UIRecord = {
  message: string
  dropdown: string
  selection: number[]
  messages: MessageRecord[]
}

type CoordsRecord = {
  clientX: number
  clientY: number
  mouseDownX: number
  MouseDownY: number
}

type CoordsStyleRecord = {
  borderColor: string
  top: string
  left: string
  width: string
  height: string
}

type OverlayRecord = {
  mouseDownX: number
  mouseDownY: number
  width: number
  height: number
  coords: CoordsRecord
  style: CoordsStyleRecord
}

type ConfigRecord = {
  times: string[]
}

type DataRecord = {
  mouseDown: boolean
  touchDown: boolean
  lastTouchMove: number
  config: ConfigRecord
  undo: string[]
  overlay: OverlayRecord
  ui: UIRecord
}

export default Vue.extend({
  name: 'IndexPage',

  data(): DataRecord {
    return {
      mouseDown: false,
      touchDown: false,
      lastTouchMove: -1,
      config: {
        times: times,
      },
      undo: [],
      overlay: {
        mouseDownX: 0,
        mouseDownY: 0,
        width: 0,
        height: 0,
        coords: {
          clientX: 0,
          clientY: 0,
          mouseDownX: 0,
          MouseDownY: 0,
        },
        style: {
          borderColor: '',
          top: '0',
          left: '0',
          width: '0',
          height: '0',
        },
      },
      ui: {
        message: '',
        dropdown: '',
        selection: [],
        messages: [],
      },
    }
  },

  computed: {
    overlayStyle(): { borderColor: string; width: string; height: string } {
      return {
        borderColor: this.overlay.style.borderColor,
        width: `${this.overlay.style.width}px`,
        height: `${this.overlay.style.height}px`,
      }
    },
    // perform a group by on the messages that are logged for the day
    // key is the message, value is the amount of time logged for these entries
    summary(): { [key: string]: { sum: number; green: number } } {
      return this.buildSummary()
    },
    // return the total amount of time logged for the day
    totalTime(): number {
      return this.ui.messages.length * 0.25
    },
  },

  watch: {
    // watch for changes to `ui` and write to localStorage
    ui: {
      handler(newVal, oldVal) {
        window.localStorage.setItem('schedule', JSON.stringify(this.ui))
      },
      deep: true,
    },
    touchDown(newVal, oldVal) {
      const body = document.querySelector('body')
      if (body) {
        if (newVal) {
          body.style.overflow = 'hidden'
        } else {
          body.style.overflow = 'auto'
        }
      }
    },
  },

  methods: {
    changeMessage(text: string) {
      this.ui.message = text
    },
    buildSummary() {
      let green = 255
      const res = this.ui.messages
        .filter((x: MessageRecord) => x.idx !== -1)
        .reduce(
          (
            acc: { [key: string]: { sum: number; green: number } },
            curr: MessageRecord
          ) => {
            if (curr.message in acc) {
              acc[curr.message].sum += 0.25
            } else {
              acc[curr.message] = { sum: 0.25, green: green }
              green -= 20
            }
            return acc
          },
          {}
        )
      return res
    },
    getRGB(idx: number, text: string) {
      if (this.ui.selection.includes(idx)) {
        return {}
      }

      const rec = this.buildSummary()[text]
      if (rec && rec.green) {
        const rgb = `rgb(0,${rec.green},0)`
        return {
          backgroundColor: rgb,
          borderColor: rgb,
          color: 'black',
        }
      }
    },
    copyText(text: string) {
      navigator.clipboard.writeText(text)
    },
    // if it exists, get the message property based on the array position of the timestamp
    getMessage(idx: number): string {
      return this.ui.messages.find((x) => x.idx === idx)?.message ?? ''
    },
    // restore to the previous version on the stack
    undoLast() {
      if (this.undo) {
        this.ui = JSON.parse(
          this.undo.pop() ||
            "{message: '',dropdown: '',selection: [],messages: [],}"
        )
      }
    },
    // reset the settings to the default values
    reset() {
      this.undo.push(JSON.stringify(this.ui))

      this.ui = {
        message: '',
        dropdown: '',
        selection: [],
        messages: [],
      }
    },
    // save the current data to local storage
    save() {
      this.undo.push(JSON.stringify(this.ui))

      this.ui.messages = this.ui.messages
        .filter((x) => !this.ui.selection.includes(x.idx))
        .concat(
          this.ui.selection.map((x) => ({
            idx: x,
            message: this.ui.message,
          }))
        )
      this.ui.selection = []
      this.ui.message = ''
      this.ui.dropdown = ''
    },
    // clear local storage
    clearCache() {
      window.localStorage.clear()
    },
    // unhighlight anything the user has selected
    clearSelections() {
      this.undo.push(JSON.stringify(this.ui))

      this.ui.messages = this.ui.messages.filter(
        (x) => !this.ui.selection.includes(x.idx)
      )
      this.ui.selection = []
      this.ui.message = ''
    },
    legacyMouseDown(idx: number) {
      if (this.ui.selection.length === 0) {
        this.ui.selection.push(idx)
      } else if (this.ui.selection.includes(idx)) {
        this.ui.selection = this.ui.selection.filter((x) => x !== idx)
      } else {
        this.ui.selection.push(idx)
      }
    },
    legacyMouseOver(idx: number) {
      if (this.mouseDown || this.touchDown) {
        if (!this.ui.selection.includes(idx)) {
          this.ui.selection.push(idx)
        } else {
          this.ui.selection = this.ui.selection.filter((x) => x !== idx)
        }
      }
    },
    overlayMouseDown(e: MouseEvent) {
      this.overlay.mouseDownX = e.clientX
      this.overlay.mouseDownY = e.clientY
      this.overlay.width = 0
      this.overlay.height = 0
      this.overlay.style.borderColor = 'green'
      this.overlay.style.top = `${e.clientY}px`
      this.overlay.style.left = `${e.clientX}px`
      this.overlay.style.width = '0'
      this.overlay.style.height = '0'
    },
    // track when cells should be highlighted
    mousedown(e: MouseEvent, idx: number) {
      this.legacyMouseDown(idx)
      this.overlayMouseDown(e)
    },
    mouseup(e: MouseEvent, idx: number) {},
    // track when cells should be highlighted
    mouseover(e: MouseEvent, idx: number) {
      this.legacyMouseOver(idx)
    },
    touchmove(e: TouchEvent) {
      const touch = e.touches[0]
      const x = touch.clientX
      const y = touch.clientY

      const cells = Array.from(document.querySelectorAll('#cells div'))
      const cell = cells.find(
        (c) =>
          c.getBoundingClientRect().left < x &&
          c.getBoundingClientRect().right > x &&
          c.getBoundingClientRect().top < y &&
          c.getBoundingClientRect().bottom > y
      )

      if (cell) {
        const idx = parseInt(cell.getAttribute('data-idx') || '0')
        if (idx !== this.lastTouchMove) {
          this.legacyMouseOver(idx)
          this.lastTouchMove = idx
        }
      }
    },
  },
  mounted() {
    // listen for the global mousedown event
    // when the mousedown is true then we know the user could click/drag to highlight
    document.addEventListener('mousedown', () => {
      this.mouseDown = true
    })

    document.addEventListener('mouseup', () => {
      this.mouseDown = false
    })

    document.addEventListener('touchstart', () => {
      console.log('touchstart')
      this.touchDown = true
    })

    document.addEventListener('touchend', () => {
      console.log('touchend')
      this.touchDown = false
    })

    // retrieve the last state from the browser and restore into the app
    const schedule = window.localStorage.getItem('schedule')
    if (schedule) {
      this.ui = JSON.parse(schedule)
    }
  },
})
</script>

<style>
#overlay {
  border-width: 1px;
  border-style: solid;
  position: absolute;
}

.highlight {
  background-color: blueviolet;
  border-color: blueviolet;
}

/* .filled {
  background-color: rgb(0, 255, 0);
  border-color: rgb(0, 255, 0);
} */

.danger {
  background-color: red;
  border-color: red;
}

.disableHighlight {
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  -khtml-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}
</style>
