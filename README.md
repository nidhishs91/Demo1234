async function resolveStartupAuthentication() {

    const config =
        loadConfig();

    /*
     * -----------------------------------------
     * 1. INSTANCE NOT CONFIGURED
     * -----------------------------------------
     */

    if (
        !config ||
        !config.instanceUrl
    ) {
        return {
            authenticated: false,
            authorized: false,
            state: 'instance_required'
        };
    }


    /*
     * -----------------------------------------
     * 2. NO SAVED SESSION
     * -----------------------------------------
     */

    if (
        !config.accessToken &&
        !config.refreshToken
    ) {
        return {
            authenticated: false,
            authorized: false,
            state: 'login_required'
        };
    }


    /*
     * -----------------------------------------
     * 3. RESTORE / REFRESH SESSION
     * -----------------------------------------
     */

    try {

        /*
         * ensureValidAccessToken() currently
         * expects an access token.
         *
         * If only a refresh token remains,
         * refresh it directly.
         */

        if (
            !config.accessToken &&
            config.refreshToken
        ) {
            await refreshAccessToken();
        }
        else {
            await ensureValidAccessToken();
        }


        /*
         * -------------------------------------
         * 4. CHECK SERVICENOW IDENTITY + ROLE
         * -------------------------------------
         */

        const currentUser =
            await getCurrentServiceCallUser();

        const authorization =
            currentUser?.authorization || {};

        const user =
            currentUser?.user || null;

        /*
 * -----------------------------------------
 * STORE CURRENT ACCOUNT IDENTITY
 * -----------------------------------------
 */

currentServiceCallUser =
    user;

currentServiceCallAuthorization =
    authorization;


        /*
         * -------------------------------------
         * 5. AUTHENTICATED BUT NOT AUTHORIZED
         * -------------------------------------
         */

        if (
            authorization.allowed !== true
        ) {
            return {
                authenticated: true,
                authorized: false,
                state: 'access_denied',
                user: user,
                authorization: authorization
            };
        }


        /*
         * -------------------------------------
         * 6. AUTHENTICATED + AUTHORIZED
         * -------------------------------------
         */

        return {
            authenticated: true,
            authorized: true,
            state: 'ready',
            user: user,
            authorization: authorization
        };

    }
    catch (error) {

        console.error(
            'ServiceCall startup authentication failed:',
            error
        );

        return {
            authenticated: false,
            authorized: false,
            state: 'login_required',
            error:
                error?.message ||
                'Authentication could not be restored.'
        };
    }
}
