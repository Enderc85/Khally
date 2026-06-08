<template>
  <div id="app">
    <div class="hero">
      <h2>Welcome to Khally</h2>
      <p>Click any day to add an event. Drag events between days once they appear.</p>
      <div class="view-controls">
        <label>
          View:
          <select v-model="view">
            <option value="monthly">Month</option>
            <option value="weekly">Week</option>
            <option value="daily">Day</option>
          </select>
        </label>
      </div>
    </div>

    <div v-if="view === 'daily'" class="day-view-panel">
      <div class="day-view-header">
        <button @click="moveDay(-1)">←</button>
        <div>
          <div class="day-view-title">{{ formatLongDate(currentDay) }}</div>
          <div class="day-view-subtitle">Day view shows only the selected date.</div>
        </div>
        <button @click="moveDay(1)">→</button>
      </div>

      <div class="day-view-day-card">
        <div class="date-label-large">{{ formatDayLabel(currentDay) }}</div>
        <div class="events day-events">
          <div v-if="eventsForDate(currentDay).length === 0" class="empty-day">No events for this day. Click a day in Month or Week view to add one.</div>
          <div v-for="ev in eventsForDate(currentDay)" :key="ev.id" class="event compact daily-event" draggable="true" @dragstart="onDragStart($event, ev.id)">
            <div class="event-text">
              <span class="title">{{ ev.title }}</span>
            </div>
            <img v-if="ev.image" :src="ev.image" class="thumb day-thumb" />
            <div class="event-text event-text--center">
              <span class="description">{{ ev.description || 'No description provided.' }}</span>
            </div>
            <div class="event-actions" v-if="view !== 'monthly'">
              <button class="edit-button" data-tooltip="Edit event" @click.stop="openEditEvent(ev)">✎</button>
              <button class="delete-button" data-tooltip="Delete event" @click.stop="openDeleteConfirm(ev)">×</button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div v-else class="calendar-wrapper" :class="view">
      <calendar
        :attributes="[]"
        :rows="1"
        :columns="1"
        :view="view"
        :transition="'fade'"
        @dayclick="onDayClick"
      >
      <template #day-content="{ day }">
        <div class="day-cell" @click="onDayClick({ date: day.date })" @dragover.prevent @drop="onDrop($event, day.date)">
          <div class="date-label">{{ day.day }}</div>
          <div class="events">
            <div v-if="view === 'monthly'">
              <div v-if="eventsForDate(day.date).length > 0" class="event-count">
                <span>{{ eventsForDate(day.date).length }}</span>
                <small>{{ eventsForDate(day.date).length === 1 ? 'event' : 'events' }}</small>
              </div>
            </div>
            <div v-else>
              <div v-for="ev in eventsForDate(day.date)" :key="ev.id" class="event compact" draggable="true" @dragstart="onDragStart($event, ev.id)">
                <span class="title">{{ ev.title }}</span>
                <div class="event-actions" v-if="view !== 'monthly'">
                  <button class="edit-button" data-tooltip="Edit event" @click.stop="openEditEvent(ev)">✎</button>
                  <button class="delete-button" data-tooltip="Delete event" @click.stop="openDeleteConfirm(ev)">×</button>
                </div>
              </div>
            </div>
          </div>
        </div>
      </template>
      </calendar>
    </div>

    <div v-if="showModal" class="modal">
      <div class="modal-content">
        <h3>Add Event - {{ formatDate(selectedDate) }}</h3>
        <input v-model="newEvent.title" placeholder="Title" />
        <textarea v-model="newEvent.description" placeholder="Description"></textarea>
        <input type="file" @change="onFileChange" accept="image/*" />
        <div v-if="newEvent.image"><img :src="newEvent.image" class="preview" /></div>
        <div class="actions">
          <button @click="saveEvent">{{ isEditing ? 'Save' : 'Add' }}</button>
          <button @click="closeModal">Cancel</button>
        </div>
      </div>
    </div>

    <div v-if="showConfirmModal" class="modal">
      <div class="modal-content">
        <h3>Delete event?</h3>
        <p class="confirm-text">Are you sure you want to delete this event?</p>
        <div class="actions confirm-actions">
          <button @click="confirmDeleteEvent">Delete</button>
          <button @click="cancelDelete">Cancel</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed } from 'vue'
import { Calendar } from 'v-calendar'
import 'v-calendar/style.css'

let idCounter = 1

export default {
  name: 'App',
  components: { Calendar },
  setup() {
    const view = ref('monthly')
    const today = new Date().toISOString().slice(0, 10)
    const events = ref([
      { id: idCounter++, title: 'Welcome event', description: 'This is your first calendar event.', date: today, image: null }
    ])
    const showModal = ref(false)
    const showConfirmModal = ref(false)
    const selectedDate = ref(null)
    const currentDay = ref(new Date())
    const newEvent = ref({ title: '', description: '', image: null })
    const isEditing = ref(false)
    const editingEventId = ref(null)
    const pendingDeleteEventId = ref(null)
    const pendingDeleteTitle = ref('')
    const dragEventId = ref(null)

    const isDayView = computed(() => view.value === 'daily')

    const shouldShowImage = (event, date) => {
      if (!isDayView.value || !event.image) return false
      const count = eventsForDate(date).length
      return count <= 3
    }

    const shouldShowDescription = (event, date) => {
      if (!isDayView.value) return false
      return !!event.description
    }

    const moveDay = (offset) => {
      currentDay.value = new Date(currentDay.value.valueOf() + offset * 24 * 60 * 60 * 1000)
      selectedDate.value = currentDay.value
    }

    const formatLongDate = (date) => {
      if (!date) return ''
      return new Intl.DateTimeFormat('es-ES', { weekday: 'long', day: 'numeric', month: 'long', year: 'numeric' }).format(date)
    }

    const formatDayLabel = (date) => {
      if (!date) return ''
      return new Intl.DateTimeFormat('es-ES', { weekday: 'long', day: 'numeric', month: 'short' }).format(date)
    }

    const onDayClick = ({ date }) => {
      selectedDate.value = date
      currentDay.value = date
      newEvent.value = { title: '', description: '', image: null }
      isEditing.value = false
      editingEventId.value = null
      showModal.value = true
    }

    const closeModal = () => {
      showModal.value = false
      isEditing.value = false
      editingEventId.value = null
    }

    const addEvent = () => {
      if (!newEvent.value.title) return
      events.value.push({
        id: idCounter++,
        title: newEvent.value.title,
        description: newEvent.value.description,
        date: selectedDate.value.toISOString().slice(0,10),
        image: newEvent.value.image
      })
      showModal.value = false
    }

    const saveEvent = () => {
      if (!newEvent.value.title) return
      if (isEditing.value && editingEventId.value !== null) {
        const index = events.value.findIndex(ev => ev.id === editingEventId.value)
        if (index >= 0) {
          events.value[index] = {
            ...events.value[index],
            title: newEvent.value.title,
            description: newEvent.value.description,
            image: newEvent.value.image
          }
        }
      } else {
        addEvent()
        return
      }
      showModal.value = false
      isEditing.value = false
      editingEventId.value = null
    }

    const openEditEvent = (event) => {
      isEditing.value = true
      editingEventId.value = event.id
      selectedDate.value = new Date(event.date)
      newEvent.value = {
        title: event.title,
        description: event.description,
        image: event.image
      }
      showModal.value = true
    }

    const openDeleteConfirm = (event) => {
      pendingDeleteEventId.value = event.id
      pendingDeleteTitle.value = event.title
      showConfirmModal.value = true
    }

    const confirmDeleteEvent = () => {
      if (pendingDeleteEventId.value !== null) {
        deleteEvent(pendingDeleteEventId.value)
      }
      pendingDeleteEventId.value = null
      pendingDeleteTitle.value = ''
      showConfirmModal.value = false
    }

    const cancelDelete = () => {
      pendingDeleteEventId.value = null
      pendingDeleteTitle.value = ''
      showConfirmModal.value = false
    }

    const onFileChange = (e) => {
      const f = e.target.files[0]
      if (!f) return
      const reader = new FileReader()
      reader.onload = () => { newEvent.value.image = reader.result }
      reader.readAsDataURL(f)
    }

    const eventsForDate = (date) => {
      const key = date.toISOString().slice(0,10)
      return events.value.filter(e => e.date === key)
    }

    const onDragStart = (e, id) => { dragEventId.value = id; e.dataTransfer.setData('text/plain', id) }

    const deleteEvent = (id) => {
      events.value = events.value.filter((ev) => ev.id !== id)
    }

    const onDrop = (e, date) => {
      const id = e.dataTransfer.getData('text/plain') || dragEventId.value
      if (!id) return
      const ev = events.value.find(x => String(x.id) === String(id))
      if (ev) ev.date = date.toISOString().slice(0,10)
      dragEventId.value = null
    }

    const formatDate = (d) => d ? d.toISOString().slice(0,10) : ''

    return {
      view,
      events,
      showModal,
      showConfirmModal,
      selectedDate,
      currentDay,
      newEvent,
      isEditing,
      onDayClick,
      closeModal,
      saveEvent,
      addEvent,
      onFileChange,
      eventsForDate,
      onDragStart,
      onDrop,
      deleteEvent,
      openEditEvent,
      openDeleteConfirm,
      confirmDeleteEvent,
      cancelDelete,
      moveDay,
      formatDate,
      formatLongDate,
      formatDayLabel,
      isDayView,
      shouldShowImage,
      shouldShowDescription,
    }
  }
}
</script>

<style scoped>
#app {
  font-family: 'Inter', system-ui, sans-serif;
  width: 90vw;
  max-width: 960px;
  padding: 32px 16px;
  margin: 0 auto;
  min-height: 100vh;
  background: linear-gradient(180deg, #f9fbff 0%, #eef5ff 45%, #f7f2ff 100%);
}

.hero {
  text-align: center;
  margin-bottom: 24px;
  padding: 22px 24px;
  border-radius: 32px;
  background: rgba(255, 255, 255, 0.85);
  box-shadow: 0 20px 60px rgba(27, 59, 118, 0.12);
  backdrop-filter: blur(12px);
}

.hero h2 {
  margin-bottom: 8px;
  color: #102a62;
  font-size: clamp(1.9rem, 2.8vw, 2.6rem);
}

.hero p {
  color: #475569;
  font-size: 1rem;
  max-width: 680px;
  margin: 0 auto;
}

.view-controls {
  margin-top: 18px;
  display: inline-flex;
  align-items: center;
  gap: 12px;
  justify-content: center;
}

.view-controls select {
  border-radius: 16px;
  padding: 10px 14px;
  border: 1px solid rgba(148, 163, 184, 0.38);
  background: white;
  color: #0f172a;
  font-weight: 600;
}

:deep(.vc-container) {
  width: 100%;
  max-width: 100%;
  min-height: 720px;
  border-radius: 34px;
  overflow: hidden;
  border: 1px solid rgba(255, 255, 255, 0.8);
  background: rgba(255, 255, 255, 0.82);
  box-shadow: 0 30px 80px rgba(16, 42, 98, 0.12);
}

:deep(.vc-container),
:deep(.vc-pane),
:deep(.vc-pane-layout),
:deep(.vc-weeks),
:deep(.vc-week),
:deep(.vc-day) {
  width: 100% !important;
  min-width: 100% !important;
}

:deep(.vc-container) {
  min-height: 720px;
}

:deep(.vc-weeks) {
  min-height: 520px;
}

:deep(.vc-week) {
  align-items: stretch;
}

:deep(.vc-day) {
  min-height: 120px;
}

:deep(.vc-pane-layout) {
  gap: 14px;
  padding: 28px;
}

.events {
  display: flex;
  flex-direction: column;
  gap: 8px;
  width: 100%;
}

.event {
  background: linear-gradient(135deg, #6d5dfc, #4b84ff);
  color: #fff;
  padding: 6px 10px;
  border-radius: 16px;
  cursor: grab;
  font-size: 0.8rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  box-shadow: 0 10px 20px rgba(76, 140, 255, 0.14);
  min-height: 32px;
  line-height: 1.2;
  width: 100%;
}

.event.large {
  min-height: 50px;
  padding: 8px 10px;
}

.event.compact {
  min-height: 32px;
  padding: 6px 10px;
}

.vc-pane-header-wrapper {
  position: relative;
  padding-top: 24px;
}

.day-cell {
  min-height: 120px;
  padding: 12px;
  border-radius: 24px;
  border: 1px solid rgba(16, 42, 98, 0.08);
  background: linear-gradient(180deg, rgba(255,255,255,0.96), rgba(235,245,255,0.95));
  transition: transform 250ms ease, box-shadow 250ms ease, border-color 250ms ease;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.day-cell:hover {
  transform: translateY(-3px);
  box-shadow: 0 18px 35px rgba(35, 66, 147, 0.12);
  border-color: rgba(16, 42, 98, 0.16);
}

.date-label {
  font-weight: 700;
  color: #1d3a70;
  margin-bottom: 10px;
  font-size: 0.95rem;
}

.empty-day {
  color: #64748b;
  font-size: 0.85rem;
  padding: 10px 12px;
  border-radius: 18px;
  background: rgba(99, 102, 241, 0.08);
  border: 1px dashed rgba(99, 102, 241, 0.3);
  text-align: center;
}

.thumb {
  width: 30px;
  height: 30px;
  border-radius: 10px;
  object-fit: cover;
  border: 1px solid rgba(255, 255, 255, 0.75);
}

.event-text {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.event-actions {
  display: inline-flex;
  align-items: center;
  gap: 4px;
}

.event-count {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  border-radius: 16px;
  background: rgba(99, 102, 241, 0.12);
  color: #3730a3;
  font-weight: 700;
  font-size: 0.82rem;
}

.event-count span {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 26px;
  height: 26px;
  border-radius: 999px;
  background: rgba(99, 102, 241, 0.92);
  color: white;
  font-size: 0.8rem;
}

.event-count small {
  color: #475569;
  font-size: 0.72rem;
  font-weight: 600;
}

.edit-button,
.delete-button {
  border: 1px solid rgba(99, 102, 241, 0.28);
  background: rgba(99, 102, 241, 0.22);
  color: #ffffff;
  width: 20px;
  height: 20px;
  border-radius: 999px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 0.8rem;
  transition: background 150ms ease, transform 150ms ease, box-shadow 150ms ease;
}

.edit-button:hover,
.delete-button:hover {
  background: rgba(99, 102, 241, 0.34);
  transform: scale(1.05);
  box-shadow: 0 8px 18px rgba(99, 102, 241, 0.24);
}

.edit-button::after,
.delete-button::after {
  content: attr(data-tooltip);
  position: absolute;
  bottom: calc(100% + 10px);
  left: 50%;
  transform: translateX(-50%);
  white-space: nowrap;
  background: rgba(99, 102, 241, 0.95);
  color: #f8fbff;
  padding: 7px 12px;
  border-radius: 14px;
  font-size: 0.75rem;
  font-weight: 700;
  box-shadow: 0 16px 30px rgba(16, 42, 98, 0.18);
  opacity: 0;
  pointer-events: none;
  transition: opacity 0ms ease;
  z-index: 10;
}

.edit-button:hover::after,
.delete-button:hover::after {
  opacity: 1;
}

.edit-button::before,
.delete-button::before {
  content: '';
  position: absolute;
  bottom: 100%;
  left: 50%;
  transform: translateX(-50%);
  border-width: 6px;
  border-style: solid;
  border-color: transparent transparent rgba(99, 102, 241, 0.95) transparent;
  opacity: 0;
  transition: opacity 0ms ease;
}

.edit-button:hover::before,
.delete-button:hover::before {
  opacity: 1;
}

.edit-button,
.delete-button {
  position: relative;
}

.confirm-text {
  margin: 12px 0 0;
  color: #475569;
  line-height: 1.5;
}

.confirm-actions {
  justify-content: flex-end;
}

.confirm-actions button:first-child {
  background: #ef4444;
}

.confirm-actions button:last-child {
  background: #64748b;
}

.calendar-wrapper.weekly :deep(.vc-container),
.calendar-wrapper.weekly :deep(.vc-pane),
.calendar-wrapper.weekly :deep(.vc-weeks),
.calendar-wrapper.weekly :deep(.vc-week) {
  min-height: 720px;
}

.calendar-wrapper.weekly :deep(.vc-day) {
  min-height: 520px;
}

.calendar-wrapper.weekly :deep(.vc-day-content) {
  height: 100%;
}

.description {
  font-size: 0.78rem;
  color: rgba(255, 255, 255, 0.88);
  margin-top: 4px;
  line-height: 1.25;
}

.title {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(15, 23, 42, 0.55);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
  padding: 28px;
  border-radius: 30px;
  box-shadow: 0 30px 80px rgba(15, 23, 42, 0.18);
  max-width: 420px;
  width: 100%;
  border: 1px solid rgba(99, 102, 241, 0.16);
}

.modal-content h3 {
  margin: 0 0 18px 0;
  color: #102a62;
}

.modal-content input,
.modal-content textarea {
  width: 100%;
  padding: 12px 14px;
  margin-bottom: 12px;
  border: 1px solid rgba(148, 163, 184, 0.32);
  border-radius: 16px;
  box-sizing: border-box;
  font-size: 0.96rem;
  transition: border-color 200ms ease, box-shadow 200ms ease;
}

.modal-content textarea {
  min-height: 100px;
  resize: vertical;
}

.modal-content input:focus,
.modal-content textarea:focus {
  outline: none;
  border-color: #6366f1;
  box-shadow: 0 0 0 4px rgba(99, 102, 241, 0.14);
}

.preview {
  max-width: 100%;
  max-height: 150px;
  margin-bottom: 14px;
  border-radius: 18px;
}

.actions {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
  margin-top: 18px;
}

button {
  padding: 12px 22px;
  border: none;
  border-radius: 18px;
  cursor: pointer;
  font-size: 0.95rem;
  transition: transform 200ms ease, box-shadow 200ms ease;
}

button:first-child {
  background: linear-gradient(135deg, #6366f1, #4f46e5);
  color: white;
  box-shadow: 0 12px 24px rgba(99, 102, 241, 0.24);
}

button:first-child:hover {
  transform: translateY(-1px);
}

button:last-child {
  background: #f97316;
  color: white;
  box-shadow: 0 12px 24px rgba(249, 115, 22, 0.24);
}

button:last-child:hover {
  transform: translateY(-1px);
}

.day-view-panel {
  width: 100%;
  min-height: 720px;
  box-sizing: border-box;
  background: rgba(255, 255, 255, 0.9);
  border-radius: 32px;
  padding: 26px;
  box-shadow: 0 30px 70px rgba(16, 42, 98, 0.12);
  border: 1px solid rgba(147, 197, 253, 0.35);
}

.day-view-day-card {
  min-height: 520px;
}

.day-view-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 18px;
  margin-bottom: 22px;
}

.day-view-header button {
  background: rgba(99, 102, 241, 0.12);
  border: 1px solid rgba(99, 102, 241, 0.25);
  color: #334155;
  font-weight: 700;
  border-radius: 14px;
  width: 48px;
  height: 48px;
}

.day-view-title {
  font-size: 1.2rem;
  font-weight: 700;
  color: #0f172a;
}

.day-view-subtitle {
  color: #64748b;
  font-size: 0.95rem;
}

.day-view-day-card {
  border-radius: 28px;
  background: linear-gradient(180deg, #f8fbff 0%, #eef4ff 100%);
  padding: 24px;
  border: 1px solid rgba(99, 102, 241, 0.18);
}

.date-label-large {
  color: #0f172a;
  font-size: 1rem;
  font-weight: 700;
  margin-bottom: 16px;
}

.day-events {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.day-view-panel .event {
  min-height: 260px;
  padding: 16px 16px;
  flex-direction: column;
  align-items: center;
  text-align: center;
}

.day-view-panel .event.compact {
  min-height: 260px;
}

.day-view-panel .event-text {
  gap: 8px;
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}

.day-view-panel .event-text--center {
  width: 100%;
}

.day-view-panel .day-thumb {
  width: 128px;
  height: 128px;
  border-radius: 22px;
  object-fit: cover;
  border: 1px solid rgba(255, 255, 255, 0.85);
  margin: 14px 0;
}

.day-view-panel .title {
  font-size: 1.1rem;
  font-weight: 800;
}

.day-view-panel .description {
  color: rgba(15, 23, 42, 0.85);
}
</style>
