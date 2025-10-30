<template>
  <div class="backdrop">
    <section class="panel" role="dialog" aria-modal="true" aria-labelledby="cropper-title">
      <header class="panel-header">
        <h2 id="cropper-title">Adjust your photo</h2>
        <p>Drag to reposition and zoom to crop before saving.</p>
      </header>

      <div
          ref="cropArea"
          class="crop-area"
          :style="cropAreaStyle"
          @pointerdown="startDrag"
          @pointermove="onDrag"
          @pointerup="endDrag"
          @pointerleave="endDrag"
      >
        <img
            ref="imageEl"
            :src="src"
            alt="Photo awaiting crop"
            class="crop-image"
            :style="imageStyle"
            @load="onImageLoad"
        />
        <div class="crop-frame" aria-hidden="true"></div>
      </div>

      <div class="controls">
        <label>
          <span>Zoom</span>
          <input
              type="range"
              min="1"
              max="3"
              step="0.01"
              v-model.number="scale"
              @input="clampOffsets"
          />
        </label>
        <div class="actions">
          <button type="button" class="link" @click="cancel">Cancel</button>
          <button type="button" class="primary" @click="confirmCrop" :disabled="!ready">
            Apply crop
          </button>
        </div>
      </div>
    </section>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue';

import type { PhotoDraft } from './CameraCapture.vue';

const props = defineProps<{
  src: string;
  format?: string;
  aspectRatio?: number;
}>();

const emit = defineEmits<{ (e: 'cancel'): void; (e: 'confirm', draft: PhotoDraft): void }>();

const cropArea = ref<HTMLElement | null>(null);
const imageEl = ref<HTMLImageElement | null>(null);
const ready = ref(false);

const containerRect = reactive({ width: 0, height: 0 });

const naturalSize = reactive({ width: 0, height: 0 });

const scale = ref(1);
const offset = reactive({ x: 0, y: 0 });

const dragging = ref(false);
const dragStart = reactive({ x: 0, y: 0 });
const dragOrigin = reactive({ x: 0, y: 0 });

let resizeObserver: ResizeObserver | null = null;

const aspect = computed(() => props.aspectRatio ?? 4 / 5);

const cropAreaStyle = computed(() => ({
  aspectRatio: `${aspect.value} / 1`,
}));

const hasDimensions = computed(() => naturalSize.width > 0 && naturalSize.height > 0);

const baseScale = computed(() => {
  if (!hasDimensions.value || !containerRect.width || !containerRect.height) {
    return 1;
  }

  const widthRatio = containerRect.width / naturalSize.width;
  const heightRatio = containerRect.height / naturalSize.height;

  return Math.max(widthRatio, heightRatio);
});

const displayWidth = computed(() => naturalSize.width * baseScale.value);
const displayHeight = computed(() => naturalSize.height * baseScale.value);

const scaledWidth = computed(() => displayWidth.value * scale.value);
const scaledHeight = computed(() => displayHeight.value * scale.value);

const imageStyle = computed(() => {
  if (!hasDimensions.value) {
    return {};
  }

  return {
    width: `${scaledWidth.value}px`,
    height: `${scaledHeight.value}px`,
    top: `calc(50% + ${offset.y}px)`,
    left: `calc(50% + ${offset.x}px)`,
  };
});

const resetState = () => {
  scale.value = 1;
  offset.x = 0;
  offset.y = 0;
  ready.value = false;
};

const resolveContainerWidth = () => {
  if (containerRect.width) {
    return containerRect.width;
  }

  if (cropArea.value) {
    return cropArea.value.clientWidth;
  }

  return imageEl.value?.clientWidth ?? imageEl.value?.naturalWidth ?? 0;
};

const resolveContainerHeight = () => {
  if (containerRect.height) {
    return containerRect.height;
  }

  if (cropArea.value) {
    return cropArea.value.clientHeight;
  }

  return imageEl.value?.clientHeight ?? imageEl.value?.naturalHeight ?? 0;
};

const measure = () => {
  if (!cropArea.value) {
    return;
  }

  const rect = cropArea.value.getBoundingClientRect();
  const width = rect.width || cropArea.value.clientWidth;
  const height = rect.height || cropArea.value.clientHeight;

  containerRect.width = width || imageEl.value?.clientWidth || imageEl.value?.naturalWidth || 0;
  containerRect.height = height || imageEl.value?.clientHeight || imageEl.value?.naturalHeight || 0;
  clampOffsets();
};

const setupObserver = () => {
  if (typeof window === 'undefined' || !('ResizeObserver' in window)) {
    return;
  }

  resizeObserver = new ResizeObserver(() => measure());
  if (cropArea.value) {
    resizeObserver.observe(cropArea.value);
  }
};

const cleanupObserver = () => {
  if (resizeObserver) {
    resizeObserver.disconnect();
    resizeObserver = null;
  }
};

const onImageLoad = () => {
  if (!imageEl.value) {
    return;
  }

  naturalSize.width = imageEl.value.naturalWidth;
  naturalSize.height = imageEl.value.naturalHeight;
  ready.value = true;
  nextTick(() => {
    measure();
    clampOffsets();
  });
};

const clampOffsets = () => {
  const containerWidth = resolveContainerWidth();
  const containerHeight = resolveContainerHeight();

  if (!containerWidth || !containerHeight) {
    return;
  }

  const maxX = Math.max(0, (scaledWidth.value - containerWidth) / 2);
  const maxY = Math.max(0, (scaledHeight.value - containerHeight) / 2);

  offset.x = Math.min(maxX, Math.max(-maxX, offset.x));
  offset.y = Math.min(maxY, Math.max(-maxY, offset.y));
};

const startDrag = (event: PointerEvent) => {
  if (!ready.value) {
    return;
  }

  dragging.value = true;
  dragStart.x = event.clientX;
  dragStart.y = event.clientY;
  dragOrigin.x = offset.x;
  dragOrigin.y = offset.y;
  try {
    (event.currentTarget as HTMLElement).setPointerCapture(event.pointerId);
  } catch (error) {
    // ignore when pointer capture is unsupported
  }
};

const onDrag = (event: PointerEvent) => {
  if (!dragging.value) {
    return;
  }

  const deltaX = event.clientX - dragStart.x;
  const deltaY = event.clientY - dragStart.y;

  offset.x = dragOrigin.x + deltaX;
  offset.y = dragOrigin.y + deltaY;

  clampOffsets();
};

const endDrag = (event: PointerEvent) => {
  if (!dragging.value) {
    return;
  }

  dragging.value = false;
  try {
    (event.currentTarget as HTMLElement).releasePointerCapture(event.pointerId);
  } catch (error) {
    // ignore when pointer capture wasn't set
  }
};

const cancel = () => {
  emit('cancel');
};

const confirmCrop = () => {
  if (!imageEl.value) {
    return;
  }

  const naturalWidth = imageEl.value.naturalWidth;
  const naturalHeight = imageEl.value.naturalHeight;

  const containerWidth = resolveContainerWidth();
  const containerHeight = resolveContainerHeight();

  if (!containerWidth || !containerHeight) {
    emit('cancel');
    return;
  }

  const scaleFactorX = naturalWidth / scaledWidth.value;
  const scaleFactorY = naturalHeight / scaledHeight.value;

  const imageLeftInContainer = containerWidth / 2 + offset.x - scaledWidth.value / 2;
  const imageTopInContainer = containerHeight / 2 + offset.y - scaledHeight.value / 2;

  const cropX = Math.max(0, -imageLeftInContainer * scaleFactorX);
  const cropY = Math.max(0, -imageTopInContainer * scaleFactorY);
  const cropWidth = Math.min(naturalWidth, containerWidth * scaleFactorX);
  const cropHeight = Math.min(naturalHeight, containerHeight * scaleFactorY);

  const canvas = document.createElement('canvas');
  canvas.width = Math.round(cropWidth);
  canvas.height = Math.round(cropHeight);
  const context = canvas.getContext('2d');

  if (!context) {
    emit('cancel');
    return;
  }

  context.drawImage(
      imageEl.value,
      cropX,
      cropY,
      cropWidth,
      cropHeight,
      0,
      0,
      canvas.width,
      canvas.height
  );

  const preferredFormat = (props.format && props.format.startsWith('image/'))
      ? props.format
      : `image/${props.format || 'jpeg'}`;

  const quality = preferredFormat.includes('jpeg') ? 0.92 : undefined;
  const dataUrl = canvas.toDataURL(preferredFormat, quality);

  emit('confirm', {
    dataUrl,
    format: props.format || preferredFormat.replace('image/', ''),
  });
};

watch(
    () => props.src,
    () => {
      resetState();
      nextTick(() => {
        measure();
        if (imageEl.value?.complete) {
          onImageLoad();
        }
      });
    }
);

onMounted(() => {
  nextTick(() => {
    measure();
    setupObserver();
  });
});

onBeforeUnmount(() => {
  cleanupObserver();
});
</script>

<style scoped>
.backdrop {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.65);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1.5rem;
  z-index: 20;
}

.panel {
  background: white;
  border-radius: 1.2rem;
  width: min(640px, 100%);
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1.2rem;
  box-shadow: 0 24px 60px -30px rgba(15, 23, 42, 0.7);
}

.panel-header h2 {
  margin: 0;
  font-size: 1.35rem;
  color: #0f172a;
}

.panel-header p {
  margin: 0.2rem 0 0;
  color: #475569;
}

.crop-area {
  position: relative;
  width: 100%;
  border-radius: 1rem;
  overflow: hidden;
  background: #0f172a;
  cursor: grab;
  touch-action: none;
}

.crop-area:active {
  cursor: grabbing;
}

.crop-image {
  position: absolute;
  transform: translate(-50%, -50%);
  user-select: none;
  touch-action: none;
}

.crop-frame {
  position: absolute;
  inset: 0;
  border: 2px solid rgba(255, 255, 255, 0.8);
  pointer-events: none;
  box-shadow: inset 0 0 0 2000px rgba(15, 23, 42, 0.35);
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.controls label {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
  color: #0f172a;
  font-weight: 600;
}

.controls input[type='range'] {
  width: 100%;
}

.actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.primary {
  background: linear-gradient(135deg, #22d3ee, #6366f1);
  color: white;
  border: none;
  padding: 0.75rem 1.5rem;
  border-radius: 999px;
  cursor: pointer;
  font-weight: 600;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.primary:not(:disabled):hover {
  transform: translateY(-1px);
  box-shadow: 0 12px 24px -18px rgba(99, 102, 241, 0.8);
}

.link {
  background: transparent;
  border: none;
  color: #6366f1;
  font-weight: 600;
  cursor: pointer;
  padding: 0.75rem 1.2rem;
  border-radius: 999px;
}

.link:hover {
  background: rgba(99, 102, 241, 0.12);
}
</style>