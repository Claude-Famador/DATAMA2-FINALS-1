<script setup>
import { ref, onMounted } from 'vue';
import { supabase } from '../lib/supabaseClient'

const appointments = ref([]);
const userType = ref('customer');
const loading = ref(false);
const newAppointment = ref({ 
  APPT_Date: '', 
  APPT_Client_Name: '', 
  APPT_Dentist_Name: '', 
  APPT_Type: '',
  APPT_Status: 'Pending'
});

const fetchAppointments = async () => {
  loading.value = true;
  const { data, error } = await supabase
    .from('APPOINTMENT')
    .select(`
      *,
      DENTIST (DENTIST_Name),
      CLIENT (CLIENT_Name)
    `)
    .order('APPT_Date', { ascending: true });
    
  if (!error) {
    appointments.value = data;
  } else {
    alert('Could not load appointments');
  }
  loading.value = false;
};

const bookAppointment = async () => {
  if (!validateAppointment()) return;
  
  loading.value = true;
  const { error } = await supabase
    .from('APPOINTMENT')
    .insert([{ 
      ...newAppointment.value,
      APPT_Status: 'Pending',
      CREATED_Date: new Date().toISOString()
    }]);

  if (!error) {
    alert('Appointment booked successfully!');
    clearForm();
    fetchAppointments();
  } else {
    alert('Failed to book appointment');
  }
  loading.value = false;
};

const validateAppointment = () => {
  const required = ['APPT_Date', 'APPT_Client_Name', 'APPT_Dentist_Name', 'APPT_Type'];
  const missing = required.filter(field => !newAppointment.value[field]);
  
  if (missing.length > 0) {
    alert('Please fill in all required fields');
    return false;
  }
  return true;
};

const clearForm = () => {
  newAppointment.value = { 
    APPT_Date: '', 
    APPT_Client_Name: '', 
    APPT_Dentist_Name: '', 
    APPT_Type: '',
    APPT_Status: 'Pending'
  };
};

const updateStatus = async (id, status) => {
  loading.value = true;
  const { error } = await supabase
    .from('APPOINTMENT')
    .update({ 
      APPT_Status: status,
      UPDATED_Date: new Date().toISOString()
    })
    .match({ APPT_ID: id });

  if (!error) {
    fetchAppointments();
  } else {
    alert('Failed to update appointment status');
  }
  loading.value = false;
};

onMounted(fetchAppointments);
</script>

<template>
  <div class="dashboard">
    <div class="dashboard-container">
      <div class="dashboard-header">
        <h1>Dental Appointment System</h1>
        <select v-model="userType" class="border p-2">
          <option value="customer">Customer</option>
          <option value="staff">Staff</option>
        </select>
      </div>

      <div v-if="userType === 'customer'" class="border p-4 bg-white rounded-lg shadow">
        <h2 class="text-xl font-semibold mb-2">Book an Appointment</h2>
        <input v-model="newAppointment.APPT_Date" type="date" class="border p-2 mb-2 w-full" />
        <input v-model="newAppointment.APPT_Client_Name" placeholder="Your Name" class="border p-2 mb-2 w-full" />
        <select v-model="newAppointment.APPT_Dentist_Name" class="border p-2 mb-2 w-full">
          <option value="">Select Dentist</option>
          <option v-for="dentist in dentists" :key="dentist.DENTIST_ID" :value="dentist.DENTIST_Name">
            {{ dentist.DENTIST_Name }}
          </option>
        </select>
        <select v-model="newAppointment.APPT_Type" class="border p-2 mb-2 w-full">
          <option value="">Select Service Type</option>
          <option value="Cleaning">Cleaning</option>
          <option value="Checkup">Checkup</option>
          <option value="Treatment">Treatment</option>
          <option value="Surgery">Surgery</option>
        </select>
        <button 
          @click="bookAppointment" 
          class="bg-blue-500 text-white p-2 rounded w-full"
          :disabled="loading"
        >
          {{ loading ? 'Booking...' : 'Book' }}
        </button>
      </div>

      <div v-if="userType === 'staff'" class="border p-4 mt-4 bg-white rounded-lg shadow">
        <h2 class="text-xl font-semibold mb-2">Manage Appointments</h2>
        <div class="mb-4">
          <select v-model="filterStatus" class="border p-2 mr-2">
            <option value="">All Status</option>
            <option value="Pending">Pending</option>
            <option value="Confirmed">Confirmed</option>
            <option value="Cancelled">Cancelled</option>
          </select>
          <input type="date" v-model="filterDate" class="border p-2" />
        </div>
        <table class="w-full border-collapse">
          <thead>
            <tr class="bg-gray-200">
              <th class="p-2">Date</th>
              <th class="p-2">Time</th>
              <th class="p-2">Client</th>
              <th class="p-2">Dentist</th>
              <th class="p-2">Type</th>
              <th class="p-2">Status</th>
              <th class="p-2">Action</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="appt in filteredAppointments" :key="appt.APPT_ID" class="border-b">
              <td class="p-2">{{ formatDate(appt.APPT_Date) }}</td>
              <td class="p-2">{{ formatTime(appt.APPT_Date) }}</td>
              <td class="p-2">{{ appt.CLIENT?.CLIENT_Name || appt.APPT_Client_Name }}</td>
              <td class="p-2">{{ appt.DENTIST?.DENTIST_Name || appt.APPT_Dentist_Name }}</td>
              <td class="p-2">{{ appt.APPT_Type }}</td>
              <td class="p-2">
                <span :class="getStatusClass(appt.APPT_Status)">
                  {{ appt.APPT_Status }}
                </span>
              </td>
              <td class="p-2 space-x-2">
                <button 
                  v-if="appt.APPT_Status === 'Pending'"
                  @click="updateStatus(appt.APPT_ID, 'Confirmed')" 
                  class="bg-green-500 text-white p-1 rounded"
                  :disabled="loading"
                >
                  {{ loading ? '...' : 'Confirm' }}
                </button>
                <button 
                  v-if="appt.APPT_Status === 'Pending'"
                  @click="updateStatus(appt.APPT_ID, 'Cancelled')" 
                  class="bg-red-500 text-white p-1 rounded"
                  :disabled="loading"
                >
                  Cancel
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<style scoped>
.dashboard {
  min-height: 100vh;
  width: 100%;
  background: linear-gradient(135deg, #f5f7fa 0%, #c3cfe2 100%);
  padding: 2rem;
}

.dashboard-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
}

.dashboard-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
}

.dashboard-header h1 {
  color: #2c3e50;
  font-size: 1.8rem;
  font-weight: 600;
}

.dashboard-header select {
  padding: 0.5rem 1rem;
  border-radius: 4px;
  border: 1px solid #e2e8f0;
  background-color: white;
  cursor: pointer;
}

input {
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  padding: 0.75rem;
  margin-bottom: 1rem;
  width: 100%;
  font-size: 1rem;
}

button {
  background-color: #3b82f6;
  color: white;
  padding: 0.75rem 1.5rem;
  border-radius: 4px;
  border: none;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.2s ease;
}

button:hover {
  background-color: #2563eb;
  transform: translateY(-1px);
}

.border {
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}
</style>
button {
  background-color: #3b82f6;
  color: white;
  padding: 0.75rem 1.5rem;
  border-radius: 4px;
  border: none;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.2s ease;
}

button:hover {
  background-color: #2563eb;
  transform: translateY(-1px);
}

.border {
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}
</style>
```