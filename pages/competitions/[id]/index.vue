<script setup lang="ts">
import { DanceStyle } from '~/types/competition'
import { UserRole } from '~/types/user'

definePageMeta({
  layout: 'dashboard',
  middleware: ['auth']
})

const route = useRoute()
const router = useRouter()
const auth = useAuth()
const competitions = useCompetitions()

// Get competition data
const competition = computed(() => {
  const comp = competitions.getCompetitionById(route.params.id as string)
  if (!comp) {
    console.error(`Competition not found: ${route.params.id}`)
    return null
  }
  return comp
})

// Helper function to format dance style
function formatDanceStyle(style: string) {
  const danceStyles = {
    [DanceStyle.BACHATA]: 'Bachata',
    [DanceStyle.ZOUK]: 'Zouk',
    [DanceStyle.WESTCOAST_SWING]: 'West Coast Swing',
    [DanceStyle.SALSA]: 'Salsa',
    [DanceStyle.KIZOMBA]: 'Kizomba',
    [DanceStyle.OTHER]: 'Other'
  }
  return danceStyles[style as DanceStyle] || style
}

// Check if registration is still open
const isRegistrationOpen = computed(() => {
  if (!competition.value) return false
  return new Date() <= new Date(competition.value.registrationDeadline)
})

// Check if competition is full
const isCompetitionFull = computed(() => {
  if (!competition.value) return false
  return competition.value.maxDancers > 0 && 
         (competition.value.registrations?.length || 0) >= competition.value.maxDancers
})

// Get registration status text and color
const registrationStatus = computed(() => {
  if (!isRegistrationOpen.value) {
    return { text: 'Registration Closed', color: 'text-gray-500' }
  }
  if (isCompetitionFull.value) {
    return { text: 'Competition Full', color: 'text-destructive' }
  }
  return { text: 'Open for Registration', color: 'text-green-600' }
})

// Check if user is already registered
const isRegistered = computed(() => {
  return competition.value?.registrations?.some(reg => reg.userId === auth.user?.id)
})

// Load competition data
onMounted(async () => {
  try {
    if (competitions.competitions.length === 0) {
      await competitions.fetchCompetitions()
    }
  } catch (error) {
    console.error('Failed to load competitions:', error)
  }
})

function handleRegister() {
  if (!competition.value) return
  router.push(`/competitions/${route.params.id}/register`)
}
</script>

<template>
  <div class="max-w-4xl mx-auto py-8 px-4">
    <!-- Loading State -->
    <div v-if="competitions.loading" class="text-center py-8">
      <div class="animate-spin h-8 w-8 border-4 border-primary border-t-transparent rounded-full mx-auto"></div>
      <p class="mt-2 text-gray-500">Loading competition details...</p>
    </div>

    <!-- Error State -->
    <Alert v-else-if="!competition" variant="destructive">
      Competition not found or failed to load
    </Alert>

    <!-- Competition Details -->
    <div v-else class="space-y-8">
      <!-- Banner -->
      <div class="relative aspect-video rounded-lg overflow-hidden shadow-lg">
        <img
          :src="competition.banner || 'https://placehold.co/600x400/png?text=Dance+Competition'"
          :alt="competition.name"
          class="w-full h-full object-cover"
        />
      </div>

      <!-- Header -->
      <div class="space-y-4">
        <div class="flex justify-between items-start">
          <div>
            <h1 class="text-3xl font-bold">{{ competition.name }}</h1>
            <p class="text-gray-500">
              Organized by: <span class="font-medium">{{ auth.getUserById(competition.organizerId)?.name || 'Unknown' }}</span>
            </p>
            <p :class="registrationStatus.color" class="mt-2 font-medium">
              {{ registrationStatus.text }}
            </p>
          </div>
          <Button 
            v-if="!isRegistered" 
            @click="handleRegister"
            class="w-full md:w-auto"
            :disabled="!competition"
          >
            Register Now
          </Button>
          <div 
            v-else 
            class="text-green-600 font-medium"
          >
            You are registered for this competition
          </div>
        </div>

        <!-- Details Grid -->
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 mt-8">
          <div>
            <h3 class="font-semibold">Date & Time</h3>
            <p>{{ new Date(competition.date).toLocaleDateString() }}</p>
          </div>
          <div>
            <h3 class="font-semibold">Location</h3>
            <p>{{ competition.location }}</p>
          </div>
          <div>
            <h3 class="font-semibold">Dance Style</h3>
            <p>{{ formatDanceStyle(competition.danceStyle) }}</p>
          </div>
          <div>
            <h3 class="font-semibold">Entry Fee</h3>
            <p>€{{ competition.entryFee }}</p>
          </div>
          <div>
            <h3 class="font-semibold">Registration Deadline</h3>
            <p>{{ new Date(competition.registrationDeadline).toLocaleDateString() }}</p>
          </div>
          <div>
            <h3 class="font-semibold">Available Spots</h3>
            <p>{{ competition.maxDancers === 0 ? 'Unlimited' : `${(competition.registrations?.length || 0)}/${competition.maxDancers}` }}</p>
          </div>
        </div>

        <!-- Description -->
        <div class="mt-8">
          <h2 class="text-xl font-semibold mb-2">About the Competition</h2>
          <p class="text-gray-600 whitespace-pre-wrap">{{ competition.description }}</p>
        </div>

        <!-- Rules -->
        <div class="mt-8">
          <h2 class="text-xl font-semibold mb-2">Rules & Guidelines</h2>
          <p class="text-gray-600 whitespace-pre-wrap">{{ competition.rules }}</p>
        </div>

        <!-- Payment Methods -->
        <div>
          <h2 class="text-xl font-semibold mb-2">Accepted Payment Methods</h2>
          <div class="flex gap-4">
            <span v-if="competition.paymentMethods.paypal" class="text-gray-600">PayPal</span>
            <span v-if="competition.paymentMethods.stripe" class="text-gray-600">Credit/Debit Card</span>
            <span v-if="competition.paymentMethods.cash" class="text-gray-600">Cash</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template> 