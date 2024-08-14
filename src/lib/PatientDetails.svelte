<script lang="ts">
    import axios from 'axios';
    import { onMount } from 'svelte';
    import { FHIR_BASE_URL } from '../config';
    import type { Patient } from 'fhir/r4';

    export let accessToken: string;
    export let patientId: string;

    const getPatientDetails = async () => {
      try {
        const patientResponse = await axios.get<Patient>(`${FHIR_BASE_URL}/Patient/${patientId}`, {
          headers: {
            'Authorization': `Bearer ${accessToken}`
          }
        });
        return patientResponse.data;
      } catch (error) {
        console.error('Error fetching patient details:', error);
        throw error;
      }
    };
</script>

<div class="mt-10 mx-10 p-6 border rounded-lg shadow-md bg-white max-w-4xl">
  {#await getPatientDetails() then patient}
    <div class="mb-4">
      <h1 class="text-3xl font-bold mb-2">Hello, {patient?.name?.[0]?.given?.[0]}</h1>
      <p class="text-lg text-gray-700">Patient Overview</p>
    </div>
    <div class="space-y-2 text-left">
      <p><span class="font-semibold">Full Name:</span> {patient?.name?.[0]?.text}</p>
      <p><span class="font-semibold">Epic Identifier:</span> {patient?.identifier?.find(i => i.system === 'urn:oid:1.2.840.114350.1.13.0.1.7.5.737384.0')?.value}</p>
      <p><span class="font-semibold">Date of Birth:</span> {patient?.birthDate}</p>
      <p><span class="font-semibold">Gender:</span> <span class="capitalize">{patient?.gender}</span></p>
    </div>
  {:catch error}
    <div>Error loading patient details: {error.message}</div>
  {/await}
</div>

<style>
  .border {
    border-color: #e5e7eb; 
  }
  .shadow-md {
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  }
</style>
