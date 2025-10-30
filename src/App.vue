<template>
  <main>
    <header class="hero">
      <h1>Camera Journal</h1>
      <p>Capture photos, add a short story, and revisit your memories anywhere.</p>
    </header>

    <CameraCapture @captured="startDescription" />

    <Transition name="fade">
      <form v-if="pendingPhoto" class="description-card" @submit.prevent="savePhoto">
        <div class="preview">
          <img :src="pendingPhoto.dataUrl" alt="Preview of the captured photo" />
          <button
              type="button"
              class="ghost"
              @click="openCropperForPending"
              :disabled="isSaving"
          >
            Adjust crop
          </button>
        </div>
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

    <Transition name="fade">
      <form v-if="editingDraft && editingTarget" class="description-card" @submit.prevent="applyEdit">
        <div class="preview">
          <img :src="editingDraft.dataUrl" alt="Preview of the selected photo" />
          <button type="button" class="ghost" @click="openCropperForEdit" :disabled="isUpdating">
            Adjust crop
          </button>
        </div>
        <label class="field">
          <span>Update the description</span>
          <textarea
              v-model="editingDescription"
              rows="3"
              maxlength="280"
              placeholder="Add more detail to this memory"
          ></textarea>
        </label>
        <div class="form-actions">
          <button type="submit" class="primary" :disabled="isUpdating">
            <span v-if="isUpdating">Saving…</span>
            <span v-else>Save changes</span>
          </button>
          <button type="button" class="link" @click="closeEdit" :disabled="isUpdating">
            Cancel
          </button>
        </div>
      </form>
    </Transition>

    <PhotoGallery :photos="photos" @edit="beginEdit" @delete="requestDelete" />
  </main>

  <PhotoCropper
      v-if="cropDraft"
      :src="cropDraft.dataUrl"
      :format="cropDraft.format"
      @cancel="handleCropCancel"
      @confirm="handleCropConfirm"
  />
</template>

<script setup lang="ts">
import { nanoid } from 'nanoid/non-secure';
import { ref, watch } from 'vue';

import CameraCapture, { PhotoDraft } from './components/CameraCapture.vue';
import PhotoGallery, { PhotoEntry } from './components/PhotoGallery.vue';
import PhotoCropper from './components/PhotoCropper.vue';

const STORAGE_KEY = 'camera-journal-photos';

const inferFormat = (dataUrl: string): string => {
  const match = dataUrl.match(/^data:image\/([a-zA-Z0-9+.-]+);/);
  return match ? match[1] : 'jpeg';
};

const photos = ref<PhotoEntry[]>(loadFromStorage());
const pendingPhoto = ref<PhotoDraft | null>(null);
const pendingDescription = ref('');
const isSaving = ref(false);
const cropDraft = ref<PhotoDraft | null>(null);
const cropContext = ref<'new' | 'edit' | null>(null);
const editingTarget = ref<PhotoEntry | null>(null);
const editingDraft = ref<PhotoDraft | null>(null);
const editingDescription = ref('');
const isUpdating = ref(false);

function loadFromStorage(): PhotoEntry[] {
  if (typeof window === 'undefined') {
    return [];
  }

  try {
    const raw = window.localStorage.getItem(STORAGE_KEY);

    if (!raw) {
      return [];
    }

    const parsed = JSON.parse(raw) as Partial<PhotoEntry>[];

    if (!Array.isArray(parsed)) {
      return [];
    }

    return parsed
        .filter((entry): entry is PhotoEntry => typeof entry === 'object' && !!entry)
        .map((entry) => ({
          id: entry.id ?? nanoid(),
          dataUrl: entry.dataUrl ?? '',
          description: entry.description ?? '',
          createdAt: entry.createdAt ?? new Date().toISOString(),
          format: entry.format ?? inferFormat(entry.dataUrl ?? ''),
        }))
        .filter((entry) => Boolean(entry.dataUrl));
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
  cropContext.value = 'new';
  cropDraft.value = draft;
  pendingPhoto.value = null;
  pendingDescription.value = '';
};

const openCropperForPending = () => {
  if (!pendingPhoto.value) {
    return;
  }

  cropContext.value = 'new';
  cropDraft.value = { ...pendingPhoto.value };
};

const openCropperForEdit = () => {
  if (!editingDraft.value) {
    return;
  }

  cropContext.value = 'edit';
  cropDraft.value = { ...editingDraft.value };
};

const handleCropCancel = () => {
  if (cropContext.value === 'new' && !pendingPhoto.value) {
    discardPending();
  }

  cropDraft.value = null;
  cropContext.value = null;
};

const handleCropConfirm = (draft: PhotoDraft) => {
  if (cropContext.value === 'new') {
    pendingPhoto.value = draft;
  } else if (cropContext.value === 'edit') {
    editingDraft.value = draft;
  }

  cropDraft.value = null;
  cropContext.value = null;
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
    format: pendingPhoto.value.format || inferFormat(pendingPhoto.value.dataUrl),
  };

  photos.value = [newPhoto, ...photos.value];

  isSaving.value = false;
  discardPending();
};

const discardPending = () => {
  pendingPhoto.value = null;
  pendingDescription.value = '';
  if (cropContext.value === 'new') {
    cropContext.value = null;
    cropDraft.value = null;
  }
};

const beginEdit = (id: string) => {
  const target = photos.value.find((photo) => photo.id === id);

  if (!target) {
    return;
  }

  editingTarget.value = { ...target };
  editingDraft.value = {
    dataUrl: target.dataUrl,
    format: target.format || inferFormat(target.dataUrl),
  };
  editingDescription.value = target.description;
};

const applyEdit = () => {
  if (!editingTarget.value || !editingDraft.value) {
    return;
  }

  isUpdating.value = true;

  photos.value = photos.value.map((photo) =>
      photo.id === editingTarget.value?.id
          ? {
            ...photo,
            dataUrl: editingDraft.value?.dataUrl ?? photo.dataUrl,
            description: editingDescription.value.trim(),
            format: editingDraft.value?.format || photo.format,
          }
          : photo
  );

  isUpdating.value = false;
  closeEdit();
};

const closeEdit = () => {
  editingTarget.value = null;
  editingDraft.value = null;
  editingDescription.value = '';
  if (cropContext.value === 'edit') {
    cropContext.value = null;
    cropDraft.value = null;
  }
};

const requestDelete = (id: string) => {
  if (typeof window !== 'undefined') {
    const confirmed = window.confirm('Remove this photo from your gallery?');
    if (!confirmed) {
      return;
    }
  }

  photos.value = photos.value.filter((photo) => photo.id !== id);

  if (editingTarget.value?.id === id) {
    closeEdit();
  }
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

.preview {
  position: relative;
}

.description-card img {
  width: 100%;
  border-radius: 1rem;
  object-fit: cover;
  aspect-ratio: 4 / 5;
  display: block;
}

.preview .ghost {
  position: absolute;
  inset: auto 1rem 1rem auto;
  background: rgba(15, 23, 42, 0.75);
  color: white;
  border: none;
  border-radius: 999px;
  padding: 0.4rem 1rem;
  cursor: pointer;
  font-size: 0.85rem;
  font-weight: 600;
  transition: background 0.2s ease, transform 0.2s ease;
}

.preview .ghost:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.preview .ghost:not(:disabled):hover {
  background: rgba(99, 102, 241, 0.9);
  transform: translateY(-1px);
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