<template>
  <section class="gallery" v-if="photos.length">
    <h2>Your captured memories</h2>
    <div class="grid">
      <article v-for="photo in photos" :key="photo.id" class="card">
        <div class="media">
          <img :src="photo.dataUrl" :alt="photo.description || 'Captured photo'" loading="lazy" />
          <div class="card-actions">
            <button type="button" class="ghost" @click="onEdit(photo.id)">Edit</button>
            <button type="button" class="ghost danger" @click="onRemove(photo.id)">Delete</button>
          </div>
        </div>
        <div class="details">
          <p class="description" v-if="photo.description">{{ photo.description }}</p>
          <p class="timestamp">{{ formatDate(photo.createdAt) }}</p>
        </div>
      </article>
    </div>
  </section>

  <section v-else class="empty">
    <h2>No photos yet</h2>
    <p>Capture your first memory to populate the gallery.</p>
  </section>
</template>

<script setup lang="ts">
export interface PhotoEntry {
  id: string;
  dataUrl: string;
  description: string;
  createdAt: string;
  format: string;
}

defineProps<{ photos: PhotoEntry[] }>();

const emit = defineEmits<{
  (e: 'edit', id: string): void;
  (e: 'remove', id: string): void;
}>();

const onEdit = (id: string) => {
  emit('edit', id);
};

const onRemove = (id: string) => {
  emit('remove', id);
};

const formatDate = (iso: string) =>
    new Date(iso).toLocaleString(undefined, {
      dateStyle: 'medium',
      timeStyle: 'short',
    });
</script>

<style scoped>
.gallery {
  margin-top: 2.5rem;
}

.grid {
  margin-top: 1.5rem;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1.5rem;
}

.card {
  overflow: hidden;
  border-radius: 1rem;
  background: white;
  box-shadow: 0 12px 30px -20px rgba(15, 23, 42, 0.3);
  border: 1px solid rgba(15, 23, 42, 0.08);
}

.media {
  position: relative;
}

img {
  width: 100%;
  display: block;
  aspect-ratio: 4 / 5;
  object-fit: cover;
}

.card-actions {
  position: absolute;
  inset: auto 0 0;
  display: flex;
  justify-content: space-between;
  gap: 0.5rem;
  padding: 0.75rem 1rem;
  background: linear-gradient(180deg, transparent 0%, rgba(15, 23, 42, 0.75) 100%);
}

.ghost {
  background: rgba(15, 23, 42, 0.65);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.25);
  border-radius: 999px;
  padding: 0.35rem 0.9rem;
  cursor: pointer;
  font-size: 0.85rem;
  transition: background 0.2s ease, transform 0.2s ease;
}

.ghost:hover {
  background: rgba(99, 102, 241, 0.75);
  transform: translateY(-1px);
}

.ghost.danger {
  background: rgba(248, 113, 113, 0.8);
  border-color: rgba(248, 113, 113, 0.5);
}

.ghost.danger:hover {
  background: rgba(239, 68, 68, 0.9);
}

.details {
  padding: 1rem 1.2rem 1.3rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.description {
  margin: 0;
  font-weight: 600;
  color: #0f172a;
}

.timestamp {
  margin: 0;
  color: #64748b;
  font-size: 0.9rem;
}

.empty {
  margin-top: 3rem;
  text-align: center;
  color: #475569;
}
</style>