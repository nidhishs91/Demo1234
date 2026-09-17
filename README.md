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
