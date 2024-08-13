<script lang="ts">
  import { onMount } from "svelte";
  import { CLIENT_ID, CODE_VERIFIER_SESSION_STORAGE_KEY, FHIR_BASE_URL, REDIRECT_URL, SMART_AUTH_URL, SMART_TOKEN_URL, TOKEN_RESPONSE_SESSION_STORAGE_KEY } from "./config";
  import axios from 'axios';
  import pkceChallenge from 'pkce-challenge';
  import PatientDetails from "./lib/PatientDetails.svelte";

  let tokenResponse: { access_token: string, id_token: string, patient: string, scope: string } | undefined;
  let loading = true;

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

    try {
      const response = await axios.post(SMART_TOKEN_URL, tokenRequestForm);
      console.log("Response Data:", response.data);

      if (response.data && typeof response.data === 'object') {
        tokenResponse = response.data;
        sessionStorage.setItem(TOKEN_RESPONSE_SESSION_STORAGE_KEY, JSON.stringify(tokenResponse));
        console.log("Token saved to sessionStorage:", JSON.stringify(tokenResponse));
        console.log("SessionStorage Token Response:", sessionStorage.getItem(TOKEN_RESPONSE_SESSION_STORAGE_KEY));
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

    console.log("Code:", code);
    console.log("Code Verifier:", codeVerifier);
    console.log("Token Response String:", tokenResponseString);

    if (tokenResponseString) {
      try {
        console.log("Trying to parse Token Response String:", tokenResponseString);
        tokenResponse = JSON.parse(tokenResponseString);
        console.log("Parsed Token Response:", tokenResponse);
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

<main>
  {#if loading}
    Loading...
  {:else}
    {#if tokenResponse}
      <PatientDetails accessToken={tokenResponse.access_token} patientId={tokenResponse.patient} />
    {:else}
      <div class="flex justify-center my-20">
        <button on:click={initiateAuthorizationRequest} class="p-3 bg-black rounded-md text-white">Sign In With Epic</button>
      </div>
    {/if}
  {/if}
</main>
