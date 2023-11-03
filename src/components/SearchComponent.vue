<template>
  <div :class="{ 'dark': isDarkMode }" class="p-4 md:p-8 lg:p-12">
    <div class="flex justify-between items-center mb-4 w-full">
      <div class="flex flex-col md:flex-row flex-grow space-y-2 md:space-y-0 md:space-x-2">
        <!-- 部門選擇 -->
        <select v-model="selectedDivision" @change="filterByDivision" class="p-2 border rounded w-full md:w-auto">
          <option value="">所有部門 ({{ totalEmployees || 0 }}人)</option>
          <option v-for="division in divisions" :key="division" :value="division">
            {{ division }} ({{ divisionCounts[division] || 0 }}人)
          </option>
        </select>

        <select v-model="selectedDepartment" @change="filterByDepartment" class="p-2 border rounded w-full md:w-auto">
          <option value="">所有科別  </option>
          <option v-for="department in departments" :key="department" :value="department">
            {{ department }} ({{ departmentCounts[department] || 0 }}人)
          </option>
        </select>

        <input v-model="query" @keyup.enter="search" placeholder="輸入姓名、分機或職務" class="p-2 border rounded w-full md:flex-1">

        <div v-if="query.length" class="text-sm mt-1 fixed top-4 right-4 border shadow p-2  items-center hidden md:block">
          <button @click="navigateResult(-1)" class="mr-2">↑</button>
          "{{ query }}" 總共搜尋到 {{ results.length }} 筆資料
          <button @click="navigateResult(1)" class="ml-2">↓</button>
        </div>
      </div>
    </div>

    <!-- 無結果，不渲染表單 -->
    <div v-if="results.length === 0" class="text-center mt-4">
      沒有找到相關結果
    </div>

    <!-- 有查詢到資料時，渲染畫面 -->
    <div v-else class="mt-4">
      <table class="min-w-full bg-white border text-left border-gray-200 dark:bg-gray-800 dark:border-gray-600">
        <thead class="text-gray-700 uppercase bg-gray-50 dark:bg-gray-700 dark:text-gray-400">
          <tr>
            <th class="py-3 px-6 border-b">部門</th>
            <th class="py-3 px-6 border-b">科別</th>
            <th class="py-3 px-6 border-b">姓名</th>
            <th class="py-3 px-6 border-b">分機</th>
            <th class="py-3 px-6 border-b">職務</th>
          </tr>
        </thead>
        <tbody class="dark:text-white">
          <tr v-for="(result, index) in results" :key="result.id" :data-result-index="index" :class="{ 'bg-blue-200': index === highlightedIndex, 'bg-gray-100 dark:bg-gray-900': index % 2 === 1 && index !== highlightedIndex }">
            <td class="py-4 px-6 border-b md:border-b-0 md:w-1/5">{{ result.division }}</td>
            <td class="py-4 px-6 border-b md:border-b-0 md:w-1/5">{{ result.department }}</td>
            <td class="py-4 px-6 border-b md:border-b-0 md:w-1/5" v-html="highlightKeyword(result.name)"></td>
            <td class="py-4 px-6 border-b md:border-b-0 md:w-1/5" v-html="highlightKeyword(result.number)"></td>
            <td class="py-4 px-6 border-b md:border-b-0 md:w-1/5" v-html="highlightKeyword(result.position)"></td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>

  <button 
    v-if="showBackToTop"
    @click="backTopFn()" 
    class="fixed bottom-4 right-4 p-1 md:p-2 bg-blue-500 text-white rounded-full hover:bg-blue-600 active:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-400 focus:ring-opacity-50"
  >
    回頂部
  </button>
</template>
  
<script setup>
import { ref, onMounted, watch, computed } from 'vue';

// State
const query = ref(''); // User's search query
const selectedDepartment = ref(''); // Selected department filter
const results = ref([]); // Search results
const data = ref([]); // Raw data from API
const isDarkMode = ref(false); // Dark mode toggle
const isLoading = ref(true); // Loading state for API call
const timer = ref(null); // Timer for back to top animation
const selectedDivision = ref(''); // Selected division filter
const highlightedIndex = ref(-1); // Index of the currently highlighted search result

// Computed property to get totalEmployees 
const totalEmployees = computed(() => {
  return data.value.length;
});

// Computed property to get counts for each division
const divisionCounts = computed(() => {
  const counts = {};
  data.value.forEach(item => {
    if (!counts[item.division]) {
      counts[item.division] = 0;
    }
    counts[item.division]++;
  });
  return counts;
});

// Computed property to get counts for each department
const departmentCounts = computed(() => {
  const counts = {};
  data.value.forEach(item => {
    if (item.division === selectedDivision.value) {
      if (!counts[item.department]) {
        counts[item.department] = 0;
      }
      counts[item.department]++;
    }
  });
  return counts;
});

// Highlight the search query within the text
const highlightKeyword = (text) => {
  const regex = new RegExp(`(${query.value})`, 'gi');
  return text.replace(regex, '<span class="bg-yellow-200">$1</span>');
};

// Navigate through search results
const navigateResult = (direction) => {
  highlightedIndex.value += direction;
  if (highlightedIndex.value < 0) {
    highlightedIndex.value = results.value.length - 1;
  } else if (highlightedIndex.value >= results.value.length) {
    highlightedIndex.value = 0;
  }
};

// Computed list of unique departments based on selected division
const departments = computed(() => {
  const uniqueDepartments = new Set();
  data.value.forEach(item => {
    if (item.division === selectedDivision.value && item.department) {
      uniqueDepartments.add(item.department);
    }
  });
  return [...uniqueDepartments];
});

// Computed list of unique divisions
const divisions = computed(() => {
  const uniqueDivisions = new Set();
  data.value.forEach(item => {
    uniqueDivisions.add(item.division);
  });
  return [...uniqueDivisions];
});

// Filter results by selected division
const filterByDivision = () => {
  if (!selectedDivision.value) {
    results.value = flattenData(data.value);
    return;
  }
  results.value = data.value.filter(member => member.division === selectedDivision.value);
  selectedDepartment.value = '';
};

// Filter results by selected department
const filterByDepartment = () => {
  if (!selectedDepartment.value) {
    results.value = flattenData(data.value);
    return;
  }
  results.value = data.value.filter(member => member.department === selectedDepartment.value);
};

// Flatten data for easier filtering
const flattenData = (data) => {
  return data.map(member => ({
    ...member,
    department: member.department,
    division: member.division
  }));
};

// Main search function
const search = () => {
  highlightedIndex.value = -1; // Reset highlighted index
  let filteredData = flattenData(data.value);

  // Apply filters
  if (selectedDivision.value) {
    filteredData = filteredData.filter(member => member.division === selectedDivision.value);
  }
  if (selectedDepartment.value) {
    filteredData = filteredData.filter(member => member.department === selectedDepartment.value);
  }
  if (query.value) {
    filteredData = filteredData.filter(member =>
      member.name.includes(query.value) ||
      member.number.includes(query.value) ||
      member.position.includes(query.value)
    );
  }

  results.value = filteredData;
};

// Back to top function
const backTopFn = () => {
  cancelAnimationFrame(timer.value);
  timer.value = requestAnimationFrame(function fn() {
    let oTop = document.body.scrollTop || document.documentElement.scrollTop;
    if (oTop > 0) {
      window.scroll(0, oTop - 10000);
      timer.value = requestAnimationFrame(fn);
    } else {
      cancelAnimationFrame(timer.value);
    }
  });
};

// Fetch data on component mount
onMounted(async () => {
  try {
    const response = await fetch('/mergedData.json');
    if (response.ok) {
      data.value = await response.json();
      results.value = flattenData(data.value);
    } else {
      console.error('Failed to load 600.json');
    }
  } catch (error) {
    console.error('Error fetching 600.json:', error);
  } finally {
    isLoading.value = false;
  }
});

// Debounce function
const debounce = (func, delay) => {
  let timeout;
  return (...args) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), delay);
  };
};

// Debounced search function
const debouncedSearch = debounce(search, 300);

// Watchers to re-run search when filters or query change
watch(selectedDivision, search);
watch(selectedDepartment, search);
watch(query, debouncedSearch);

// Watch highlighted index to scroll the highlighted result into view
watch(highlightedIndex, (newIndex) => {
  if (newIndex >= 0) {
    const highlightedElement = document.querySelector(`[data-result-index="${newIndex}"]`);
    if (highlightedElement) {
      highlightedElement.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
    }
  }
});

// Show/hide back to top button based on scroll position
const showBackToTop = ref(false);
window.addEventListener('scroll', () => {
  showBackToTop.value = window.scrollY > 200;
});
</script>
