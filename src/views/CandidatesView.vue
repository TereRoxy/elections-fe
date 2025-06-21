<template>
  <div class="candidates-container">
    <!-- Pie Chart -->
    <div class="chart-container">
      <h2>Party Distribution</h2>
      <Pie
        :data="chartData"
        :options="chartOptions"
        class="pie-chart"
      />
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
        <div class="action-buttons">
          <button class="edit-btn" @click="startEditing">Edit</button>
          <button class="remove-btn" @click="removeCandidate">Remove</button>
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
import { faker } from '@faker-js/faker'
import { Pie } from 'vue-chartjs'
import { Chart as ChartJS, ArcElement, Tooltip, Legend } from 'chart.js'

ChartJS.register(ArcElement, Tooltip, Legend)

export default {
  name: 'CandidatesView',
  components: { Pie },
  data() {
    return {
      candidates: [
        {
          id: 1,
          name: 'Ilie Bolojan',
          description: 'Ilie-Gavril Bolojan is a Romanian politician serving as the president of the senate of Romania',
          imageUrl: 'https://duckduckgo.com/i/05d55141240b08df.jpg',
          party: 'PNL'
        },
        {
          id: 2,
          name: 'George Simion',
          description: 'George-Nicolae Simion is a Romanian politician and civic activist. He is the founder and chairman of the Alliance for the Union of Romanians, the second largest party in both houses of parliament since 2024',
          imageUrl: 'https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Ftse4.mm.bing.net%2Fth%3Fid%3DOIP.4oNTdHDnVOYorRvXrsRX3AHaHa%26pid%3DApi&f=1&ipt=63e0f897e1fc0eea17335beb56a7c7d40b38f3efe236ae8e4f71abc9a6e88188&ipo=images',
          party: 'AUR'
        },
        {
          id: 3,
          name: 'Nicusor Dan',
          description: 'Nicușor Daniel Dan is a Romanian politician, mathematician, and civic activist serving as the sixth president of Romania since 2025',
          imageUrl: 'https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Ftse1.mm.bing.net%2Fth%3Fid%3DOIP.UzscfnrmdAZkVsR3qe8rywHaHa%26pid%3DApi&f=1&ipt=e96675aea732943c322fac2e98ded32367c96366e528116b18da695e51566394&ipo=images',
          party: 'USR'
        }
      ],
      selectedCandidate: null,
      showAddPopup: false,
      isEditing: false,
      generating: false,
      generationInterval: null,
      newCandidate: {
        name: '',
        party: '',
        description: '',
        imageUrl: ''
      },
      editCandidate: {
        id: null,
        name: '',
        party: '',
        description: '',
        imageUrl: ''
      },
      chartOptions: {
        responsive: true,
        maintainAspectRatio: false,
        plugins: {
          legend: {
            position: 'bottom',
            labels: {
              color: '#ffffff'
            }
          }
        }
      }
    }
  },
  computed: {
    chartData() {
      // Calculate party distribution
      const partyCounts = this.candidates.reduce((acc, candidate) => {
        acc[candidate.party] = (acc[candidate.party] || 0) + 1
        return acc
      }, {})

      const labels = Object.keys(partyCounts)
      const data = Object.values(partyCounts)
      const backgroundColors = labels.map((_, index) => {
        const colors = ['#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0', '#9966FF']
        return colors[index % colors.length]
      })

      return {
        labels,
        datasets: [{
          data,
          backgroundColor: backgroundColors,
          borderColor: '#1e1e1e',
          borderWidth: 1
        }]
      }
    }
  },
  methods: {
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
    },
    addCandidate() {
      const newId = this.candidates.length + 1
      this.candidates.push({
        id: newId,
        ...this.newCandidate
      })
      this.newCandidate = { name: '', party: '', description: '', imageUrl: '' }
      this.closeAddPopup()
    },
    startEditing() {
      this.isEditing = true
      this.editCandidate = { ...this.selectedCandidate }
    },
    saveEdit() {
      const index = this.candidates.findIndex(c => c.id === this.editCandidate.id)
      if (index !== -1) {
        this.candidates[index] = { ...this.editCandidate }
      }
      this.closePopup()
    },
    removeCandidate() {
      this.candidates = this.candidates.filter(c => c.id !== this.selectedCandidate.id)
      this.closePopup()
    },
    async generateCandidates() {
      while (this.generating) {
        const newId = this.candidates.length + 1
        this.candidates.push({
          id: newId,
          name: faker.person.fullName(),
          party: faker.helpers.arrayElement(['PNL', 'AUR', 'USR', 'Independent', 'Green Party']),
          description: faker.lorem.sentence({ min: 10, max: 20 }),
          imageUrl: 'https://via.placeholder.com/150'
        })
        await new Promise(resolve => setTimeout(resolve, 3000))
      }
    },
    startStopGeneration() {
      this.generating = !this.generating
      if (this.generating) {
        this.generateCandidates()
      }
    }
  },
  beforeUnmount() {
    this.generating = false
  }
}
</script>

<style scoped>

</style>
