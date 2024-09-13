<template>
  <div>
    <SpeedInsights />
    <div class="container">
      <div class="wrapper">
        <div class="header">
          <h3 style="user-select: none; border: 1px solid #B6EADA;">Latihan Mengetik</h3>
          <h3 @click="logout" v-if="isLoggedIn"> Logout </h3>
          <h3 @click="login" v-else>Mau masuk leaderboard? login dulu sini</h3>
          <h3 @click="about">About?</h3>
        </div>
        <div class="content">
          <div class="prompt">
            <div class="text">
              <p v-if="this.countdown != 0" v-html="formattedPrompt"></p>
            </div>
            <div class="input-text">
              <input class="text-form" v-model="userInput" v-if="isStarted" @keyup.space="nextWord"
                placeholder="Ayo Ketikk.....">
              <button class="btn-mulai" v-if="!isStarted" @click="startTest" :disabled="isStarted">Mulai</button>
              <div class="timer animate__animated animate__bounceInRight" v-if="isStarted">{{ countdown }}</div>
            </div>
          </div>
          <div class="leaderboard">
            <p>Yaow ini adalah leaderboard || Nama kamu: {{ isLoggedIn ? userLoginData.displayName : "Login dulu wak"
            }}
            </p>
            <table>
              <thead>
                <tr>
                  <th>No</th>
                  <th>Nama</th>
                  <th>WPM (Kata permenit)</th>
                  <th>Last Update</th>
                  <th>Action</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(i, no) in leaderboardData" :key="i.id">
                  <td>{{ no + 1 }}</td>
                  <td>{{ i.name }}</td>
                  <td>{{ i.wpm }}</td>
                  <td>{{ i.date }}</td>
                  <td><button class="btn-clear" @click="removeResult(i)"
                      v-if="userLoginData.email === i.email">Remove</button>
                    <button class="btn-clear" v-else-if="!isLoggedIn" @click="login">Login dulu broh</button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>

      <!-- MODAL -->
      <div class="modal animate__animated animate__jackInTheBox" v-if="toggleModal">
        <div class="modal-header">
          <h2>STOPPP WAKTU HABIS...</h2>
        </div>
        <div class="modal-body">
          <p>Hmm kecepatan mengetik kamu adalah {{ this.howMuchWordType }} WPM (Kata permenit)</p>
          <br>
          <p>Kata yang benar: {{ this.correctWord }}</p>
          <p>Kata yang salah: {{ this.incorrectWord }}</p>
        </div>
        <small v-if="!isLoggedIn">*Login dulu kalau mau disave biar masuk leaderboard</small>
        <small v-if="apakahCurang" style="color: red;">Skor anda terlalu parah untuk di save!! ulang aja ya</small>
        <div class="modal-btn">
          <button class="btn" @click="toggleModal = false">Close</button>
          <button class="btn" @click="saveResult" v-if="isLoggedIn && !apakahCurang">Mau disave?</button>
        </div>
      </div>
      <!-- MODAL -->
    </div>

  </div>
</template>

<script>
import { SpeedInsights } from "@vercel/speed-insights/vue"
import swal from 'sweetalert'
import "animate.css"
import { addDoc, collection, getFirestore, onSnapshot, query, orderBy, doc, deleteDoc } from 'firebase/firestore'
import { db } from '../../firebase.js'
import { getAuth, GoogleAuthProvider, signInWithPopup, signOut } from 'firebase/auth';

export default {
  components: {
    SpeedInsights: SpeedInsights
  },
  data() {
    return {
      prompts: [
        ["pensil", "rumah", "lemari", "kasur", "pintu", "kemana", "siapa",
          "bukan", "karena", "kamu", "di", "dan", "mereka", "tidak", "jatuh",
          "bawah", "tahu", "oleh", "kembali", "siap", "memiliki", "dengan", "kamu",
          "anak", "harus", "ada", "baru", "lama", "tinggi", "setiap", "kembali", "mengatakan",
          "tidak", "harus", "dengan", "pergi", "lama", "bukan", "jam", "mana", "beritahu", "banyak", "tak",
          "kini", "tidak", "tentang", " sesuatu", "pada", "sini", "sekitar", "tinggi", "selalu", "terjadi",
          "seperti", "malam", "jalan", "ada", "belum", "melalui", "tiba", "jika", "sebentar", "lagi",
          "khusus", "selalu", "tetapi", "kemudian", "tak", "cara", "jumlah", "bahwa", "tahu", "naik", "sebentar",
          "baik", "bila", "melakukan", "juta", "tiba", "paling", "bisa", "kita"],
      ],
      currentPromptIndex: 0,
      currentPrompt: [],
      currentWordIndex: 0,
      howMuchWordType: 0,
      userInput: "",
      isStarted: false,
      isTimeUp: false,
      startTime: 0,
      endTime: 0,
      errorCount: 0,
      typingSpeed: 0,
      countdown: 0,
      maxPromptLines: 2,

      apakahCurang: false,
      incorrectWord: 0,
      correctWord: 0,
      toggleModal: false,
      leaderboardData: {},
      userLoginData: localStorage.getItem('isLoggedIn') ? JSON.parse(localStorage.getItem('user')) : 'broNotLogin',
      isLoggedIn: localStorage.getItem('isLoggedIn'),
    };
  },
  mounted() {
    console.log(this.currentPrompt[this.currentPromptIndex])
    console.log({ status: this.isLoggedIn })
    // console.log({userData: this.userLoginData})
    // console.log({ key: process.env.VUE_APP_API })

    // GET RESULT
    const db = getFirestore();
    const chatCollection = collection(db, 'result');

    onSnapshot(query(chatCollection, orderBy('wpm', 'desc')), (snapshot) => {
      this.leaderboardData = snapshot.docs.map((doc) => ({ ...doc.data(), id: doc.id }));
      console.log(this.leaderboardData);
    }, (error) => {
      console.error('Error getting real-time updates: ', error);
    });

    console.log(this.leaderboardData)

    // GET RESULT
  },
  computed: {
    formattedPrompt() {
      const wordsBeforeCurrent = this.currentPrompt.slice(0, this.currentWordIndex);
      const wordsAfterCurrent = this.currentPrompt.slice(this.currentWordIndex + 1);
      const currentWord = this.currentPrompt[this.currentWordIndex] ? this.currentPrompt[this.currentWordIndex] : "";

      return [
        ...wordsBeforeCurrent.map((word) => `<span>${word}</span>`),
        `<span style="color: ${wordsBeforeCurrent.length === this.currentWordIndex ? 'yellow' : 'none'}">${currentWord}</span>`,
        ...wordsAfterCurrent.map(word => `<span>${word}</span>`)
      ].join(' ');
    }
  },
  methods: {
    async login() {
      try {
        const auth = getAuth();
        const provider = new GoogleAuthProvider();
        const result = await signInWithPopup(auth, provider);
        const user = result.user;

        // Simpan data penting ke localStorage
        localStorage.setItem("user", JSON.stringify({
          displayName: user.displayName,
          email: user.email,
          uid: user.uid,
          PhotoURL: user.photoURL,
          providerData: user.providerData,
          emailVerified: user.emailVerified,
          phoneNumber: user.phoneNumber
        }));
        localStorage.setItem('isLoggedIn', true)
        location.reload()
      } catch (error) {
        console.error(error.message);
        swal({
          icon: 'error',
          title: 'Login Failed',
          button: 'waduh'
        });
      }
    },
    async logout() {
      const confirm = await swal({
        icon: 'warning',
        title: 'Yakin?',
        buttons: ['Ga deh', 'Iya'],
        dangerMode: true,
      })
      if (confirm) {
        try {
          const auth = getAuth();
          await signOut(auth)
          this.isLoggedIn = false
          localStorage.clear()
          location.reload()
        } catch (er) {
          console.log(er)
          swal({
            icon: 'error',
            text: 'Logout Failed',
          })
        }
      }
    },
    async saveResult() {
      try {
        // setTimeout(() => {
        // Menambahkan data baru ke koleksi "tasks"
        const data = {
          name: this.userLoginData.displayName,
          email: this.userLoginData.email,
          wpm: this.howMuchWordType,
          date: new Date().toDateString()
        }
        await addDoc(collection(db, 'result'), data);

        // Menambahkan data baru ke array tasks (lokal)
        // this.tasks.push({ id: docRef.id, ...this.newTask });
        console.log("berhasil")
        swal({
          icon: 'success',
          title: 'Berhasil Save',
          button: 'Close',
          closeOnClickOutside: false
        }).then(
          (close) => {
            if (close) {
              this.toggleModal = false
            }
          }
        )
        // }, 1200);
      } catch (error) {
        console.error('Error adding task:', error);
      }
    },
    async removeResult(i) {
      const warning = await swal({
        icon: 'warning',
        title: 'Yakin mau dihapus?',
        dangerMode: true,
        buttons: ['Ga', 'Iya']
      })

      try {
        if (warning) {
          console.log(`Hapus leaderboard dengan id ${i.id}`)
          const DB = getFirestore()
          const findDoc = doc(DB, 'result', i.id)
          await deleteDoc(findDoc)
          swal({
            icon: 'success',
            title: 'Berhasil dihapus',
            button: 'Oke'
          })
        }
      } catch (r) {
        console.log(r)
      }

    },
    about() {
      swal({
        icon: false,
        title: 'Sulit aseli buatnya',
        text: '--Radya--',
        button: 'Okeh'
      })
    },
    setNextPrompt() {
      this.currentPrompt = this.shuffleArray(this.prompts[this.currentPromptIndex]);
      this.currentWordIndex = 0;
    },
    shuffleArray(array) {
      let currentIndex = array.length, randomIndex;
      while (currentIndex != 0) {
        randomIndex = Math.floor(Math.random() * currentIndex);
        currentIndex--;
        [array[currentIndex], array[randomIndex]] = [array[randomIndex], array[currentIndex]];
      }
      return array;
    },
    nextWord() {
      if (this.isStarted && !this.isTimeUp) {
        this.howMuchWordType++;
        // INI DIA KATA YANG HARUS DIKETIK
        this.checkWord()
        this.currentWordIndex = (this.currentWordIndex + 1) % this.currentPrompt.length;
        this.userInput = "";
        this.scrollPrompt();
      }
    },
    checkWord() {
      console.log(`Kata yang harus diketik: ${this.currentPrompt[this.currentWordIndex]}`)
      console.log(`Kata yang sedang diketik: ${this.userInput}`)
      if (this.currentPrompt[this.currentWordIndex] === this.userInput.trim()) {
        console.log("Benar")
        this.correctWord++
      } else {
        console.log("Salah")
        this.incorrectWord++
      }
    },
    scrollPrompt() {
      const promptContainer = document.querySelector('.text');
      const currentWordElement = promptContainer.querySelector(`span:nth-child(${this.currentWordIndex + 1})`);

      if (currentWordElement) {
        const containerHeight = promptContainer.clientHeight;
        const currentWordHeight = currentWordElement.clientHeight;
        const currentWordOffsetTop = currentWordElement.offsetTop;

        const scrollTo = currentWordOffsetTop - containerHeight + currentWordHeight;

        promptContainer.scrollTo({
          top: scrollTo,
          behavior: 'smooth',
        });
      }
    },
    startTest() {
      this.isStarted = true;
      this.isTimeUp = false;
      this.startCountdown();
      this.setNextPrompt();
      this.correctWord = 0
      this.incorrectWord = 0
    },
    startCountdown() {
      this.countdown = 60;

      this.countdownInterval = setInterval(() => {
        if (this.countdown > 0) {
          this.countdown--;
        } else {
          this.endTest();
          clearInterval(this.countdownInterval);
        }
      }, 1000);
    },

    endTest() {
      if (this.incorrectWord >= 50 || this.correctWord <= 10) {
        this.apakahCurang = true
        this.isTimeUp = true;
        this.isStarted = false;
        clearInterval(this.countdownInterval);
        this.toggleModal = true
      } else {
        this.apakahCurang = false
        this.isTimeUp = true;
        this.isStarted = false;
        clearInterval(this.countdownInterval);
        this.toggleModal = true
      }
    },

  },
};
</script>

<style scoped>
.container {
  position: fixed;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
  background-color: #03001C;
  color: #B6EADA;
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal {
  position: fixed;
  width: 50%;
  height: 50vh;
  color: #03001C;
  border-radius: 20px;
  background: rgba(255, 255, 255, 0.55);
  -webkit-backdrop-filter: blur(50px);
  backdrop-filter: blur(50px);
  border: 1px solid rgba(255, 255, 255, 0.275);
}

.modal-header {
  /* border: 1px solid yellow; */
  text-align: center;
  padding-top: 10px;
  height: 10%;
  font-size: 2rem
}

.modal-body {
  /* border: 1px solid black; */
  width: 100%;
  height: 60%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  font-weight: bolder;
}

.modal-body p {
  font-size: 1.4rem
}

.modal-btn {
  /* border: 1px solid red; */
  height: 20%;
  display: flex;
  justify-content: flex-end;
  align-items: center;
}

.modal-btn .btn {
  margin-right: 30px;
  border: none;
  padding: 10px 20px;
  border-radius: 10px;
  background-color: #03001C;
  color: #B6EADA;
  font-weight: bolder;
  cursor: pointer;
}

.timer {
  width: 50px;
  height: 50px;
  background-color: #03001C;
  position: fixed;
  right: 100px;
  border-radius: 50%;
  border: 5px solid #B6EADA;
  display: flex;
  justify-content: center;
  align-items: center;
  font-weight: bolder;
}

.wrapper {
  /* border: 1px solid red; */
  width: 90%;
  height: 90vh;
}

.header {
  /* border: 1px solid black; */
  display: flex;
  height: 10%;
}

.header h3 {
  background-color: #03001C;
  padding: 10px 20px;
  border-radius: 10px 10px 0 0;
  display: flex;
  justify-content: center;
  align-items: center;
  border-bottom: 1px solid #B6EADA;
}

.wrapper h3:nth-child(n+2) {
  margin-left: 50px;
  cursor: pointer;
}

.content {
  /* border: 1px solid blue; */
  width: 100%;
  height: 90%;
}

.prompt {
  /* border: 1px solid green; */
  width: 100%;
  height: 50%;
}

.text {
  /* border: 1px solid red; */
  border-radius: 0 10px 10px 10px;
  width: 100%;
  height: 50%;
  background-color: #03001C;
  font-size: 1.8rem;
  display: flex;
  align-items: center;
  overflow-y: scroll;
  scrollbar-color: #03001C #e0e0e0;
  scrollbar-width: none;
}

.text p {
  /* border: 1px solid yellow; */
  margin-left: 15px;
  margin-top: 340px;
  line-height: 2;
}

.input-text {
  /* border: 1px solid blue; */
  border-radius: 10px;
  margin: 0 auto;
  margin-top: 20px;
  width: 70%;
  height: 30%;
  background-color: #03001C;
  display: flex;
  justify-content: center;
  align-items: center;
}

.text-form {
  border: none;
  box-sizing: border-box;
  margin-right: 25px;
  padding: 10px;
  padding-right: 200px;
  border-radius: 10px;
  font-size: 1.2rem;
}

.btn-mulai {
  border: none;
  border-radius: 10px;
  padding: 10px;
  font-size: 1.2rem;
  font-weight: bolder;
  cursor: pointer;
  background-color: #B6EADA;
  color: #03001C;
}

.btn-clear {
  border: none;
  border-radius: 10px;
  padding: 10px;
  font-size: 1rem;
  font-weight: bolder;
  cursor: pointer;
  background-color: #B6EADA;
  color: #03001C;
}

.leaderboard {
  border: 1px solid black;
  background-color: #03001C;
  width: 100%;
  height: 50%;
  overflow-y: auto;
  scrollbar-color: #03001C #e0e0e0;
  scrollbar-width: auto;
  border-radius: 10px;
}

.leaderboard p {
  text-align: center;
  margin-top: 10px;
  font-weight: bolder;
}

table {
  margin: 0 auto;
  width: 90%;
  margin-top: 10px;
}

table,
tr,
th {
  border: 2px solid #B6EADA;
  border-collapse: collapse;
  padding: 5px;
}

th {
  background-color: #B6EADA;
  color: #03001C;
}

table,
tr,
td {
  border: 2px solid #B6EADA;
  border-collapse: collapse;
  padding: 5px;
  text-align: center;
  margin-bottom: 5px;
}
</style>
