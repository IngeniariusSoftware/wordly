<template>
  <div v-if="isNotificationVisible" class="notification">
    {{ notificationMessage }}
  </div>
  <div id="main-container" class="word-grid-container">
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
      <div style="align-self: center">
        <q-btn id="refresh-button" class="generate-button" round icon="refresh" size="1.5em" @mousedown="refreshGame"
               @keydown.enter.prevent="handleInput"/>
        <q-btn class="generate-button" round icon="add" size="1.5em" @focusin="isPopupFocused = true">
          <q-popup-edit auto-save v-model="newWord">
            <q-input v-model="newWord" dense autofocus counter @keyup.enter="handleNewWord"
                     @focusout="isPopupFocused = false"/>
          </q-popup-edit>
        </q-btn>
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
      words.push(x.word.toUpperCase())
    })

    const quasar = useQuasar()
    quasar.dark.set(true)

    return {
      grid: [],
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
      hiddenWord: wordsCsv[Math.floor(Math.random() * wordsCsv.length)].word.toUpperCase(),
      isPopupFocused: false
    }
  },
  methods: {
    handleInput(event) {
      if (this.currentAttempt === this.attemptsCount || this.isPopupFocused) return

      const input = event.key

      if (input.match('^[а-яА-Я]$')) {
        this.tryAddLetter(input)
      } else if (input === 'Enter') {
        this.tryEnterWord()
      } else if (input === 'Backspace') {
        this.tryRemoveLetters(event.ctrlKey)
      }
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
      this.currentLetterPosition = 0
      this.currentAttempt++

      if (this.currentAttempt === this.attemptsCount && word !== this.hiddenWord) {
        setTimeout(() => this.showMessage('Было загадано: ' + this.hiddenWord), 1000)
      }

    },
    checkWord(word) {
      this.grid[this.currentAttempt].map((cell, i) => {
        const isEnoughLetters = this.countLetters(this.hiddenWord, cell.letter) >= this.countLetters(word, cell.letter, 0, i + 1)

        if (isEnoughLetters) {
          if (cell.letter === this.hiddenWord[i]) {
            cell.state = 'correct-letter-position'
          } else {
            cell.state = 'wrong-letter-position'
          }
        }
      })
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
      } else if (!this.tryDecrypt(this.newWord)) {
        message = 'Некорректный токен'
      }

      if (message) {
        this.showMessage(message)
      }
    },
    tryEncrypt(message) {
      const key = this.generateRandomString(this.keyLength)
      return key + CryptoJS.AES.encrypt(message, key).toString()
    },
    tryDecrypt(encrypted) {
      if (encrypted.length <= this.keyLength) return false

      const key = encrypted.slice(0, this.keyLength)
      encrypted = encrypted.slice(this.keyLength, encrypted.length)
      const decrypted = CryptoJS.AES.decrypt(encrypted, key)
      const newWord = decrypted.toString(CryptoJS.enc.Utf8).toUpperCase()

      if (newWord.match('[А-Я]{5}')) {
        this.newGame(newWord)
        return true
      }

      return false
    },
    refreshGame() {
      const newWord = this.words[Math.floor(Math.random() * this.words.length)]
      this.newGame(newWord)
      setTimeout(() => document.getElementById('main-container').focus(), 1000)
    },
    newGame(str) {
      this.resetGame()
      this.hiddenWord = str
      this.showMessage('Новая игра')
    },
    resetGame() {
      this.currentAttempt = 0
      this.currentLetterPosition = 0
      this.grid.map((arr) => arr.map((x) => x.state = 'filled'))
      const characters = 'АБВГДЕЖЗИЙКЛМНОПРСТУФХЦЧШЩЪЫЬЭЮЯ'
      let timerId = setInterval(() => this.grid.map((arr) => arr.map((x) => x.letter = characters[Math.floor(Math.random() * characters.length)])), 50)
      setTimeout(() => {
        clearInterval(timerId)
        this.grid.map((arr) => arr.map((x) => {
          x.letter = ''
          x.state = ''
        }))
      }, 1000)
    },
    showMessage(message) {
      this.quasar.notify({
        message: message,
        timeout: 2000,
        position: 'top',
      })
    },
    generateRandomString(length) {
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
  },
  mounted() {
    this.grid = []
    for (let i = 0; i < this.attemptsCount; i++) {
      this.grid.push(Array(this.wordLength))
      for (let j = 0; j < this.wordLength; j++) {
        this.grid[i][j] = {letter: '', state: ''}
      }
    }
    document.addEventListener('keydown', (event) => {
      this.handleInput(event)
    })
  },
}

</script>

<style scoped>

.word-grid-container {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100vh;
  background-color: #121212;
}

.word-grid {
  display: flex;
  flex-direction: column;
  padding-top: 50px;
  border-radius: 5px;
}

.word-row {
  display: flex;
  justify-content: center;
}

.letter-cell {
  width: 110px;
  height: 110px;
  border: 1px solid #404040;
  display: flex;
  align-items: center;
  justify-content: center;
  font: 80px bolder Roboto, -apple-system, Helvetica Neue, Helvetica, Arial, sans-serif;
  margin: 4px;
  color: #FFF;
}

.generate-button {
  width: 60px;
  align-self: center;
  color: #404040;
  border: 1px solid #404040;
  margin-top: 20px;
  margin-left: 10px;
  margin-right: 10px;
}

.filled {
  background-color: #404040;
}

.wrong-letter-position {
  transition-duration: 2s;
  background-color: #C9B458;
}

.correct-letter-position {
  transition-duration: 2s;
  background-color: #6AAA64;

}

.notification {
  position: fixed;
  top: 20px;
  right: 20px;
  padding: 10px 20px;
  background-color: #333;
  color: #FFF;
  border-radius: 4px;
}

</style>
