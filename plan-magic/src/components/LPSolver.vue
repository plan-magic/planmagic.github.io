<script setup lang="ts">
import { ref, computed, watch } from 'vue';
import type { default as Highs } from 'highs';

interface ProjectLimits {
    min: number;
    max: number;
}

interface Assignment {
    studentName: string;
    projectIndex: number;
    weight: number;
}

const props = defineProps<{
    tableData: any[];
    rawData: any[][];
    headers: string[];
    projectLimits: ProjectLimits[];
}>();

const emit = defineEmits<{
    'solution-updated': [solution: Assignment[]];
}>();

const isSolving = ref(false);
const solution = ref<Assignment[]>([]);
const objectiveValue = ref<number | null>(null);
const solveStatus = ref<string>('');
const errorMessage = ref<string>('');

const canSolve = computed(() => {
    return props.tableData.length > 0 && 
           props.projectLimits.length > 0 &&
           props.projectLimits.length === props.headers.length - 1;
});

async function solve() {
    if (!canSolve.value) {
        errorMessage.value = 'Cannot solve: Missing data or project limits';
        return;
    }

    isSolving.value = true;
    errorMessage.value = '';
    solution.value = [];
    objectiveValue.value = null;
    solveStatus.value = '';

    try {
        // Extract student names and weights
        const students: string[] = [];
        const weights: number[][] = [];
        
        for (let i = 0; i < props.rawData.length; i++) {
            const row = props.rawData[i];
            if (!row || row.length === 0) continue;
            
            // Column 1 is student name, columns 2+ are project weights
            const studentName = String(row[0] || '').trim();
            if (!studentName) continue;
            
            students.push(studentName);
            const studentWeights: number[] = [];
            
            // Extract weights for each project (columns 1 to numberOfProjects)
            for (let j = 1; j < props.headers.length; j++) {
                const weight = parseFloat(row[j]) || 0;
                studentWeights.push(weight);
            }
            
            weights.push(studentWeights);
        }

        if (students.length === 0) {
            throw new Error('No valid students found in the data');
        }

        const numStudents = students.length;
        const numProjects = props.projectLimits.length;

        // Validate that we have weights for all projects
        if (weights.some(w => w.length !== numProjects)) {
            throw new Error('Inconsistent number of project weights');
        }

        // Formulate LP problem in CPLEX format
        let lpProblem = 'Minimize\n';
        lpProblem += ' obj:';
        
        // Objective: minimize sum of weights
        const objectiveTerms: string[] = [];
        for (let i = 0; i < numStudents; i++) {
            for (let j = 0; j < numProjects; j++) {
                const varName = `x_${i}_${j}`;
                const weight = weights[i]?.[j] ?? 0;
                // Format: coefficient variable (CPLEX format uses spaces)
                if (weight !== 0) {
                    objectiveTerms.push(`${weight} ${varName}`);
                }
            }
        }
        lpProblem += ' ' + (objectiveTerms.length > 0 ? objectiveTerms.join(' + ') : '0') + '\n';
        
        lpProblem += 'Subject To\n';
        
        // Constraint 1: Each student assigned to exactly 1 project
        for (let i = 0; i < numStudents; i++) {
            const constraintTerms: string[] = [];
            for (let j = 0; j < numProjects; j++) {
                constraintTerms.push(`x_${i}_${j}`);
            }
            lpProblem += ` student_${i}: ${constraintTerms.join(' + ')} = 1\n`;
        }
        
        // Constraint 2: Project capacity constraints (min and max)
        for (let j = 0; j < numProjects; j++) {
            const constraintTerms: string[] = [];
            for (let i = 0; i < numStudents; i++) {
                constraintTerms.push(`x_${i}_${j}`);
            }
            const projectLimit = props.projectLimits[j];
            if (!projectLimit) continue;
            
            const minLimit = projectLimit.min;
            const maxLimit = projectLimit.max;
            
            if (minLimit > 0) {
                lpProblem += ` project_${j}_min: ${constraintTerms.join(' + ')} >= ${minLimit}\n`;
            }
            if (maxLimit < Infinity) {
                lpProblem += ` project_${j}_max: ${constraintTerms.join(' + ')} <= ${maxLimit}\n`;
            }
        }
        
        lpProblem += 'Bounds\n';
        
        // Binary variables: 0 <= x[i][j] <= 1
        for (let i = 0; i < numStudents; i++) {
            for (let j = 0; j < numProjects; j++) {
                lpProblem += ` 0 <= x_${i}_${j} <= 1\n`;
            }
        }
        
        lpProblem += 'Binary\n';
        for (let i = 0; i < numStudents; i++) {
            for (let j = 0; j < numProjects; j++) {
                lpProblem += ` x_${i}_${j}\n`;
            }
        }
        
        lpProblem += 'End';
        
        console.log('LP Problem:', lpProblem);
        
        // Load and solve with highs
        const highsLoader = await import('highs');
        const highs = await highsLoader.default({
            locateFile: (file: string) => {
                // In browser, load the wasm file from CDN
                if (file.endsWith('.wasm')) {
                    return 'https://lovasoa.github.io/highs-js/' + file;
                }
                return file;
            }
        });
        
        const result = highs.solve(lpProblem, {
            solver: 'choose',
            presolve: 'on'
        });
        
        console.log('LP Solution:', result);
        
        solveStatus.value = result.Status || 'Unknown';
        
        // Check if solution is optimal or feasible (using string comparison to handle type issues)
        const status = result.Status || '';
        const isOptimalOrFeasible = status === 'Optimal' || status.toLowerCase().includes('feasible');
        
        if (isOptimalOrFeasible) {
            objectiveValue.value = result.ObjectiveValue ?? 0;
            
            // Extract assignments from solution
            const assignments: Assignment[] = [];
            for (let i = 0; i < numStudents; i++) {
                const studentName = students[i];
                const studentWeights = weights[i];
                if (!studentName || !studentWeights) continue;
                
                for (let j = 0; j < numProjects; j++) {
                    const varName = `x_${i}_${j}`;
                    const column = result.Columns?.[varName];
                    // Check if column has Primal property (not an infeasible solution column)
                    if (column && 'Primal' in column && column.Primal !== undefined && column.Primal > 0.5) {
                        // Binary variable is 1 (or close to 1)
                        assignments.push({
                            studentName: studentName,
                            projectIndex: j,
                            weight: studentWeights[j] ?? 0
                        });
                    }
                }
            }
            
            solution.value = assignments;
            emit('solution-updated', assignments);
        } else {
            errorMessage.value = `Solver status: ${result.Status}. The problem may be infeasible or unbounded.`;
            emit('solution-updated', []);
        }
        
    } catch (error) {
        console.error('Error solving LP:', error);
        errorMessage.value = error instanceof Error ? error.message : 'Unknown error occurred';
    } finally {
        isSolving.value = false;
    }
}

// Watch for changes in data or limits and clear solution
watch([() => props.tableData, () => props.projectLimits], () => {
    solution.value = [];
    objectiveValue.value = null;
    solveStatus.value = '';
    errorMessage.value = '';
    emit('solution-updated', []);
}, { deep: true });
</script>

<template>
    <div v-if="canSolve || solution.length > 0" style="margin-top: 30px;">
        <h2 style="margin-bottom: 15px; font-size: 18px; font-weight: 600;">Assignment Solver</h2>
        
        <button 
            @click="solve"
            :disabled="!canSolve || isSolving"
            style="
                padding: 12px 24px;
                background: #4a90e2;
                color: white;
                border: none;
                border-radius: 6px;
                font-size: 16px;
                font-weight: 600;
                cursor: pointer;
                transition: background 0.2s;
            "
            :style="(!canSolve || isSolving) ? 'background: #ccc; cursor: not-allowed;' : ''"
            @mouseover="!canSolve || isSolving ? '' : 'this.style.background = \'#357abd\''"
            @mouseout="!canSolve || isSolving ? '' : 'this.style.background = \'#4a90e2\''">
            {{ isSolving ? 'Solving...' : 'Solve Assignment' }}
        </button>
        
        <div v-if="errorMessage" style="margin-top: 15px; padding: 12px; background: #fee; border: 1px solid #fcc; border-radius: 6px; color: #c33;">
            <strong>Error:</strong> {{ errorMessage }}
        </div>
        
        <div v-if="solveStatus && !errorMessage" style="margin-top: 15px; padding: 12px; background: #efe; border: 1px solid #cfc; border-radius: 6px; color: #3c3;">
            <strong>Status:</strong> {{ solveStatus }}
            <span v-if="objectiveValue !== null" style="margin-left: 15px;">
                <strong>Total Weight:</strong> {{ objectiveValue.toFixed(2) }}
            </span>
        </div>
        
        <div v-if="solution.length > 0" style="margin-top: 20px;">
            <h3 style="margin-bottom: 15px; font-size: 16px; font-weight: 600;">Assignments</h3>
            
            <div style="overflow-x: auto; border: 2px solid #333; border-radius: 6px; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
                <table style="width: 100%; border-collapse: collapse; background: white;">
                    <thead style="background: #2c3e50;">
                        <tr>
                            <th style="padding: 12px 16px; text-align: left; border-bottom: 2px solid #1a252f; border-right: 1px solid #34495e; font-weight: 600; color: #ffffff; font-size: 15px;">
                                Student
                            </th>
                            <th style="padding: 12px 16px; text-align: center; border-bottom: 2px solid #1a252f; border-right: 1px solid #34495e; font-weight: 600; color: #ffffff; font-size: 15px;">
                                Project
                            </th>
                            <th style="padding: 12px 16px; text-align: center; border-bottom: 2px solid #1a252f; font-weight: 600; color: #ffffff; font-size: 15px;">
                                Weight
                            </th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(assignment, index) in solution" :key="index">
                            <td style="padding: 12px 16px; border-bottom: 1px solid #e0e0e0; border-right: 1px solid #e0e0e0; color: #212121; font-size: 15px;">
                                {{ assignment.studentName }}
                            </td>
                            <td style="padding: 12px 16px; border-bottom: 1px solid #e0e0e0; border-right: 1px solid #e0e0e0; text-align: center; color: #212121; font-size: 15px; font-weight: 600;">
                                Project {{ assignment.projectIndex + 1 }}
                            </td>
                            <td style="padding: 12px 16px; border-bottom: 1px solid #e0e0e0; text-align: center; color: #212121; font-size: 15px;">
                                {{ assignment.weight.toFixed(2) }}
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>
            
            <!-- Summary by project -->
            <div style="margin-top: 20px;">
                <h3 style="margin-bottom: 15px; font-size: 16px; font-weight: 600;">Summary by Project</h3>
                <div style="overflow-x: auto; border: 2px solid #333; border-radius: 6px; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
                    <table style="width: 100%; border-collapse: collapse; background: white;">
                        <thead style="background: #2c3e50;">
                            <tr>
                                <th style="padding: 12px 16px; text-align: left; border-bottom: 2px solid #1a252f; border-right: 1px solid #34495e; font-weight: 600; color: #ffffff; font-size: 15px;">
                                    Project
                                </th>
                                <th style="padding: 12px 16px; text-align: center; border-bottom: 2px solid #1a252f; border-right: 1px solid #34495e; font-weight: 600; color: #ffffff; font-size: 15px;">
                                    Students Assigned
                                </th>
                                <th style="padding: 12px 16px; text-align: center; border-bottom: 2px solid #1a252f; border-right: 1px solid #34495e; font-weight: 600; color: #ffffff; font-size: 15px;">
                                    Min Limit
                                </th>
                                <th style="padding: 12px 16px; text-align: center; border-bottom: 2px solid #1a252f; border-right: 1px solid #34495e; font-weight: 600; color: #ffffff; font-size: 15px;">
                                    Max Limit
                                </th>
                                <th style="padding: 12px 16px; text-align: center; border-bottom: 2px solid #1a252f; font-weight: 600; color: #ffffff; font-size: 15px;">
                                    Total Weight
                                </th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr v-for="(limit, projectIndex) in projectLimits" :key="projectIndex">
                                <td style="padding: 12px 16px; border-bottom: 1px solid #e0e0e0; border-right: 1px solid #e0e0e0; color: #212121; font-size: 15px; font-weight: 600;">
                                    Project {{ projectIndex + 1 }}
                                </td>
                                <td style="padding: 12px 16px; border-bottom: 1px solid #e0e0e0; border-right: 1px solid #e0e0e0; text-align: center; color: #212121; font-size: 15px;">
                                    {{ solution.filter(a => a.projectIndex === projectIndex).length }}
                                </td>
                                <td style="padding: 12px 16px; border-bottom: 1px solid #e0e0e0; border-right: 1px solid #e0e0e0; text-align: center; color: #212121; font-size: 15px;">
                                    {{ limit.min }}
                                </td>
                                <td style="padding: 12px 16px; border-bottom: 1px solid #e0e0e0; border-right: 1px solid #e0e0e0; text-align: center; color: #212121; font-size: 15px;">
                                    {{ limit.max }}
                                </td>
                                <td style="padding: 12px 16px; border-bottom: 1px solid #e0e0e0; text-align: center; color: #212121; font-size: 15px;">
                                    {{ solution.filter(a => a.projectIndex === projectIndex).reduce((sum, a) => sum + a.weight, 0).toFixed(2) }}
                                </td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped>
tbody tr:hover {
    background-color: #f5f7fa;
}

tbody tr:nth-child(even) {
    background-color: #ffffff;
}

tbody tr:nth-child(odd) {
    background-color: #fafbfc;
}
</style>