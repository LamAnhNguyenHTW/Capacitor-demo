<template>
  <main>
    <header class="hero">
      <h1>Camera Journal</h1>
      <p>Capture photos, add a short story, and revisit your memories anywhere.</p>
    </header>

    <CameraCapture @captured="startDescription" />

    <Transition name="fade">
      <form v-if="pendingPhoto" class="description-card" @submit.prevent="savePhoto">
        <img :src="pendingPhoto.dataUrl" alt="Preview of the captured photo" />
        <label class="field">
          <span>Add a description</span>
          <textarea
            v-model="pendingDescription"
            rows="3"
            placeholder="What makes this moment special?"
            maxlength="280"
          ></textarea>
        </label>
        <div class="form-actions">
          <button type="submit" class="primary" :disabled="isSaving">
            <span v-if="isSaving">Saving…</span>
            <span v-else>Save to gallery</span>
          </button>
          <button type="button" class="link" @click="discardPending" :disabled="isSaving">
            Discard
          </button>
        </div>
      </form>
    </Transition>

    <PhotoGallery :photos="photos" />
  </main>
</template>

<script setup lang="ts">
import { nanoid } from 'nanoid/non-secure';
import { ref, watch } from 'vue';

import CameraCapture, { PhotoDraft } from './components/CameraCapture.vue';
import PhotoGallery, { PhotoEntry } from './components/PhotoGallery.vue';

const STORAGE_KEY = 'camera-journal-photos';

const photos = ref<PhotoEntry[]>(loadFromStorage());
const pendingPhoto = ref<PhotoDraft | null>(null);
const pendingDescription = ref('');
const isSaving = ref(false);

function loadFromStorage(): PhotoEntry[] {
  if (typeof window === 'undefined') {
    return [];
  }

  try {
    const raw = window.localStorage.getItem(STORAGE_KEY);

    if (!raw) {
      return [];
    }

    const parsed = JSON.parse(raw) as PhotoEntry[];

    return Array.isArray(parsed) ? parsed : [];
  } catch (error) {
    console.error('Unable to load stored photos', error);
    return [];
  }
}

watch(
  photos,
  (value) => {
    if (typeof window === 'undefined') {
      return;
    }

    window.localStorage.setItem(STORAGE_KEY, JSON.stringify(value));
  },
  { deep: true }
);

const startDescription = (draft: PhotoDraft) => {
  pendingPhoto.value = draft;
  pendingDescription.value = '';
};

const savePhoto = () => {
  if (!pendingPhoto.value) {
    return;
  }

  isSaving.value = true;

  const newPhoto: PhotoEntry = {
    id: nanoid(),
    dataUrl: pendingPhoto.value.dataUrl,
    description: pendingDescription.value.trim(),
    createdAt: new Date().toISOString(),
  };

  photos.value = [newPhoto, ...photos.value];

  isSaving.value = false;
  discardPending();
};

const discardPending = () => {
  pendingPhoto.value = null;
  pendingDescription.value = '';
};
</script>

<style scoped>
main {
  display: flex;
  flex-direction: column;
  gap: 2.5rem;
}

.hero {
  text-align: center;
}

.hero h1 {
  margin: 0;
  font-size: clamp(2.3rem, 4vw, 3rem);
  color: #0f172a;
}

.hero p {
  margin: 0.5rem auto 0;
  max-width: 32rem;
  color: #475569;
}

.description-card {
  display: grid;
  grid-template-columns: minmax(0, 320px) minmax(0, 1fr);
  gap: 1.5rem;
  padding: 1.5rem;
  border-radius: 1.2rem;
  background: white;
  border: 1px solid rgba(15, 23, 42, 0.1);
  box-shadow: 0 18px 45px -28px rgba(15, 23, 42, 0.55);
}

.description-card img {
  width: 100%;
  border-radius: 1rem;
  object-fit: cover;
  aspect-ratio: 4 / 5;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  color: #0f172a;
}

textarea {
  resize: vertical;
  min-height: 6rem;
  padding: 0.75rem 0.9rem;
  border-radius: 0.75rem;
  border: 1px solid rgba(15, 23, 42, 0.15);
  background: rgba(241, 245, 249, 0.7);
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

textarea:focus {
  border-color: #6366f1;
  box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.15);
  outline: none;
}

.form-actions {
  grid-column: span 2;
  display: flex;
  justify-content: flex-end;
  gap: 1rem;
}

.primary {
  background: linear-gradient(135deg, #22d3ee, #6366f1);
  color: white;
  border: none;
  padding: 0.8rem 1.6rem;
  border-radius: 999px;
  cursor: pointer;
  font-weight: 600;
  transition: transform 0.2s ease, box-shadow 0.2s ease, opacity 0.2s ease;
}

.primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.primary:not(:disabled):hover {
  transform: translateY(-1px);
  box-shadow: 0 10px 20px -12px rgba(34, 211, 238, 0.8);
}

.link {
  background: transparent;
  color: #64748b;
  border: none;
  cursor: pointer;
  font-weight: 600;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

@media (max-width: 768px) {
  .description-card {
    grid-template-columns: 1fr;
  }

  .form-actions {
    grid-column: span 1;
    justify-content: center;
  }
}
</style>
