<script setup lang="ts">
import { ref, watch } from 'vue';

interface ProjectLimits {
    min: number;
    max: number;
}

const props = defineProps<{
    numberOfProjects: number;
}>();

const emit = defineEmits<{
    'update:projectLimits': [limits: ProjectLimits[]];
}>();

const projectLimits = ref<ProjectLimits[]>([]);

// Initialize project limits when number of projects changes
watch(() => props.numberOfProjects, (newCount) => {
    if (newCount > 0) {
        // Initialize with default values
        projectLimits.value = Array.from({ length: newCount }, () => ({
            min: 0,
            max: 10
        }));
        emit('update:projectLimits', projectLimits.value);
    } else {
        projectLimits.value = [];
        emit('update:projectLimits', []);
    }
}, { immediate: true });

function updateMin(projectIndex: number, value: number) {
    if (projectLimits.value[projectIndex]) {
        projectLimits.value[projectIndex].min = Math.max(0, value);
        // Ensure min doesn't exceed max
        if (projectLimits.value[projectIndex].min > projectLimits.value[projectIndex].max) {
            projectLimits.value[projectIndex].max = projectLimits.value[projectIndex].min;
        }
        emit('update:projectLimits', projectLimits.value);
    }
}

function updateMax(projectIndex: number, value: number) {
    if (projectLimits.value[projectIndex]) {
        projectLimits.value[projectIndex].max = Math.max(0, value);
        // Ensure max is not less than min
        if (projectLimits.value[projectIndex].max < projectLimits.value[projectIndex].min) {
            projectLimits.value[projectIndex].min = projectLimits.value[projectIndex].max;
        }
        emit('update:projectLimits', projectLimits.value);
    }
}

function decrementMin(projectIndex: number) {
    const current = projectLimits.value[projectIndex]?.min || 0;
    updateMin(projectIndex, current - 1);
}

function incrementMin(projectIndex: number) {
    const current = projectLimits.value[projectIndex]?.min || 0;
    updateMin(projectIndex, current + 1);
}

function decrementMax(projectIndex: number) {
    const current = projectLimits.value[projectIndex]?.max || 0;
    updateMax(projectIndex, current - 1);
}

function incrementMax(projectIndex: number) {
    const current = projectLimits.value[projectIndex]?.max || 0;
    updateMax(projectIndex, current + 1);
}

function handleMinInput(event: Event, projectIndex: number) {
    const target = event.target as HTMLInputElement;
    const value = parseInt(target.value) || 0;
    updateMin(projectIndex, value);
}

function handleMaxInput(event: Event, projectIndex: number) {
    const target = event.target as HTMLInputElement;
    const value = parseInt(target.value) || 0;
    updateMax(projectIndex, value);
}
</script>

<template>
    <div v-if="numberOfProjects > 0" style="margin-top: 30px;">
        <h2 style="margin-bottom: 15px; font-size: 18px; font-weight: 600;">Project Configuration</h2>
        
        <div style="overflow-x: auto; border: 2px solid #333; border-radius: 6px; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
            <table style="width: 100%; border-collapse: collapse; background: white; min-width: 250px;">
                <thead style="background: #2c3e50;">
                    <tr>
                        <th style="padding: 6px 8px; text-align: left; border-right: 1px solid #34495e; font-weight: 600; color: #ffffff; font-size: 12px; position: sticky; left: 0; background: #2c3e50; z-index: 5;">
                            Project
                        </th>
                        <th 
                            v-for="(project, index) in projectLimits" 
                            :key="index"
                            style="padding: 6px 8px; text-align: center; border-right: 1px solid #34495e; font-weight: 600; color: #ffffff; font-size: 12px; min-width: 100px;">
                            Project {{ index + 1 }}
                        </th>
                    </tr>
                </thead>
                <tbody>
                    <!-- Minimum Students Row -->
                    <tr>
                        <td style="padding: 4px 8px; border-bottom: 1px solid #e0e0e0; border-right: 1px solid #e0e0e0; font-weight: 600; color: #2c3e50; background: #f5f7fa; position: sticky; left: 0; z-index: 3; font-size: 12px;">
                            Min
                        </td>
                        <td 
                            v-for="(project, index) in projectLimits" 
                            :key="`min-${index}`"
                            style="padding: 4px 6px; border-bottom: 1px solid #e0e0e0; border-right: 1px solid #e0e0e0; background: #ffffff;">
                            <div style="display: flex; align-items: center; justify-content: center; gap: 3px;">
                                <button 
                                    @click="decrementMin(index)"
                                    style="width: 24px; height: 24px; border: 1px solid #ccc; background: white; border-radius: 2px; cursor: pointer; font-size: 14px; font-weight: bold; display: flex; align-items: center; justify-content: center; user-select: none;"
                                    @mousedown.prevent>
                                    −
                                </button>
                                <input 
                                    type="number"
                                    :value="project.min"
                                    @input="handleMinInput($event, index)"
                                    @blur="handleMinInput($event, index)"
                                    min="0"
                                    class="number-input"
                                    style="width: 40px; height: 24px; padding: 0 4px; border: 1px solid #ccc; border-radius: 2px; font-size: 12px; text-align: center;"
                                />
                                <button 
                                    @click="incrementMin(index)"
                                    style="width: 24px; height: 24px; border: 1px solid #ccc; background: white; border-radius: 2px; cursor: pointer; font-size: 14px; font-weight: bold; display: flex; align-items: center; justify-content: center; user-select: none;"
                                    @mousedown.prevent>
                                    +
                                </button>
                            </div>
                        </td>
                    </tr>
                    <!-- Maximum Students Row -->
                    <tr>
                        <td style="padding: 4px 8px; border-bottom: 1px solid #e0e0e0; border-right: 1px solid #e0e0e0; font-weight: 600; color: #2c3e50; background: #f5f7fa; position: sticky; left: 0; z-index: 3; font-size: 12px;">
                            Max
                        </td>
                        <td 
                            v-for="(project, index) in projectLimits" 
                            :key="`max-${index}`"
                            style="padding: 4px 6px; border-bottom: 1px solid #e0e0e0; border-right: 1px solid #e0e0e0; background: #fafbfc;">
                            <div style="display: flex; align-items: center; justify-content: center; gap: 3px;">
                                <button 
                                    @click="decrementMax(index)"
                                    style="width: 24px; height: 24px; border: 1px solid #ccc; background: white; border-radius: 2px; cursor: pointer; font-size: 14px; font-weight: bold; display: flex; align-items: center; justify-content: center; user-select: none;"
                                    @mousedown.prevent>
                                    −
                                </button>
                                <input 
                                    type="number"
                                    :value="project.max"
                                    @input="handleMaxInput($event, index)"
                                    @blur="handleMaxInput($event, index)"
                                    min="0"
                                    class="number-input"
                                    style="width: 40px; height: 24px; padding: 0 4px; border: 1px solid #ccc; border-radius: 2px; font-size: 12px; text-align: center;"
                                />
                                <button 
                                    @click="incrementMax(index)"
                                    style="width: 24px; height: 24px; border: 1px solid #ccc; background: white; border-radius: 2px; cursor: pointer; font-size: 14px; font-weight: bold; display: flex; align-items: center; justify-content: center; user-select: none;"
                                    @mousedown.prevent>
                                    +
                                </button>
                            </div>
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>
    </div>
</template>

<style scoped>
button:hover {
    background-color: #f0f0f0 !important;
}

button:active {
    background-color: #e0e0e0 !important;
}

input:focus {
    outline: 2px solid #4a90e2;
    outline-offset: -2px;
    border-color: #4a90e2;
}

/* Hide number input spinners */
.number-input::-webkit-outer-spin-button,
.number-input::-webkit-inner-spin-button {
    -webkit-appearance: none;
    margin: 0;
}

.number-input[type=number] {
    -moz-appearance: textfield;
}
</style>
