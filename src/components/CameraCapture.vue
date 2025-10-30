<template>
  <section class="capture">
    <h2>Capture a new moment</h2>
    <p class="helper">Use your camera or upload a photo from your device.</p>

    <div class="actions">
      <button type="button" class="primary" @click="triggerCapture">
        <span v-if="isNative">Open Camera</span>
        <span v-else>Take or Upload Photo</span>
      </button>
      <input
        ref="fileInput"
        type="file"
        accept="image/*"
        capture="environment"
        class="hidden-input"
        @change="onFileSelected"
      />
    </div>

    <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
  </section>
</template>

<script setup lang="ts">
import { Camera, CameraResultType, CameraSource } from '@capacitor/camera';
import { Capacitor } from '@capacitor/core';
import { computed, ref } from 'vue';

export interface PhotoDraft {
  dataUrl: string;
  format: string;
}

const emit = defineEmits<{ (e: 'captured', value: PhotoDraft): void }>();

const fileInput = ref<HTMLInputElement | null>(null);
const errorMessage = ref('');

const isNative = computed(() => Capacitor.isNativePlatform());

const triggerCapture = async () => {
  errorMessage.value = '';

  if (isNative.value) {
    try {
      const photo = await Camera.getPhoto({
        quality: 85,
        allowEditing: false,
        source: CameraSource.Camera,
        resultType: CameraResultType.DataUrl,
        saveToGallery: false,
      });

      if (!photo.dataUrl) {
        throw new Error('Unable to access the captured photo.');
      }

      emit('captured', {
        dataUrl: photo.dataUrl,
        format: photo.format ?? 'jpeg',
      });
    } catch (error) {
      if (error instanceof Error) {
        errorMessage.value = error.message;
      } else {
        errorMessage.value = 'An unexpected error occurred while using the camera.';
      }
    }

    return;
  }

  fileInput.value?.click();
};

const onFileSelected = async (event: Event) => {
  errorMessage.value = '';

  const target = event.target as HTMLInputElement;
  const file = target.files?.[0];

  if (!file) {
    return;
  }

  if (!file.type.startsWith('image/')) {
    errorMessage.value = 'Please choose an image file.';
    target.value = '';
    return;
  }

  const reader = new FileReader();

  reader.onload = () => {
    const result = reader.result;

    if (typeof result === 'string') {
      emit('captured', {
        dataUrl: result,
        format: file.type.replace('image/', '') || 'jpeg',
      });
    } else {
      errorMessage.value = 'Could not read the selected file.';
    }

    target.value = '';
  };

  reader.onerror = () => {
    errorMessage.value = 'There was a problem reading the selected file.';
    target.value = '';
  };

  reader.readAsDataURL(file);
};
</script>

<style scoped>
.capture {
  padding: 1.5rem;
  border-radius: 1rem;
  background: #ffffffcc;
  border: 1px solid rgba(15, 23, 42, 0.1);
  box-shadow: 0 12px 30px -20px rgba(15, 23, 42, 0.45);
}

h2 {
  margin: 0 0 0.5rem;
  font-size: 1.4rem;
}

.helper {
  margin-top: 0;
  margin-bottom: 1.5rem;
  color: #52606d;
}

.actions {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.primary {
  background: linear-gradient(135deg, #0ea5e9, #6366f1);
  color: white;
  border: none;
  padding: 0.8rem 1.6rem;
  border-radius: 999px;
  cursor: pointer;
  font-weight: 600;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.primary:hover {
  transform: translateY(-1px);
  box-shadow: 0 10px 20px -12px rgba(14, 165, 233, 0.9);
}

.hidden-input {
  display: none;
}

.error {
  margin-top: 1rem;
  color: #dc2626;
  font-weight: 500;
}
</style>
