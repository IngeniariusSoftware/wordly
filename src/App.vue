<template>
  <div class="word-grid-container">
    <div class="word-grid">
      <div v-for="(row, y) in grid" :key="y" class="word-row">
        <div
            v-for="(element, x) in row"
            :key="x"
            :class="['letter-cell', element.state]"
        >
          {{ element.letter }}
        </div>
      </div>
      <div class="button-row">
        <q-btn class="button" round icon="refresh" @mousedown="refreshGame" @mouseleave="focusBody"
               :disabled="isGeneratingNewGame"/>
        <q-btn class="button" round icon="add" @focusin="isPopupFocused = true" :disabled="isGeneratingNewGame">
          <q-popup-edit auto-save v-model="newWord">
            <q-input v-model="newWord" dense autofocus counter @keyup.enter="handleNewWord"
                     @focusout="isPopupFocused = false"/>
          </q-popup-edit>
        </q-btn>
        <q-btn class="button" round @mousedown="switchDifficulty" @mouseleave="focusBody">
          {{ difficulty }}
        </q-btn>
      </div>
      <div v-for="(row, y) in keyboard.slice(0, 2)" :key="y" class="word-row">
        <q-btn
            v-for="(element, x) in row"
            :key="x"
            :class="['key', element.state]"
            @mousedown="handleInput({key: element.key})"
            @mouseleave="focusBody"
        >
          {{ element.key }}
        </q-btn>
      </div>
      <div class="word-row">
        <q-btn class="key wide-key" icon="input" @mousedown="handleInput({key: 'Enter'})" @mouseleave="focusBody"/>
        <q-btn
            v-for="(element, x) in keyboard[2]"
            :key="x"
            :class="['key', element.state]"
            @mousedown="handleInput({key: element.key})"
            @mouseleave="focusBody"
        >
          {{ element.key }}
        </q-btn>
        <q-btn class="key wide-key" icon="backspace" @mousedown="handleInput({key: 'Backspace'})" @mouseleave="focusBody"/>
      </div>
    </div>
  </div>
</template>

<script>

import {useQuasar} from 'quasar'
import wordsCsv from '@/data/words.csv'
import CryptoJS from 'crypto-js'

export default {
  data() {
    const words = []

    wordsCsv.forEach(function (x) {
      x.word = x.word.toUpperCase()
      words.push(x.word)
    })

    const quasar = useQuasar()
    quasar.dark.set(true)

    const keyboard = []
    const letters = {}
    const russianLetters = [['Й', 'Ц', 'У', 'К', 'Е', 'Н', 'Г', 'Ш', 'Щ', 'З', 'Х', 'Ъ'],
      ['Ф', 'Ы', 'В', 'А', 'П', 'Р', 'О', 'Л', 'Д', 'Ж', 'Э', 'Я'], ['Ч', 'С', 'М', 'И', 'Т', 'Ь', 'Б', 'Ю']]

    for (let i = 0; i < russianLetters.length; i++) {
      keyboard.push(Array(russianLetters[i].length))
      for (let j = 0; j < russianLetters[i].length; j++) {
        const key = {key: russianLetters[i][j], state: ''}
        letters[russianLetters[i][j]] = key
        keyboard[i][j] = key
      }
    }

    return {
      grid: [],
      keyboard,
      letters,
      wordLength: 5,
      attemptsCount: 6,
      currentAttempt: 0,
      currentLetterPosition: 0,
      keyLength: 8,
      notificationMessage: '',
      isNotificationVisible: false,
      quasar,
      usedWords: [],
      newWord: '',
      words,
      wordsFrequencies: wordsCsv,
      isPopupFocused: false,
      isGeneratingNewGame: false,
      difficulty: 1,
      maxDifficulty: 3,
      hiddenWord: '',
    }
  },
  methods: {
    handleInput(event) {
      if (this.currentAttempt === this.attemptsCount || this.isPopupFocused || this.isGeneratingNewGame) return

      const input = event.key

      if (input.match('^[а-яА-Я]$')) {
        this.tryAddLetter(input)
      } else if (input === 'Enter') {
        this.tryEnterWord()
      } else if (input === 'Backspace') {
        this.tryRemoveLetters(event.ctrlKey)
      }
    },
    focusBody() {
      document.activeElement.blur()
    },
    tryAddLetter(input) {
      if (this.currentLetterPosition >= this.wordLength) return

      const cell = this.grid[this.currentAttempt][this.currentLetterPosition]
      cell.letter = input.toUpperCase()
      cell.state = 'filled'
      this.currentLetterPosition++
    },
    tryEnterWord() {
      const word = this.grid[this.currentAttempt].map(x => x.letter).join('')

      if (word.length === 0) return

      let errorMessage = ''

      if (word.length !== this.wordLength) {
        errorMessage = 'В слове недостаточно букв'
      } else if (this.usedWords.includes(word)) {
        errorMessage = 'Вы уже использовали данное слово'
      } else if (!this.isWordExists(word)) {
        errorMessage = 'В словаре игры нет такого слова, попробуйте другое'
      }

      if (errorMessage) {
        this.showMessage(errorMessage)
        return
      }

      this.usedWords.push(word)
      this.checkWord(word)
      this.checkWin(word)
      this.currentLetterPosition = 0
      this.currentAttempt++

      if (this.currentAttempt === this.attemptsCount && word !== this.hiddenWord) {
        setTimeout(() => this.showMessage('Было загадано: ' + this.hiddenWord), 1000)
      }

    },
    checkWord(word) {
      this.grid[this.currentAttempt].map((cell, i) => {
        const wordLetterCount = this.countLetters(this.hiddenWord, cell.letter)
        const isEnoughLetters = wordLetterCount >= this.countLetters(word, cell.letter, 0, i + 1)

        if (isEnoughLetters) {
          if (cell.letter === this.hiddenWord[i]) {
            cell.state = 'correct-letter-position'
            this.letters[cell.letter].state = 'correct-letter-position'
          } else {
            cell.state = 'wrong-letter-position'
            if (this.letters[cell.letter].state !== 'correct-letter-position') {
              this.letters[cell.letter].state = 'wrong-letter-position'
            }
          }
        } else if (wordLetterCount === 0) {
          this.letters[cell.letter].state = 'absent-letter-position'
        }
      })
    },
    checkWin() {
      for (let i = 0; i < this.wordLength; i++) {
        if (this.grid[this.currentAttempt][i].state !== 'correct-letter-position') return
      }

      setTimeout(() => this.showMessage('😊🎉🌟😻'), 1000)
    },
    tryRemoveLetters(isRemovingAllLetters) {
      if (this.currentLetterPosition === 0) return

      if (isRemovingAllLetters) {
        this.currentLetterPosition = 0
        this.grid[this.currentAttempt].map(cell => {
          cell.letter = ''
          cell.state = ''
        })
      } else {
        this.currentLetterPosition--
        const cell = this.grid[this.currentAttempt][this.currentLetterPosition]
        cell.letter = ''
        cell.state = ''
      }
    },
    isWordExists(word) {
      return this.words.includes(word)
    },
    countLetters(str, letter, start = 0, end = str.length) {
      let count = 0
      for (let i = start; i < end; i++) {
        if (letter === str[i]) count++
      }

      return count
    },
    handleNewWord() {
      if (this.newWord.length === 0) return

      let message = ''
      if (this.newWord.match('^[а-яА-Я]+$')) {
        if (this.newWord.length < this.wordLength) {
          message = 'В слове недостаточно букв'
        } else if (this.newWord.length > this.wordLength) {
          message = 'В слове слишком много букв'
        } else if (!this.isWordExists(this.newWord.toUpperCase())) {
          message = 'В словаре игры нет такого слова, попробуйте другое'
        } else {
          const encrypted = this.tryEncrypt(this.newWord.toUpperCase())
          navigator.clipboard.writeText(encrypted)
          message = 'Токен слова успешно скопирован в буфер обмена'
          this.newWord = ''
        }
      } else {
        const newWord = this.tryDecrypt(this.newWord)
        if (newWord.match('[А-Я]{5}')) {
          this.newGame(newWord)
        } else {
          message = 'Некорректный токен'
        }
      }

      if (message) {
        this.showMessage(message)
      }
    },
    tryEncrypt(message) {
      const key = generateRandomString(this.keyLength)
      return key + CryptoJS.AES.encrypt(message, key).toString()
    },
    tryDecrypt(encrypted) {
      if (encrypted.length <= this.keyLength) return ''

      const key = encrypted.slice(0, this.keyLength)
      encrypted = encrypted.slice(this.keyLength, encrypted.length)
      const decrypted = CryptoJS.AES.decrypt(encrypted, key)

      return  decrypted.toString(CryptoJS.enc.Utf8).toUpperCase()
    },
    refreshGame() {
      this.newGame(this.chooseNewWord())
    },
    newGame(word) {
      this.isGeneratingNewGame = true
      this.resetGame()
      this.hiddenWord = word
      this.showMessage('Новая игра')
      this.newWord = ''
    },
    resetGame() {
      this.currentAttempt = 0
      this.currentLetterPosition = 0
      this.usedWords = []
      this.grid.map((arr) => arr.map((x) => x.state = 'filled'))
      const characters = 'АБВГДЕЖЗИЙКЛМНОПРСТУФХЦЧШЩЪЫЬЭЮЯ'
      this.keyboard.map((arr) => arr.map((x) => {
        x.state = ''
      }))
      let timerId = setInterval(() => this.grid.map((arr) => arr.map((x) => x.letter = randomChoice(characters))), 50)
      setTimeout(() => {
        clearInterval(timerId)
        this.grid.map((arr) => arr.map((x) => {
          x.letter = ''
          x.state = ''
        }))
        this.isGeneratingNewGame = false
        this.focusBody()
      }, 1000)
    },
    showMessage(message) {
      this.quasar.notify({
        message: message,
        timeout: 2000,
        position: 'top',
      })
    },
    switchDifficulty() {
      this.difficulty = 1 + (this.difficulty % this.maxDifficulty)
      this.showMessage(`Сложность следующего загаданного слова изменена на ${this.difficulty}`)
    },
    chooseNewWord() {
      const length = this.wordsFrequencies.length / this.maxDifficulty * this.difficulty
      return randomChoice(this.wordsFrequencies, length).word
    }
  },
  mounted() {
    for (let i = 0; i < this.attemptsCount; i++) {
      this.grid.push(Array(this.wordLength))
      for (let j = 0; j < this.wordLength; j++) {
        this.grid[i][j] = {letter: '', state: ''}
      }
    }
    document.addEventListener('keydown', (event) => {
      this.handleInput(event)
    })
    this.refreshGame()
  },
}

function generateRandomString(length) {
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'
  const charactersLength = characters.length
  let result = ''
  const randomValues = new Uint32Array(length)
  window.crypto.getRandomValues(randomValues)
  randomValues.forEach((value) => {
    result += characters.charAt(value % charactersLength)
  })

  return result
}

function randomChoice(choices, length = choices.length) {
  return choices[Math.floor(length * Math.random())];
}

</script>

<style scoped>

.word-grid-container {
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #121212;
  height: 100vh;
}

.word-grid {
  display: flex;
  flex-direction: column;
  padding-top: min(5vh, 5vw);
}

.word-row {
  display: flex;
  justify-content: center;
}

.button-row {
  display: flex;
  justify-content: center;
  margin-top: min(3vw, 2vh);
  margin-bottom: min(3.5vw, 2.5vh);
}

.letter-cell {
  width: min(18.5vw, 8vh);
  height: min(18.5vw, 8vh);
  border: 1px solid #404040;
  display: flex;
  align-items: center;
  justify-content: center;
  font: min(11vw, 6vh) bolder ui-sans-serif, system-ui, -apple-system,
  BlinkMacSystemFont, Segoe UI, Roboto, Helvetica Neue, Arial, Noto Sans, sans-serif,
  Apple Color Emoji, Segoe UI Emoji, Segoe UI Symbol, Noto Color Emoji;
  margin: min(0.4vw, 0.4vh);
  color: #FFF;
}

.key {
  border-radius: min(1vw, 1vh);
  display: flex;
  width: min(7.3vw, 6.9vh);
  height: min(7.3vw, 6.9vh);
  font: min(2.6vw, 2.4vh) ui-sans-serif, system-ui, -apple-system,
  BlinkMacSystemFont, Segoe UI, Roboto, Helvetica Neue, Arial, Noto Sans, sans-serif,
  Apple Color Emoji, Segoe UI Emoji, Segoe UI Symbol, Noto Color Emoji;
  padding: 1%;
  margin: min(0.4vw, 0.4vh);
  background-color: #818384;
  color: #FFF;
}

.wide-key {
  width: min(15.3vw, 14.3vh);
}

.button {
  width: min(11vw, 6vh);
  height: min(11vw, 6vh);
  font-size: min(4vw, 2vh);
  color: #404040;
  border: 1px solid #404040;
  margin-left: min(3vw, 2vh);
  margin-right: min(3vw, 2vh);
}

.filled {
  background-color: #404040;
}

.absent-letter-position {
  transition: background-color 2s;
  background-color: #404040;
}

.wrong-letter-position {
  transition: background-color 2s;
  background-color: #C9B458;
}

.correct-letter-position {
  transition: background-color 2s;
  background-color: #6AAA64;

}

</style>
