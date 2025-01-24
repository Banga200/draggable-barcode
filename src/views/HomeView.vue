<template>
  <div class="min-h-screen bg-gray-100 p-4">
    <div class="flex gap-4">
      <!-- Design Area -->
      <div class="flex flex-col gap-4 w-2/3">
        <!-- Alignment Controls -->
        <div class="bg-white p-4 rounded-lg shadow flex gap-2">
          <button 
            v-for="align in alignments" 
            :key="align.value"
            @click="alignElements(align.value)"
            class="px-4 py-2 bg-gray-100 hover:bg-gray-200 rounded"
          >
            {{ align.label }}
          </button>
        </div>

        <div 
          ref="designArea"
          class="bg-white border-2 border-dashed border-gray-300 rounded-lg p-4 min-h-[600px] relative"
          @dragover.prevent
          @drop.prevent="handleDrop"
          @click.self="selectedIndex = -1"
        >
          <div v-for="(item, index) in designElements" 
               :key="index"
               :style="{
                 position: 'absolute',
                 left: `${item.x}px`,
                 top: `${item.y}px`,
               }"
               class="cursor-move relative"
               :class="{ 'ring-2 ring-blue-500': selectedIndex === index }"
               @click.stop="selectElement(index)"
               draggable="true"
               @dragstart="handleDragStart($event, index)"
               @drag="handleDrag"
               @dragend="handleDragEnd"
          >
            <!-- Coordinates Popover -->
            <div v-if="selectedIndex === index"
                 class="absolute bottom-full left-1/2 transform -translate-x-1/2 mb-2 bg-white shadow-lg rounded-lg p-3 z-10 whitespace-nowrap">
              <div class="flex gap-4">
                <div>
                  <label class="text-sm text-gray-600">X:</label>
                  <input 
                    type="number" 
                    v-model="item.x"
                    class="w-20 px-2 py-1 border rounded"
                  >
                </div>
                <div>
                  <label class="text-sm text-gray-600">Y:</label>
                  <input 
                    type="number" 
                    v-model="item.y"
                    class="w-20 px-2 py-1 border rounded"
                  >
                </div>
              </div>
              <div class="absolute -bottom-2 left-1/2 transform -translate-x-1/2 w-3 h-3 bg-white rotate-45"></div>
            </div>

            <template v-if="item.type === 'barcode'">
              <div class="bg-white p-2 border border-gray-300 rounded">
                <svg :ref="el => setBarcodeRef(el, item.text)"></svg>
              </div>
            </template>
            <template v-else>
              <span>{{ item.text }}</span>
            </template>
          </div>
        </div>
      </div>

      <!-- Properties Panel -->
      <div class="w-1/3 bg-white rounded-lg p-4 shadow">
        <h2 class="text-xl font-bold mb-4">Properties</h2>
        <div class="space-y-2">
          <div v-for="(prop, index) in properties" 
               :key="index" 
               class="flex items-center gap-2">
            <input 
              type="checkbox" 
              v-model="prop.selected"
              class="rounded"
            >
            <span>{{ prop.name }}</span>
          </div>
        </div>

        <!-- Barcode Input -->
        <div class="mt-6">
          <h3 class="font-bold mb-2">Add Barcode</h3>
          <div class="flex gap-2">
            <input 
              v-model="barcodeValue"
              type="text"
              placeholder="Enter barcode value"
              class="border rounded px-2 py-1 flex-1"
            >
            <button 
              @click="addBarcode"
              class="px-4 py-1 bg-blue-500 text-white rounded hover:bg-blue-600"
            >
              Add
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import JsBarcode from 'jsbarcode';
import {ref, onMounted, onUnmounted, watch} from 'vue';
const properties = ref([
  { name: 'Product Name', selected: false },
  { name: 'SKU', selected: false },
  { name: 'Price', selected: false },
  { name: 'Barcode', selected: false }
]);

const alignments = [
  { label: 'Left Align', value: 'left' },
  { label: 'Center Align', value: 'center' },
  { label: 'Right Align', value: 'right' },
  { label: 'Top Align', value: 'top' },
  { label: 'Middle Align', value: 'middle' },
  { label: 'Bottom Align', value: 'bottom' }
];

const designElements = ref([]);
const selectedIndex = ref(-1);
const designArea = ref(null);
const barcodeValue = ref('');
const dragStartPosition = ref({ x: 0, y: 0 });

// Handle property selection
watch(properties, (newProps) => {
  const selectedProp = newProps.find(p => p.selected);
  if (selectedProp) {
    designElements.value.push({
      text: selectedProp.name,
      type: 'text',
      x: 20,
      y: 20
    });
    selectedProp.selected = false;
  }
}, { deep: true });

// Handle element selection
const selectElement = (index) => {
  selectedIndex.value = index;
};

// Handle keyboard movement
onMounted(() => {
  window.addEventListener('keydown', handleKeyPress);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyPress);
});

const handleKeyPress = (e) => {
  if (selectedIndex.value === -1) return;

  const step = 10;
  const element = designElements.value[selectedIndex.value];

  switch (e.key) {
    case 'ArrowLeft':
      element.x = Math.max(0, element.x - step);
      break;
    case 'ArrowRight':
      element.x = Math.min(designArea.value.clientWidth - 100, element.x + step);
      break;
    case 'ArrowUp':
      element.y = Math.max(0, element.y - step);
      break;
    case 'ArrowDown':
      element.y = Math.min(designArea.value.clientHeight - 50, element.y + step);
      break;
  }
};

// Enhanced drag and drop functionality
const handleDragStart = (e, index) => {
  selectElement(index);
  const element = designElements.value[index];
  dragStartPosition.value = {
    x: e.clientX - element.x,
    y: e.clientY - element.y
  };
};

const handleDrag = (e) => {
  if (!e.clientX || !e.clientY) return; // Ignore invalid drag events
  
  if (selectedIndex.value !== -1) {
    const rect = designArea.value.getBoundingClientRect();
    const x = e.clientX - rect.left - dragStartPosition.value.x;
    const y = e.clientY - rect.top - dragStartPosition.value.y;
    
    const element = designElements.value[selectedIndex.value];
    element.x = Math.max(0, Math.min(x, designArea.value.clientWidth - 100));
    element.y = Math.max(0, Math.min(y, designArea.value.clientHeight - 50));
  }
};

const handleDragEnd = () => {
  // Reset drag start position
  dragStartPosition.value = { x: 0, y: 0 };
};

// Handle drop (for compatibility)
const handleDrop = (e) => {
  if (selectedIndex.value === -1) return;

  const rect = designArea.value.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;

  const element = designElements.value[selectedIndex.value];
  element.x = Math.max(0, Math.min(x, designArea.value.clientWidth - 100));
  element.y = Math.max(0, Math.min(y, designArea.value.clientHeight - 50));
};

// Add barcode
const addBarcode = () => {
  if (!barcodeValue.value) return;

  designElements.value.push({
    text: barcodeValue.value,
    type: 'barcode',
    x: 20,
    y: 20
  });

  barcodeValue.value = '';
};

// Set barcode reference and generate barcode
const setBarcodeRef = (el, value) => {
  if (el && value) {
    JsBarcode(el, value, {
      format: 'CODE128',
      width: 2,
      height: 50,
      displayValue: true
    });
  }
};

// Alignment functions
const alignElements = (alignment) => {
  if (selectedIndex.value === -1) return;

  const area = designArea.value;
  const element = designElements.value[selectedIndex.value];

  switch (alignment) {
    case 'left':
      element.x = 0;
      break;
    case 'center':
      element.x = (area.clientWidth - 100) / 2;
      break;
    case 'right':
      element.x = area.clientWidth - 100;
      break;
    case 'top':
      element.y = 0;
      break;
    case 'middle':
      element.y = (area.clientHeight - 50) / 2;
      break;
    case 'bottom':
      element.y = area.clientHeight - 50;
      break;
  }
};
</script>