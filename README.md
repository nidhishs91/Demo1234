async function getCurrentServiceCallUser() {
 
    const result =
        await serviceCallApiRequest(
            '/me',
            'GET'
        );
 
    if (
        !result ||
        result.success !== true
    ) {
        throw new Error(
            result?.message ||
            'Unable to retrieve the current ServiceCall user.'
        );
    }
 
    return result;
}
 
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
 
async function stopCurrentServiceCallSession() {
 
    /*
     * -----------------------------------------
     * STOP HEARTBEAT
     * -----------------------------------------
     */
 
    if (heartbeatTimer) {
 
        clearInterval(
            heartbeatTimer
        );
 
        heartbeatTimer =
            null;
    }
 
 
    /*
     * -----------------------------------------
     * STOP INCOMING CALL MONITOR
     * -----------------------------------------
     */
 
    if (incomingCallTimer) {
 
        clearInterval(
            incomingCallTimer
        );
 
        incomingCallTimer =
            null;
    }
 
 
    /*
     * -----------------------------------------
     * STOP OUTGOING CALL MONITOR
     * -----------------------------------------
     */
 
    if (outgoingCallTimer) {
 
        clearInterval(
            outgoingCallTimer
        );
 
        outgoingCallTimer =
            null;
    }
 
 
    /*
     * -----------------------------------------
     * MARK DESKTOP OFFLINE
     * -----------------------------------------
     *
     * Do this BEFORE clearing the active
     * authenticated account.
     */
 
    try {
 
    await signOutDesktopSession();
 
} catch (error) {
 
    console.error(
        'ServiceCall explicit sign out failed:',
        error
    );
 
 
    /*
     * Fallback:
     *
     * If the dedicated sign-out endpoint
     * fails, still try to mark this device
     * registration offline.
     */
    try {
 
        await updateDesktopState(
            'offline'
        );
 
    } catch (fallbackError) {
 
        console.error(
            'ServiceCall offline fallback failed:',
            fallbackError
        );
    }
}
 
 
    /*
     * -----------------------------------------
     * CLEAR ACTIVE IN-MEMORY IDENTITY
     * -----------------------------------------
     */
 
    currentServiceCallUser =
        null;
 
    currentServiceCallAuthorization =
        null;
 
 
    activeIncomingCallId =
        null;
 
    activeOutgoingCallId =
        null;
 
 
    console.log(
        'Current ServiceCall session stopped.'
    );
 
 
    return {
        success: true
    };
}
 
async function startHeartbeatLoop() {
 
    stopHeartbeatLoop();
 
    try {
 
        const result =
            await sendHeartbeatOnce();
 
        console.log(
            'ServiceCall heartbeat:',
            result
        );
 
        sendAuthStatus(
            'connected',
            'ServiceCall Desktop is connected.'
        );
 
    } catch (error) {
 
        console.error(
            'Initial heartbeat failed:',
            error
        );
 
        if (
            error.code ===
            'AUTHENTICATION_REQUIRED'
        ) {
 
            stopHeartbeatLoop();
 
            sendAuthStatus(
                'authentication_required',
                'Your ServiceNow session has expired. Please sign in again.'
            );
 
            return;
        }
 
        sendAuthStatus(
            'warning',
            'ServiceCall Desktop heartbeat failed: ' +
            error.message
        );
 
        return;
    }
 
    heartbeatTimer =
        setInterval(
            async () => {
 
                try {
 
                    const result =
                        await sendHeartbeatOnce();
 
                    console.log(
                        'ServiceCall heartbeat:',
                        result
                    );
 
                } catch (error) {
 
                    console.error(
                        'Heartbeat failed:',
                        error
                    );
 
                    if (
                        error.code ===
                        'AUTHENTICATION_REQUIRED'
                    ) {
 
                        stopHeartbeatLoop();
 
                        sendAuthStatus(
                            'authentication_required',
                            'Your ServiceNow session has expired. Please sign in again.'
                        );
 
                        return;
                    }
 
                    sendAuthStatus(
                        'warning',
                        'ServiceCall Desktop heartbeat failed: ' +
                        error.message
                    );
                }
 
            },
            30000
        );
}
 
async function sendHeartbeatOnce() {
 
    if (isDeviceSuspended) {
 
        console.log(
            'ServiceCall heartbeat skipped because device is suspended.'
        );
 
        return {
            success: true,
            skipped: true,
            reason: 'device_suspended'
        };
    }
 
    const config =
        loadConfig();
 
    if (!config.instanceUrl) {
        throw new Error(
            'ServiceNow instance is not configured.'
        );
    }
 
    if (!config.accessToken) {
        throw new Error(
            'ServiceNow access token was not found.'
        );
    }
 
    const validAccessToken = await ensureValidAccessToken();
 
    const heartbeatUrl =
        config.instanceUrl +
        config.heartbeatPath;
 
    const payload = {
        device_id:
            getOrCreateDeviceId(),
 
        device_name:
            os.hostname(),
 
        platform:
            process.platform === 'win32'
                ? 'Windows'
                : process.platform,
 
        app_version:
            app.getVersion()
    };
 
    let response =
    await fetch(
        heartbeatUrl,
        {
            method: 'POST',
 
            headers: {
                'Authorization':
                    'Bearer ' +
                    validAccessToken,
 
                'Content-Type':
                    'application/json',
 
                'Accept':
                    'application/json'
            },
 
            body:
                JSON.stringify(
                    payload
                )
        }
    );
 
 
/*
* If the short-lived access token expired,
* automatically renew it and retry heartbeat once.
*/
if (
    response.status === 401 ||
    response.status === 403
) {
 
    console.log(
        'Heartbeat authorization expired. Attempting automatic renewal...'
    );
 
 
    try {
 
        const newAccessToken =
            await refreshAccessToken();
 
 
        response =
            await fetch(
                heartbeatUrl,
                {
                    method: 'POST',
 
                    headers: {
                        'Authorization':
                            'Bearer ' +
                            newAccessToken,
 
                        'Content-Type':
                            'application/json',
 
                        'Accept':
                            'application/json'
                    },
 
                    body:
                        JSON.stringify(
                            payload
                        )
                }
            );
 
 
    } catch (refreshError) {
 
        const authError =
            new Error(
                'Your ServiceCall authorization has expired. Please sign in again.'
            );
 
        authError.code =
            'AUTHENTICATION_REQUIRED';
 
        throw authError;
    }
}
 
    const responseText =
        await response.text();
 
    let data;
 
    try {
 
        data =
            JSON.parse(
                responseText
            );
 
    } catch (error) {
 
        throw new Error(
            'Heartbeat API returned an invalid response.'
        );
    }
 
    if (!response.ok) {
 
    console.error(
        'Heartbeat HTTP status:',
        response.status
    );
 
    console.error(
        'Heartbeat response:',
        JSON.stringify(
            data,
            null,
            2
        )
    );
 
    if (
    response.status === 401 ||
    response.status === 403
) {
 
    const authError =
        new Error(
            'Your ServiceNow session has expired. Please sign in again.'
        );
 
    authError.code =
        'AUTHENTICATION_REQUIRED';
 
    throw authError;
}
 
    var errorMessage =
        'Heartbeat failed with HTTP ' +
        response.status;
 
    if (
        data &&
        typeof data.error === 'object' &&
        data.error
    ) {
 
        errorMessage =
            data.error.message ||
            data.error.detail ||
            errorMessage;
 
    } else if (
        data &&
        typeof data.error === 'string'
    ) {
 
        errorMessage =
            data.error;
 
    } else if (
        data &&
        typeof data.message === 'string'
    ) {
 
        errorMessage =
            data.message;
    }
 
    throw new Error(
        errorMessage
    );
}
 
    return data;
}
 
