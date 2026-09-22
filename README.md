ipcMain.handle(
    'servicecall-check-access',
    async () => {

        try {

            /*
             * Make sure the saved OAuth
             * session is still usable.
             */

            await ensureValidAccessToken();


            /*
             * Re-check identity + current
             * ServiceCall roles.
             */

            const currentUser =
                await getCurrentServiceCallUser();

            const authorization =
                currentUser?.authorization || {};

            currentServiceCallUser =
    currentUser?.user || null;

currentServiceCallAuthorization =
    authorization;


            if (
                authorization.allowed !== true
            ) {

                return {
                    success: true,
                    authenticated: true,
                    authorized: false,
                    state: 'access_denied',
                    user:
                        currentUser?.user || null,
                    authorization:
                        authorization
                };
            }


            /*
             * ACCESS HAS NOW BEEN GRANTED
             */

            await startHeartbeatLoop();

            startIncomingCallLoop();

            startOutgoingCallLoop();


            return {
                success: true,
                authenticated: true,
                authorized: true,
                state: 'ready',
                user:
                    currentUser?.user || null,
                authorization:
                    authorization
            };

        }
        catch (error) {

            console.error(
                'ServiceCall access check failed:',
                error
            );

            return {
                success: false,
                authenticated: false,
                authorized: false,
                state: 'login_required',
                message:
                    error?.message ||
                    'Unable to check ServiceCall access.'
            };
        }
    }
);
