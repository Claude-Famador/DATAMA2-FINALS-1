<script setup>
import { ref, onMounted } from 'vue';
import { supabase } from '../lib/supabaseClient'

const appointments = ref([]);
const dentists = ref([]);
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

const fetchDentists = async () => {
  const { data, error } = await supabase
    .from('DENTIST')
    .select('DENTIST_ID, DENTIST_Name');

  if (!error) {
    dentists.value = data;
  } else {
    alert('Could not load dentists');
  }
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

onMounted(async () => {
  await fetchAppointments();
  await fetchDentists();
});
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
</style>
