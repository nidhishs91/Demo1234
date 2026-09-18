    callWindow.loadFile(
        'call-window.html',
        {
            query: {

    mode:
        mode,

    callSysId:
        callSysId,

    callNumber:
        callData.callNumber ||
        '',

    name:
        callData.name ||
        'Unknown User',

    department:
        callData.department ||
        '',

    isConference:
        callData.isConference
            ? 'true'
            : 'false'
}
        }
    );


    callWindow.once(
        'ready-to-show',
        () => {

            if (
                callWindow &&
                !callWindow.isDestroyed()
            ) {

                callWindow.show();
                callWindow.focus();
            }
        }
    );

    callWindow.on(
    'close',
    async (event) => {

        /*
         * Allow the window to actually close
         * after we finish our own handling.
         */
        if (callWindowClosing) {
            return;
        }


        event.preventDefault();


        /*
         * No call sys_id means there is nothing
         * to update in ServiceNow.
         */
        if (!callSysId) {

            callWindowClosing = true;

            callWindow.close();

            return;
        }


        try {

            const result =
                await serviceCallApiRequest(
                    '/call-status?call_sys_id=' +
                    encodeURIComponent(
                        callSysId
                    ),
                    'GET'
                );


            const state =
                result.state || '';

/*
             * -------------------------------------------------
             * RINGING / INVITED PARTICIPANT
             * -------------------------------------------------
             *
             * Direct outgoing:
             * X = Cancel Call
             *
             * Direct incoming:
             * X = Decline Call
             *
             * Conference invitation:
             * Call itself may already be connected,
             * but this participant is still ringing.
             * X = Decline conference invitation.
             */
            const participantStatus =
                result.participant_status || '';


            if (
                participantStatus === 'ringing' ||
                participantStatus === 'invited'
            ) {

                /*
                 * Original caller cancelling
                 * an unanswered direct call.
                 */
                if (
                    mode === 'calling' &&
                    state === 'ringing'
                ) {

                    await serviceCallApiRequest(
                        '/cancel-call',
                        'POST',
                        {
                            call_sys_id:
                                callSysId
                        }
                    );

                } else {

                    /*
                     * Direct incoming receiver
                     * OR conference invite receiver.
                     */
                    await serviceCallApiRequest(
                        '/decline-call',
                        'POST',
                        {
                            call_sys_id:
                                callSysId
                        }
                    );
                }


                callWindowClosing =
                    true;

                callWindow.close();

                return;
            }


            /*
             * -------------------------------------------------
             * CONNECTED
             * -------------------------------------------------
             *
             * X does NOT end or leave a connected call.
             *
             * It only hides the call window.
             * The call/audio continues in the background.
             */
            if (
                state === 'connected' &&
                participantStatus === 'connected'
            ) {

                callWindow.hide();

                return;
            }


            /*
             * -------------------------------------------------
             * TERMINAL STATES
             * -------------------------------------------------
             *
             * The call has already finished,
             * so the window can close normally.
             */
            if (
                state === 'completed' ||
                state === 'cancelled' ||
                state === 'declined' ||
                state === 'failed'
            ) {

                callWindowClosing =
                    true;

                callWindow.close();

                return;
            }


            /*
             * Unknown/unexpected state:
             *
             * Safest behavior is to hide the
             * window rather than accidentally
             * terminating a live call.
             */
            callWindow.hide();


        } catch (error) {

            console.error(
                'ServiceCall window close handling failed:',
                error
            );


            /*
             * If ServiceNow cannot be reached,
             * do NOT accidentally terminate
             * a live call.
             */
            if (
                callWindow &&
                !callWindow.isDestroyed()
            ) {

                callWindow.hide();
            }
        }
    }
);


    callWindow.on(
        'closed',
        () => {

            callWindow =
                null;

            callWindowClosing =
                false;

            activeCallWindowId =
                null;


            if (
                activeIncomingCallId ===
                callSysId
            ) {

                activeIncomingCallId =
                    null;
            }


            if (
                activeOutgoingCallId ===
                callSysId
            ) {

                activeOutgoingCallId =
                    null;
            }
        }
    );
}
        
async function serviceCallApiRequest(
    pathName,
    method = 'GET',
    body = null,
    allowRefresh = true
) {
 
    let config =
        loadConfig();

    const validAccessToken =
    await ensureValidAccessToken();
 
    if (
        !config ||
        !config.instanceUrl ||
        !config.accessToken
    ) {
        throw new Error(
            'ServiceCall Desktop is not connected to ServiceNow.'
        );
    }
 
 
    const url =
        config.instanceUrl.replace(/\/$/, '') +
        '/api/x_1806573_servic_0/servicecall_desktop_api' +
        pathName;
 
 
    const options = {
        method: method,
 
        headers: {
            'Accept':
                'application/json',
 
            'Authorization':
                'Bearer ' +
                validAccessToken
        }
    };
 
 
    if (body) {
 
        options.headers['Content-Type'] =
            'application/json';
 
        options.body =
            JSON.stringify(body);
    }
 
 
    let response =
        await fetch(
            url,
            options
        );
 
 
    /*
     * -------------------------------------------------
     * ACCESS TOKEN EXPIRED
     * -------------------------------------------------
     *
     * Try ONE automatic refresh.
     *
     * We only retry once so a bad/revoked refresh
     * token cannot create an infinite loop.
     */
    if (
        (
            response.status === 401 ||
            response.status === 403
        ) &&
        allowRefresh
    ) {
 
        console.log(
            'ServiceCall API authorization expired. Trying automatic renewal...'
        );
 
 
        try {
 
            const newAccessToken =
                await refreshAccessToken();
 
 
            /*
             * Retry the ORIGINAL request using
             * the newly issued access token.
             */
            options.headers['Authorization'] =
                'Bearer ' +
                newAccessToken;
 
 
            response =
                await fetch(
                    url,
                    options
                );
 
 
        } catch (refreshError) {
 
            console.error(
                'Automatic ServiceCall authorization renewal failed:',
                refreshError.message
            );
 
 
            const error =
                new Error(
                    'Your ServiceCall authorization has expired. Please sign in to ServiceNow again.'
                );
 
            error.code =
                'AUTHENTICATION_REQUIRED';
 
            throw error;
        }
    }
 
 
    let data = {};
 
    try {
 
        data =
            await response.json();
 
    } catch (error) {
 
        data = {};
    }
 
 
    const result =
        data.result || data;
 
 
    /*
     * If we're STILL unauthorized after refreshing,
     * the long-lived authorization is no longer usable.
     */
    if (
        response.status === 401 ||
        response.status === 403
    ) {
 
        const error =
            new Error(
                'Your ServiceCall authorization has expired. Please sign in to ServiceNow again.'
            );
 
        error.code =
            'AUTHENTICATION_REQUIRED';
 
        throw error;
    }
 
 
    if (!response.ok) {
 
        const error =
            new Error(
                result.message ||
                'ServiceCall request failed.'
            );
 
        error.code =
            result.code ||
            'SERVICECALL_API_ERROR';
 
        throw error;
    }
 
 
    return result;
}

ipcMain.handle(
    'servicecall-accept-call',
    async (
        event,
        callSysId
    ) => {

        return await serviceCallApiRequest(
            '/accept-call',
            'POST',
            {
                call_sys_id:
                    callSysId
            }
        );
    }
);


ipcMain.handle(
    'servicecall-decline-call',
    async (
        event,
        callSysId
    ) => {

        return await serviceCallApiRequest(
            '/decline-call',
            'POST',
            {
                call_sys_id:
                    callSysId
            }
        );
    }
);


ipcMain.handle(
    'servicecall-cancel-call',
    async (
        event,
        callSysId
    ) => {

        return await serviceCallApiRequest(
            '/cancel-call',
            'POST',
            {
                call_sys_id:
                    callSysId
            }
        );
    }
);


ipcMain.handle(
    'servicecall-end-call',
    async (
        event,
        callSysId
    ) => {

        return await serviceCallApiRequest(
            '/end-call',
            'POST',
            {
                call_sys_id:
                    callSysId
            }
        );
    }
);


ipcMain.handle(
    'servicecall-get-call-status',
    async (
        event,
        callSysId
    ) => {

        return await serviceCallApiRequest(
            '/call-status?call_sys_id=' +
            encodeURIComponent(
                callSysId
            ),
            'GET'
        );
    }
);

async function checkOutgoingCallOnce() {
 
    try {
 
        const result =
            await serviceCallApiRequest(
                '/outgoing-call',
                'GET'
            );
 
 
        if (
            result &&
            result.success &&
            result.outgoing_call
        ) {
 
            const outgoingCallId =
                result.call_sys_id;
 
 
            if (
                outgoingCallId &&
                outgoingCallId !==
                    activeOutgoingCallId
            ) {
 
                activeOutgoingCallId =
                    outgoingCallId;
 
 
                console.log(
                    'Outgoing ServiceCall:',
                    result
                );
 
 
                showCallWindow(
                    result.state === 'connected'
                        ? 'connected'
                        : 'calling',
 
                    {
                        callSysId:
                            result.call_sys_id,
 
                        callNumber:
                            result.call_number,
 
                        name:
                            result.target_user_name ||
                            'Unknown User',
 
                        department:
                            result.target_department ||
                            ''
                    }
                );
            }
 
 
            return;
        }
 
 
        /*
         * No outgoing ringing/connected call.
         */
        activeOutgoingCallId =
            null;
 
 
    } catch (error) {
 
        console.error(
            'Outgoing call check failed:',
            error
        );
 
 
        if (
            error.code ===
            'AUTHENTICATION_REQUIRED'
        ) {
 
            stopOutgoingCallLoop();
 
 
            sendAuthStatus(
                'authentication_required',
                'Your ServiceCall authorization has expired. Please sign in to ServiceNow again.'
            );
        }
    }
}

function startOutgoingCallLoop() {

    stopOutgoingCallLoop();

    checkOutgoingCallOnce();

    outgoingCallTimer =
        setInterval(
            checkOutgoingCallOnce,
            3000
        );
}


function stopOutgoingCallLoop() {

    if (outgoingCallTimer) {

        clearInterval(
            outgoingCallTimer
        );

        outgoingCallTimer =
            null;
    }
}

ipcMain.handle(
    'servicecall-open-active-call',
    async () => {

        if (
            callWindow &&
            !callWindow.isDestroyed()
        ) {

            if (callWindow.isMinimized()) {
                callWindow.restore();
            }

            callWindow.show();
            callWindow.focus();

            return {
                success: true,
                active_call: true
            };
        }


        return {
            success: true,
            active_call: false,
            message: 'No active call window is currently available.'
        };
    }
);

/* -------------------------------------------------------
   DYNAMIC MEDIA CREDENTIALS
------------------------------------------------------- */

ipcMain.handle(
    'servicecall-get-media-credentials',

    async (
        event,
        callSysId
    ) => {

        if (!callSysId) {

            return {
                success: false,
                code: 'CALL_ID_REQUIRED',
                message:
                    'Call ID was not provided.'
            };
        }


        try {

            const result =
                await serviceCallApiRequest(
                    '/media-credentials?call_sys_id=' +
                    encodeURIComponent(
                        callSysId
                    ),
                    'GET'
                );


            return result;


        } catch (error) {

            console.error(
                'Unable to get ServiceCall media credentials:',
                error.message
            );


            return {
                success: false,

                code:
                    error.code ||
                    'MEDIA_CREDENTIALS_FAILED',

                message:
                    error.message ||
                    'Unable to obtain media credentials.'
            };
        }
    }
);

ipcMain.handle(
    'servicecall-invite-participant',

    async (
        event,
        callSysId,
        userSysId
    ) => {

        if (
            !callSysId ||
            !userSysId
        ) {

            return {
                success: false,
                code:
                    'INVITE_DATA_REQUIRED',
                message:
                    'Call ID and user ID are required.'
            };
        }


        try {

            const result =
                await serviceCallApiRequest(
                    '/invite-participant',
                    'POST',
                    {
                        call_sys_id:
                            callSysId,

                        user_sys_id:
                            userSysId
                    }
                );


            return result;


        } catch (error) {

            console.error(
                'Unable to invite ServiceCall participant:',
                error.message
            );


            return {
                success: false,
                code:
                    error.code ||
                    'INVITE_PARTICIPANT_FAILED',
                message:
                    error.message ||
                    'Unable to invite participant.'
            };
        }
    }
);

ipcMain.handle(
    'servicecall-search-users',

    async (
        event,
        searchText
    ) => {

        const search =
            String(
                searchText || ''
            ).trim();


        if (
            search.length < 2
        ) {

            return {
                success: true,
                users: []
            };
        }


        try {

            return await serviceCallApiRequest(
                '/users?search=' +
                encodeURIComponent(
                    search
                ),
                'GET'
            );


        } catch (error) {

            console.error(
                'Unable to search ServiceCall users:',
                error.message
            );


            return {
                success: false,

                code:
                    error.code ||
                    'USER_SEARCH_FAILED',

                message:
                    error.message ||
                    'Unable to search users.',

                users: []
            };
        }
    }
);

ipcMain.handle(
    'servicecall-leave-call',

    async (
        event,
        callSysId
    ) => {

        if (!callSysId) {

            return {
                success: false,
                code: 'CALL_ID_REQUIRED',
                message:
                    'Call ID was not provided.'
            };
        }


        try {

            return await serviceCallApiRequest(
                '/leave-call',
                'POST',
                {
                    call_sys_id:
                        callSysId
                }
            );


        } catch (error) {

            console.error(
                'Unable to leave ServiceCall:',
                error.message
            );


            return {
                success: false,

                code:
                    error.code ||
                    'LEAVE_CALL_FAILED',

                message:
                    error.message ||
                    'Unable to leave the call.'
            };
        }
    }
);

/* -------------------------------------------------------
   UPLOAD RECORDING ATTACHMENT
------------------------------------------------------- */

async function uploadRecordingAttachment(
    recordingSysId,
    fileData,
    fileName,
    format
) {

    if (!recordingSysId) {
        throw new Error(
            'Recording ID was not provided.'
        );
    }


    if (!fileData) {
        throw new Error(
            'Recording file data was not provided.'
        );
    }


    const normalizedFormat =
        String(
            format || ''
        )
            .trim()
            .toLowerCase();


    if (
        normalizedFormat !== 'mp3' &&
        normalizedFormat !== 'mp4'
    ) {

        throw new Error(
            'Recording format must be mp3 or mp4.'
        );
    }


    const expectedExtension =
        '.' + normalizedFormat;


    let safeFileName =
        String(
            fileName || ''
        )
            .trim()
            .replace(
                /[^a-zA-Z0-9._-]/g,
                '_'
            );


    if (!safeFileName) {

        safeFileName =
            'servicecall-recording-' +
            Date.now() +
            expectedExtension;
    }


    /*
     * Ensure the filename agrees with the
     * recording format.
     */
    if (
        !safeFileName
            .toLowerCase()
            .endsWith(
                expectedExtension
            )
    ) {

        safeFileName +=
            expectedExtension;
    }


    const config =
        loadConfig();


    if (
        !config ||
        !config.instanceUrl
    ) {

        throw new Error(
            'ServiceCall Desktop is not connected to ServiceNow.'
        );
    }


    /*
     * Get a valid OAuth access token.
     *
     * The renderer never receives this token.
     */
    let accessToken =
        await ensureValidAccessToken();


    /*
     * IPC can give us a Uint8Array rather
     * than a Node Buffer.
     */
    const recordingBuffer =
        Buffer.isBuffer(
            fileData
        )
            ? fileData
            : Buffer.from(
                fileData
            );


    if (
        recordingBuffer.length <= 0
    ) {

        throw new Error(
            'Recording file is empty.'
        );
    }


    const contentType =
        normalizedFormat === 'mp3'
            ? 'audio/mpeg'
            : 'video/mp4';


    const tableName =
        'x_1806573_servic_0_servicecall_recording';


    const uploadUrl =
        config.instanceUrl
            .replace(
                /\/$/,
                ''
            ) +
        '/api/now/attachment/file' +
        '?table_name=' +
        encodeURIComponent(
            tableName
        ) +
        '&table_sys_id=' +
        encodeURIComponent(
            recordingSysId
        ) +
        '&file_name=' +
        encodeURIComponent(
            safeFileName
        );


    async function performUpload(
        token
    ) {

        return await fetch(
            uploadUrl,
            {
                method: 'POST',

                headers: {

                    'Authorization':
                        'Bearer ' +
                        token,

                    'Accept':
                        'application/json',

                    'Content-Type':
                        contentType
                },

                /*
                 * IMPORTANT:
                 *
                 * Raw binary.
                 * No JSON.
                 * No Base64.
                 */
                body:
                    recordingBuffer
            }
        );
    }


    let response =
        await performUpload(
            accessToken
        );


    /*
     * If OAuth expired between obtaining the
     * token and uploading the file, refresh
     * once and retry the same binary upload.
     */
    if (
        response.status === 401 ||
        response.status === 403
    ) {

        console.log(
            'Recording upload authorization expired. Attempting automatic renewal...'
        );


        try {

            accessToken =
                await refreshAccessToken();


            response =
                await performUpload(
                    accessToken
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


    let data = {};


    try {

        data =
            responseText
                ? JSON.parse(
                    responseText
                )
                : {};

    } catch (error) {

        throw new Error(
            'ServiceNow returned an invalid attachment upload response.'
        );
    }


    const result =
        data.result ||
        data;


    if (!response.ok) {

        const uploadError =
            new Error(
                (
                    result &&
                    result.error &&
                    (
                        result.error.message ||
                        result.error.detail
                    )
                ) ||
                (
                    result &&
                    result.message
                ) ||
                (
                    data &&
                    data.error &&
                    (
                        data.error.message ||
                        data.error.detail
                    )
                ) ||
                'ServiceNow recording upload failed.'
            );


        uploadError.code =
            'RECORDING_UPLOAD_FAILED';


        throw uploadError;
    }


    if (
        !result ||
        !result.sys_id
    ) {

        throw new Error(
            'ServiceNow did not return an attachment ID.'
        );
    }


    /*
     * Extra verification using the metadata
     * ServiceNow returned from Attachment API.
     */
    if (
        result.table_sys_id &&
        result.table_sys_id !==
            recordingSysId
    ) {

        throw new Error(
            'Uploaded attachment was associated with the wrong recording.'
        );
    }


    if (
        result.table_name &&
        result.table_name !==
            tableName
    ) {

        throw new Error(
            'Uploaded attachment was associated with the wrong table.'
        );
    }


    console.log(
        'ServiceCall recording uploaded successfully.',
        'Attachment:',
        result.sys_id,
        'Size:',
        result.size_bytes || recordingBuffer.length
    );


    /*
     * Never return the OAuth token.
     */
    return {

        success: true,

        attachment_sys_id:
            result.sys_id,

        file_name:
            result.file_name ||
            safeFileName,

        file_size:
            Number(
                result.size_bytes ||
                recordingBuffer.length
            ),

        content_type:
            result.content_type ||
            contentType
    };
}

ipcMain.handle(
    'servicecall-upload-recording',

    async (
        event,
        recordingSysId,
        fileData,
        fileName,
        format
    ) => {

        try {

            return await uploadRecordingAttachment(
                recordingSysId,
                fileData,
                fileName,
                format
            );


        } catch (error) {

            console.error(
                'Unable to upload ServiceCall recording:',
                error.message
            );


            return {

                success: false,

                code:
                    error.code ||
                    'RECORDING_UPLOAD_FAILED',

                message:
                    error.message ||
                    'Unable to upload recording.'
            };
        }
    }
);

/* -------------------------------------------------------
   SERVICECALL RECORDING
------------------------------------------------------- */

/*
 * Create the ServiceCall Recording record.
 *
 * This does NOT start local media capture by itself.
 * It creates the authoritative recording session
 * in ServiceNow.
 */
ipcMain.handle(
    'servicecall-start-recording',

    async (
        event,
        callSysId
    ) => {

        if (!callSysId) {

            return {
                success: false,
                code: 'CALL_ID_REQUIRED',
                message:
                    'Call ID was not provided.'
            };
        }


        try {

            return await serviceCallApiRequest(
                '/start-recording',
                'POST',
                {
                    call_sys_id:
                        callSysId
                }
            );


        } catch (error) {

            console.error(
                'Unable to start ServiceCall recording:',
                error.message
            );


            return {
                success: false,

                code:
                    error.code ||
                    'START_RECORDING_FAILED',

                message:
                    error.message ||
                    'Unable to start recording.'
            };
        }
    }
);


/*
 * Tell ServiceNow that media capture has stopped
 * and the recording is now being processed.
 */
ipcMain.handle(
    'servicecall-finish-recording',

    async (
        event,
        recordingSysId
    ) => {

        if (!recordingSysId) {

            return {
                success: false,
                code: 'RECORDING_ID_REQUIRED',
                message:
                    'Recording ID was not provided.'
            };
        }


        try {

            return await serviceCallApiRequest(
                '/finish-recording',
                'POST',
                {
                    recording_sys_id:
                        recordingSysId
                }
            );


        } catch (error) {

            console.error(
                'Unable to finish ServiceCall recording:',
                error.message
            );


            return {
                success: false,

                code:
                    error.code ||
                    'FINISH_RECORDING_FAILED',

                message:
                    error.message ||
                    'Unable to finish recording.'
            };
        }
    }
);


/*
 * After the binary file has been successfully
 * uploaded to sys_attachment, this finalizes the
 * ServiceCall Recording record.
 */
ipcMain.handle(
    'servicecall-complete-recording',

    async (
        event,
        recordingSysId,
        attachmentSysId,
        format
    ) => {

        if (
            !recordingSysId ||
            !attachmentSysId ||
            !format
        ) {

            return {
                success: false,
                code: 'RECORDING_DATA_REQUIRED',
                message:
                    'Recording ID, attachment ID and format are required.'
            };
        }


        try {

            return await serviceCallApiRequest(
                '/complete-recording',
                'POST',
                {
                    recording_sys_id:
                        recordingSysId,

                    attachment_sys_id:
                        attachmentSysId,

                    format:
                        format
                }
            );


        } catch (error) {

            console.error(
                'Unable to complete ServiceCall recording:',
                error.message
            );


            return {
                success: false,

                code:
                    error.code ||
                    'COMPLETE_RECORDING_FAILED',

                message:
                    error.message ||
                    'Unable to complete recording.'
            };
        }
    }
);

/* -------------------------------------------------------
   FINALIZE VOICE RECORDING
------------------------------------------------------- */

ipcMain.handle(
    'servicecall-finalize-voice-recording',

    async (
        event,
        recordingSysId,
        webmData
    ) => {

        if (!recordingSysId) {

            return {
                success: false,
                code:
                    'RECORDING_ID_REQUIRED',
                message:
                    'Recording ID was not provided.'
            };
        }


        if (!webmData) {

            return {
                success: false,
                code:
                    'RECORDING_DATA_REQUIRED',
                message:
                    'Recording data was not provided.'
            };
        }


        try {

            /*
             * IPC structured cloning commonly
             * gives the main process a Uint8Array.
             */
            const webmBuffer =
                Buffer.isBuffer(
                    webmData
                )
                    ? webmData
                    : Buffer.from(
                        webmData
                    );


            if (
                webmBuffer.length <= 0
            ) {

                throw new Error(
                    'Recording data is empty.'
                );
            }


            console.log(
                'ServiceCall voice recording received.',
                'WebM size:',
                webmBuffer.length
            );


            /* -----------------------------------------
               1. WEBM -> REAL MP3
            ----------------------------------------- */

            const converted =
                await convertWebmToMp3(
                    webmBuffer
                );


            if (
                !converted ||
                !converted.success ||
                !converted.buffer
            ) {

                throw new Error(
                    'Unable to convert ServiceCall recording to MP3.'
                );
            }


            /* -----------------------------------------
               2. MARK RECORDING PROCESSING
            ----------------------------------------- */

            const finishResult =
                await serviceCallApiRequest(
                    '/finish-recording',
                    'POST',
                    {
                        recording_sys_id:
                            recordingSysId
                    }
                );


            if (
                !finishResult ||
                finishResult.success !== true
            ) {

                throw new Error(
                    finishResult &&
                    finishResult.message
                        ? finishResult.message
                        : 'Unable to finish ServiceCall recording.'
                );
            }


            /* -----------------------------------------
               3. UPLOAD FINAL MP3
            ----------------------------------------- */

            const fileName =
                'servicecall-recording-' +
                recordingSysId +
                '.mp3';


            const uploadResult =
                await uploadRecordingAttachment(
                    recordingSysId,
                    converted.buffer,
                    fileName,
                    'mp3'
                );


            if (
                !uploadResult ||
                !uploadResult.success ||
                !uploadResult.attachment_sys_id
            ) {

                throw new Error(
                    uploadResult &&
                    uploadResult.message
                        ? uploadResult.message
                        : 'Unable to upload ServiceCall recording.'
                );
            }


            /* -----------------------------------------
               4. COMPLETE RECORDING
            ----------------------------------------- */

            const completeResult =
                await serviceCallApiRequest(
                    '/complete-recording',
                    'POST',
                    {
                        recording_sys_id:
                            recordingSysId,

                        attachment_sys_id:
                            uploadResult
                                .attachment_sys_id,

                        format:
                            'mp3'
                    }
                );


            if (
                !completeResult ||
                completeResult.success !== true
            ) {

                throw new Error(
                    completeResult &&
                    completeResult.message
                        ? completeResult.message
                        : 'Unable to complete ServiceCall recording.'
                );
            }


            console.log(
                'ServiceCall voice recording finalized successfully.',
                recordingSysId
            );


            return {

                success: true,

                code:
                    'VOICE_RECORDING_AVAILABLE',

                recording_sys_id:
                    recordingSysId,

                attachment_sys_id:
                    uploadResult
                        .attachment_sys_id,

                format:
                    'mp3',

                file_name:
                    uploadResult.file_name,

                file_size:
                    uploadResult.file_size,

                status:
                    completeResult.status ||
                    'available',

                expires_at:
                    completeResult.expires_at ||
                    ''
            };


        } catch (error) {

            console.error(
                'ServiceCall voice recording finalization failed:',
                error
            );


            return {

                success: false,

                code:
                    error.code ||
                    'VOICE_RECORDING_FINALIZATION_FAILED',

                message:
                    error.message ||
                    'Unable to finalize ServiceCall recording.'
            };
        }
    }
);

/* -------------------------------------------------------
   FINALIZE SCREEN RECORDING
------------------------------------------------------- */

/*
 * Finalizes a ServiceCall recording that contained
 * screen sharing at least once.
 *
 * Renderer WebM:
 *   VP8 canvas video
 *   +
 *   Opus mixed conference audio
 *
 * Main process:
 *   WebM
 *   -> FFmpeg
 *   -> H.264 + AAC MP4
 *   -> /finish-recording
 *   -> ServiceNow Attachment API
 *   -> /complete-recording
 */
ipcMain.handle(
    'servicecall-finalize-screen-recording',

    async (
        event,
        recordingSysId,
        webmData
    ) => {

        if (!recordingSysId) {

            return {
                success: false,

                code:
                    'RECORDING_ID_REQUIRED',

                message:
                    'Recording ID was not provided.'
            };
        }


        if (!webmData) {

            return {
                success: false,

                code:
                    'RECORDING_DATA_REQUIRED',

                message:
                    'Recording data was not provided.'
            };
        }


        try {

            /*
             * IPC structured cloning normally
             * gives the main process a Uint8Array.
             */
            const webmBuffer =
                Buffer.isBuffer(
                    webmData
                )
                    ? webmData
                    : Buffer.from(
                        webmData
                    );


            if (
                webmBuffer.length <= 0
            ) {

                throw new Error(
                    'Recording data is empty.'
                );
            }


            console.log(
                'ServiceCall screen recording received.',
                'WebM size:',
                webmBuffer.length
            );


            /* -----------------------------------------
               1. WEBM -> REAL MP4
            ----------------------------------------- */

            const converted =
                await convertWebmToMp4(
                    webmBuffer
                );


            if (
                !converted ||
                !converted.success ||
                !converted.buffer
            ) {

                throw new Error(
                    'Unable to convert ServiceCall recording to MP4.'
                );
            }


            console.log(
                'ServiceCall screen recording converted.',
                'MP4 size:',
                converted.size
            );


            /* -----------------------------------------
               2. MARK RECORDING PROCESSING
            ----------------------------------------- */

            const finishResult =
                await serviceCallApiRequest(
                    '/finish-recording',
                    'POST',
                    {
                        recording_sys_id:
                            recordingSysId
                    }
                );


            if (
                !finishResult ||
                finishResult.success !== true
            ) {

                throw new Error(
                    finishResult &&
                    finishResult.message
                        ? finishResult.message
                        : 'Unable to finish ServiceCall recording.'
                );
            }


            /* -----------------------------------------
               3. UPLOAD FINAL MP4
            ----------------------------------------- */

            const fileName =
                'servicecall-recording-' +
                recordingSysId +
                '.mp4';


            const uploadResult =
                await uploadRecordingAttachment(
                    recordingSysId,
                    converted.buffer,
                    fileName,
                    'mp4'
                );


            if (
                !uploadResult ||
                !uploadResult.success ||
                !uploadResult.attachment_sys_id
            ) {

                throw new Error(
                    uploadResult &&
                    uploadResult.message
                        ? uploadResult.message
                        : 'Unable to upload ServiceCall screen recording.'
                );
            }


            /* -----------------------------------------
               4. COMPLETE RECORDING
            ----------------------------------------- */

            const completeResult =
                await serviceCallApiRequest(
                    '/complete-recording',
                    'POST',
                    {
                        recording_sys_id:
                            recordingSysId,

                        attachment_sys_id:
                            uploadResult
                                .attachment_sys_id,

                        format:
                            'mp4'
                    }
                );


            if (
                !completeResult ||
                completeResult.success !== true
            ) {

                throw new Error(
                    completeResult &&
                    completeResult.message
                        ? completeResult.message
                        : 'Unable to complete ServiceCall screen recording.'
                );
            }


            console.log(
                'ServiceCall screen recording finalized successfully.',
                recordingSysId
            );


            return {

                success:
                    true,

                code:
                    'SCREEN_RECORDING_AVAILABLE',

                recording_sys_id:
                    recordingSysId,

                attachment_sys_id:
                    uploadResult
                        .attachment_sys_id,

                format:
                    'mp4',

                file_name:
                    uploadResult.file_name,

                file_size:
                    uploadResult.file_size,

                status:
                    completeResult.status ||
                    'available',

                expires_at:
                    completeResult.expires_at ||
                    ''
            };


        } catch (error) {

            console.error(
                'ServiceCall screen recording finalization failed:',
                error
            );


            return {

                success:
                    false,

                code:
                    error.code ||
                    'SCREEN_RECORDING_FINALIZATION_FAILED',

                message:
                    error.message ||
                    'Unable to finalize ServiceCall screen recording.'
            };
        }
    }
);

/* -------------------------------------------------------
   SERVICECALL SCREEN SHARE SOURCES
------------------------------------------------------- */

/*
 * Return the screens/windows that Electron
 * can capture.
 *
 * IMPORTANT:
 *
 * We return only serializable metadata to
 * the renderer.
 *
 * The renderer will use the selected source ID
 * to request the actual MediaStream.
 */
ipcMain.handle(
    'servicecall-get-screen-sources',

    async () => {

        try {

            const sources =
                await desktopCapturer
                    .getSources({
                        types: [
                            'screen',
                            'window'
                        ],

                        thumbnailSize: {
                            width: 320,
                            height: 180
                        },

                        fetchWindowIcons: true
                    });


            const safeSources =
                sources.map(
                    (source) => {

                        return {

                            id:
                                source.id,

                            name:
                                source.name,

                            thumbnail:
                                source.thumbnail &&
                                !source.thumbnail.isEmpty()
                                    ? source.thumbnail
                                        .toDataURL()
                                    : '',

                            appIcon:
                                source.appIcon &&
                                !source.appIcon.isEmpty()
                                    ? source.appIcon
                                        .toDataURL()
                                    : ''
                        };
                    }
                );


            console.log(
                'ServiceCall screen sources available:',
                safeSources.length
            );


            return {

                success: true,

                sources:
                    safeSources
            };


        } catch (error) {

            console.error(
                'Unable to retrieve ServiceCall screen sources:',
                error
            );


            return {

                success: false,

                code:
                    'SCREEN_SOURCE_FAILED',

                message:
                    error.message ||
                    'Unable to retrieve screens and windows.',

                sources: []
            };
        }
    }
);

/* -------------------------------------------------------
   SERVICECALL CALL WINDOW LAYOUT
------------------------------------------------------- */

ipcMain.handle(
    'servicecall-set-call-window-layout',

    async (
        event,
        layout
    ) => {

        try {

            const senderWindow =
                BrowserWindow.fromWebContents(
                    event.sender
                );


            if (
                !senderWindow ||
                senderWindow.isDestroyed() ||
                senderWindow !== callWindow
            ) {

                return {
                    success: false,
                    message:
                        'ServiceCall call window is unavailable.'
                };
            }


            if (
                layout === 'screen'
            ) {

                /*
                 * Allow the existing call window
                 * to become a larger screen-share
                 * experience.
                 */
                senderWindow.setResizable(
                    true
                );


                senderWindow.setMinimumSize(
                    760,
                    620
                );


                senderWindow.setSize(
                    1000,
                    760,
                    true
                );


                senderWindow.center();


                return {
                    success: true,
                    layout: 'screen'
                };
            }


            /*
             * Return to compact voice-call mode.
             */
            senderWindow.setMinimumSize(
                440,
                560
            );


            senderWindow.setSize(
                440,
                560,
                true
            );


            senderWindow.setResizable(
                false
            );


            senderWindow.center();


            return {
                success: true,
                layout: 'compact'
            };


        } catch (error) {

            console.error(
                'Unable to change ServiceCall call window layout:',
                error
            );


            return {
                success: false,
                message:
                    error.message ||
                    'Unable to change call window layout.'
            };
        }
    }
);

/* -------------------------------------------------------
   SERVICECALL MEETINGS
------------------------------------------------------- */

/*
 * Get all meetings relevant to the currently
 * authenticated ServiceCall user.
 *
 * ServiceNow decides which meetings the user
 * is allowed to see.
 */
ipcMain.handle(
    'servicecall-get-my-meetings',

    async () => {

        try {

            const result =
                await serviceCallApiRequest(
                    '/my-meetings',
                    'GET'
                );


            return result;


        } catch (error) {

            console.error(
                'Unable to get ServiceCall meetings:',
                error.message
            );


            return {

                success: false,

                code:
                    error.code ||
                    'GET_MEETINGS_FAILED',

                message:
                    error.message ||
                    'Unable to retrieve meetings.',

                meetings: []
            };
        }
    }
);

/* -------------------------------------------------------
   CREATE SERVICECALL MEETING
------------------------------------------------------- */

/*
 * Create a new ServiceCall meeting.
 *
 * The renderer sends the meeting details here.
 * The main process forwards them securely to
 * ServiceNow using the authenticated OAuth session.
 */
ipcMain.handle(
    'servicecall-create-meeting',

    async (
        event,
        meetingData
    ) => {

        try {

            if (
                !meetingData ||
                !meetingData.title ||
                !meetingData.scheduled_start ||
                !meetingData.scheduled_end
            ) {

                return {

                    success: false,

                    code:
                        'MEETING_DATA_REQUIRED',

                    message:
                        'Title, start time and end time are required.'
                };
            }


            const payload = {

    title:
        String(
            meetingData.title
        ).trim(),

    description:
        String(
            meetingData.description || ''
        ).trim(),

    scheduled_start:
        meetingData.scheduled_start,

    scheduled_end:
        meetingData.scheduled_end,

    participants:
        Array.isArray(
            meetingData.participants
        )
            ? meetingData.participants
            : []
};


            const result =
                await serviceCallApiRequest(
                    '/create-meeting',
                    'POST',
                    payload
                );


            return result;


        } catch (error) {

            console.error(
                'Unable to create ServiceCall meeting:',
                error.message
            );


            return {

                success: false,

                code:
                    error.code ||
                    'CREATE_MEETING_FAILED',

                message:
                    error.message ||
                    'Unable to create meeting.'
            };
        }
    }
);

ipcMain.handle(
    'servicecall-start-meeting',

    async (
        event,
        meetingSysId
    ) => {

        try {

            if (!meetingSysId) {

                return {
                    success: false,
                    code: 'MEETING_REQUIRED',
                    message: 'Meeting sys_id is required.'
                };
            }


            const payload = {
                meeting_sys_id:
                    String(
                        meetingSysId
                    ).trim()
            };


            const result =
                await serviceCallApiRequest(
                    '/start-meeting',
                    'POST',
                    payload
                );


            return result;


        } catch (error) {

            console.error(
                'Unable to start ServiceCall meeting:',
                error.message
            );


            return {
                success: false,
                code:
                    error.code ||
                    'START_MEETING_FAILED',
                message:
                    error.message ||
                    'Unable to start meeting.'
            };
        }
    }
);
