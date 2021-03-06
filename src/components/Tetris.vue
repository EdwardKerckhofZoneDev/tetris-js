<template>
  <div class="tetris">
    <div class="tetris-actions">
      <button v-if="state !== 1" @click="startPause">
        {{ state === 0 ? 'Pause ' : 'Play' }} (esc)
      </button>
      <button v-else @click="restartGame">Restart</button>
      <h3>
        Score
        <p>{{ score }}</p>
      </h3>
      <h3>
        Level
        <p>{{ level }}</p>
      </h3>
      <h3>
        State
        <p>
          {{ state === 0 ? 'Playing' : state === 1 ? 'Game Over' : 'Paused' }}
        </p>
      </h3>
      <h3>
        Controls
        <p class="control">ArrowDown: Move Down</p>
        <p class="control">ArrowRight: Move Right</p>
        <p class="control">ArrowLeft: Move Left</p>
        <p class="control">ArrowUp: Rotate</p>
      </h3>
    </div>
    <div class="canvas-wrap">
      <div v-if="state === 1 || state === 2" class="state-modal">
        <h1>{{ state === 1 ? 'Game Over' : 'Paused' }}</h1>
      </div>
      <canvas id="tetris-canvas" ref="canvas"> </canvas>
    </div>
    <div class="tetris-info">
      <div>
        <h3>Up next</h3>
        <div class="info">
          <div v-for="i in 4" :key="i">
            <div
              v-for="(el, j) in 2"
              :key="j"
              :data-x="i - 1"
              :data-y="j"
              class="nextData"
            ></div>
          </div>
        </div>
      </div>
      <div class="hiscores">
        <h3>High scores</h3>
        <div v-if="hiscores.length > 0">
          <p v-for="(hiscore, i) in hiscores" :key="i">
            {{ formatDate(hiscore.date) }}: {{ hiscore.score }}
          </p>
        </div>
        <div v-else>
          <p>/</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, nextTick, onMounted, ref } from 'vue'

class coordinate {
  x: number
  y: number
  constructor(x: number, y: number) {
    this.x = x
    this.y = y
  }
}

const DIRECTION = {
  IDLE: 0,
  DOWN: 1,
  LEFT: 2,
  RIGHT: 3
}

const STATE = {
  PLAYING: 0,
  GAME_OVER: 1,
  PAUSED: 2
}

export default defineComponent({
  name: 'Tetris',

  setup() {
    const canvas = ref(null) // vue ref for canvas element
    let level = ref(1) // vue ref for level, start at 1
    let score = ref(0) // vue ref for the score, start at 0
    let state = ref(STATE.PLAYING) // vue ref for state, starting at paused
    let nextTetromino = ref() // vue ref to display the next tetromino
    let hiscores = ref([]) // vue ref for the hiscores
    let timer: any // interval for automatically moving tetromino down (starts at 1 second, decrease each level)
    let movingSpeed = 1
    let levelUp = 15

    let ctx: CanvasRenderingContext2D // canvas context 2d
    let arrayHeight = 20 // cells(21x21) in array height
    let arrayWidth = 12 // cells(21x21) in array width
    let startX = 4 // X start pos for tetromino (4 = (21*4 = 84 (pixels)))
    let startY = 0 // Y start pos for tetromino (0 = (21*0 = 0 (pixels)))

    // Used as a lookup table (12x20) where each value contains the x and y positions
    // as pixels so we can easily use them to draw on the canvas
    let coordArray = [...Array(arrayHeight)].map(() =>
      Array(arrayWidth).fill(0)
    )
    // T-shape
    // 1 = fill
    let currTetromino = [
      [1, 0], // x = 1, y = 0
      [0, 1], // x = 0, y = 1
      [1, 1],
      [2, 1]
    ]

    // Array that will hold all possible tetrominos
    let tetrominos: Array<number[][]> = []

    // Array that holds all possible colors for the tetrominos
    let tetrominosColors = [
      'purple',
      'cyan',
      'blue',
      'yellow',
      'orange',
      'green',
      'red'
    ]

    // Holds current tetromino color
    let currTetrominoColor: string

    // Makes a 2d array of the game board and fills it with zeros (12x20)
    // This keeps track of where all the current squares are
    let tetrisArray = [...Array(arrayHeight)].map(() =>
      Array(arrayWidth).fill(0)
    )

    // Makes a 2d array of the board and fills the already placed tetrominos (12x20)
    // Adds the color of the tetromino when it stops
    let stoppedShapesArray = [...Array(arrayHeight)].map(() =>
      Array(arrayWidth).fill(0)
    )

    // Holds current direction of new tetromino
    let direction: number

    // Automatically calls for a tetromino to fall down
    const setSpeed = (speed: number) => {
      timer = window.setInterval(() => {
        if (state.value === STATE.PLAYING) {
          handleTetrominoArrowDown()
        }
      }, 1000 / speed)
    }

    // setupCanvas and setInterval for when state = playing to automatically move the tetromino down
    // on component load
    onMounted(() => {
      setupCanvas()
      setupNextTetromino()
      setupHiscores()
      setSpeed(movingSpeed)
    })

    // Populates the coordArray with square coordinates [x (pixels), y (pixels)]
    // [0, 0] = [11, 9]; [1, 0] = [34, 9]
    const populateCoordArray = () => {
      let i = 0
      let j = 0
      let increment = 23
      // start at y = 9 = 9px
      // until 446 pixels (height of canvas)
      // +23 pixels each time (1 box = 21x21px)
      for (let y = 9; y <= 446; y += increment) {
        // start at x = 11 = 11px
        // 12 * 23 = 276 - 12 = 264
        // until 264 pixels (width of canvas)
        for (let x = 11; x <= 264; x += increment) {
          coordArray[i][j] = new coordinate(x, y)
          // console.log(i + ':' + j + ' = ' + coordArray[i][j].x + 'px:' + coordArray[i][j].y + 'px')
          i++
        }
        j++
        i = 0
      }
      // console.log(coordArray)
    }

    const setupCanvas = () => {
      const scale = 1
      ctx = ((canvas.value as unknown) as HTMLCanvasElement).getContext('2d')!
      ;((canvas.value as unknown) as HTMLCanvasElement).width = 296 * scale
      ;((canvas.value as unknown) as HTMLCanvasElement).height = 478 * scale

      ctx.scale(scale, scale)
      ctx.fillStyle = 'white'
      ctx.fillRect(
        0,
        0,
        ((canvas.value as unknown) as HTMLCanvasElement).width,
        ((canvas.value as unknown) as HTMLCanvasElement).height
      )
      ctx.strokeStyle = 'black'
      ctx.strokeRect(8, 8, 280, 462)

      // Handle keyboard press
      document.addEventListener('keydown', handleKeyDown)

      // Create all possible tetrominos
      createTetrominos()

      // Creates random next tetromino
      createNextTetromino()

      // Populate the lookup table
      populateCoordArray()

      drawTetromino()
    }

    const drawTetromino = () => {
      // Cycle through x and y array for the tetromino looking for all the places a square would be drawn
      // 1 = draw square
      for (let i = 0; i < currTetromino.length; i++) {
        let x = currTetromino[i][0] + startX // new tetromino starts 4 boxes from the right
        let y = currTetromino[i][1] + startY // new tetromino starts 0 boxes from the top
        tetrisArray[x][y] = 1 // put the shape in the board array
        let coordX = coordArray[x][y].x // gets x value from the lookup table (pixels)
        let coordY = coordArray[x][y].y // gets y value from the lookup table (pixels)
        ctx.fillStyle = currTetrominoColor // set current ctx color
        ctx.fillRect(coordX, coordY, 21, 21) // draw square at x and y from lookup table
      }
    }

    // Every time a key is pressed we change either the x or y value to where we want to the new tetromino to be drawn
    // Deletes the previously drawn shape, then draws the new one
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') startPause()
      switch (state.value) {
        case STATE.PLAYING:
          switch (e.key) {
            case 'ArrowLeft':
              direction = DIRECTION.LEFT
              // Check if it doesn't hit the wall or had any collisions
              if (!hitsWall() && !horizontalCollision()) {
                deleteTetromino()
                startX-- // x-- -> go left
                drawTetromino()
              }
              break
            case 'ArrowRight':
              direction = DIRECTION.RIGHT
              // Check if it doesn't hit the wall or had any collisions
              if (!hitsWall() && !horizontalCollision()) {
                deleteTetromino()
                startX++ // x++ -> go right
                drawTetromino()
              }
              break
            case 'ArrowDown':
              handleTetrominoArrowDown()
              break
            case 'ArrowUp':
              handleTetrominoRotation()
              break
          }
          break
      }
    }

    const handleTetrominoArrowDown = () => {
      direction = DIRECTION.DOWN
      // Check for any vertical collisions (bottom or another tetromino)
      if (!verticalCollision()) {
        deleteTetromino()
        startY++ // y++ -> go down
        drawTetromino()
      }
    }

    // Clears previous tetromino
    const deleteTetromino = () => {
      for (let i = 0; i < currTetromino.length; i++) {
        let x = currTetromino[i][0] + startX // tetromino to remove x
        let y = currTetromino[i][1] + startY // tetromino to remove y
        tetrisArray[x][y] = 0 // delete it from board array, 0 = empty
        let coordX = coordArray[x][y].x // gets x value from the lookup table
        let coordY = coordArray[x][y].y // gets y value from the lookup table
        ctx.fillStyle = 'white' // sets ctx color to white
        ctx.fillRect(coordX, coordY, 21, 21) // draw white squares where previous tetromino was
      }
    }

    // Creates all possible tetromino shapes
    const createTetrominos = () => {
      // I
      tetrominos.push([
        [0, 0],
        [1, 0],
        [2, 0],
        [3, 0]
      ])
      // J
      tetrominos.push([
        [0, 0],
        [0, 1],
        [1, 1],
        [2, 1]
      ])
      // T
      tetrominos.push([
        [1, 0],
        [0, 1],
        [1, 1],
        [2, 1]
      ])
      // L
      tetrominos.push([
        [2, 0],
        [0, 1],
        [1, 1],
        [2, 1]
      ])
      // S
      tetrominos.push([
        [1, 0],
        [2, 0],
        [0, 1],
        [1, 1]
      ])
      // Z
      tetrominos.push([
        [0, 0],
        [1, 0],
        [1, 1],
        [2, 1]
      ])
      // O or square
      tetrominos.push([
        [0, 0],
        [1, 0],
        [0, 1],
        [1, 1]
      ])
    }

    // Sets a random tetromino and its color to the currentTetromino
    const setCurrentTetromino = (tetromino: number[][]) => {
      currTetromino = tetromino
    }

    // Sets a random tetromino and its color to the nextTetromino
    const createNextTetromino = () => {
      let randomTetroIndex = Math.floor(Math.random() * tetrominos.length) // get a random index
      nextTetromino.value = tetrominos[randomTetroIndex] // set currect tetromino to a random one with the created index
      currTetrominoColor = tetrominosColors[randomTetroIndex] // set new random tetromino color
      setupNextTetromino()
    }

    const setupNextTetromino = () => {
      nextTick(() => {
        document.querySelectorAll('.nextData').forEach((div) => {
          ;(div as HTMLDivElement).style.background = 'transparent'

          let coords = [
            (div as HTMLDivElement).dataset.x,
            (div as HTMLDivElement).dataset.y
          ]

          nextTetromino.value.forEach((square: any) => {
            if (
              Number(square[0]) === Number(coords[0]) &&
              Number(square[1]) === Number(coords[1])
            ) {
              ;(div as HTMLDivElement).style.background = 'white'
            }
          })
        })
      })
    }

    // Check if tetromino hits the wall
    // Cycle through the squares at upper left hand corner to see if((x <= 0) || (x >= 11) && moving left or right) -> stop movement
    const hitsWall = () => {
      for (let i = 0; i < currTetromino.length; i++) {
        let x = currTetromino[i][0] + startX
        // console.log(x)
        if (x <= 0 && direction === DIRECTION.LEFT) return true
        else if (x >= 11 && direction === DIRECTION.RIGHT) return true
      }
      return false
    }

    // Check for vertical collision
    const verticalCollision = () => {
      // Copy currTetromino to try and move this "fake" tetromino
      // and check for any collisions before actually moving the currTetromino
      let tetromino = currTetromino
      let collision = false

      // Cycle through the squares
      for (let i = 0; i < tetromino.length; i++) {
        // Get the square and move it into position using the upper left had coords
        // to check for collisions
        let square = tetromino[i]

        let x = square[0] + startX
        let y = square[1] + startY

        if (direction === DIRECTION.DOWN) y++ // moving down

        // Check to check if it's going to hit a previousle set pieve
        // If there is a string in the next down square it means it has a color so there is already a square in place
        if (typeof stoppedShapesArray[x][y + 1] === 'string') {
          // If so, delete the tetromino
          deleteTetromino()
          // Increment down to put into correct place and draw
          startY++
          drawTetromino()
          collision = true
          break
        }
        // Bottom
        if (y >= 20) {
          collision = true
          break
        }
      }
      if (collision) {
        // Check for game over
        // If startY of new tetromino <= 1 -> game over
        if (startY <= 1) {
          state.value = STATE.GAME_OVER

          // Add score to localstorage if score > 0
          if (score.value > 0) {
            // Get previously stored scores
            let scores = []
            // Object to store
            const obj = {
              score: score.value,
              date: new Date()
            }
            // If scores are already stored, get those first
            if (localStorage.getItem('tetris-score')) {
              scores = JSON.parse(localStorage.getItem('tetris-score')!)
            }
            // Push new score object
            scores.push(obj)
            localStorage.setItem('tetris-score', JSON.stringify(scores))
          }
        } else {
          // No collisions so add stopped tetromino to stoppedShapesArray
          for (let i = 0; i < tetromino.length; i++) {
            let square = tetromino[i]
            let x = square[0] + startX
            let y = square[1] + startY
            // Add the current color for later checks
            stoppedShapesArray[x][y] = currTetrominoColor
          }

          // Check for any completed rows
          checkCompletedRows()

          // Creates next new tetromino
          setCurrentTetromino(nextTetromino.value)
          createNextTetromino()

          // Reset its values
          direction = DIRECTION.IDLE
          startX = 4
          startY = 0

          // Draw the new random tetromino
          drawTetromino()
        }
      }
      return collision
    }

    // Check for horizontal shape collision
    const horizontalCollision = () => {
      // Copy currTetromino to try and move this "fake" tetromino
      // and check for any collisions with a stopped tetromino
      // before actually moving the currTetromino
      let tetrominoFake = currTetromino
      let collision = false

      // Cycle through the squares
      for (let i = 0; i < tetrominoFake.length; i++) {
        // Get the square and move it into position using the upper left had coords
        let square = tetrominoFake[i]
        let x = square[0] + startX
        let y = square[1] + startY

        // Move the clone into position base on direction
        if (direction === DIRECTION.LEFT) {
          x--
        } else if (direction === DIRECTION.RIGHT) {
          x++
        }

        // Get the potential stopped square
        let stoppedShape = stoppedShapesArray[x][y]
        // If there is a string in the next down square it means it has a color so there is already a square in place
        if (typeof stoppedShape === 'string') {
          collision = true
          break
        }
      }
      return collision
    }

    // Check for any completed rows
    // Removes them and slides down
    const checkCompletedRows = () => {
      let rowsToDelete = 0 // tracks how many rows to delete
      let startOfDeletion = 0 // where to start deleting

      // Check every row to see if it has been completed or not
      for (let y = 0; y < arrayHeight; y++) {
        let completed = true
        // Cycle through x's of the row
        for (let x = 0; x < arrayWidth; x++) {
          // Get vaues stored in the stoppedShapesArray
          let square = stoppedShapesArray[x][y]
          // square = 0 (nothing there) or it's of type undefined (so not string) -> not completed
          if (square === 0 || typeof square === 'undefined') {
            completed = false
            break
          }
        }

        // A row has been completed
        if (completed) {
          // Set start of deletion to the completed row
          if (startOfDeletion === 0) startOfDeletion = y
          rowsToDelete++

          // Delete it from everywhere
          for (let i = 0; i < arrayWidth; i++) {
            // set values in stopped and current board array (tetrisArray) to 0, meaning nothing is there
            stoppedShapesArray[i][y] = 0
            tetrisArray[i][y] = 0
            let coordX = coordArray[i][y].x // get x from lookup table
            let coordY = coordArray[i][y].y // get y from lookup table
            ctx.fillStyle = 'white' // ctx fill to white
            ctx.fillRect(coordX, coordY, 21, 21) // color current tetromino white to remove it
          }
        }
      }

      // If a row needs to be deleted add 10 to score
      if (rowsToDelete > 0) {
        score.value += 10 * level.value // increment score
        moveAllRowsDown(rowsToDelete, startOfDeletion) // move all rows down

        if (score.value % levelUp === 0) {
          movingSpeed++
          level.value++
          clearInterval(timer)
          setSpeed(movingSpeed)
        }
      }
    }

    // Moves rows down after deletion of a row
    const moveAllRowsDown = (rowsToDelete: number, start: number) => {
      // Move from start to 0
      for (let i = start - 1; i >= 0; i--) {
        for (let x = 0; x < arrayWidth; x++) {
          let y = i + rowsToDelete
          let square = stoppedShapesArray[x][i]
          let nextSquare = stoppedShapesArray[x][y]

          if (typeof square === 'string') {
            nextSquare = square
            tetrisArray[x][y] = 1 // put square in board array
            stoppedShapesArray[x][y] = square // draw its color in stoppedShapesArray
            let coordX = coordArray[x][y].x // get x from lookup table
            let coordY = coordArray[x][y].y // get y from lookup table
            ctx.fillStyle = nextSquare // ctx fill color of next square
            ctx.fillRect(coordX, coordY, 21, 21) // add that next square to the canvas (will be moved down)

            // set old square to 0 everywhere
            square = 0
            tetrisArray[x][i] = 0
            stoppedShapesArray[x][i] = 0
            coordX = coordArray[x][i].x
            coordY = coordArray[x][i].y
            ctx.fillStyle = 'white'
            ctx.fillRect(coordX, coordY, 21, 21) // colors old square white to remove it from canvas
          }
        }
      }
    }

    // Rotates the tetromino
    const handleTetrominoRotation = () => {
      let newRotation = []
      let tetromino = currTetromino
      let currTetrominoBak: number[][] = []

      for (let i = 0; i < tetromino.length; i++) {
        // Make a backup to handle any rotation errors to go back to
        // Cloning the array instead of creating a refference
        currTetrominoBak = [...currTetromino]

        // Find new rotation by getting the x value of the last square of the tetromino
        // Orientate other squares based on it
        let x = tetromino[i][0]
        let y = tetromino[i][1]
        let newX = getLastSquareX() - y
        let newY = x
        newRotation.push([newX, newY])
      }

      deleteTetromino()

      // Try to draw new rotation
      try {
        currTetromino = newRotation
        drawTetromino()
      } catch (error) {
        // TypeError -> try to find value that doesn't exist (drawing outside canvas)
        // Draw backup
        if (error instanceof TypeError) {
          currTetromino = currTetrominoBak
          deleteTetromino()
          drawTetromino()
        }
      }
    }

    // Gets the x value for the last square in the tetromino
    // This is used to orientate all other squares using that as a boundary
    // Simulates rotating the shape
    const getLastSquareX = (): number => {
      let lastX = 0
      for (let i = 0; i < currTetromino.length; i++) {
        let square = currTetromino[i]
        if (square[0] > lastX) lastX = square[0]
      }
      return lastX
    }

    // Plays / pauses game
    const startPause = () => {
      if (state.value !== STATE.GAME_OVER) {
        if (state.value === STATE.PAUSED) state.value = 0
        else if (state.value === STATE.PLAYING) state.value = 2
      }
    }

    // Restarts game
    const restartGame = () => {
      // Reloads all variables
      // Could do this manually but this ensures everything works correctly
      location.reload()
    }

    // Gets scores from localstorage at start of page load
    // Stores them in a ref
    const setupHiscores = () => {
      if (localStorage.getItem('tetris-score')) {
        hiscores.value = JSON.parse(localStorage.getItem('tetris-score')!).sort(
          compare
        )
      } else hiscores.value = []
    }

    const compare = (
      a: { score: number; date: Date },
      b: { score: number; date: Date }
    ) => {
      if (a.score < b.score) return 1
      if (a.score > b.score) return -1
      return 0
    }

    // Formats the date
    const formatDate = (value: string) => {
      if (value)
        return `${value.split('-')[2].split('T')[0]}/${value.split('-')[1]}`
    }

    return {
      score,
      level,
      state,
      canvas,
      startPause,
      restartGame,
      hiscores,
      formatDate
    }
  }
})
</script>

<style lang="scss" scoped>
@import '../assets/scss/tetris.scss';
</style>
