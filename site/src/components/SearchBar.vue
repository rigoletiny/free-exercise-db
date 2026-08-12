<script>
import exercises from '../../../dist/exercises.json'
import ExerciseInstructions from './ExerciseInstructions.vue'
import PhotoGallery from './PhotoGallery.vue'

// import bookmark icon from heroicons
import { BookmarkIcon as BookmarkIconOutline } from '@heroicons/vue/24/outline'
import { BookmarkIcon as BookmarkIconSolid } from '@heroicons/vue/24/solid'

const FIELD_CANDIDATES = {
  region: ['bodyRegion', 'bodyRegions', 'regions', 'BodyRegion', 'BodyRegions'],
  goal: ['goal', 'goals', 'therapeuticGoal', 'therapeuticGoals', 'TherapeuticGoal'],
  specialty: ['specialty', 'specialties', 'Specialty'],
  equipment: ['equipment', 'equipments', 'Equipment'],
  startPosition: ['startPosition', 'startPositions', 'StartPosition']
}

function normalizeCode(value) {
  if (value === null || value === undefined) {
    return ''
  }

  return String(value).trim().toLowerCase()
}

function toArray(value) {
  if (Array.isArray(value)) {
    return value
  }

  if (value === null || value === undefined || value === '') {
    return []
  }

  return [value]
}

function extractCode(value) {
  if (value === null || value === undefined) {
    return ''
  }

  if (typeof value === 'string' || typeof value === 'number') {
    return normalizeCode(value)
  }

  if (typeof value === 'object') {
    return normalizeCode(value.code ?? value.id ?? value.value ?? value.name)
  }

  return ''
}

function getCodesByCandidates(exercise, candidates) {
  const codes = []

  candidates.forEach((field) => {
    toArray(exercise[field]).forEach((entry) => {
      const code = extractCode(entry)
      if (code) {
        codes.push(code)
      }
    })
  })

  return [...new Set(codes)]
}

function getDifficultyValue(exercise) {
  if (typeof exercise.difficulty === 'number') {
    return exercise.difficulty
  }

  if (typeof exercise.difficulty === 'string') {
    const asNumber = Number(exercise.difficulty)
    if (!Number.isNaN(asNumber)) {
      return asNumber
    }
  }

  const levelToDifficulty = {
    beginner: 1,
    intermediate: 3,
    expert: 5
  }

  const mapped = levelToDifficulty[normalizeCode(exercise.level)]
  return mapped ?? null
}

function resolveLocaleMatch(translations, locale) {
  if (!Array.isArray(translations) || translations.length === 0) {
    return null
  }

  if (!locale) {
    return translations[0]
  }

  const normalizedLocale = locale.toLowerCase()

  return (
    translations.find((item) => normalizeCode(item.locale) === normalizedLocale) ||
    translations.find((item) => {
      const itemLocale = normalizeCode(item.locale)
      return itemLocale && normalizedLocale.startsWith(itemLocale)
    }) ||
    translations[0]
  )
}

function getLocalizedNameAndAka(exercise, locale) {
  const translations =
    exercise.exerciseTranslations || exercise.translations || exercise.ExerciseTranslation || []
  const selectedTranslation = resolveLocaleMatch(toArray(translations), locale)

  const localizedName =
    selectedTranslation?.name || selectedTranslation?.title || exercise.name || exercise.id || ''

  const localizedAka = [
    ...toArray(selectedTranslation?.aka),
    ...toArray(selectedTranslation?.aliases),
    ...toArray(exercise.aka)
  ]
    .map((value) => String(value).trim())
    .filter(Boolean)

  return {
    localizedName,
    localizedAka
  }
}

function getUniqueOptionValues(exercisesList, fieldKey) {
  const values = new Set()

  exercisesList.forEach((exercise) => {
    getCodesByCandidates(exercise, FIELD_CANDIDATES[fieldKey]).forEach((code) => values.add(code))
  })

  return [...values].sort((a, b) => a.localeCompare(b))
}

export default {
  components: {
    ExerciseInstructions,
    PhotoGallery,
    BookmarkIconOutline,
    BookmarkIconSolid
  },
  data() {
    return {
      query: '',
      locale: typeof navigator !== 'undefined' ? navigator.language : 'en',
      exercises,
      searchResults: exercises,
      pageSize: 50,
      currentPage: 0,
      savedExercises: [],
      showSavedExercises: false,
      filters: {
        region: [],
        goal: '',
        specialty: '',
        equipment: '',
        difficultyMin: 1,
        difficultyMax: 5,
        startPosition: ''
      }
    }
  },
  computed: {
    // TODO: Refactor this, it's a mess
    savedItemClasses() {
      let color = this.showSavedExercises ? 'blue' : 'gray'

      let colors = {
        [`bg-${color}-700`]: true,
        [`hover:bg-${color}-800`]: true,
        [`dark:bg-${color}-800`]: true,
        [`dark:hover:bg-${color}-700`]: true,
        [`dark:focus:ring-${color}-800`]: true,
        [`focus:ring-${color}-300`]: true
      }

      return colors
    },
    paginatedItems() {
      const startIndex = this.currentPage * this.pageSize
      const endIndex = startIndex + this.pageSize
      return this.searchResults.slice(0, endIndex)
    },
    regionOptions() {
      return getUniqueOptionValues(this.exercises, 'region')
    },
    goalOptions() {
      return getUniqueOptionValues(this.exercises, 'goal')
    },
    specialtyOptions() {
      return getUniqueOptionValues(this.exercises, 'specialty')
    },
    equipmentOptions() {
      return getUniqueOptionValues(this.exercises, 'equipment')
    },
    startPositionOptions() {
      return getUniqueOptionValues(this.exercises, 'startPosition')
    }
  },
  methods: {
    totalPages() {
      return Math.ceil(this.searchResults.length / this.pageSize)
    },
    saveExercise(exercise) {
      // add the exercise if it's not already in the array otherwise remove it
      if (!this.isBookedMarked(exercise)) {
        this.savedExercises.push(exercise)
      } else {
        this.savedExercises = this.savedExercises.filter((e) => e !== exercise)

        // if we ended up with no exercises then let's clear the search and reset the results
        if (this.savedExercises.length == 0) {
          this.query = ''
          this.showSavedExercises = false
        }
      }

      this.applyFilters()
    },
    toggleSavedExercises() {
      // toggle between showing all exercises and saved exercises
      if (this.showSavedExercises) {
        this.showSavedExercises = false
      } else if (this.savedExercises.length > 0) {
        this.showSavedExercises = true
      }

      this.currentPage = 0
      this.applyFilters()
    },
    isBookedMarked(exercise) {
      return this.savedExercises.includes(exercise)
    },
    onRegionChange(event) {
      this.filters.region = [...event.target.selectedOptions].map((option) => option.value)
    },
    matchesSelection(exerciseValues, selectedValue) {
      if (!selectedValue) {
        return true
      }

      return exerciseValues.includes(normalizeCode(selectedValue))
    },
    matchesMultiSelection(exerciseValues, selectedValues) {
      if (!Array.isArray(selectedValues) || selectedValues.length === 0) {
        return true
      }

      const normalizedSelected = selectedValues.map((value) => normalizeCode(value))
      return normalizedSelected.some((value) => exerciseValues.includes(value))
    },
    matchesDifficulty(exercise) {
      const difficulty = getDifficultyValue(exercise)

      if (difficulty === null) {
        return true
      }

      return (
        difficulty >= Number(this.filters.difficultyMin) &&
        difficulty <= Number(this.filters.difficultyMax)
      )
    },
    matchesFullText(exercise) {
      if (!this.query.trim()) {
        return true
      }

      const { localizedName, localizedAka } = getLocalizedNameAndAka(exercise, this.locale)
      const haystack = [localizedName, ...localizedAka]
        .map((value) => String(value).toLowerCase())
        .join(' ')

      return haystack.includes(this.query.trim().toLowerCase())
    },
    matchesFilters(exercise) {
      const regionCodes = getCodesByCandidates(exercise, FIELD_CANDIDATES.region)
      const goalCodes = getCodesByCandidates(exercise, FIELD_CANDIDATES.goal)
      const specialtyCodes = getCodesByCandidates(exercise, FIELD_CANDIDATES.specialty)
      const equipmentCodes = getCodesByCandidates(exercise, FIELD_CANDIDATES.equipment)
      const startPositionCodes = getCodesByCandidates(exercise, FIELD_CANDIDATES.startPosition)

      return (
        this.matchesMultiSelection(regionCodes, this.filters.region) &&
        this.matchesSelection(goalCodes, this.filters.goal) &&
        this.matchesSelection(specialtyCodes, this.filters.specialty) &&
        this.matchesSelection(equipmentCodes, this.filters.equipment) &&
        this.matchesSelection(startPositionCodes, this.filters.startPosition) &&
        this.matchesDifficulty(exercise) &&
        this.matchesFullText(exercise)
      )
    },
    applyFilters() {
      const source = this.showSavedExercises ? this.savedExercises : this.exercises
      this.searchResults = source.filter((exercise) => this.matchesFilters(exercise))
    }
  },
  mounted() {
    window.addEventListener('scroll', () => {
      if (window.scrollY + window.innerHeight >= document.documentElement.scrollHeight) {
        if (this.totalPages() >= this.currentPage + 1) {
          // FIXME: Add a slight delay to the endless scroll
          // as it causes repaint issues otherwise
          setTimeout(() => {
            this.currentPage = this.currentPage + 1
          }, 400)
        }
      }
    })

    // load saved exercises from local storage
    if (localStorage.getItem('savedExercises')) {
      this.savedExercises = JSON.parse(localStorage.getItem('savedExercises'))
    }

    this.applyFilters()
  },
  watch: {
    savedExercises: {
      handler: function (val) {
        localStorage.setItem('savedExercises', JSON.stringify(val))

        if (this.showSavedExercises) {
          this.currentPage = 0
          this.applyFilters()
        }
      },
      deep: true
    },
    query() {
      this.currentPage = 0
      this.applyFilters()
    },
    filters: {
      handler() {
        this.currentPage = 0
        this.applyFilters()
      },
      deep: true
    }
  }
}
</script>
<template>
  <div class="flex gap-4 items-start">
    <div class="w-full">
      <form @submit.prevent="onSubmit">
        <label
          for="default-search"
          class="mb-2 text-sm font-medium text-gray-900 sr-only dark:text-white"
          >Search</label
        >
        <div class="relative">
          <div class="absolute inset-y-0 left-0 flex items-center pl-3 pointer-events-none">
            <svg
              aria-hidden="true"
              class="w-5 h-5 text-gray-500 dark:text-gray-400"
              fill="none"
              stroke="currentColor"
              viewBox="0 0 24 24"
              xmlns="http://www.w3.org/2000/svg"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"
              ></path>
            </svg>
          </div>
          <input
            v-model="query"
            name="search"
            type="search"
            autofocus="autofocus"
            id="search"
            class="block w-full p-4 pl-10 text-sm text-gray-900 border border-gray-300 rounded-lg bg-gray-50 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
            placeholder="Search by translation name or AKA"
          />
        </div>
      </form>
    </div>
    <div class="w-24 relative">
      <button
        type="button"
        @click.prevent="toggleSavedExercises"
        :class="savedItemClasses"
        class="text-white absolute right-2.5 bottom-2.5 focus:ring-4 focus:outline-none font-medium rounded-lg text-sm px-4 py-2"
      >
        <BookmarkIconOutline
          class="w-5 h-5 mr-2 -ml-1 text-white"
          v-if="savedExercises.length == 0"
        />
        <BookmarkIconSolid class="w-5 h-5 mr-2 -ml-1 text-white" v-if="savedExercises.length > 0" />
        <span class="sr-only">Saved</span>
        <div
          class="absolute inline-flex items-center justify-center w-6 h-6 text-xs font-bold text-white bg-red-500 border-2 border-white rounded-full -top-2 -right-2 dark:border-gray-900"
        >
          {{ savedExercises.length }}
        </div>
      </button>
    </div>
  </div>

  <div class="mt-4 grid grid-cols-1 gap-3 md:grid-cols-2 lg:grid-cols-4">
    <div>
      <label class="block mb-1 text-sm text-gray-700 dark:text-gray-200">Region</label>
      <select
        multiple
        class="block w-full p-2 text-sm text-gray-900 border border-gray-300 rounded-lg bg-gray-50 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
        @change="onRegionChange"
      >
        <option v-for="region in regionOptions" :key="region" :value="region">{{ region }}</option>
      </select>
    </div>

    <div>
      <label class="block mb-1 text-sm text-gray-700 dark:text-gray-200">Goal</label>
      <select
        v-model="filters.goal"
        class="block w-full p-2 text-sm text-gray-900 border border-gray-300 rounded-lg bg-gray-50 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
      >
        <option value="">All</option>
        <option v-for="goal in goalOptions" :key="goal" :value="goal">{{ goal }}</option>
      </select>
    </div>

    <div>
      <label class="block mb-1 text-sm text-gray-700 dark:text-gray-200">Specialty</label>
      <select
        v-model="filters.specialty"
        class="block w-full p-2 text-sm text-gray-900 border border-gray-300 rounded-lg bg-gray-50 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
      >
        <option value="">All</option>
        <option v-for="specialty in specialtyOptions" :key="specialty" :value="specialty">
          {{ specialty }}
        </option>
      </select>
    </div>

    <div>
      <label class="block mb-1 text-sm text-gray-700 dark:text-gray-200">Equipment</label>
      <select
        v-model="filters.equipment"
        class="block w-full p-2 text-sm text-gray-900 border border-gray-300 rounded-lg bg-gray-50 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
      >
        <option value="">All</option>
        <option v-for="equipment in equipmentOptions" :key="equipment" :value="equipment">
          {{ equipment }}
        </option>
      </select>
    </div>

    <div>
      <label class="block mb-1 text-sm text-gray-700 dark:text-gray-200">Start Position</label>
      <select
        v-model="filters.startPosition"
        class="block w-full p-2 text-sm text-gray-900 border border-gray-300 rounded-lg bg-gray-50 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
      >
        <option value="">All</option>
        <option v-for="startPosition in startPositionOptions" :key="startPosition" :value="startPosition">
          {{ startPosition }}
        </option>
      </select>
    </div>

    <div>
      <label class="block mb-1 text-sm text-gray-700 dark:text-gray-200">Difficulty Min</label>
      <input
        v-model.number="filters.difficultyMin"
        type="number"
        min="1"
        max="5"
        class="block w-full p-2 text-sm text-gray-900 border border-gray-300 rounded-lg bg-gray-50 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
      />
    </div>

    <div>
      <label class="block mb-1 text-sm text-gray-700 dark:text-gray-200">Difficulty Max</label>
      <input
        v-model.number="filters.difficultyMax"
        type="number"
        min="1"
        max="5"
        class="block w-full p-2 text-sm text-gray-900 border border-gray-300 rounded-lg bg-gray-50 dark:bg-gray-700 dark:border-gray-600 dark:text-white"
      />
    </div>
  </div>

  <div id="infinite-list">
    <div
      v-for="exercise in paginatedItems"
      v-bind:key="exercise.name"
      :class="savedItemClasses"
      class="exercise flex flex-col relative mt-4 items-start justify-between bg-white border border-gray-200 rounded-lg shadow md:flex-row md:max-w-xl dark:border-gray-700"
    >
      <div class="w-full md:h-auto md:w-60">
        <PhotoGallery :photos="exercise.images" />
      </div>
      <div class="w-96 p-4 leading-normal" :class="{ bookedmarked: isBookedMarked(exercise) }">
        <a
          href=""
          @click.prevent="saveExercise(exercise)"
          class="text-blue-500 hover:text-blue-600 dark:text-blue-400 dark:hover:text-blue-500"
        >
          <BookmarkIconOutline
            class="absolute drop-shadow top-4 right-4 md:top-2 md:right-2 w-8 md:w-5 text-gray-400 hover:text-gray-500 cursor-pointer"
            v-if="!isBookedMarked(exercise)"
          />
          <BookmarkIconSolid
            class="absolute drop-shadow top-4 right-4 md:top-2 md:right-2 w-8 md:w-5 text-gray-400 hover:text-gray-500 cursor-pointer"
            v-if="isBookedMarked(exercise)"
          />
        </a>
        <h5 class="mb-2 text-2xl font-bold tracking-tight text-gray-900 dark:text-white">
          {{ exercise.name }}
        </h5>
        <ExerciseInstructions :text="exercise.instructions" />
      </div>
    </div>
  </div>
</template>