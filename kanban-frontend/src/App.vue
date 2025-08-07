<template>
  <div class="kanban-board">
    <h1>📝 Kanban</h1>

    <form @submit.prevent="addTask">
      <input
        v-model="newTask.title"
        type="text"
        placeholder="Titre de la tâche"
        required
      />
      <input
        v-model="newTask.description"
        type="text"
        placeholder="Description (optionnelle)"
      />
      <button type="submit">Ajouter</button>
    </form>

    <div class="columns">
      <div class="column" v-for="col in columns" :key="col">
        <h2 class="h2">{{ col }}</h2>
        <draggable
          :list="tasksByColumn(col)"
          group="tasks"
          @end="onDragEnd"
          item-key="id"
          class="task-list"
        >
          <template #item="{ element }">
            <div class="task">
              <!-- Mode édition -->
              <div v-if="editTaskId === element.id" class="edit-form">
                <input v-model="editTitle" placeholder="Titre" />
                <input v-model="editDescription" placeholder="Description" />
                <button @click="saveEdit(element)">💾</button>
                <button @click="cancelEdit">✖</button>
              </div>

              <!-- Mode affichage -->
              <div v-else>
                <strong>{{ element.title }}</strong>
                <p class="description">{{ element.description }}</p>
              </div>

              <!-- Boutons -->
              <div class="task-buttons">
                <button v-if="editTaskId !== element.id" @click="startEdit(element)" title="Modifier">✎</button>
                <button @click="deleteTask(element)" class="delete-btn" title="Supprimer la tâche">✖</button>
              </div>
            </div>
          </template>
        </draggable>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import axios from "axios";
import draggable from "vuedraggable";

const columns = ["To Do", "In Progress", "Done"];
const tasks = ref([]);
const newTask = ref({ title: "", description: "", columnName: "To Do" });

const editTaskId = ref(null);
const editTitle = ref("");
const editDescription = ref("");

onMounted(async () => {
  const res = await axios.get("http://localhost:8080/api/tasks");
  tasks.value = res.data;
});

async function addTask() {
  if (!newTask.value.title) return;

  try {
    const res = await axios.post("http://localhost:8080/api/tasks", newTask.value);
    tasks.value.push(res.data);
    newTask.value = { title: "", description: "", columnName: "To Do" };
  } catch {
    alert("Erreur lors de l’ajout de la tâche");
  }
}

function tasksByColumn(col) {
  return tasks.value.filter((t) => t.columnName === col);
}

async function deleteTask(task) {
  if (!confirm(`Supprimer la tâche "${task.title}" ?`)) return;

  try {
    await axios.delete(`http://localhost:8080/api/tasks/${task.id}`);
    tasks.value = tasks.value.filter((t) => t.id !== task.id);
  } catch {
    alert("Erreur lors de la suppression de la tâche");
  }
}

function startEdit(task) {
  editTaskId.value = task.id;
  editTitle.value = task.title;
  editDescription.value = task.description || "";
}

function cancelEdit() {
  editTaskId.value = null;
  editTitle.value = "";
  editDescription.value = "";
}

async function saveEdit(task) {
  if (!editTitle.value.trim()) {
    alert("Le titre ne peut pas être vide.");
    return;
  }

  const updatedTask = {
    ...task,
    title: editTitle.value,
    description: editDescription.value,
  };

  try {
    await axios.put(`http://localhost:8080/api/tasks/${task.id}`, updatedTask);

    // Mettre à jour localement
    const index = tasks.value.findIndex((t) => t.id === task.id);
    if (index !== -1) {
      tasks.value[index] = updatedTask;
    }

    cancelEdit();
  } catch {
    alert("Erreur lors de la mise à jour de la tâche");
  }
}

async function onDragEnd(event) {
  const movedTask = event.item._underlying_vm_;
  const newColumn = event.to.parentElement.querySelector("h2").textContent;

  if (movedTask.columnName !== newColumn) {
    movedTask.columnName = newColumn;
    try {
      await axios.put(`http://localhost:8080/api/tasks/${movedTask.id}`, movedTask);
    } catch {
      alert("Erreur lors de la mise à jour de la tâche");
    }
  }
}
</script>

<style scoped>
/* Styles précédents conservés */
.kanban-board {
  max-width: 1000px;
  margin: auto;
  font-family: "Segoe UI", sans-serif;
  padding: 20px;
}

h1 {
  text-align: center;
  color: #444;
}

form {
  display: flex;
  justify-content: center;
  margin-bottom: 20px;
  gap: 10px;
  flex-wrap: wrap;
}

input {
  padding: 10px;
  width: 45%;
  border-radius: 6px;
  border: 1px solid #ccc;
  font-size: 16px;
}

button {
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: bold;
  cursor: pointer;
}

button:hover {
  background-color: #0056b3;
}

.columns {
  display: flex;
  justify-content: space-between;
  gap: 20px;
}

.column {
  flex: 1;
  background-color: #f8f9fa;
  border-radius: 10px;
  padding: 15px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.05);
  min-height: 300px;
}

h2 {
  text-align: center;
  color: #555;
}

.task-list {
  min-height: 200px;
}

.task {
  background-color: #4db664;
  margin: 10px 0;
  padding: 12px 14px;
  border-radius: 8px;
  border-left: 6px solid #007bff;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  position: relative;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.task .description {
  font-size: 13px;
  margin-top: 4px;
  color: #fff;
  opacity: 0.8;
}

.task-buttons {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.task-buttons button {
  background: transparent;
  border: none;
  color: white;
  font-size: 16px;
  cursor: pointer;
  padding: 0;
}

.task-buttons button:hover {
  color: #b02a37;
}

.delete-btn {
  color: #e63946;
}

.delete-btn:hover {
  color: #b02a37;
}

.edit-form input {
  margin-bottom: 6px;
  padding: 6px 8px;
  width: 200px;
  border-radius: 4px;
  border: 1px solid #ccc;
  font-size: 14px;
}

.edit-form button {
  margin-right: 6px;
}
</style>
