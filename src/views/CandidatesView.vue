<template>
  <div class="auth-container" v-if="!isAuthenticated">
    <!-- Login Form -->
    <div v-if="!showRegister" class="auth-form">
      <h2>Login</h2>
      <form @submit.prevent="login">
        <label>
          CNP:
          <input v-model="loginData.cnp" type="text" pattern="\d{13}" required />
        </label>
        <button type="submit">Login</button>
        <p>Don't have an account? <a @click="showRegister = true">Register</a></p>
      </form>
    </div>

    <!-- Register Form -->
    <div v-if="showRegister" class="auth-form">
      <h2>Register</h2>
      <form @submit.prevent="register">
        <label>
          CNP:
          <input v-model="registerData.cnp" type="text" pattern="\d{13}" required />
        </label>
        <button type="submit">Register</button>
        <p>Already have an account? <a @click="showRegister = false">Login</a></p>
      </form>
    </div>
  </div>

  <div v-else class="candidates-container">
    <!-- Logout Button -->
    <button class="logout-btn" @click="logout">Logout</button>

    <!-- Pie Chart -->
    <div class="chart-container">
      <h2>Vote Distribution</h2>
      <Pie :data="chartData" :options="chartOptions" class="pie-chart" />
    </div>

    <!-- Main Content -->
    <div class="master-list">
      <h2>Candidates</h2>
      <div class="candidates-grid">
        <div
          v-for="candidate in candidates"
          :key="candidate.id"
          class="candidate-item"
          @click="selectCandidate(candidate)"
        >
          <img :src="candidate.imageUrl" alt="Candidate Image" class="candidate-item-image" />
          <span class="candidate-name">{{ candidate.name }}</span>
          <span class="vote-count">Votes: {{ candidate.voteCount }}</span>
        </div>
      </div>
    </div>

    <!-- Detail Pop-up -->
    <div v-if="selectedCandidate && !isEditing" class="popup-overlay" @click="closePopup">
      <div class="popup-content" @click.stop>
        <button class="close-btn" @click="closePopup">×</button>
        <h3>{{ selectedCandidate.name }}</h3>
        <img :src="selectedCandidate.imageUrl" alt="Candidate Image" class="candidate-image" />
        <p><strong>Party:</strong> {{ selectedCandidate.party }}</p>
        <p><strong>Description:</strong> {{ selectedCandidate.description }}</p>
        <p><strong>Votes:</strong> {{ selectedCandidate.voteCount }}</p>
        <div class="action-buttons">
          <button class="edit-btn" @click="startEditing">Edit</button>
          <button class="remove-btn" @click="removeCandidate">Remove</button>
          <button class="vote-btn" @click="voteCandidate" :disabled="hasVoted">Vote</button>
        </div>
      </div>
    </div>

    <!-- Edit Candidate Pop-up -->
    <div v-if="selectedCandidate && isEditing" class="popup-overlay" @click="closePopup">
      <div class="popup-content" @click.stop>
        <button class="close-btn" @click="closePopup">×</button>
        <h3>Edit Candidate</h3>
        <form @submit.prevent="saveEdit">
          <label>
            Name:
            <input v-model="editCandidate.name" type="text" required />
          </label>
          <label>
            Party:
            <input v-model="editCandidate.party" type="text" required />
          </label>
          <label>
            Description:
            <textarea v-model="editCandidate.description" required></textarea>
          </label>
          <label>
            Image URL:
            <input v-model="editCandidate.imageUrl" type="text" required />
          </label>
          <button type="submit">Save Changes</button>
        </form>
      </div>
    </div>

    <!-- Add Candidate Pop-up -->
    <div v-if="showAddPopup" class="popup-overlay" @click="closeAddPopup">
      <div class="popup-content" @click.stop>
        <button class="close-btn" @click="closeAddPopup">×</button>
        <h3>Add New Candidate</h3>
        <form @submit.prevent="addCandidate">
          <label>
            Name:
            <input v-model="newCandidate.name" type="text" required />
          </label>
          <label>
            Party:
            <input v-model="newCandidate.party" type="text" required />
          </label>
          <label>
            Description:
            <textarea v-model="newCandidate.description" required></textarea>
          </label>
          <label>
            Image URL:
            <input v-model="newCandidate.imageUrl" type="text" required />
          </label>
          <button type="submit">Add Candidate</button>
        </form>
      </div>
    </div>

    <!-- Floating Add Button -->
    <button class="add-btn" @click="openAddPopup">+</button>

    <!-- Start/Stop Generation Button -->
    <button class="start-stop-btn" @click="startStopGeneration">
      {{ generating ? 'Stop Generating' : 'Start Generating' }}
    </button>
  </div>
</template>

<script>
import { Pie } from 'vue-chartjs'
import { Chart as ChartJS, ArcElement, Tooltip, Legend } from 'chart.js'

ChartJS.register(ArcElement, Tooltip, Legend)

const BACKEND_URL = 'https://elections-be-production.up.railway.app'

export default {
  name: 'CandidatesView',
  components: { Pie },
  data() {
    return {
      isAuthenticated: false,
      showRegister: false,
      hasVoted: false,
      token: null,
      loginData: {
        cnp: '',
      },
      registerData: {
        cnp: '',
      },
      candidates: [],
      selectedCandidate: null,
      showAddPopup: false,
      isEditing: false,
      generating: false,
      ws: null,
      newCandidate: {
        name: '',
        party: '',
        description: '',
        imageUrl: '',
      },
      editCandidate: {
        id: null,
        name: '',
        party: '',
        description: '',
        imageUrl: '',
      },
      chartOptions: {
        responsive: true,
        maintainAspectRatio: false,
        plugins: {
          legend: {
            position: 'bottom',
            labels: {
              color: '#ffffff',
            },
          },
        },
      },
    }
  },
  computed: {
    chartData() {
      const partyVotes = this.candidates.reduce((acc, candidate) => {
        acc[candidate.party] = (acc[candidate.party] || 0) + candidate.voteCount
        return acc
      }, {})

      const labels = Object.keys(partyVotes)
      const data = Object.values(partyVotes)
      const backgroundColors = labels.map((_, index) => {
        const colors = ['#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0', '#9966FF']
        return colors[index % colors.length]
      })

      return {
        labels,
        datasets: [
          {
            data,
            backgroundColor: backgroundColors,
            borderColor: '#1e1e1e',
            borderWidth: 1,
          },
        ],
      }
    },
  },
  methods: {
    async login() {
      try {
        const response = await fetch(`${BACKEND_URL}/api/login`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(this.loginData),
        })
        if (!response.ok) throw new Error('Failed to login')
        const { token, hasVoted } = await response.json()
        this.token = token
        localStorage.setItem('jwt', token)
        this.isAuthenticated = true
        this.hasVoted = hasVoted
        await this.fetchCandidates()
        this.connectWebSocket()
      } catch (error) {
        console.error('Error logging in:', error)
        alert('Login failed: ' + error.message)
      }
    },
    async register() {
      try {
        const response = await fetch(`${BACKEND_URL}/api/register`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(this.registerData),
        })
        if (!response.ok) throw new Error('Failed to register')
        const { token } = await response.json()
        this.token = token
        localStorage.setItem('jwt', token)
        this.isAuthenticated = true
        this.showRegister = false
        await this.fetchCandidates()
        this.connectWebSocket()
      } catch (error) {
        console.error('Error registering:', error)
        alert('Registration failed: ' + error.message)
      }
    },
    async logout() {
      try {
        const response = await fetch(`${BACKEND_URL}/api/logout`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${this.token}`,
          },
        })
        if (!response.ok) throw new Error('Failed to logout')
        this.isAuthenticated = false
        this.candidates = []
        this.hasVoted = false
        this.token = null
        localStorage.removeItem('jwt')
        if (this.ws) {
          this.ws.close()
        }
      } catch (error) {
        console.error('Error logging out:', error)
        alert('Logout failed: ' + error.message)
      }
    },
    async fetchCandidates() {
      try {
        const response = await fetch(`${BACKEND_URL}/api/candidates`, {
          headers: {
            Authorization: `Bearer ${this.token}`,
          },
        })
        if (!response.ok) throw new Error('Failed to fetch candidates')
        this.candidates = await response.json()
      } catch (error) {
        console.error('Error fetching candidates:', error)
        this.isAuthenticated = false
        localStorage.removeItem('jwt')
      }
    },
    async checkSession() {
      const token = localStorage.getItem('jwt')
      if (!token) {
        this.isAuthenticated = false
        return
      }
      try {
        const response = await fetch(`${BACKEND_URL}/api/check-session`, {
          headers: {
            Authorization: `Bearer ${token}`,
          },
        })
        if (!response.ok) throw new Error('Session check failed')
        const { isAuthenticated, hasVoted } = await response.json()
        this.isAuthenticated = isAuthenticated
        this.hasVoted = hasVoted
        this.token = token
        if (isAuthenticated) {
          await this.fetchCandidates()
          this.connectWebSocket()
        } else {
          localStorage.removeItem('jwt')
        }
      } catch (error) {
        console.error('Error checking session:', error)
        this.isAuthenticated = false
        localStorage.removeItem('jwt')
      }
    },
    connectWebSocket() {
      const wsUrl = BACKEND_URL.replace('https', 'wss')
      this.ws = new WebSocket(wsUrl, [], {
        headers: {
          Authorization: `Bearer ${this.token}`,
        },
      })

      this.ws.onopen = () => {
        console.log('WebSocket connected')
      }

      this.ws.onmessage = (event) => {
        const message = JSON.parse(event.data)
        if (message.type === 'candidates') {
          this.candidates = message.data
        }
      }

      this.ws.onclose = () => {
        console.log('WebSocket disconnected, reconnecting...')
        if (this.isAuthenticated) {
          setTimeout(() => this.connectWebSocket(), 1000)
        }
      }

      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error)
      }
    },
    selectCandidate(candidate) {
      this.selectedCandidate = candidate
    },
    closePopup() {
      this.selectedCandidate = null
      this.isEditing = false
      this.editCandidate = { id: null, name: '', party: '', description: '', imageUrl: '' }
    },
    openAddPopup() {
      this.showAddPopup = true
    },
    closeAddPopup() {
      this.showAddPopup = false
      this.newCandidate = { name: '', party: '', description: '', imageUrl: '' }
    },
    async addCandidate() {
      try {
        const response = await fetch(`${BACKEND_URL}/api/candidates`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${this.token}`,
          },
          body: JSON.stringify(this.newCandidate),
        })
        if (!response.ok) throw new Error('Failed to add candidate')
        this.closeAddPopup()
      } catch (error) {
        console.error('Error adding candidate:', error)
        alert('Failed to add candidate: ' + error.message)
      }
    },
    startEditing() {
      this.isEditing = true
      this.editCandidate = { ...this.selectedCandidate }
    },
    async saveEdit() {
      try {
        const response = await fetch(`${BACKEND_URL}/api/candidates/${this.editCandidate.id}`, {
          method: 'PUT',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${this.token}`,
          },
          body: JSON.stringify({
            name: this.editCandidate.name,
            party: this.editCandidate.party,
            description: this.editCandidate.description,
            imageUrl: this.editCandidate.imageUrl,
          }),
        })
        if (!response.ok) throw new Error('Failed to update candidate')
        this.closePopup()
      } catch (error) {
        console.error('Error updating candidate:', error)
        alert('Failed to update candidate: ' + error.message)
      }
    },
    async removeCandidate() {
      try {
        const response = await fetch(`${BACKEND_URL}/api/candidates/${this.selectedCandidate.id}`, {
          method: 'DELETE',
          headers: {
            Authorization: `Bearer ${this.token}`,
          },
        })
        if (!response.ok) throw new Error('Failed to delete candidate')
        this.closePopup()
      } catch (error) {
        console.error('Error deleting candidate:', error)
        alert('Failed to delete candidate: ' + error.message)
      }
    },
    async voteCandidate() {
      try {
        const response = await fetch(`${BACKEND_URL}/api/vote/${this.selectedCandidate.id}`, {
          method: 'POST',
          headers: {
            Authorization: `Bearer ${this.token}`,
          },
        })
        if (!response.ok) throw new Error('Failed to vote')
        this.hasVoted = true
        alert('Vote recorded successfully!')
      } catch (error) {
        console.error('Error voting:', error)
        alert('Voting failed: ' + error.message)
      }
    },
    async generateCandidate() {
      try {
        const response = await fetch(`${BACKEND_URL}/api/candidates/generate`, {
          method: 'POST',
          headers: {
            Authorization: `Bearer ${this.token}`,
          },
        })
        if (!response.ok) throw new Error('Failed to generate candidate')
      } catch (error) {
        console.error('Error generating candidate:', error)
        alert('Failed to generate candidate: ' + error.message)
      }
    },
    async generateCandidates() {
      while (this.generating) {
        await this.generateCandidate()
        await new Promise((resolve) => setTimeout(resolve, 3000))
      }
    },
    startStopGeneration() {
      this.generating = !this.generating
      if (this.generating) {
        this.generateCandidates()
      }
    },
  },
  mounted() {
    this.checkSession()
  },
  beforeUnmount() {
    this.generating = false
    if (this.ws) {
      this.ws.close()
    }
  },
}
</script>

<style scoped>
.auth-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background-color: #1e1e1e;
}

.auth-form {
  background-color: #2a2a2a;
  padding: 2rem;
  border-radius: 8px;
  width: 300px;
  color: #ffffff;
}

.auth-form h2 {
  text-align: center;
  margin-bottom: 1rem;
}

.auth-form label {
  display: block;
  margin-bottom: 0.5rem;
}

.auth-form input {
  width: 100%;
  padding: 0.5rem;
  margin-bottom: 1rem;
  border: none;
  border-radius: 4px;
  background-color: #3a3a3a;
  color: #ffffff;
}

.auth-form button {
  width: 100%;
  padding: 0.5rem;
  background-color: #36a2eb;
  border: none;
  border-radius: 4px;
  color: #ffffff;
  cursor: pointer;
}

.auth-form button:hover {
  background-color: #2a8cd7;
}

.auth-form p {
  text-align: center;
  margin-top: 1rem;
}

.auth-form a {
  color: #36a2eb;
  cursor: pointer;
  text-decoration: underline;
}

.logout-btn {
  position: fixed;
  top: 1rem;
  right: 1rem;
  padding: 0.5rem 1rem;
  background-color: #ff6384;
  border: none;
  border-radius: 4px;
  color: #ffffff;
  cursor: pointer;
}

.vote-btn {
  background-color: #4bc0c0;
  color: #ffffff;
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-left: 0.5rem;
}

.vote-btn:disabled {
  background-color: #666;
  cursor: not-allowed;
}

.vote-count {
  font-size: 0.8rem;
  color: #ffffff;
  margin-top: 0.5rem;
  display: block;
}
</style>
