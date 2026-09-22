async function checkRuntimeAuthorizationOnce() {
 
    try {
 
        /*
         * No active authenticated ServiceCall identity
         * means there is nothing to monitor.
         */
        if (!currentServiceCallUser) {
            return;
        }
 
 
        /*
         * Ensure the OAuth session itself is still valid.
         */
        await ensureValidAccessToken();
 
 
        /*
         * Re-check the CURRENT ServiceNow roles.
         *
         * Do not trust the role snapshot stored when
         * the user originally signed in.
         */
        const currentUser =
            await getCurrentServiceCallUser();
 
 
        const authorization =
            currentUser?.authorization || {};
 
 
        currentServiceCallUser =
            currentUser?.user || null;
 
        currentServiceCallAuthorization =
            authorization;
 
 
        /*
         * -----------------------------------------
         * SERVICECALL ACCESS REMOVED
         * -----------------------------------------
         */
 
        if (
            authorization.allowed !== true
        ) {
 
            if (!serviceCallAccessUnavailable) {
 
                serviceCallAccessUnavailable =
                    true;
 
 
                console.warn(
                    'ServiceCall runtime access removed.',
                    {
                        user:
                            currentServiceCallUser,
 
                        authorization:
                            authorization
                    }
                );
 
                suspendServiceCallRuntime();

                await showServiceCallUnavailablePage();
 
                sendAuthStatus(
                    'access_removed',
                    'ServiceCall access is currently unavailable for this account.'
                );
            }
 
 
            return;
        }
 
 
        /*
         * -----------------------------------------
         * SERVICECALL ACCESS RESTORED
         * -----------------------------------------
         */
 
        if (serviceCallAccessUnavailable) {
 
    serviceCallAccessUnavailable =
        false;
 
 
    console.log(
        'ServiceCall runtime access restored.',
        {
            user:
                currentServiceCallUser,
 
            authorization:
                authorization
        }
    );
 
 
    try {
 
        await resumeServiceCallRuntime();

        await restoreServiceCallApplication();
 
    } catch (error) {
 
        console.error(
            'Unable to resume ServiceCall runtime:',
            error
        );
    }
 
 
    sendAuthStatus(
        'access_restored',
        'ServiceCall access has been restored.'
    );
}
 
 
    } catch (error) {
 
 
        console.error(
            'ServiceCall runtime authorization check failed:',
            error
        );
    }
}
