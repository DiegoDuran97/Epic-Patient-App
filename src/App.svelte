<script lang="ts">
  import { onMount } from "svelte";
  import { CLIENT_ID, CODE_VERIFIER_SESSION_STORAGE_KEY, FHIR_BASE_URL, REDIRECT_URL, SMART_AUTH_URL, SMART_TOKEN_URL, TOKEN_RESPONSE_SESSION_STORAGE_KEY } from "./config";
  import axios from 'axios';
  import pkceChallenge from 'pkce-challenge';
  import PatientDetails from "./lib/PatientDetails.svelte";
  import MedicationDetails from "./lib/MedicationDetails.svelte";
  import ObservationViewer from "./lib/ObservationViewer.svelte";

  let tokenResponse: { access_token: string, id_token: string, patient: string, scope: string, expires_in?: number } | undefined;
  let loading = true;

  const getSecs = (date: Date) => {
    return Math.round((date).getTime() / 1000);
  };

  const generateCodeChallenge = async () => {
    const { code_challenge, code_verifier } = await pkceChallenge();
    sessionStorage.setItem(CODE_VERIFIER_SESSION_STORAGE_KEY, code_verifier);
    return code_challenge;
  };

  const generateRedirectedURL = (codeChallenge: string) => {
    const authorizationURL = new URL(SMART_AUTH_URL);
    authorizationURL.searchParams.set('client_id', CLIENT_ID);
    authorizationURL.searchParams.set('scope', 'openid fhirUser');
    authorizationURL.searchParams.set('redirect_uri', REDIRECT_URL);
    authorizationURL.searchParams.set('response_type', 'code');
    authorizationURL.searchParams.set('state', '');
    authorizationURL.searchParams.set('aud', FHIR_BASE_URL);
    authorizationURL.searchParams.set('code_challenge', codeChallenge);
    authorizationURL.searchParams.set('code_challenge_method', 'S256');
    return authorizationURL.href;
  };

  const makeTokenRequest = async (code: string, codeVerifier: string) => {
    const tokenRequestForm = new FormData();
    tokenRequestForm.set('grant_type', 'authorization_code');
    tokenRequestForm.set('code', code);
    tokenRequestForm.set('redirect_uri', REDIRECT_URL);
    tokenRequestForm.set('client_id', CLIENT_ID);
    tokenRequestForm.set('code_verifier', codeVerifier);
    const tokenGeneratedAt = getSecs(new Date());
    try {
      const response = await axios.post(SMART_TOKEN_URL, tokenRequestForm);
      console.log("Response Data:", response.data);

      if (response.data && typeof response.data === 'object') {
        tokenResponse = response.data;
        sessionStorage.setItem(TOKEN_RESPONSE_SESSION_STORAGE_KEY, JSON.stringify({ ...tokenResponse, issued_at_in_sec: tokenGeneratedAt }));
      } else {
        console.error("Invalid token response format:", response.data);
      }
    } catch (error) {
      console.error("Error in token request:", error);
    }
  };

  onMount(() => {
    const code = new URL(window.location.href).searchParams.get('code');
    const codeVerifier = sessionStorage.getItem(CODE_VERIFIER_SESSION_STORAGE_KEY);
    const tokenResponseString = sessionStorage.getItem(TOKEN_RESPONSE_SESSION_STORAGE_KEY);

    if (tokenResponseString) {
      try {
        const tokenResponseTemp = JSON.parse(tokenResponseString);
        const { issued_at_in_sec, expires_in } = tokenResponseTemp;
        const expires_in_sec = issued_at_in_sec + expires_in;
        const now_in_sec = getSecs(new Date());
        if (now_in_sec >= expires_in_sec) {
          sessionStorage.removeItem(TOKEN_RESPONSE_SESSION_STORAGE_KEY);
        } else {
          tokenResponse = tokenResponseTemp;
        }
      } catch (e) {
        console.error("JSON parsing error:", e, "Token Response String:", tokenResponseString);
      }
    } else if (code && codeVerifier) {
      makeTokenRequest(code, codeVerifier).then(() => {
        sessionStorage.removeItem(CODE_VERIFIER_SESSION_STORAGE_KEY);
      }).catch((e) => {
        console.error("Error during token request:", e);
      });
    }

    loading = false;
  });

  const initiateAuthorizationRequest = async () => {
    const codeChallenge = await generateCodeChallenge();
    const redirectUrl = generateRedirectedURL(codeChallenge);
    console.log("Redirecting to:", redirectUrl);
    window.location.href = redirectUrl;
  };
</script>

<main class="flex flex-col items-center justify-center min-h-screen">
  {#if loading}
    <p>Loading...</p>
  {:else}
    {#if tokenResponse}
      <PatientDetails accessToken={tokenResponse.access_token} patientId={tokenResponse.patient} />
      <MedicationDetails accessToken={tokenResponse.access_token} patientId={tokenResponse.patient}></MedicationDetails>
      <ObservationViewer accessToken={tokenResponse.access_token} patientId={tokenResponse.patient}></ObservationViewer>
      <ObservationViewer title="Vital Signs" category='vital-signs' accessToken={tokenResponse.access_token} patientId={tokenResponse.patient}></ObservationViewer>
    {:else}
      <div class="text-center">
        <h1 class="text-3xl font-bold mb-4">MyHealth App</h1>
        <h2 class="text-xl mb-6">Click the Sign In button to login with Epic to get started!</h2>
        <button on:click={initiateAuthorizationRequest} class="p-3 bg-black rounded-md text-white">Sign In</button>
      </div>
    {/if}
  {/if}
</main>
