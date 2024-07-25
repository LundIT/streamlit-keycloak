<script lang="ts">
    import { createEventDispatcher, getContext, onMount } from 'svelte'
    import type { LabelMap } from './localization'

    export let loginUrl: string
    export let redirectUri: string

    const authenticate = async (): Promise<Record<string, string>> => {
        const urlParams = new URLSearchParams(window.location.search)
        const code = urlParams.get('code')
        if (code) {
            return { code }
        } else {
            throw new Error(labels.errorNoCode)
        }
    }

    const redirectToLogin = (): void => {
        const state = Math.random().toString(36).substring(2)
        const loginWithRedirect = `${loginUrl}?response_type=code&client_id=your-client-id&redirect_uri=${redirectUri}&state=${state}`
        window.location.href = loginWithRedirect
    }

    const labels: LabelMap = getContext('localization')
    const dispatch = createEventDispatcher()

    let showError = false
    let errorMessage = ''

    onMount(async () => {
        try {
            const result = await authenticate()
            if (result && result.code) {
                dispatch('loggedin')
            }
        } catch (error) {
            showError = true
            errorMessage = error.message
        }
    })
</script>

<div class="alert alert-warning" on:loggedin>
    <button type="button" class="btn btn-primary" on:click={redirectToLogin}>
        <span>{labels.labelButton}</span>
    </button>
    <span class="mx-3">{labels.labelLogin}</span>
    {#if showError}
        <div class="alert alert-danger mt-3 mb-0">{errorMessage}</div>
    {/if}
</div>
