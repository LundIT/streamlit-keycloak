<script lang="ts">
    import LoginDialog from './LoginDialog.svelte'
    import { afterUpdate, onMount, setContext } from 'svelte'
    import { Streamlit } from './streamlit'
    import Keycloak from 'keycloak-js'
    import type { KeycloakInitOptions, KeycloakLoginOptions } from 'keycloak-js'
    import { LabelMap, defaultLabels } from './localization'

    export let url: string
    export let realm: string
    export let clientId: string
    export let autoRefresh: boolean = true
    export let initOptions: KeycloakInitOptions = {}
    export let loginOptions: KeycloakLoginOptions = {}
    export let customLabels: LabelMap = {}

    const rewritePage = (newPage: string): string => {
        return (
            window.location.origin +
            window.location.pathname.replace(/\/[^/]*$/, newPage)
        )
    }

    const getLoginUrl = (): string => {
        return keycloak.createLoginUrl(loginOptions)
    }

    // Set up the response to Streamlit
    const setComponentValue = async (): Promise<void> => {
        if (!keycloak.userInfo && keycloak.authenticated) {
            await keycloak.loadUserInfo()
        }

        let value
        if (keycloak.authenticated) {
            value = {
                authenticated: true,
                access_token: keycloak.token,
                refresh_token: keycloak.refreshToken,
                user_info: keycloak.userInfo,
                id_token: keycloak.idToken,
            }
        } else {
            value = { authenticated: false }
        }

        Streamlit.setComponentValue(value)
    }

    // Set up Keycloak events listeners to send state to Streamlit
    const setKeycloakEventListeners = (autoRefresh: boolean): void => {
        keycloak.onAuthError = async () => await setComponentValue()
        keycloak.onAuthRefreshError = async () => await setComponentValue()
        keycloak.onAuthSuccess = async () => await setComponentValue()
        keycloak.onAuthRefreshSuccess = async () => await setComponentValue()
        keycloak.onTokenExpired = async () => {
            if (!autoRefresh || !keycloak.refreshToken) return
            await keycloak.updateToken(10)
        }
    }

    const authenticate = async (): Promise<boolean> => {
        keycloak = new Keycloak({
            url: url,
            realm: realm,
            clientId: clientId,
        })

        setKeycloakEventListeners(autoRefresh)

        // Check if user is already logged in
        return keycloak.init({
            ...initOptions,
            onLoad: 'check-sso',
            silentCheckSsoRedirectUri: rewritePage('/check-sso.html'),
        })
    }

    // Adapted authentication with popup logic
    const createLoginPopup = (): void => {
        if (currentPopup && !currentPopup.closed) {
            currentPopup.focus()
            return
        }

        openPopup(getLoginUrl())
        showPopup = true
    }

    const openPopup = (url: string): void => {
        const width = 400
        const height = 600
        const left = window.screenX + (window.innerWidth - width) / 2
        const top = window.screenY + (window.innerHeight - height) / 2

        currentPopup = window.open(
            url,
            'keycloak:authorize:popup',
            `left=${left},top=${top},width=${width},height=${height},resizable,scrollbars=yes,status=1`
        )
    }

    const runPopup = async (popup: Window): Promise<Record<string, string>> => {
        return new Promise((resolve, reject) => {
            // Throw exception if popup is closed manually
            const popupTimer = setInterval(() => {
                if (popup.closed) {
                    window.removeEventListener(
                        'message',
                        popupEventListener,
                        false
                    )
                    clearInterval(popupTimer)

                    reject(new Error(labels.errorPopupClosed))
                }
            }, 1000)

            // Wait for postMessage from popup if login is successful
            const popupEventListener = function (event: MessageEvent): void {
                if (event.origin !== window.location.origin) return
                if (!Object.keys(event.data).includes('code')) return

                window.removeEventListener('message', popupEventListener, false)
                clearInterval(popupTimer)

                popup.close()
                resolve(event.data)
            }

            window.addEventListener('message', popupEventListener)
        })
    }

    const authenticateWithPopup = async (
        popup: Window | null
    ): Promise<void> => {
        if (!popup) {
            throw new Error(labels.errorNoPopup)
        }

        await runPopup(popup)
        dispatch('loggedin')
    }

    const labels = {
        ...defaultLabels,
        ...customLabels,
    }

    const dispatch = createEventDispatcher()
    let currentPopup: Window | null
    let showPopup = false

    onMount((): void => {
        Streamlit.setFrameHeight()
    })

    afterUpdate((): void => {
        Streamlit.setFrameHeight(clientHeight)
    })

    let keycloak: Keycloak
    let clientHeight: number

    setContext('localization', labels)
</script>

<div bind:clientHeight>
    {#await authenticate() then authenticated}
        {#if !authenticated}
            <LoginDialog
                loginUrl={getLoginUrl()}
                on:loggedin={() => {
                    keycloak.login(loginOptions)
                }}
            />
            <div class="alert alert-warning">
                <button type="button" class="btn btn-primary" on:click={createLoginPopup}>
                    <span>{labels.labelButton}</span>
                </button>
                <span class="mx-3">{labels.labelLogin}</span>
                {#if showPopup}
                    {#await authenticateWithPopup(currentPopup) catch error}
                        <div class="alert alert-danger mt-3 mb-0">{error.message}</div>
                    {/await}
                {/if}
            </div>
        {/if}
    {:catch exception}
        <div class="alert alert-danger">
            <span>{labels.errorFatal}</span>
            <br />
            <span>{exception.error}</span>
        </div>
    {/await}
</div>
