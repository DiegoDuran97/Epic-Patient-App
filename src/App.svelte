<script lang="ts">
  import { onMount } from "svelte";
  import {CLIENT_ID, CODE_VERIFIER_LOCAL_STORAGE_KEY, FHIR_BASE_URL, REDIRECT_URL, SMART_AUTH_URL, SMART_TOKEN_URL} from "./config"
  import axios from 'axios';
  import pkceChallenge from 'pkce-challenge';

  let tokenResponse: {
      access_token: string,
      id_token: string,
      patient: string,
      scope: string
    }
  let loading = true

 
  const generateCodeChallenge = async () => {
    const {code_challenge, code_verifier} = await  pkceChallenge()
    localStorage.setItem(CODE_VERIFIER_LOCAL_STORAGE_KEY, code_verifier)
    return code_challenge
  }


  const generateRedirectedURL = (codeChallenge: string) => {
   const authorizationURL = new URL(SMART_AUTH_URL)
   authorizationURL.searchParams.set('clinet_id', CLIENT_ID)
   authorizationURL.searchParams.set('scope', 'openid fhirUser')
   authorizationURL.searchParams.set('redirect_uri', REDIRECT_URL)
   authorizationURL.searchParams.set('response_type', 'code')
   authorizationURL.searchParams.set('state', '')
   authorizationURL.searchParams.set('aud', FHIR_BASE_URL)
   authorizationURL.searchParams.set('code_challenge',  codeChallenge)
   authorizationURL.searchParams.set('code_challenge_method', 'S256')
  
    return authorizationURL.href

  }
  const makeTokenRequest = async (code:string, codeVerifier:string) => {
    const tokenRequestForm = new FormData();
    tokenRequestForm.set('grant_type', 'authorization_code') 
    tokenRequestForm.set('code', code)
    tokenRequestForm.set('redirect_uri', REDIRECT_URL)
    tokenRequestForm.set('clinet_id', CLIENT_ID)
    tokenRequestForm.set('code_verifier', codeVerifier)


    const response = await axios.postForm(SMART_TOKEN_URL, tokenRequestForm)
    console.log(response)
    tokenResponse = response.data 
  }

    onMount(async () => {
      const code = new URL(window.location.href).searchParams.get('code')
      const codeVerifier = localStorage.getItem(CODE_VERIFIER_LOCAL_STORAGE_KEY) 
      try {
        if (code  && codeVerifier) {
          await makeTokenRequest(code, codeVerifier)
          localStorage.removeItem(CODE_VERIFIER_LOCAL_STORAGE_KEY)
      } 
    } catch (e) {
      console.error(e)
    }
    loading = false
  })



const initiateAuthorizationRequest = async () => {
  const codeChallenge = await generateCodeChallenge()
  const redirectUrl = generateRedirectedURL(codeChallenge)
  window.location.href = redirectUrl
}
</script>

<main>

  {#if loading}
    Loading...
  {:else}
    {#if tokenResponse}
      {JSON.stringify(tokenResponse)}
    {:else}
    <div class="flex justify-center my-20">
    <button on:click={initiateAuthorizationRequest} class ="p-3 bg-black rounded-md text-white">Sign In With Epic</button>
    </div>
  {/if}
  {/if}
</main>


