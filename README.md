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


const {
    app,
    BrowserWindow,
    ipcMain,
    shell,
    Tray,
    Menu,
    desktopCapturer,
} = require('electron');

const path = require('path');
const fs = require('fs');
const crypto = require('crypto');
const http = require('http');
const os = require('os');

const {
    convertWebmToMp3,
    convertWebmToMp4
} = require(
    './media-converter'
);
