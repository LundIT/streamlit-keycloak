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
    export let loginPage: LoginPageProps = {background: "", logo: ""}

    const rewritePage = (newPage: string): string => {
        return (
            window.location.origin +
            window.location.pathname.replace(/\/[^/]*$/, newPage)
        )
    }

    const getLoginUrl = (): string => {
        return keycloak.createLoginUrl({
            ...loginOptions,
           redirectUri: rewritePage('/login.html')
        })
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

    // Set up Keycloak events listeners to send state to Steamlit
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

    onMount((): void => {
        Streamlit.setFrameHeight()
    })

    afterUpdate((): void => {
        Streamlit.setFrameHeight(clientHeight)
    })

    let keycloak: Keycloak
    let clientHeight: number

    const labels = {
        ...defaultLabels,
        ...customLabels,
    }

    setContext('localization', labels)
</script>

<style>
   .block-container {
     padding-right: 0;
     padding-left: 0;
     padding-bottom: 0;
     height: calc(100vh - 160px);
   }
    .background {
        position: fixed;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background-size: cover;
        background-position: center;
        z-index: -1;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        gap: 10rem;
    }

    .logo {
        max-width: 200px;
    }

</style>


<div bind:clientHeight>
    {#await authenticate() then authenticated}
        {#if !authenticated}
        <div class="background" style="background-image: url({loginPage.background});">
            <div>
                <img class="logo" src={loginPage.logo} alt="Logo" />
            </div>
            <LoginDialog
                loginUrl={getLoginUrl()}
                on:loggedin={() => {
                    keycloak.login(loginOptions)
                }}
                loginPage={loginPage}
            />
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
