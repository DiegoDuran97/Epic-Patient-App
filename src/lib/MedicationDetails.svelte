<script lang="ts">
  import axios from 'axios';
  import { onMount } from 'svelte';
  import { FHIR_BASE_URL } from '../config';
  import type { MedicationRequest, OperationOutcome, Bundle, BundleEntry } from 'fhir/r4';

  export let accessToken: string;
  export let patientId: string;

  const getMedications = async () => {
      try {
          const medicationRequestResponse = await axios.get<Bundle<MedicationRequest | OperationOutcome>>(`${FHIR_BASE_URL}/MedicationRequest`, {
              params: {
                  subject: patientId,
              },
              headers: {
                  'Authorization': `Bearer ${accessToken}`,
              },
          });
          return medicationRequestResponse.data;
      } catch (error) {
          console.error('Error fetching patient details:', error);
          throw error;
      }
  };

  const getMedicationRequestEntries = (bundle: Bundle<MedicationRequest | OperationOutcome>): BundleEntry<MedicationRequest>[] => {
      if (!bundle?.entry) {
          return [];
      }
      return bundle?.entry?.filter((entry) => entry.resource?.resourceType === 'MedicationRequest') as BundleEntry<MedicationRequest>[];
  };
</script>

<div class="mt-10 mx-auto p-6 border rounded-lg shadow-md bg-white max-w-4xl">
  {#await getMedications()}
      loading...
  {:then medicationList}
      <h1 class="text-3xl font-bold mb-4">Medication List</h1>
      {#each getMedicationRequestEntries(medicationList) as medication, i}
          <div class="mb-4">
              <p class="font-medium text-lg">
                  {i + 1}. {medication.resource?.medicationReference?.display}
              </p>
              <div class="ml-4 text-left">
                  {#if medication?.resource?.reasonCode?.[0]?.text}
                      <p>
                          Diagnosis: {medication?.resource?.reasonCode?.[0]?.text}
                      </p>
                  {/if}

                  {#if medication?.resource?.dosageInstruction?.[0]?.patientInstruction}
                      <p>
                          Dosage: {medication?.resource?.dosageInstruction?.[0]?.patientInstruction}
                      </p>
                  {/if}
              </div>
          </div>
      {/each}
  {/await}
</div>

<style>
  .border {
    border-color: #e5e7eb; /* Light gray border color */
  }
  .shadow-md {
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  }
</style>
