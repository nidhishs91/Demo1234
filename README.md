const {
    contextBridge,
    ipcRenderer
} = require('electron');


contextBridge.exposeInMainWorld(
    'serviceCall',
    {

        /* -------------------------
           INSTANCE / CONNECTION
        ------------------------- */

        saveInstance:
            (instanceUrl) =>
                ipcRenderer.invoke(
                    'servicecall-save-instance',
                    instanceUrl
                ),


        getInstance:
            () =>
                ipcRenderer.invoke(
                    'servicecall-get-instance'
                ),


        getConnectionStatus:
            () =>
                ipcRenderer.invoke(
                    'servicecall-get-connection-status'
                ),


        /* -------------------------
           AUTHENTICATION
        ------------------------- */

        startLogin:
            () =>
                ipcRenderer.invoke(
                    'servicecall-start-login'
                ),


        onAuthStatus:
            (callback) => {

                ipcRenderer.on(
                    'servicecall-auth-status',
                    (
                        event,
                        data
                    ) => {

                        callback(
                            data
                        );
                    }
                );
            },


        /* -------------------------
           ACTIVE CALL
        ------------------------- */

        openActiveCall:
            () =>
                ipcRenderer.invoke(
                    'servicecall-open-active-call'
                ),


        /* -------------------------
           CALL ACTIONS
        ------------------------- */

        acceptCall:
            (callSysId) =>
                ipcRenderer.invoke(
                    'servicecall-accept-call',
                    callSysId
                ),


        declineCall:
            (callSysId) =>
                ipcRenderer.invoke(
                    'servicecall-decline-call',
                    callSysId
                ),


        cancelCall:
            (callSysId) =>
                ipcRenderer.invoke(
                    'servicecall-cancel-call',
                    callSysId
                ),


        endCall:
            (callSysId) =>
                ipcRenderer.invoke(
                    'servicecall-end-call',
                    callSysId
                ),

        leaveCall:
    (callSysId) =>
        ipcRenderer.invoke(
            'servicecall-leave-call',
            callSysId
        ),


        getCallStatus:
            (callSysId) =>
                ipcRenderer.invoke(
                    'servicecall-get-call-status',
                    callSysId
                ),


        /* -------------------------
           PARTICIPANTS
        ------------------------- */

        searchUsers:
            (searchText) =>
                ipcRenderer.invoke(
                    'servicecall-search-users',
                    searchText
                ),


        inviteParticipant:
            (
                callSysId,
                userSysId
            ) =>
                ipcRenderer.invoke(
                    'servicecall-invite-participant',
                    callSysId,
                    userSysId
                ),


        /* -------------------------
           DYNAMIC MEDIA CREDENTIALS
        ------------------------- */

        getMediaCredentials:
    (callSysId) =>
        ipcRenderer.invoke(
            'servicecall-get-media-credentials',
            callSysId
        ),


/* -------------------------
   SCREEN SHARING
------------------------- */

getScreenSources:
    () =>
        ipcRenderer.invoke(
            'servicecall-get-screen-sources'
        ),

        setCallWindowLayout:
    (layout) =>
        ipcRenderer.invoke(
            'servicecall-set-call-window-layout',
            layout
        ),


/* -------------------------
   RECORDING
------------------------- */

startRecording:
    (callSysId) =>
        ipcRenderer.invoke(
            'servicecall-start-recording',
            callSysId
        ),


finishRecording:
    (recordingSysId) =>
        ipcRenderer.invoke(
            'servicecall-finish-recording',
            recordingSysId
        ),

uploadRecording:
    (
        recordingSysId,
        fileData,
        fileName,
        format
    ) =>
        ipcRenderer.invoke(
            'servicecall-upload-recording',
            recordingSysId,
            fileData,
            fileName,
            format
        ),

finalizeVoiceRecording:
    (
        recordingSysId,
        webmData
    ) =>
        ipcRenderer.invoke(
            'servicecall-finalize-voice-recording',
            recordingSysId,
            webmData
        ),

        finalizeScreenRecording:
    (
        recordingSysId,
        webmData
    ) =>
        ipcRenderer.invoke(
            'servicecall-finalize-screen-recording',
            recordingSysId,
            webmData
        ),

        downloadRecording:
    (recordingSysId) =>
        ipcRenderer.invoke(
            'servicecall-download-recording',
            recordingSysId
        ),


completeRecording:
    (
        recordingSysId,
        attachmentSysId,
        format
    ) =>
        ipcRenderer.invoke(
            'servicecall-complete-recording',
            recordingSysId,
            attachmentSysId,
            format
        )

    }
);

