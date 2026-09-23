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

    stopNotificationLoop();

    stopAuthorizationMonitor();

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
