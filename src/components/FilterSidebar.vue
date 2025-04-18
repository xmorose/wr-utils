<template>
  <div class="filter-sidebar p-4 bg-gray-900 text-white h-full relative">
    <h2 class="font-bold mb-2 text-lg">Filter by Sound Category</h2>
    <select
        @change="updateFolder($event)"
        class="border border-gray-700 p-2 w-full bg-gray-800 text-white rounded-md"
        :value="selectedFolder"
    >
      <option value="">All Categories</option>
      <option v-for="folder in folders" :key="folder" :value="folder">
        {{ folder }}
      </option>
    </select>

    <h2 class="font-bold mt-4 mb-2 text-lg">Search Sounds</h2>
    <div class="relative">
      <input
          type="text"
          v-model="localSearchQuery"
          @input="updateSearch"
          @keydown.down="onArrowDown"
          @keydown.up="onArrowUp"
          @keydown.enter="onEnter"
          placeholder="Search sounds..."
          class="border border-gray-700 p-2 w-full bg-gray-800 text-white rounded-md mb-2"
          ref="searchInput"
      />
      <div class="flex flex-col space-y-2 mt-2">
        <label class="flex items-center space-x-2">
          <input type="checkbox" v-model="searchOptions.includeDescriptions" @change="updateSearchOptions"
                 class="form-checkbox text-blue-500"/>
          <span class="text-sm">Include sound names in search</span>
        </label>
        <label class="flex items-center space-x-2">
          <input type="checkbox" v-model="searchOptions.groupSimilarSounds" @change="updateSearchOptions"
                 class="form-checkbox text-blue-500"/>
          <span class="text-sm">Group similar sounds</span>
        </label>
      </div>
      <div v-if="showSuggestions" class="fixed inset-0 bg-black bg-opacity-50 z-50"
           @click="showSuggestions = false"></div>
      <ul
          v-if="showSuggestions"
          class="fixed bg-gray-800 border border-gray-700 rounded-md mt-1 max-h-60 overflow-y-auto scrollbar-hide z-50"
          :style="suggestionStyle"
      >
        <li
            v-for="(suggestion, index) in suggestions"
            :key="index"
            @click="selectSuggestion(suggestion)"
            class="px-4 py-2 hover:bg-gray-700 cursor-pointer"
            :class="{ 'bg-gray-700': index === arrowCounter }"
        >
          <div class="font-semibold">{{ suggestion.displayName }}</div>
          <div v-for="(sound, soundIndex) in suggestion.sounds.slice(0, 3)" :key="soundIndex"
               class="text-xs text-gray-400">
            {{ sound }}
          </div>
          <div v-if="suggestion.sounds.length > 3" class="text-xs text-gray-500">
            ... and {{ suggestion.sounds.length - 3 }} more
          </div>
        </li>
      </ul>
    </div>
  </div>
</template>
<script>
export default {
  props: {
    folders: Array,
    availableTags: Array,
    selectedTags: Array,
    selectedFolder: String,
    searchQuery: String,
    allSounds: {
      type: Array,
      default: () => []
    },
    tagCounts: {
      type: Object,
      required: true
    },
    soundTags: {
      type: Object,
      required: true
    }
  },
  mounted() {
    this.localSearchQuery = this.searchQuery || '';
    window.addEventListener('resize', this.updateSuggestionPosition);
  },
  beforeUnmount() {
    window.removeEventListener('resize', this.updateSuggestionPosition);
  },
  data() {
    return {
      localSearchQuery: this.searchQuery || '',
      suggestions: [],
      showSuggestions: false,
      arrowCounter: -1,
      suggestionStyle: {
        top: '0px',
        left: '0px',
        width: '400px',
      },
      searchOptions: {
        includeDescriptions: false,
        groupSimilarSounds: false
      }
    };
  },
  watch: {
    searchQuery(newVal) {
      this.localSearchQuery = newVal;
    },
  },
  methods: {
    updateFolder(event) {
      this.$emit('update:selectedFolder', event.target.value);
    },
    updateSearchOptions() {
      this.$emit('update:searchOptions', this.searchOptions);
    },
    updateSearch() {
      this.$emit('update:searchQuery', this.localSearchQuery);
      this.getSuggestions();
      this.$nextTick(this.updateSuggestionPosition);
    },
    updateSuggestionPosition() {
      const inputEl = this.$refs.searchInput;
      if (inputEl) {
        const rect = inputEl.getBoundingClientRect();
        this.suggestionStyle.top = `${rect.bottom + window.scrollY}px`;
        this.suggestionStyle.left = `${rect.left + window.scrollX}px`;
        this.suggestionStyle.width = `${Math.max(300, rect.width)}px`;
      }
    },
    getSuggestions() {
      if (this.localSearchQuery.length < 2) {
        this.suggestions = [];
        this.showSuggestions = false;
        return;
      }

      const searchTerms = this.localSearchQuery.toLowerCase().split(' ').filter(term => term.length > 0);

      try {
        this.suggestions = this.allSounds
            .filter(sound => {
              if (!sound || typeof sound.displayName !== 'string' || !Array.isArray(sound.sounds)) {
                console.warn('Invalid sound object:', sound);
                return false;
              }

              const normalizedName = sound.displayName.toLowerCase();
              const normalizedSoundNames = sound.sounds.map(s =>
                  (typeof s === 'string' ? s : s.name || '').toLowerCase()
              );

              const matches = searchTerms.every(term => {
                if (this.searchOptions.groupSimilarSounds) {
                  const regex = new RegExp(`${term.replace(/\d+$/, '')}\\d*$`);
                  return regex.test(normalizedName) || normalizedSoundNames.some(name => regex.test(name));
                } else {
                  return normalizedName.includes(term) || normalizedSoundNames.some(name => name.includes(term));
                }
              });
              return matches;
            })
            .map(sound => ({
              displayName: sound.displayName,
              sounds: sound.sounds.map(s => typeof s === 'string' ? s : s.name),
              tags: this.getTagsForSound(sound)
            }))
            .slice(0, 100);

        console.log("Filtered suggestions:", this.suggestions);

        this.showSuggestions = this.suggestions.length > 0;
        this.arrowCounter = -1;
      } catch (error) {
        console.error('Error in getSuggestions:', error);
        this.suggestions = [];
      }
    },
    getTagsForSound(sound) {
      const soundTags = {};
      for (const [tag, count] of Object.entries(this.tagCounts)) {
        if (this.soundTags[tag]?.includes(sound.displayName)) {
          soundTags[tag] = count;
        }
      }
      return soundTags;
    },
    onArrowDown() {
      if (this.arrowCounter < this.suggestions.length - 1) {
        this.arrowCounter++;
      }
    },
    onArrowUp() {
      if (this.arrowCounter > 0) {
        this.arrowCounter--;
      }
    },
    onEnter() {
      if (this.arrowCounter > -1) {
        this.selectSuggestion(this.suggestions[this.arrowCounter]);
      }
      this.showSuggestions = false;
    },
    selectSuggestion(suggestion) {
      this.localSearchQuery = suggestion.displayName;
      this.$emit('update:searchQuery', suggestion.displayName);
      this.showSuggestions = false;
    }
  },
};
</script>

<style scoped>
.filter-sidebar {
  width: 350px;
  height: 300px
}
</style>