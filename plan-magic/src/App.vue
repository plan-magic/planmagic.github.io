<script setup lang="ts">
import { ref, computed } from 'vue';
import FileUpload from "./components/FileUpload.vue";
import ProjectConfig from "./components/ProjectConfig.vue";
import LPSolver from "./components/LPSolver.vue";

interface ProjectLimits {
    min: number;
    max: number;
}

interface Assignment {
    studentName: string;
    projectIndex: number;
    weight: number;
}

const fileUploadRef = ref<InstanceType<typeof FileUpload> | null>(null);
const projectLimits = ref<ProjectLimits[]>([]);
const numberOfColumns = ref<number>(0);
const solution = ref<Assignment[]>([]);

function handleColumnsChanged(count: number) {
    numberOfColumns.value = count;
}

const numberOfProjects = computed(() => {
    return numberOfColumns.value > 0 ? numberOfColumns.value - 1 : 0;
});

function handleProjectLimitsUpdate(limits: ProjectLimits[]) {
    projectLimits.value = limits;
    console.log('Project limits updated:', projectLimits.value);
}

// Computed properties to access FileUpload data
const tableData = computed(() => fileUploadRef.value?.tableData || []);
const rawData = computed(() => fileUploadRef.value?.rawData || []);
const headers = computed(() => fileUploadRef.value?.headers || []);
</script>

<template>
  <div style="padding: 20px; max-width: 1200px; margin: 0 auto;">
    <h1 style="margin-bottom: 30px;">Plan Magic</h1>
    <FileUpload 
      ref="fileUploadRef" 
      @columns-changed="handleColumnsChanged"
      :solution="solution"
    />
    <ProjectConfig 
      :number-of-projects="numberOfProjects"
      @update:project-limits="handleProjectLimitsUpdate"
    />
    <LPSolver
      :table-data="tableData"
      :raw-data="rawData"
      :headers="headers"
      :project-limits="projectLimits"
      @solution-updated="solution = $event"
    />
  </div>
</template>

<style scoped>

</style>
