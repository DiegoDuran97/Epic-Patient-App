<script lang="ts">
    import { onMount } from "svelte";
  import {CLIENT_ID, FHIR_BASE_URL, REDIRECT_URL, SMART_AUTH_URL, SMART_TOKEN_URL} from "./config"
  import axios from 'axios';
  const generateRedirectedURL = () => {
   const authorizationURL = new URL(SMART_AUTH_URL)
   authorizationURL.searchParams.set('clinet_id', CLIENT_ID)
   authorizationURL.searchParams.set('scope', 'openid fhirUser')
   authorizationURL.searchParams.set('redirect_uri', REDIRECT_URL)
   authorizationURL.searchParams.set('response_type', 'code')
   authorizationURL.searchParams.set('state', '')
   authorizationURL.searchParams.set('aud', FHIR_BASE_URL)
   authorizationURL.searchParams.set('code_challenge', '')
   authorizationURL.searchParams.set('code_challenge_method', 'S256')
  
    return authorizationURL.href

  }
  const makeTokenRequest = async (code:string) => {
    const tokenRequestForm = new FormData();
    tokenRequestForm.set('grant_type', 'authorization_code') 
    tokenRequestForm.set('code', code)
    tokenRequestForm.set('redirect_uri', REDIRECT_URL)
    tokenRequestForm.set('clinet_id', CLIENT_ID)

    const response = await axios.postForm(SMART_TOKEN_URL, tokenRequestForm)
    console.log(response)

    onMount(async () =>{
      const code = new URL(window.location.href).searchParams.get('code')
      if (code) {
        await makeTokenRequest(code)
      } 
    })

  }
</script>

<main>
  <div class="flex justify-center my-20">
  <button on:click={()=>{console.log(generateRedirectedURL())}} class ="p-3 bg-black rounded-md text-white">Sign In With Epic</button>
  </div>
</main>


