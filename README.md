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

let mainWindow = null;
let callbackServer = null;
let heartbeatTimer = null;
let tray = null;
let isQuitting = false;
let incomingCallTimer = null;
let activeIncomingCallId = null;
let outgoingCallTimer = null;
let activeOutgoingCallId = null;
let activeCallWindowId = null;
let callWindowClosing = false;

const CALLBACK_HOST = '127.0.0.1';
const CALLBACK_PORT = 42813;
const SERVICECALL_PROTOCOL = 'servicecall';


/* -------------------------------------------------------
   CONFIG
------------------------------------------------------- */

function getConfigPath() {

    return path.join(
        app.getPath('userData'),
        'servicecall-config.json'
    );
}


function saveConfig(config) {

    fs.writeFileSync(
        getConfigPath(),
        JSON.stringify(
            config,
            null,
            2
        ),
        'utf8'
    );
}


function loadConfig() {

    const configPath =
        getConfigPath();

    if (!fs.existsSync(configPath)) {
        return {};
    }

    return JSON.parse(
        fs.readFileSync(
            configPath,
            'utf8'
        )
    );
}

function getOrCreateDeviceId() {

    const config =
        loadConfig();

    if (config.deviceId) {
        return config.deviceId;
    }

    const deviceId =
        crypto.randomUUID();

    config.deviceId =
        deviceId;

    saveConfig(config);

    return deviceId;
}

async function sendHeartbeatOnce() {

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

async function checkIncomingCallOnce() {

    try {

        const result =
            await serviceCallApiRequest(
                '/incoming-call',
                'GET'
            );


        if (
            result &&
            result.success &&
            result.incoming_call
        ) {

            const incomingCallId =
                result.call_sys_id;


            if (
                incomingCallId &&
                incomingCallId !==
                    activeIncomingCallId
            ) {

                activeIncomingCallId =
                    incomingCallId;


                console.log(
                    'Incoming ServiceCall:',
                    result
                );


                const isConference =
                    result.is_conference === true ||
                    result.call_type ===
                        'conference';


                /*
                 * For an existing conference,
                 * show the connected participant
                 * summary returned by ServiceNow.
                 *
                 * Example:
                 *
                 * Nidhish, Divyani
                 *
                 * or
                 *
                 * Nidhish, Divyani +2
                 */
                const displayName =
                    isConference
                        ? (
                            result.conference_display ||
                            'Conference Call'
                        )
                        : (
                            result.caller_name ||
                            'Unknown User'
                        );


                const displayDepartment =
                    isConference
                        ? 'Conference Call'
                        : (
                            result.caller_department ||
                            ''
                        );


                showCallWindow(
                    'incoming',
                    {
                        callSysId:
                            result.call_sys_id,

                        callNumber:
                            result.call_number,

                        name:
                            displayName,

                        department:
                            displayDepartment,

                        isConference:
                            isConference
                    }
                );
            }


            return;
        }


        /*
         * No incoming ringing invitation.
         */
        activeIncomingCallId =
            null;


    } catch (error) {

        console.error(
            'Incoming call check failed:',
            error
        );


        if (
            error.code ===
            'AUTHENTICATION_REQUIRED'
        ) {

            stopIncomingCallLoop();


            sendAuthStatus(
                'authentication_required',
                'Your ServiceCall authorization has expired. Please sign in to ServiceNow again.'
            );
        }
    }
}

function stopIncomingCallLoop() {

    if (incomingCallTimer) {

        clearInterval(
            incomingCallTimer
        );

        incomingCallTimer =
            null;
    }
}

function stopHeartbeatLoop() {

    if (heartbeatTimer) {

        clearInterval(
            heartbeatTimer
        );

        heartbeatTimer = null;
    }
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

function startIncomingCallLoop() {

    stopIncomingCallLoop();

    checkIncomingCallOnce();

    incomingCallTimer =
        setInterval(
            checkIncomingCallOnce,
            3000
        );
}

async function restoreSavedConnection() {

    const config =
        loadConfig();

    if (
        !config.instanceUrl ||
        !config.accessToken
    ) {

        console.log(
            'No saved ServiceCall connection found.'
        );

        return;
    }


    console.log(
        'Saved ServiceCall connection found. Restoring...'
    );


    try {

        await startHeartbeatLoop();

        startIncomingCallLoop();

        startOutgoingCallLoop();


        console.log(
            'ServiceCall connection restored.'
        );

    } catch (error) {

        console.error(
            'Unable to restore ServiceCall connection:',
            error
        );


        sendAuthStatus(
            'warning',
            'Saved ServiceNow connection could not be restored: ' +
            error.message
        );
    }
}

/* -------------------------------------------------------
   INSTANCE URL
------------------------------------------------------- */

function normalizeInstanceUrl(value) {

    if (!value) {
        return '';
    }

    value =
        value.trim();

    if (
        !value.startsWith(
            'https://'
        )
    ) {
        value =
            'https://' + value;
    }

    value =
        value.replace(
            /\/+$/,
            ''
        );

    return value;
}


function isValidServiceNowUrl(value) {

    try {

        const parsed =
            new URL(value);

        return (
            parsed.protocol === 'https:' &&
            parsed.hostname.endsWith(
                '.service-now.com'
            )
        );

    } catch (error) {

        return false;
    }
}


/* -------------------------------------------------------
   PKCE
------------------------------------------------------- */

function base64UrlEncode(buffer) {

    return buffer
        .toString('base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=+$/, '');
}


function generateCodeVerifier() {

    return base64UrlEncode(
        crypto.randomBytes(64)
    );
}


function generateCodeChallenge(
    verifier
) {

    const hash =
        crypto
            .createHash('sha256')
            .update(verifier)
            .digest();

    return base64UrlEncode(
        hash
    );
}


function generateState() {

    return base64UrlEncode(
        crypto.randomBytes(32)
    );
}


/* -------------------------------------------------------
   SEND STATUS TO DESKTOP WINDOW
------------------------------------------------------- */

function sendAuthStatus(
    status,
    message
) {

    if (
        mainWindow &&
        !mainWindow.isDestroyed()
    ) {

        mainWindow.webContents.send(
            'servicecall-auth-status',
            {
                status: status,
                message: message
            }
        );
    }
}


/* -------------------------------------------------------
   TOKEN EXCHANGE
------------------------------------------------------- */

async function exchangeAuthorizationCode(
    code,
    returnedState
) {

    const config =
        loadConfig();

    if (
        !config.oauthState ||
        returnedState !==
            config.oauthState
    ) {

        throw new Error(
            'OAuth state validation failed.'
        );
    }

    if (
        !config.pkceCodeVerifier
    ) {

        throw new Error(
            'PKCE code verifier was not found.'
        );
    }

    const tokenUrl =
        config.instanceUrl +
        '/oauth_token.do';

    const body =
        new URLSearchParams();

    body.set(
        'grant_type',
        'authorization_code'
    );

    body.set(
        'code',
        code
    );

    body.set(
        'redirect_uri',
        config.redirectUri
    );

    body.set(
        'client_id',
        config.oauthClientId
    );

    body.set(
        'code_verifier',
        config.pkceCodeVerifier
    );

    body.set(
        'state',
        returnedState
    );

    const response =
        await fetch(
            tokenUrl,
            {
                method: 'POST',

                headers: {
                    'Content-Type':
                        'application/x-www-form-urlencoded'
                },

                body:
                    body.toString()
            }
        );

    const responseText =
        await response.text();

    let tokenData;

    try {

        tokenData =
            JSON.parse(
                responseText
            );

    } catch (error) {

        throw new Error(
            'ServiceNow returned an invalid token response.'
        );
    }

    if (
        !response.ok ||
        !tokenData.access_token
    ) {

        throw new Error(
            tokenData.error_description ||
            tokenData.error ||
            'Unable to obtain OAuth access token.'
        );
    }

    config.accessToken =
        tokenData.access_token;

    config.tokenType =
        tokenData.token_type ||
        'Bearer';

    config.expiresIn =
        tokenData.expires_in || 0;

    config.tokenObtainedAt =
        Date.now();

    if (tokenData.refresh_token) {
 
    config.refreshToken =
        tokenData.refresh_token;
 
    console.log(
        'ServiceCall refresh token received successfully.'
    );
 
} else {
 
    console.log(
        'ServiceCall refresh token was NOT returned.'
    );
}

    /*
       We no longer need these after
       successful authentication.
    */

    delete config.pkceCodeVerifier;
delete config.oauthState;

saveConfig(config);

await startHeartbeatLoop();

startIncomingCallLoop();
startOutgoingCallLoop();
return tokenData;
}

async function ensureValidAccessToken() {
 
    const config =
        loadConfig();
 
 
    if (!config.accessToken) {
 
        const error =
            new Error(
                'ServiceCall is not authenticated.'
            );
 
        error.code =
            'AUTHENTICATION_REQUIRED';
 
        throw error;
    }
 
 
    /*
     * If expiry information is unavailable,
     * keep using the current token.
     *
     * The normal 401/403 refresh mechanism
     * remains our fallback.
     */
    if (
        !config.expiresIn ||
        !config.tokenObtainedAt
    ) {
        return config.accessToken;
    }
 
 
    const expiresAt =
        config.tokenObtainedAt +
        (Number(config.expiresIn) * 1000);
 
 
    /*
     * Refresh 60 seconds before actual expiry.
     */
    const refreshAt =
        expiresAt - 60000;
 
 
    if (Date.now() >= refreshAt) {
 
        console.log(
            'ServiceCall access token is close to expiry. Renewing automatically...'
        );
 
        return await refreshAccessToken();
    }
 
 
    return config.accessToken;
}

async function refreshAccessToken() {
 
    const config =
        loadConfig();
 
    if (
        !config.instanceUrl ||
        !config.oauthClientId ||
        !config.refreshToken
    ) {
 
        const error =
            new Error(
                'A ServiceCall refresh token is not available.'
            );
 
        error.code =
            'REAUTHENTICATION_REQUIRED';
 
        throw error;
    }
 
 
    console.log(
        'ServiceCall access token expired. Attempting automatic renewal...'
    );
 
 
    const tokenUrl =
        config.instanceUrl.replace(/\/$/, '') +
        '/oauth_token.do';
 
 
    const body =
        new URLSearchParams();
 
    body.set(
        'grant_type',
        'refresh_token'
    );
 
    body.set(
        'refresh_token',
        config.refreshToken
    );
 
    body.set(
        'client_id',
        config.oauthClientId
    );
 
 
    const response =
        await fetch(
            tokenUrl,
            {
                method: 'POST',
 
                headers: {
                    'Content-Type':
                        'application/x-www-form-urlencoded',
 
                    'Accept':
                        'application/json'
                },
 
                body:
                    body.toString()
            }
        );
 
 
    const responseText =
        await response.text();
 
 
    let tokenData;
 
    try {
 
        tokenData =
            JSON.parse(
                responseText
            );
 
    } catch (error) {
 
        const refreshError =
            new Error(
                'ServiceNow returned an invalid token renewal response.'
            );
 
        refreshError.code =
            'TOKEN_REFRESH_FAILED';
 
        throw refreshError;
    }
 
 
    if (
        !response.ok ||
        !tokenData.access_token
    ) {
 
        console.error(
            'ServiceCall automatic token renewal failed.'
        );
 
 
        const refreshError =
            new Error(
                tokenData.error_description ||
                tokenData.error ||
                'ServiceNow authorization must be renewed.'
            );
 
        refreshError.code =
            'REAUTHENTICATION_REQUIRED';
 
        throw refreshError;
    }
 
 
    /*
     * Store the NEW short-lived access token.
     */
    config.accessToken =
        tokenData.access_token;
 
    config.tokenType =
        tokenData.token_type ||
        'Bearer';
 
    config.expiresIn =
        tokenData.expires_in || 0;
 
    config.tokenObtainedAt =
        Date.now();
 
 
    /*
     * Some OAuth servers rotate refresh tokens.
     * If ServiceNow gives us a new one,
     * replace the previous refresh token.
     */
    if (tokenData.refresh_token) {
 
        config.refreshToken =
            tokenData.refresh_token;
    }
 
 
    saveConfig(config);
 
 
    console.log(
        'ServiceCall access token renewed automatically.'
    );
 
 
    return config.accessToken;
}
 
/* -------------------------------------------------------
   LOCAL OAUTH CALLBACK SERVER
------------------------------------------------------- */

function stopCallbackServer() {

    if (callbackServer) {

        try {
            callbackServer.close();
        } catch (error) {
            // Ignore shutdown errors.
        }

        callbackServer = null;
    }
}


function startCallbackServer() {

    return new Promise(
        (resolve, reject) => {

            stopCallbackServer();

            callbackServer =
                http.createServer(
                    async (
                        request,
                        response
                    ) => {

                        try {

                            const callbackUrl =
                                new URL(
                                    request.url,
                                    `http://${CALLBACK_HOST}:${CALLBACK_PORT}`
                                );

                            if (
                                callbackUrl.pathname !==
                                '/callback'
                            ) {

                                response.writeHead(
                                    404,
                                    {
                                        'Content-Type':
                                            'text/plain'
                                    }
                                );

                                response.end(
                                    'Not found.'
                                );

                                return;
                            }

                            const oauthError =
                                callbackUrl
                                    .searchParams
                                    .get(
                                        'error'
                                    );

                            if (oauthError) {

                                const description =
                                    callbackUrl
                                        .searchParams
                                        .get(
                                            'error_description'
                                        ) ||
                                    oauthError;

                                response.writeHead(
                                    400,
                                    {
                                        'Content-Type':
                                            'text/html; charset=utf-8'
                                    }
                                );

                                response.end(`
                                    <html>
                                        <body style="
                                            font-family: Arial, sans-serif;
                                            text-align: center;
                                            padding-top: 80px;
                                        ">
                                            <h2>ServiceCall authorization was not completed.</h2>
                                            <p>You can close this browser window.</p>
                                        </body>
                                    </html>
                                `);

                                sendAuthStatus(
                                    'error',
                                    description
                                );

                                stopCallbackServer();

                                return;
                            }

                            const code =
                                callbackUrl
                                    .searchParams
                                    .get(
                                        'code'
                                    );

                            const state =
                                callbackUrl
                                    .searchParams
                                    .get(
                                        'state'
                                    );

                            if (
                                !code ||
                                !state
                            ) {

                                throw new Error(
                                    'Authorization code or state was missing.'
                                );
                            }

                            await exchangeAuthorizationCode(
                                code,
                                state
                            );

                            response.writeHead(
                                200,
                                {
                                    'Content-Type':
                                        'text/html; charset=utf-8'
                                }
                            );

                            response.end(`
                                <!DOCTYPE html>
                                <html>
                                <head>
                                    <title>ServiceCall Desktop</title>
                                </head>

                                <body style="
                                    margin: 0;
                                    background: #f3f7f6;
                                    font-family: Arial, sans-serif;
                                    color: #1f2d2a;
                                ">

                                    <div style="
                                        max-width: 520px;
                                        margin: 100px auto;
                                        padding: 40px;
                                        background: white;
                                        border-radius: 12px;
                                        text-align: center;
                                        box-shadow: 0 8px 28px rgba(0,0,0,0.08);
                                    ">

                                        <h1 style="
                                            margin-bottom: 12px;
                                        ">
                                            ServiceCall Desktop
                                        </h1>

                                        <h2 style="
                                            color: #0b6b58;
                                        ">
                                            Connected successfully
                                        </h2>

                                        <p>
                                            Your ServiceNow account is now connected
                                            to ServiceCall Desktop.
                                        </p>

                                        <p>
                                            You can close this browser window
                                            and return to ServiceCall Desktop.
                                        </p>

                                    </div>

                                </body>
                                </html>
                            `);

                            sendAuthStatus(
                                'connected',
                                'ServiceNow sign-in completed successfully.'
                            );

                            if (
                                mainWindow &&
                                !mainWindow.isDestroyed()
                            ) {

                                mainWindow.show();
                                mainWindow.focus();
                            }

                            setTimeout(
                                stopCallbackServer,
                                1000
                            );

                        } catch (error) {

                            response.writeHead(
                                500,
                                {
                                    'Content-Type':
                                        'text/html; charset=utf-8'
                                }
                            );

                            response.end(`
                                <html>
                                    <body style="
                                        font-family: Arial, sans-serif;
                                        text-align: center;
                                        padding-top: 80px;
                                    ">
                                        <h2>ServiceCall connection failed.</h2>
                                        <p>
                                            Return to ServiceCall Desktop
                                            and try again.
                                        </p>
                                    </body>
                                </html>
                            `);

                            sendAuthStatus(
                                'error',
                                error.message
                            );

                            stopCallbackServer();
                        }
                    }
                );

            callbackServer.on(
                'error',
                (error) => {

                    callbackServer =
                        null;

                    reject(error);
                }
            );

            callbackServer.listen(
                CALLBACK_PORT,
                CALLBACK_HOST,
                () => {

                    resolve();
                }
            );
        }
    );
}


/* -------------------------------------------------------
   WINDOW
------------------------------------------------------- */

function registerServiceCallProtocol() {

    if (process.defaultApp) {

        if (
            process.argv.length >= 2
        ) {

            app.setAsDefaultProtocolClient(
                SERVICECALL_PROTOCOL,
                process.execPath,
                [
                    path.resolve(
                        process.argv[1]
                    )
                ]
            );
        }

    } else {

        app.setAsDefaultProtocolClient(
            SERVICECALL_PROTOCOL
        );
    }
}

function showMainWindow() {

    if (
        mainWindow &&
        !mainWindow.isDestroyed()
    ) {

        if (
            mainWindow.isMinimized()
        ) {
            mainWindow.restore();
        }

        mainWindow.show();
        mainWindow.focus();

        return;
    }

    createWindow();
}

function createWindow() {

    mainWindow =
        new BrowserWindow({
            width: 900,
            height: 650,

            minWidth: 700,
            minHeight: 500,

            title:
                'ServiceCall Desktop',

            webPreferences: {

                preload:
                    path.join(
                        __dirname,
                        'preload.js'
                    ),

                contextIsolation:
                    true,

                nodeIntegration:
                    false
            }
        });

    mainWindow.loadFile(
        'index.html'
    );

    mainWindow.on(
    'close',
    (event) => {

        if (!isQuitting) {

            event.preventDefault();

            mainWindow.hide();
        }
    }
);
}

function createTray() {

    if (tray) {
        return;
    }

    /*
       Temporary tray icon:
       Electron will use the app icon later.
       For now we can create the tray only
       after we add a proper icon file.
    */

    const trayMenu =
        Menu.buildFromTemplate([
            {
                label: 'Open ServiceCall',
                click: () => {

                    if (
                        mainWindow &&
                        !mainWindow.isDestroyed()
                    ) {

                        mainWindow.show();
                        mainWindow.focus();
                    }
                }
            },

            {
                type: 'separator'
            },

            {
                label: 'Quit ServiceCall',
                click: () => {

                    isQuitting = true;

                    stopHeartbeatLoop();
                    stopCallbackServer();

                    app.quit();
                }
            }
        ]);

    return trayMenu;
}

/* -------------------------------------------------------
   SAVE INSTANCE
------------------------------------------------------- */

ipcMain.handle(
    'servicecall-get-connection-status',

    async () => {

        const config =
            loadConfig();

        if (
            config.instanceUrl &&
            config.accessToken &&
            heartbeatTimer
        ) {

            return {
                connected: true,

                message:
                    'ServiceCall Desktop is connected.'
            };
        }

        return {
            connected: false,

            message:
                'Sign in to ServiceNow to connect ServiceCall Desktop.'
        };
    }
);

ipcMain.handle(
    'servicecall-save-instance',

    async (
        event,
        instanceUrl
    ) => {

        const normalizedUrl =
            normalizeInstanceUrl(
                instanceUrl
            );

        if (
            !isValidServiceNowUrl(
                normalizedUrl
            )
        ) {

            return {
                success: false,

                message:
                    'Please enter a valid ServiceNow instance URL, for example https://dev12345.service-now.com'
            };
        }

        const config =
            loadConfig();

        config.instanceUrl =
    normalizedUrl;

const configUrl =
    normalizedUrl +
    '/api/x_1806573_servic_0/servicecall_desktop_api/config';

const configResponse =
    await fetch(
        configUrl,
        {
            method: 'GET',

            headers: {
                'Accept':
                    'application/json'
            }
        }
    );

const configText =
    await configResponse.text();

let serviceCallConfig;

try {

    serviceCallConfig =
        JSON.parse(
            configText
        );

} catch (error) {

    return {
        success: false,
        message:
            'The ServiceNow instance returned an invalid ServiceCall configuration.'
    };
}

const result =
    serviceCallConfig.result ||
    serviceCallConfig;

if (
    !configResponse.ok ||
    !result.success
) {

    return {
        success: false,
        message:
            result.message ||
            'Unable to retrieve ServiceCall configuration from this instance.'
    };
}

if (
    !result.oauth_client_id ||
    !result.redirect_uri ||
    !result.oauth_scope ||
    !result.heartbeat_path
) {

    return {
        success: false,
        message:
            'ServiceCall Desktop is not fully configured on this ServiceNow instance.'
    };
}

config.oauthClientId =
    result.oauth_client_id;

config.redirectUri =
    result.redirect_uri;

config.oauthScope =
    result.oauth_scope;

config.heartbeatPath =
    result.heartbeat_path;

        saveConfig(config);

        return {
            success: true,

            message:
                'ServiceNow instance saved successfully.',

            instanceUrl:
                normalizedUrl
        };
    }
);


/* -------------------------------------------------------
   GET INSTANCE
------------------------------------------------------- */

ipcMain.handle(
    'servicecall-get-instance',

    async () => {

        const config =
            loadConfig();

        return {
            success: true,

            instanceUrl:
                config.instanceUrl ||
                ''
        };
    }
);


/* -------------------------------------------------------
   START OAUTH LOGIN
------------------------------------------------------- */

ipcMain.handle(
    'servicecall-start-login',

    async () => {

        try {

            const config =
                loadConfig();

            if (
                !config.instanceUrl ||
                !config.oauthClientId
            ) {

                return {
                    success: false,

                    message:
                        'Please save your ServiceNow instance first.'
                };
            }

            /*
               Start localhost listener BEFORE
               opening the browser.
            */

            await startCallbackServer();

            const codeVerifier =
                generateCodeVerifier();

            const codeChallenge =
                generateCodeChallenge(
                    codeVerifier
                );

            const state =
                generateState();

            config.pkceCodeVerifier =
                codeVerifier;

            config.oauthState =
                state;

            saveConfig(config);

            const authorizeUrl =
                new URL(
                    config.instanceUrl +
                    '/oauth_auth.do'
                );

            authorizeUrl
                .searchParams
                .set(
                    'response_type',
                    'code'
                );

            authorizeUrl
                .searchParams
                .set(
                    'client_id',
                    config.oauthClientId
                );

            authorizeUrl
                .searchParams
                .set(
                    'redirect_uri',
                    config.redirectUri
                );

            authorizeUrl
                .searchParams
                .set(
                    'scope',
                    config.oauthScope
                );

            authorizeUrl
                .searchParams
                .set(
                    'code_challenge',
                    codeChallenge
                );

            authorizeUrl
                .searchParams
                .set(
                    'code_challenge_method',
                    'S256'
                );

            authorizeUrl
                .searchParams
                .set(
                    'state',
                    state
                );

            await shell.openExternal(
                authorizeUrl.toString()
            );

            return {
                success: true,

                message:
                    'ServiceNow sign-in opened in your browser.'
            };

        } catch (error) {

            stopCallbackServer();

            return {
                success: false,

                message:
                    error.message
            };
        }
    }
);


/* -------------------------------------------------------
   APP START
------------------------------------------------------- */

const gotSingleInstanceLock =
    app.requestSingleInstanceLock();

if (!gotSingleInstanceLock) {

    app.quit();

} else {

    app.on(
        'second-instance',
        (
            event,
            commandLine
        ) => {

            const protocolUrl =
                commandLine.find(
                    function(arg) {

                        return arg.startsWith(
                            SERVICECALL_PROTOCOL +
                            '://'
                        );
                    }
                );

            if (protocolUrl) {

                console.log(
                    'ServiceCall protocol opened:',
                    protocolUrl
                );
            }

            showMainWindow();
        }
    );
}

app.whenReady().then(
    async () => {

        registerServiceCallProtocol();

        createWindow();

        await restoreSavedConnection();

        app.on(
            'activate',

            () => {

                if (
                    BrowserWindow
                        .getAllWindows()
                        .length === 0
                ) {

                    createWindow();
                }
            }
        );
    }
);

app.on(
    'window-all-closed',

    () => {

        // stopHeartbeatLoop();
        // stopCallbackServer();

        // if (
        //     process.platform !==
        //     'darwin'
        // ) {

        //     app.quit();
        // }
    }
);

app.on(
    'before-quit',
    () => {

        isQuitting = true;


        stopHeartbeatLoop();
        stopIncomingCallLoop();
        stopCallbackServer();
        stopOutgoingCallLoop();
    }
);

app.on(
    'open-url',
    (
        event,
        url
    ) => {

        event.preventDefault();

        console.log(
            'ServiceCall protocol opened:',
            url
        );

        showMainWindow();
    }
);

let callWindow = null;

function showCallWindow(
    mode,
    callData
) {

    const callSysId =
        callData.callSysId || '';


    /*
     * If this exact call window is already open,
     * simply bring it to the front.
     */
    if (
        callWindow &&
        !callWindow.isDestroyed() &&
        activeCallWindowId === callSysId
    ) {

        if (callWindow.isMinimized()) {
            callWindow.restore();
        }

        callWindow.show();
        callWindow.focus();

        return;
    }


    /*
     * Do not replace an existing active call
     * with another incoming/outgoing call.
     */
    if (
        callWindow &&
        !callWindow.isDestroyed() &&
        activeCallWindowId &&
        activeCallWindowId !== callSysId
    ) {

        console.log(
            'Another ServiceCall window is already active:',
            activeCallWindowId
        );

        return;
    }


    activeCallWindowId =
        callSysId;


    callWindow =
        new BrowserWindow({
            width: 440,
            height: 560,

            minWidth: 440,
            minHeight: 560,

            resizable: false,

            title:
                'ServiceCall',

            autoHideMenuBar:
                true,

            show:
                false,

            webPreferences: {

                preload:
                    path.join(
                        __dirname,
                        'preload.js'
                    ),

                contextIsolation:
                    true,

                nodeIntegration:
                    false
            }
        });


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
