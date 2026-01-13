<script setup lang="ts">
import { ref, computed } from 'vue';
import * as XLSX from 'xlsx';

interface ExcelRow {
    [key: string]: any;
}

interface Assignment {
    studentName: string;
    projectIndex: number;
    weight: number;
}

const props = defineProps<{
    solution?: Assignment[];
}>();

const emit = defineEmits<{
    'columns-changed': [count: number];
}>();

const fileName = ref<string>('');
const tableData = ref<ExcelRow[]>([]);
const headers = ref<string[]>([]);
const rawData = ref<any[][]>([]); // 2D array for raw access
const numberOfColumns = ref<number>(0);

// Function to check if a cell should be highlighted
function isCellHighlighted(rowIndex: number, colIndex: number): boolean {
    if (!props.solution || props.solution.length === 0) {
        return false;
    }
    
    // Column 0 is student name, columns 1+ are projects
    if (colIndex === 0) {
        return false; // Don't highlight student name column
    }
    
    const row = rawData.value[rowIndex];
    if (!row || row.length === 0) {
        return false;
    }
    
    // Get student name from first column and trim for comparison
    const studentName = String(row[0] || '').trim();
    if (!studentName) {
        return false;
    }
    
    // Column 1 = project 0, Column 2 = project 1, etc.
    const projectIndex = colIndex - 1;
    
    // Check if this cell is part of the solution
    // Compare trimmed student names to handle whitespace differences
    return props.solution.some(assignment => {
        const assignmentStudentName = String(assignment.studentName || '').trim();
        return assignmentStudentName === studentName && 
               assignment.projectIndex === projectIndex;
    });
}

function handleFileChange(event: Event) {
    const target = event.target as HTMLInputElement;
    const file = target.files?.[0];
    
    if (!file) {
        return;
    }
    
    // Validate that the file is an Excel file
    const validExtensions = ['.xlsx', '.xls'];
    const fileExtension = file.name.toLowerCase().substring(file.name.lastIndexOf('.'));
    if (!validExtensions.includes(fileExtension)) {
        alert('Please select an Excel file (.xlsx or .xls)');
        target.value = ''; // Reset the input
        return;
    }
    
    fileName.value = file.name;
    
    const reader = new FileReader();
    
    reader.onload = (e) => {
        try {
            const data = e.target?.result;
            if (!data) {
                throw new Error('No data read from file');
            }
            
            const workbook = XLSX.read(data, { type: 'binary' });
            
            // Get the first sheet
            if (workbook.SheetNames.length === 0) {
                throw new Error('Excel file has no sheets');
            }
            
            const firstSheetName = workbook.SheetNames[0];
            if (!firstSheetName) {
                throw new Error('Could not get sheet name');
            }
            
            const worksheet = workbook.Sheets[firstSheetName];
            
            if (!worksheet) {
                throw new Error('Could not read worksheet');
            }
            
            // Convert to 2D array for raw access
            const arrayData = XLSX.utils.sheet_to_json(worksheet, { header: 1, defval: '' }) as any[][];
            
            // Find the maximum number of columns across all rows
            const maxColumns = arrayData.length > 0 
                ? Math.max(...arrayData.map(row => row ? row.length : 0))
                : 0;
            
            // Generate default headers (Column 1, Column 2, etc.)
            headers.value = [];
            for (let i = 0; i < maxColumns; i++) {
                headers.value.push(`Column ${i + 1}`);
            }
            
            // Convert ALL rows (including first row) to objects using headers
            const rows: ExcelRow[] = [];
            for (let i = 0; i < arrayData.length; i++) {
                const row: ExcelRow = {};
                const rowData = arrayData[i] || [];
                headers.value.forEach((header, colIndex) => {
                    row[header] = rowData[colIndex] ?? '';
                });
                rows.push(row);
            }
            
            // Store data
            tableData.value = rows;
            rawData.value = arrayData;
            numberOfColumns.value = maxColumns;
            emit('columns-changed', maxColumns);
            
            // Note: Solution will be cleared by App.vue when tableData changes
            
            console.log('Excel file loaded:', fileName.value);
            console.log('Headers:', headers.value);
            console.log('Table data:', tableData.value);
            console.log('Raw data (2D array):', rawData.value);
            console.log('Number of columns:', numberOfColumns.value);
        } catch (error) {
            console.error('Error parsing Excel file:', error);
            alert('Error reading Excel file. Please make sure it is a valid Excel file.');
            target.value = '';
            fileName.value = '';
            tableData.value = [];
            headers.value = [];
            rawData.value = [];
            numberOfColumns.value = 0;
            emit('columns-changed', 0);
        }
    };
    
    reader.onerror = (error) => {
        console.error('Error reading file:', error);
        alert('Error reading file. Please try again.');
    };
    
    // Read as binary string for Excel files
    reader.readAsBinaryString(file);
}

// Expose data for parent component access
defineExpose({
    fileName,
    tableData,
    headers,
    rawData,
    numberOfColumns
});
</script>

<template>
    <div>
        <input 
            type="file"
            id="file-upload" 
            accept=".xlsx,.xls,application/vnd.openxmlformats-officedocument.spreadsheetml.sheet,application/vnd.ms-excel"
            @change="handleFileChange" />
        
        <div v-if="fileName" style="margin-top: 10px;">
            <p><strong>File:</strong> {{ fileName }}</p>
            <p v-if="tableData.length > 0"><strong>Rows:</strong> {{ tableData.length }}</p>
        </div>
        
        <div v-if="tableData.length > 0" style="margin-top: 20px;">
            <div style="overflow-x: auto; max-height: 500px; overflow-y: auto; border: 2px solid #333; border-radius: 6px; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">
                <table style="width: 100%; border-collapse: collapse; background: white;">
                    <thead style="position: sticky; top: 0; background: #2c3e50; z-index: 10;">
                        <tr>
                            <th 
                                v-for="(header, index) in headers" 
                                :key="index"
                                style="padding: 12px 16px; text-align: left; border-bottom: 2px solid #1a252f; border-right: 1px solid #34495e; font-weight: 600; color: #ffffff; font-size: 15px;">
                                {{ header }}
                            </th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(row, rowIndex) in tableData" :key="rowIndex">
                            <td 
                                v-for="(header, colIndex) in headers" 
                                :key="colIndex"
                                :style="{
                                    padding: '12px 16px',
                                    borderBottom: '1px solid #e0e0e0',
                                    borderRight: '1px solid #e0e0e0',
                                    color: '#212121',
                                    fontSize: '15px',
                                    lineHeight: '1.5',
                                    backgroundColor: isCellHighlighted(rowIndex, colIndex) ? '#d4edda' : 'transparent',
                                    fontWeight: isCellHighlighted(rowIndex, colIndex) ? '600' : 'normal',
                                    border: isCellHighlighted(rowIndex, colIndex) ? '2px solid #28a745' : 'none',
                                    boxShadow: isCellHighlighted(rowIndex, colIndex) ? '0 0 0 1px rgba(40, 167, 69, 0.2)' : 'none'
                                }">
                                {{ row[header] ?? '' }}
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</template>

<style scoped>
table {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
}

tbody tr:hover {
    background-color: #f5f7fa;
}

tbody tr:nth-child(even) {
    background-color: #ffffff;
}

tbody tr:nth-child(odd) {
    background-color: #fafbfc;
}

tbody tr:nth-child(even):hover {
    background-color: #eef2f5;
}

tbody tr:nth-child(odd):hover {
    background-color: #eef2f5;
}
</style>