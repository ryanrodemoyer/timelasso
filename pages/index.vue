<template>
  <div>
    <div>
      <h2>
        lasso (click and drag, or just click!) to create a group of time cells
        and then add a description
      </h2>
      <p>green cells have a logged entry</p>
      <p>purple cells will be updated with your message</p>
      <p>red cells will be cleared</p>
    </div>
    <!-- <pre>{{ ui }}</pre> -->

    <div class="flex-container">
      <div class="flex-column">
        <div
          v-for="(time, i) in config.times"
          :key="i"
          @mousedown="mousedown($event, i)"
          @mouseup="mouseup($event, i)"
          @mouseover="mouseover($event, i)"
          class="cell"
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
      <div class="flex-column">
        <div
          style="display: flex; flex-wrap: wrap; position: fixed; padding: 15px"
        >
          <div style="flex-basis: 100%; margin-bottom: 10px">
            <div style="margin-bottom: 2px">
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
                style="width: 100%"
                @change="ui.message = $event.target.value"
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
                style="width: 100%"
                v-model="ui.message"
              >
              </textarea>
            </div>
            <div><button @click="save">save</button></div>
          </div>

          <div style="flex-basis: 25%; font-weight: bold">
            {{ totalTime.toFixed(2) }} hours
          </div>
          <div style="flex-basis: 75%; font-weight: bold">Total Time</div>

          <template v-for="item in Object.keys(summary)">
            <div style="flex-basis: 25%">
              {{ summary[item].sum.toFixed(2) }} hours
            </div>
            <div style="flex-basis: 75%">
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

      <div></div>
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
  },

  methods: {
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
    getMessage(idx: number) {
      return this.ui.messages.find((x) => x.idx === idx)?.message
    },
    // restore to the previous version on the stack
    undoLast() {
      this.ui = JSON.parse(this.undo.pop() || '{}')
    },
    // reset the settings to the default values
    reset() {
      this.undo.push(JSON.stringify(this.ui))

      this.ui = {
        message: '',
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
      if (this.mouseDown) {
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

.flex-container {
  display: flex;
}

.flex-column {
  flex: 1;
  padding: 10px;
}

.cell {
  margin: 5px;
  padding: 5px;
  /* background-color: green; */
  width: 100%;
  border-radius: 10px;
  border-color: black;
  border-style: solid;
  border-width: thin;
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
