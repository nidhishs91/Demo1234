function stopAuthorizationRevalidationLoop() {
 
    if (authorizationRevalidationTimer) {
 
        clearInterval(
            authorizationRevalidationTimer
        );
 
        authorizationRevalidationTimer =
            null;
    }
}
 
 
function startAuthorizationRevalidationLoop() {
 
    stopAuthorizationRevalidationLoop();
 
    authorizationRevalidationTimer =
        setInterval(
            async () => {
 
                try {
 
                    const currentUser =
                        await getCurrentServiceCallUser();
 
                    const authorization =
                        currentUser?.authorization || {};
 
                    /*
                     * Keep our runtime identity/role snapshot
                     * synchronized with ServiceNow.
                     */
                    currentServiceCallUser =
                        currentUser?.user || null;
 
                    currentServiceCallAuthorization =
                        authorization;
 
 
                    /*
                     * ServiceNow successfully answered /me,
                     * but this user no longer has ServiceCall
                     * authorization.
                     *
                     * This is different from a temporary
                     * network failure.
                     */
                    if (
                        authorization.allowed !== true
                    ) {
 
                        console.log(
                            'ServiceCall authorization was revoked.'
                        );
 
                        stopAuthorizationRevalidationLoop();
 
                        await stopCurrentServiceCallSession();
 
                        sendAuthStatus(
                            'access_denied',
                            'Your ServiceCall access is no longer assigned.'
                        );
 
                        return;
                    }
 
                } catch (error) {
 
                    /*
                     * IMPORTANT:
                     *
                     * Do not remove access because of one failed
                     * network request.
                     *
                     * Role revocation is only trusted when /me
                     * successfully returns allowed = false.
                     */
                    console.error(
                        'ServiceCall authorization revalidation failed:',
                        error
                    );
                }
 
            },
 
            60000
        );
}
 
