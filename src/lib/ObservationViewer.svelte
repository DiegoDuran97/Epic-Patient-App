<script lang="ts">
  import axios from 'axios';
  import { formatRelative } from 'date-fns';
  import { FHIR_BASE_URL } from '../config';
  import type { OperationOutcome, Bundle, BundleEntry, Observation } from 'fhir/r4';

  export let accessToken: string;
  export let patientId: string;
  export let category: string = 'laboratory';
  export let title: string = 'Lab Results';

  const getLabResults = async () => {
      try {
          const labObservationResponse = await axios.get<Bundle<Observation | OperationOutcome>>(`${FHIR_BASE_URL}/Observation`, {
              params: {
                  subject: patientId,
                  category,
                  sort: 'date'
              },
              headers: {
                  'Authorization': `Bearer ${accessToken}`,
              },
          });
          return labObservationResponse.data;
      } catch (error) {
          console.error('Error fetching lab results:', error);
          throw error;
      }
  };

  const getVitalSigns = async () => {
      try {
          const vitalSignsResponse = await axios.get<Bundle<Observation | OperationOutcome>>(`${FHIR_BASE_URL}/Observation`, {
              params: {
                  subject: patientId,
                  category: 'vital-signs',
                  sort: 'date'
              },
              headers: {
                  'Authorization': `Bearer ${accessToken}`,
              },
          });
          return vitalSignsResponse.data;
      } catch (error) {
          console.error('Error fetching vital signs:', error);
          throw error;
      }
  };

  const getObservationDisplay = (Observation: Observation | undefined) => {
      if (!Observation) {
          return '';
      }
      const isBp = Observation?.code?.coding?.find(a => a.code == '55284-4');
      if (isBp) {
          const systolicComponent = Observation.component?.find(a => a.code?.coding?.find(b => b.code == '8480-6'));
          const systolic = systolicComponent?.valueQuantity?.value;
          const diastolicComponent = Observation.component?.find(a => a.code?.coding?.find(b => b.code == '8462-4'));
          const diastolic = diastolicComponent?.valueQuantity?.value;
          return `${systolic}/${diastolic}`;
      } 
      if (!Observation?.valueQuantity?.unit) {
          return Observation?.valueQuantity?.value;
      }
      return `${Observation?.valueQuantity?.value} ${Observation?.valueQuantity?.unit}`;
  };

  const getObservationEntries = (bundle: Bundle<Observation | OperationOutcome>): BundleEntry<Observation>[] => {
      if (!bundle?.entry) {
          return [];
      }
      const results = bundle.entry.filter((entry) => entry.resource?.resourceType === 'Observation') as BundleEntry<Observation>[];
      return results.sort((a, b) => {
          if (a?.resource?.effectiveDateTime && b?.resource?.effectiveDateTime) {
              return new Date(b?.resource?.effectiveDateTime).getTime() - new Date(a?.resource?.effectiveDateTime).getTime();
          }
          return 0;
      });
  };
</script>

<!-- Lab Results Section -->
<div class="mt-10 mx-auto p-6 border rounded-lg shadow-md bg-white max-w-4xl">
  {#await getLabResults()}
      loading...
  {:then labResults}
      <h1 class="text-3xl font-bold mb-4">{title}</h1>
      {#each getObservationEntries(labResults) as Observation}
          <p>
              <span class='font-medium'>
                  {#if Observation?.resource?.effectiveDateTime}
                      {formatRelative(new Date(Observation?.resource.effectiveDateTime), new Date())} -
                  {/if}
                  {Observation.resource?.code?.text} :  
              </span>
              {getObservationDisplay(Observation?.resource)}
          </p>
      {/each}
  {/await}
</div>

<!-- Vital Signs Section -->
<div class="mt-10 mx-auto p-6 border rounded-lg shadow-md bg-white max-w-4xl">
  {#await getVitalSigns()}
      loading...
  {:then vitalSigns}
      <h1 class="text-3xl font-bold mb-4">Vital Signs</h1>
      <table class="min-w-full divide-y divide-gray-200">
          <thead class="bg-gray-50">
              <tr>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Date</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Vital Sign</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Value</th>
              </tr>
          </thead>
          <tbody class="bg-white divide-y divide-gray-200">
              {#each getObservationEntries(vitalSigns) as Observation}
                  <tr>
                      <td class="px-6 py-4 whitespace-nowrap text-sm font-medium text-gray-900">
                          {#if Observation?.resource?.effectiveDateTime}
                              {formatRelative(new Date(Observation?.resource.effectiveDateTime), new Date())}
                          {/if}
                      </td>
                      <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                          {Observation.resource?.code?.text}
                      </td>
                      <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                          {getObservationDisplay(Observation?.resource)}
                      </td>
                  </tr>
              {/each}
          </tbody>
      </table>
  {/await}
</div>

<style>
  .border {
      border-color: #e5e7eb; /* Light gray border color */
  }
  .shadow-md {
      box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  }
  table {
      border-collapse: collapse;
      width: 100%;
  }
  th, td {
      border: 1px solid #e5e7eb; /* Light gray border color */
      padding: 0.75rem;
  }
</style>
