const {
    app,
    BrowserWindow,
    ipcMain,
    shell,
    Tray,
    Menu,
    desktopCapturer,
    dialog,
    powerMonitor
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
let isDeviceSuspended = false;
let outgoingCallTimer = null;
let activeOutgoingCallId = null;
let activeCallWindowId = null;
let callWindowClosing = false;
let currentServiceCallUser = null;
let notificationTimer = null;
const surfacedNotificationIds = new Set();
let currentServiceCallAuthorization = null; 
let authorizationMonitorTimer = null;
let serviceCallAccessUnavailable = false;
const intentionallyLeftCallIds = new Set();
const CALLBACK_HOST = '127.0.0.1';
const CALLBACK_PORT = 42813;
const SERVICECALL_PROTOCOL = 'servicecall';

/*
 * -----------------------------------------
 * DESKTOP NOTIFICATION STACK
 * -----------------------------------------
 *
 * Maximum 3 notification popups are shown
 * on screen at the same time.
 *
 * Additional notifications wait in the
 * queue until a visible slot becomes free.
 */
 
const notificationPopupWindows =
    [];
 
const notificationPopupQueue =
    [];
 
const MAX_VISIBLE_NOTIFICATION_POPUPS =
    3;
 
const NOTIFICATION_POPUP_WIDTH =
    380;
 
const NOTIFICATION_POPUP_HEIGHT =
    150;
 
const NOTIFICATION_POPUP_MARGIN =
    18;
 
const NOTIFICATION_POPUP_GAP =
    10;

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

function ensureAccountStore(
    config
) {

    if (
        !config ||
        typeof config !== 'object'
    ) {

        config = {};
    }


    /*
     * Saved ServiceCall accounts.
     */
    if (
        !Array.isArray(
            config.accounts
        )
    ) {

        config.accounts = [];
    }


    /*
     * Currently selected saved account.
     */
    if (
        typeof config.activeAccountId !==
        'string'
    ) {

        config.activeAccountId =
            '';
    }


    return config;
}

/* =========================================================
   SAVED ACCOUNT HELPERS
========================================================= */

function getSavedAccountKey(
    instanceUrl,
    userSysId
) {

    const normalizedInstance =
        String(instanceUrl || '')
            .trim()
            .replace(/\/+$/, '')
            .toLowerCase();

    const normalizedUser =
        String(userSysId || '')
            .trim()
            .toLowerCase();

    if (
        !normalizedInstance ||
        !normalizedUser
    ) {
        return '';
    }

    return (
        normalizedInstance +
        '::' +
        normalizedUser
    );
}

function ensureSavedAccountStructure(config) {

    if (
        !config ||
        typeof config !== 'object'
    ) {
        config = {};
    }

    if (
        !config.savedAccounts ||
        typeof config.savedAccounts !== 'object' ||
        Array.isArray(config.savedAccounts)
    ) {
        config.savedAccounts = {};
    }

    if (
        typeof config.activeAccountKey !== 'string'
    ) {
        config.activeAccountKey = '';
    }

    return config;
}

/* =========================================================
   SAVE AUTHENTICATED ACCOUNT
========================================================= */

function saveAuthenticatedAccount(
    config,
    user,
    authorization
) {
 
    config =
        ensureSavedAccountStructure(
            config
        );
 
 
    if (
        !user ||
        !user.sys_id
    ) {
 
        throw new Error(
            'Authenticated ServiceCall user is missing.'
        );
    }
 
 
    const accountKey =
        getSavedAccountKey(
            config.instanceUrl,
            user.sys_id
        );
 
 
    if (!accountKey) {
 
        throw new Error(
            'Unable to create the ServiceCall account key.'
        );
    }
 
 
    /*
     * Preserve anything already stored
     * for this account.
     */
 
    const existingAccount =
        config.savedAccounts[
            accountKey
        ] || {};
 
 
    const now =
        new Date().toISOString();
 
 
    config.savedAccounts[
        accountKey
    ] = {
 
        /*
         * -----------------------------------------
         * STABLE ACCOUNT IDENTITY
         * -----------------------------------------
         */
 
        accountKey:
            accountKey,
 
        instanceUrl:
            config.instanceUrl || '',
 
        userSysId:
            user.sys_id,
 
        name:
            user.name || '',
 
        userName:
            user.user_name || '',
 
        email:
            user.email || '',
 
        serviceCallId:
            user.servicecall_id || '',
 
 
        /*
         * -----------------------------------------
         * AUTHORIZATION SNAPSHOT
         * -----------------------------------------
         *
         * This is only stored for UI/account
         * information.
         *
         * /me remains the authority whenever
         * the account is activated.
         */
 
        isServiceCallUser:
            authorization
                ?.is_servicecall_user ===
            true,
 
        isServiceCallAdmin:
            authorization
                ?.is_servicecall_admin ===
            true,
 
 
        /*
         * -----------------------------------------
         * OAUTH CREDENTIALS
         * -----------------------------------------
         *
         * Each saved account owns its own
         * OAuth credentials.
         */
 
        accessToken:
            config.accessToken ||
            existingAccount.accessToken ||
            '',
 
        refreshToken:
            config.refreshToken ||
            existingAccount.refreshToken ||
            '',
 
        tokenType:
            config.tokenType ||
            existingAccount.tokenType ||
            'Bearer',
 
        expiresIn:
            config.expiresIn ||
            existingAccount.expiresIn ||
            0,
 
        tokenObtainedAt:
            config.tokenObtainedAt ||
            existingAccount.tokenObtainedAt ||
            0,
 
 
        /*
         * -----------------------------------------
         * ACCOUNT TIMESTAMPS
         * -----------------------------------------
         */
 
        addedAt:
            existingAccount.addedAt ||
            now,
 
        lastUsedAt:
            now,
 
 
        /*
         * -----------------------------------------
         * DESKTOP NOTIFICATION CHECKPOINT
         * -----------------------------------------
         *
         * This is separate from the notification's
         * read/unread state.
         *
         * It remembers the newest notification
         * already known by the desktop for THIS
         * ServiceCall account.
         *
         * IMPORTANT:
         * Preserve the existing checkpoint when
         * this account authenticates again.
         */
 
        notificationCheckpoint:
            existingAccount
                .notificationCheckpoint || {
                    createdAt: '',
                    sysId: ''
                }
    };
 
 
    /*
     * This account becomes the currently
     * selected ServiceCall account.
     */
 
    config.activeAccountKey =
        accountKey;
 
 
    return {
 
        config:
            config,
 
        accountKey:
            accountKey,
 
        account:
            config.savedAccounts[
                accountKey
            ]
    };
}

function getActiveNotificationCheckpoint() {
 
    try {
 
        const config =
            ensureSavedAccountStructure(
                loadConfig()
            );
 
 
        const activeAccountKey =
            String(
                config.activeAccountKey || ''
            ).trim();
 
 
        if (
            !activeAccountKey ||
            !config.savedAccounts[
                activeAccountKey
            ]
        ) {
 
            return {
                createdAt: '',
                sysId: ''
            };
        }
 
 
        const account =
            config.savedAccounts[
                activeAccountKey
            ];
 
 
        const checkpoint =
            account.notificationCheckpoint ||
            {};
 
 
        return {
 
            createdAt:
                String(
                    checkpoint.createdAt || ''
                ).trim(),
 
            sysId:
                String(
                    checkpoint.sysId || ''
                ).trim()
        };
 
 
    } catch (error) {
 
        console.error(
            'Unable to read ServiceCall notification checkpoint:',
            error
        );
 
 
        return {
            createdAt: '',
            sysId: ''
        };
    }
}
 
 
 
function saveActiveNotificationCheckpoint(
    notification
) {
 
    try {
 
        if (!notification) {
            return false;
        }
 
 
        const notificationSysId =
            String(
                notification.sys_id || ''
            ).trim();
 
 
        const createdAt =
            String(
                notification.created_at || ''
            ).trim();
 
 
        if (
            !notificationSysId ||
            !createdAt
        ) {
 
            console.warn(
                'ServiceCall notification checkpoint not saved because notification identity is incomplete:',
                notification
            );
 
            return false;
        }
 
 
        const config =
            ensureSavedAccountStructure(
                loadConfig()
            );
 
 
        const activeAccountKey =
            String(
                config.activeAccountKey || ''
            ).trim();
 
 
        if (
            !activeAccountKey ||
            !config.savedAccounts[
                activeAccountKey
            ]
        ) {
 
            console.warn(
                'ServiceCall notification checkpoint not saved because no active saved account exists.'
            );
 
            return false;
        }
 
 
        config.savedAccounts[
            activeAccountKey
        ].notificationCheckpoint = {
 
            createdAt:
                createdAt,
 
            sysId:
                notificationSysId
        };
 
 
        saveConfig(
            config
        );
 
 
        console.log(
            'ServiceCall notification checkpoint saved:',
            {
                accountKey:
                    activeAccountKey,
 
                createdAt:
                    createdAt,
 
                sysId:
                    notificationSysId
            }
        );
 
 
        return true;
 
 
    } catch (error) {
 
        console.error(
            'Unable to save ServiceCall notification checkpoint:',
            error
        );
 
 
        return false;
    }
}

/* =========================================================
   SYNC ACTIVE ACCOUNT OAUTH TOKENS
========================================================= */

function syncActiveAccountTokens(config) {

    config =
        ensureSavedAccountStructure(
            config
        );


    const accountKey =
        String(
            config.activeAccountKey || ''
        ).trim();


    /*
     * No active saved account yet.
     *
     * This is expected during our migration
     * from the old single-account config.
     */
    if (!accountKey) {

        return config;
    }


    const account =
        config.savedAccounts[
            accountKey
        ];


    /*
     * Never create an account here.
     *
     * Account creation happens only after
     * /me tells us who authenticated.
     */
    if (!account) {

        console.warn(
            'ServiceCall active saved account was not found:',
            accountKey
        );

        return config;
    }


    /* -----------------------------------------
       COPY CURRENT OAUTH SESSION
       INTO THE ACTIVE ACCOUNT
    ----------------------------------------- */

    account.accessToken =
        config.accessToken || '';

    account.refreshToken =
        config.refreshToken || '';

    account.tokenType =
        config.tokenType ||
        'Bearer';

    account.expiresIn =
        config.expiresIn || 0;

    account.tokenObtainedAt =
        config.tokenObtainedAt || 0;


    /*
     * Keep instance information synchronized.
     */
    account.instanceUrl =
        config.instanceUrl ||
        account.instanceUrl ||
        '';


    /*
     * Token refresh counts as account activity.
     */
    account.lastUsedAt =
        new Date().toISOString();


    config.savedAccounts[
        accountKey
    ] = account;


    return config;
}

/* =========================================================
   GET SAVED ACCOUNTS
========================================================= */

function getSavedAccounts() {

    let config =
        loadConfig();

    config =
        ensureSavedAccountStructure(
            config
        );


    const accounts =
        Object.values(
            config.savedAccounts
        );


    /*
     * Most recently used accounts first.
     */
    accounts.sort(
        (a, b) => {

            const aTime =
                new Date(
                    a.lastUsedAt ||
                    a.addedAt ||
                    0
                ).getTime();

            const bTime =
                new Date(
                    b.lastUsedAt ||
                    b.addedAt ||
                    0
                ).getTime();

            return bTime - aTime;
        }
    );


    /*
     * SECURITY:
     *
     * Never send OAuth tokens to the renderer.
     */
    return accounts.map(
        (account) => ({

            accountKey:
                account.accountKey || '',

            instanceUrl:
                account.instanceUrl || '',

            userSysId:
                account.userSysId || '',

            name:
                account.name || '',

            userName:
                account.userName || '',

            email:
                account.email || '',

            serviceCallId:
                account.serviceCallId || '',

            isServiceCallUser:
                account.isServiceCallUser ===
                true,

            isServiceCallAdmin:
                account.isServiceCallAdmin ===
                true,

            addedAt:
                account.addedAt || '',

            lastUsedAt:
                account.lastUsedAt || '',

            isActive:
                account.accountKey ===
                config.activeAccountKey

        })
    );
}

function updateActiveSavedAccountAuthorization(
    authorization
) {

    let config =
        loadConfig();

    config =
        ensureSavedAccountStructure(
            config
        );


    const accountKey =
        config.activeAccountKey;


    if (
        !accountKey ||
        !config.savedAccounts[
            accountKey
        ]
    ) {

        return;
    }


    const account =
        config.savedAccounts[
            accountKey
        ];


    /*
     * DISPLAY METADATA ONLY.
     *
     * These cached values must never be
     * used to authorize ServiceCall.
     */

    account.isServiceCallUser =
        authorization
            ?.is_servicecall_user ===
        true;


    account.isServiceCallAdmin =
        authorization
            ?.is_servicecall_admin ===
        true;


    config.savedAccounts[
        accountKey
    ] =
        account;


    saveConfig(
        config
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

async function updateDesktopState(
    state
) {

    try {

        const config =
            loadConfig();


        if (
            !config.instanceUrl ||
            !config.accessToken
        ) {

            return {
                success: false,
                code: 'NOT_CONNECTED'
            };
        }


        const result =
            await serviceCallApiRequest(
                '/desktop-state',
                'POST',
                {
                    device_id:
                        getOrCreateDeviceId(),

                    state:
                        state
                }
            );


        console.log(
            'ServiceCall desktop state:',
            result
        );


        return result;


    } catch (error) {

        console.error(
            'Unable to update ServiceCall desktop state:',
            error.message
        );


        return {
            success: false,
            code:
                error.code ||
                'DESKTOP_STATE_FAILED',
            message:
                error.message
        };
    }
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
    !intentionallyLeftCallIds.has(
        incomingCallId
    ) &&
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

/* =========================================================
   SERVICECALL NOTIFICATION MONITOR
========================================================= */

async function checkNotificationsOnce() {
 
    try {
 
        const result =
            await serviceCallApiRequest(
                '/notifications?page=1&page_size=20&filter=unread',
                'GET'
            );
 
 
        if (
            !result ||
            result.success !== true
        ) {
 
            console.warn(
                'ServiceCall notification check returned no valid result:',
                result
            );
 
            return;
        }
 
 
        const notifications =
            Array.isArray(
                result.notifications
            )
                ? result.notifications
                : [];
 
 
        /*
         * -----------------------------------------
         * NO UNREAD NOTIFICATIONS
         * -----------------------------------------
         */
 
        if (
            notifications.length === 0
        ) {
 
            console.log(
                'ServiceCall notification check: no unread notifications.'
            );
 
            return;
        }
 
 
        /*
         * -----------------------------------------
         * CURRENT ACCOUNT CHECKPOINT
         * -----------------------------------------
         */
 
        const checkpoint =
            getActiveNotificationCheckpoint();
 
 
        console.log(
            'ServiceCall notification checkpoint:',
            checkpoint
        );
 
 
        /*
         * -----------------------------------------
         * FIRST-TIME CHECKPOINT
         * -----------------------------------------
         *
         * If this account has never had a desktop
         * notification checkpoint before, treat
         * the newest currently existing
         * notification as the starting point.
         *
         * Existing unread notifications remain
         * unread, but are NOT replayed as desktop
         * popups.
         */
 
        if (
            !checkpoint.createdAt ||
            !checkpoint.sysId
        ) {
 
            const newestNotification =
                notifications[0];
 
 
            if (
                newestNotification &&
                newestNotification.sys_id &&
                newestNotification.created_at
            ) {
 
                saveActiveNotificationCheckpoint(
                    newestNotification
                );
 
 
                /*
                 * Also remember all notifications
                 * returned by this poll for the
                 * current Electron session.
                 */
 
                notifications.forEach(
                    (notification) => {
 
                        const sysId =
                            String(
                                notification.sys_id ||
                                ''
                            ).trim();
 
 
                        if (sysId) {
 
                            surfacedNotificationIds.add(
                                sysId
                            );
                        }
                    }
                );
 
 
                console.log(
                    'ServiceCall initial notification checkpoint established:',
                    {
                        createdAt:
                            newestNotification.created_at,
 
                        sysId:
                            newestNotification.sys_id
                    }
                );
            }
 
 
            return;
        }
 
 
        /*
         * -----------------------------------------
         * FIND NOTIFICATIONS NEWER THAN CHECKPOINT
         * -----------------------------------------
         *
         * ServiceNow sys_created_on uses:
         *
         * YYYY-MM-DD HH:mm:ss
         *
         * Because the components are ordered from
         * largest to smallest, normalized values
         * can be compared lexically.
         *
         * sys_id acts as the secondary identity
         * when timestamps are equal.
         */
 
        const newNotifications = [];
 
 
        for (
            const notification
            of notifications
        ) {
 
            const notificationSysId =
                String(
                    notification.sys_id || ''
                ).trim();
 
 
            const createdAt =
                String(
                    notification.created_at || ''
                ).trim();
 
 
            if (
                !notificationSysId ||
                !createdAt
            ) {
 
                continue;
            }
 
 
            /*
             * Once we reach the exact checkpoint,
             * everything after it is older because
             * the API is newest-first.
             */
 
            if (
                notificationSysId ===
                checkpoint.sysId
            ) {
 
                break;
            }
 
 
            /*
             * Anything created after the saved
             * checkpoint is new.
             */
 
            if (
                createdAt >
                checkpoint.createdAt
            ) {
 
                newNotifications.push(
                    notification
                );
 
                continue;
            }
 
 
            /*
             * Same timestamp but different sys_id.
             *
             * This can happen when multiple
             * notifications are inserted within
             * the same second.
             *
             * Since we have not yet reached the
             * exact checkpoint record and the API
             * is newest-first, treat it as new.
             */
 
            if (
                createdAt ===
                checkpoint.createdAt &&
                notificationSysId !==
                checkpoint.sysId
            ) {
 
                newNotifications.push(
                    notification
                );
            }
        }
 
 
        if (
            newNotifications.length === 0
        ) {
 
            return;
        }
 
 
        console.log(
            'ServiceCall genuinely new notifications detected:',
            newNotifications.length
        );
 
 
        /*
         * -----------------------------------------
         * SURFACE OLDEST → NEWEST
         * -----------------------------------------
         *
         * API gives newest-first.
         *
         * Reverse the newly discovered set so
         * desktop notifications appear in natural
         * chronological order.
         */
 
        const notificationsToSurface =
            newNotifications.reverse();
 
 
        notificationsToSurface.forEach(
            (notification) => {
 
                const notificationSysId =
                    String(
                        notification.sys_id || ''
                    ).trim();
 
 
                if (
                    !notificationSysId
                ) {
 
                    return;
                }
 
 
                /*
                 * Same-session duplicate protection.
                 */
 
                if (
                    surfacedNotificationIds.has(
                        notificationSysId
                    )
                ) {
 
                    return;
                }
 
 
                surfacedNotificationIds.add(
                    notificationSysId
                );
 
 
                console.log(
                    'ServiceCall NEW notification detected:',
                    {
                        sys_id:
                            notificationSysId,
 
                        type:
                            notification.type || '',
 
                        title:
                            notification.title || '',
 
                        message:
                            notification.message || '',
 
                        meeting_sys_id:
                            notification.meeting_sys_id ||
                            '',
 
                        created_at:
                            notification.created_at ||
                            ''
                    }
                );
 
 
                showNotificationPopup(
                    notification
                );
            }
        );
 
 
        /*
         * -----------------------------------------
         * ADVANCE CHECKPOINT
         * -----------------------------------------
         *
         * newNotifications was reversed above,
         * therefore its last item is now the
         * newest notification we processed.
         */
 
        const newestProcessedNotification =
            notificationsToSurface[
                notificationsToSurface.length -
                1
            ];
 
 
        if (
            newestProcessedNotification
        ) {
 
            saveActiveNotificationCheckpoint(
                newestProcessedNotification
            );
        }
 
 
    } catch (error) {
 
        console.error(
            'ServiceCall notification check failed:',
            error
        );
 
 
        if (
            error.code ===
            'AUTHENTICATION_REQUIRED'
        ) {
 
            stopNotificationLoop();
 
 
            sendAuthStatus(
                'authentication_required',
                'Your ServiceCall authorization has expired. Please sign in again.'
            );
        }
    }
}

function stopNotificationLoop() {

    if (notificationTimer) {

        clearInterval(
            notificationTimer
        );

        notificationTimer =
            null;
    }
}


// function startNotificationLoop() {

//     /*
//      * Prevent duplicate notification timers.
//      */

//     stopNotificationLoop();


//     /*
//      * Check immediately.
//      */

//     checkNotificationsOnce();


//     /*
//      * Then check every 5 seconds.
//      *
//      * This gives meeting-started notifications
//      * reasonably fast desktop delivery.
//      */

//     notificationTimer =
//         setInterval(
//             checkNotificationsOnce,
//             5000
//         );
// }

function startNotificationLoop() {

    console.log(
        '>>> SERVICECALL NOTIFICATION LOOP STARTED <<<'
    );

    stopNotificationLoop();

    checkNotificationsOnce();

    notificationTimer =
        setInterval(
            checkNotificationsOnce,
            5000
        );
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

/* =========================================================
   RUNTIME SERVICECALL AUTHORIZATION MONITOR
========================================================= */
 
function stopAuthorizationMonitor() {
 
    if (authorizationMonitorTimer) {
 
        clearInterval(
            authorizationMonitorTimer
        );
 
        authorizationMonitorTimer =
            null;
    }
}

/* =========================================================
   SUSPEND / RESUME SERVICECALL RUNTIME
========================================================= */
 
function suspendServiceCallRuntime() {
 
    console.warn(
        'Suspending ServiceCall runtime...'
    );
 
 
    /*
     * STOP HEARTBEAT
     */
    if (heartbeatTimer) {
 
        clearInterval(
            heartbeatTimer
        );
 
        heartbeatTimer = null;
    }
 
 
    /*
     * STOP INCOMING CALL MONITOR
     */
    if (incomingCallTimer) {
 
        clearInterval(
            incomingCallTimer
        );
 
        incomingCallTimer = null;
    }
 
 
    /*
     * STOP OUTGOING CALL MONITOR
     */
    if (outgoingCallTimer) {
 
        clearInterval(
            outgoingCallTimer
        );
 
        outgoingCallTimer = null;
    }

    stopNotificationLoop();
 
 
    /*
     * IMPORTANT:
     *
     * DO NOT stop authorizationMonitorTimer.
     *
     * DO NOT clear OAuth tokens.
     *
     * DO NOT clear the active account.
     *
     * DO NOT call desktop-sign-out.
     *
     * The user is still authenticated.
     * Only ServiceCall authorization is unavailable.
     */
 
    console.log(
        'ServiceCall runtime suspended.'
    );
}

async function terminateActiveSessionForAccessLoss() {
 
    console.warn(
        'Terminating active ServiceCall desktop session because access was removed...'
    );
 
 
    /*
     * Remember the active call before clearing
     * the desktop state.
     */
 
    const callSysId =
        activeCallWindowId ||
        activeOutgoingCallId ||
        activeIncomingCallId ||
        null;
 
 
    /*
     * Prevent polling from reopening this call
     * if ServiceCall access is restored later.
     */
 
    if (callSysId) {
 
        intentionallyLeftCallIds.add(
            callSysId
        );
 
 
        console.log(
            'ServiceCall call blocked from automatic reopen:',
            callSysId
        );
    }
 
 
    /*
     * Force the call window to close.
     *
     * IMPORTANT:
     * callWindowClosing = true bypasses the
     * normal close handler.
     *
     * Normally X on a connected call only hides
     * the window. Authorization loss must close it.
     */
 
    if (
        callWindow &&
        !callWindow.isDestroyed()
    ) {
 
        callWindowClosing =
            true;
 
 
        try {
 
            callWindow.close();
 
        } catch (error) {
 
            console.error(
                'Unable to close ServiceCall call window during access loss:',
                error
            );
        }
    }
 
 
    /*
     * Clear desktop call tracking.
     *
     * Restoring ServiceCall authorization must
     * NOT restore the previous call automatically.
     */
 
    activeCallWindowId =
        null;
 
    activeIncomingCallId =
        null;
 
    activeOutgoingCallId =
        null;
 
 
    console.log(
        'Active ServiceCall desktop session cleared.'
    );
}

async function showServiceCallUnavailablePage() {
 
    if (
        !mainWindow ||
        mainWindow.isDestroyed()
    ) {
        return;
    }
 
 
    try {
 
        await mainWindow.loadFile(
            'access-unavailable.html'
        );
 
 
        mainWindow.show();
 
        mainWindow.focus();
 
 
        console.log(
            'ServiceCall unavailable page displayed.'
        );
 
    } catch (error) {
 
        console.error(
            'Unable to display ServiceCall unavailable page:',
            error
        );
    }
}
 
 
async function restoreServiceCallApplication() {
 
    if (
        !mainWindow ||
        mainWindow.isDestroyed()
    ) {
        return;
    }
 
 
    try {
 
        await mainWindow.loadFile(
            'index.html'
        );
 
 
        mainWindow.show();
 
        mainWindow.focus();
 
 
        console.log(
            'ServiceCall application restored.'
        );
 
    } catch (error) {
 
        console.error(
            'Unable to restore ServiceCall application:',
            error
        );
    }
}
 
 
 
async function resumeServiceCallRuntime() {
 
    console.log(
        'Resuming ServiceCall runtime...'
    );
 
 
    /*
     * Restart operational background services.
     */
 
    await startHeartbeatLoop();
 
    startIncomingCallLoop();
 
    startOutgoingCallLoop();

    startNotificationLoop();
 
 
    console.log(
        'ServiceCall runtime resumed.'
    );
}
 
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
         * Ensure the OAuth session itself
         * is still valid.
         */
 
        await ensureValidAccessToken();
 
 
        /*
         * Re-check the CURRENT ServiceNow roles.
         *
         * Do not trust the role snapshot stored
         * when the user originally signed in.
         */
 
        const currentUser =
            await getCurrentServiceCallUser();
 
 
        const authorization =
            currentUser?.authorization || {};
 
 
        /*
         * Keep current identity and authorization
         * information synchronized.
         */
 
        currentServiceCallUser =
            currentUser?.user || null;
 
        currentServiceCallAuthorization =
            authorization;

        updateActiveSavedAccountAuthorization(authorization);
 
 
        /*
         * -----------------------------------------
         * SERVICECALL ACCESS REMOVED
         * -----------------------------------------
         */
 
        if (
            authorization.allowed !== true
        ) {
 
            /*
             * Only perform the suspension/page
             * transition once.
             */
 
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
 
 
                /*
                 * Stop ServiceCall operational
                 * background activity.
                 *
                 * The authorization monitor itself
                 * must remain running.
                 */
 
                suspendServiceCallRuntime();
 
 
await terminateActiveSessionForAccessLoss();
 
 
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

        /*
         * Tell the CURRENT unavailable page
         * that access has been restored
         * BEFORE replacing that page.
         */

        mainWindow?.webContents.send(
            'servicecall-authorization-status',
            {
                status:
                    'access_restored',

                message:
                    'ServiceCall access has been restored.'
            }
        );


        /*
         * Keep the success/reconnecting
         * message visible for 3.5 seconds.
         */

        await new Promise(
            (resolve) => {

                setTimeout(
                    resolve,
                    3500
                );
            }
        );


        /*
         * Restart operational ServiceCall
         * background services.
         */

        await resumeServiceCallRuntime();


        /*
         * Return from the unavailable page
         * to the normal ServiceCall UI.
         */

        await restoreServiceCallApplication();


        /*
         * Restoration completed successfully.
         */

        serviceCallAccessUnavailable =
            false;


        console.log(
            'ServiceCall runtime restored successfully.'
        );

    }
    catch (restoreError) {

        /*
         * Keep ServiceCall unavailable so
         * the authorization monitor can
         * retry restoration later.
         */

        serviceCallAccessUnavailable =
            true;


        console.error(
            'Unable to restore ServiceCall runtime:',
            restoreError
        );
    }
}
 
    }
    catch (error) {
 
        console.error(
            'ServiceCall runtime authorization check failed:',
            error
        );
    }
}
 
function startAuthorizationMonitor() {
 
    stopAuthorizationMonitor();
 
 
    /*
     * Check immediately.
     */
    checkRuntimeAuthorizationOnce();
 
 
    /*
     * Then re-check every 30 seconds.
     */
    authorizationMonitorTimer =
        setInterval(
            checkRuntimeAuthorizationOnce,
            30000
        );
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

saveConfig(
    config
);


/*
 * -----------------------------------------
 * VERIFY SERVICECALL AUTHORIZATION
 * -----------------------------------------
 */

const currentUser =
    await getCurrentServiceCallUser();

const authorization =
    currentUser?.authorization || {};


/*
 * -----------------------------------------
 * ESTABLISH ACTIVE RUNTIME IDENTITY
 * -----------------------------------------
 *
 * OAuth authentication has completed and
 * ServiceNow has resolved the current user.
 *
 * Keep the active Electron runtime identity
 * synchronized with the authenticated
 * ServiceNow identity.
 */

currentServiceCallUser =
    currentUser?.user || null;

currentServiceCallAuthorization =
    authorization;


/*
 * -----------------------------------------
 * SAVE AUTHENTICATED ACCOUNT
 * -----------------------------------------
 */

const savedAccountResult =
    saveAuthenticatedAccount(
        config,
        currentUser?.user || null,
        authorization
    );


saveConfig(
    savedAccountResult.config
);


console.log(
    'ServiceCall authenticated account saved:',
    {
        accountKey:
            savedAccountResult.accountKey,

        name:
            savedAccountResult.account?.name,

        userName:
            savedAccountResult.account?.userName
    }
);


/*
 * -----------------------------------------
 * ACCESS DENIED
 * -----------------------------------------
 */

if (
    authorization.allowed !== true
) {

    console.warn(
        'ServiceCall access denied:',
        currentUser
    );


    /*
     * OAuth succeeded, so the person is
     * authenticated.
     *
     * But we DO NOT start heartbeat,
     * calls, meetings, etc.
     */

    return {
        success: true,
        authenticated: true,
        authorized: false,
        state: 'access_denied',
        user:
            currentUser?.user || null,
        authorization: authorization
    };
}


/*
 * -----------------------------------------
 * AUTHORIZED
 * -----------------------------------------
 */

console.log(
    'ServiceCall authorization successful:',
    {
        user:
            currentUser?.user,

        authorization:
            authorization
    }
);


/*
 * Background services may start only
 * after ServiceCall authorization succeeds.
 */

await startHeartbeatLoop();

startIncomingCallLoop();

startOutgoingCallLoop();

startNotificationLoop();

startAuthorizationMonitor();


return {
    success: true,
    authenticated: true,
    authorized: true,
    state: 'ready',
    user:
        currentUser?.user || null,
    authorization:
        authorization,
    tokenData:
        tokenData
};
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


                            /*
                             * -----------------------------------------
                             * VALIDATE CALLBACK PATH
                             * -----------------------------------------
                             */

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


                            /*
                             * -----------------------------------------
                             * OAUTH ERROR
                             * -----------------------------------------
                             */

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
                                            <h2>
                                                ServiceCall authorization
                                                was not completed.
                                            </h2>

                                            <p>
                                                You can close this
                                                browser window.
                                            </p>
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


                            /*
                             * -----------------------------------------
                             * READ AUTHORIZATION CODE
                             * -----------------------------------------
                             */

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


                            /*
                             * -----------------------------------------
                             * EXCHANGE CODE + CHECK SERVICECALL ACCESS
                             * -----------------------------------------
                             */

                            const authResult =
                                await exchangeAuthorizationCode(
                                    code,
                                    state
                                );


                            /*
                             * -----------------------------------------
                             * AUTHENTICATED BUT ACCESS DENIED
                             * -----------------------------------------
                             */

                            if (
                                authResult?.state ===
                                'access_denied'
                            ) {

                                console.warn(
                                    'ServiceCall OAuth succeeded but access was denied:',
                                    authResult
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
                                        <title>
                                            ServiceCall Desktop
                                        </title>
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

                                            <h1>
                                                ServiceCall Desktop
                                            </h1>

                                            <h2>
                                                Access unavailable
                                            </h2>

                                            <p>
                                                Your ServiceNow sign-in
                                                was successful, but
                                                ServiceCall access has
                                                not been assigned to
                                                this account.
                                            </p>

                                            <p>
                                                You can close this
                                                browser window and
                                                return to ServiceCall
                                                Desktop.
                                            </p>

                                        </div>

                                    </body>

                                    </html>
                                `);


                                sendAuthStatus(
                                    'access_denied',
                                    'ServiceCall access has not been assigned to this account.'
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

                                return;
                            }


                            /*
                             * -----------------------------------------
                             * AUTHORIZED
                             * -----------------------------------------
                             */

                            if (
                                authResult?.state ===
                                'ready'
                            ) {

                                console.log(
                                    'ServiceCall login authorized:',
                                    authResult.user
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
                                        <title>
                                            ServiceCall Desktop
                                        </title>
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
                                                Your ServiceNow account
                                                is now connected to
                                                ServiceCall Desktop.
                                            </p>

                                            <p>
                                                You can close this
                                                browser window and
                                                return to ServiceCall
                                                Desktop.
                                            </p>

                                        </div>

                                    </body>

                                    </html>
                                `);


                                sendAuthStatus(
                                    'connected',
                                    'Connected to ServiceCall.'
                                );


                                /*
                                 * OAuth + /me authorization passed.
                                 *
                                 * exchangeAuthorizationCode()
                                 * has already started the background
                                 * ServiceCall services.
                                 *
                                 * Now enter the application.
                                 */

                                if (
                                    mainWindow &&
                                    !mainWindow.isDestroyed()
                                ) {

                                    await mainWindow.loadFile(
                                        'index.html'
                                    );

                                    mainWindow.show();

                                    mainWindow.focus();
                                }


                                setTimeout(
                                    stopCallbackServer,
                                    1000
                                );

                                return;
                            }


                            /*
                             * -----------------------------------------
                             * UNEXPECTED AUTH RESULT
                             * -----------------------------------------
                             */

                            throw new Error(
                                'ServiceCall authentication completed with an unexpected result.'
                            );

                        }
                        catch (error) {

                            console.error(
                                'ServiceCall OAuth callback failed:',
                                error
                            );


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

                                        <h2>
                                            ServiceCall connection
                                            failed.
                                        </h2>

                                        <p>
                                            Return to ServiceCall
                                            Desktop and try again.
                                        </p>

                                    </body>
                                </html>
                            `);


                            sendAuthStatus(
                                'error',
                                error?.message ||
                                'ServiceCall connection failed.'
                            );


                            stopCallbackServer();
                        }
                    }
                );


            /*
             * -----------------------------------------
             * CALLBACK SERVER ERROR
             * -----------------------------------------
             */

            callbackServer.on(
                'error',
                (error) => {

                    callbackServer =
                        null;

                    reject(
                        error
                    );
                }
            );


            /*
             * -----------------------------------------
             * START CALLBACK SERVER
             * -----------------------------------------
             */

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

/* =====================================================
   SERVICECALL DEEP LINK HANDLING
===================================================== */

let pendingDeepLink = null;


/*
 * Extract a ServiceCall deep link from
 * Electron command-line arguments.
 */
function getServiceCallDeepLink(
    args
) {

    if (!Array.isArray(args)) {
        return null;
    }


    const deepLink =
        args.find(
            arg =>
                typeof arg === 'string' &&
                arg.startsWith(
                    'servicecall://'
                )
        );


    return deepLink || null;
}


/*
 * Handle the received deep link.
 *
 * For now we only store and log it.
 * Renderer navigation comes next.
 */
function handleServiceCallDeepLink(
    deepLink
) {

    if (
        !deepLink ||
        !deepLink.startsWith(
            SERVICECALL_PROTOCOL +
            '://'
        )
    ) {
        return;
    }


    pendingDeepLink =
        deepLink;


    console.log(
        'ServiceCall deep link received:',
        deepLink
    );


    /*
     * If the renderer is already loaded,
     * send the link immediately.
     */
    if (
        mainWindow &&
        !mainWindow.isDestroyed() &&
        !mainWindow.webContents.isLoading()
    ) {

        mainWindow.webContents.send(
            'servicecall-deep-link',
            {
                url:
                    deepLink
            }
        );


        /*
         * The renderer now owns this link,
         * so it is no longer pending.
         */
        pendingDeepLink =
            null;
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

async function createWindow() {
 
    mainWindow = new BrowserWindow({
        width: 900,
        height: 650,
        minWidth: 700,
        minHeight: 500,
        title: 'ServiceCall Desktop',
 
        webPreferences: {
            preload: path.join(
                __dirname,
                'preload.js'
            ),
 
            contextIsolation: true,
            nodeIntegration: false
        }
    });
 
 
    /*
     * -----------------------------------------
     * RESOLVE STARTUP AUTHENTICATION
     * -----------------------------------------
     */
 
    const startupState =
        await resolveStartupAuthentication();
 
 
    console.log(
        'ServiceCall startup state:',
        startupState
    );
 
 
    /*
     * -----------------------------------------
     * AUTHORIZED USER
     * -----------------------------------------
     */
 
    if (
        startupState.state === 'ready'
    ) {
 
        /*
         * Make sure we begin in the
         * available state.
         */
 
        serviceCallAccessUnavailable =
            false;
 
 
        await mainWindow.loadFile(
            'index.html'
        );
 
 
        /*
         * Start background ServiceCall services
         * ONLY after authorization succeeds.
         */
 
        await startHeartbeatLoop();
 
        startIncomingCallLoop();
 
        startOutgoingCallLoop();

        startNotificationLoop();
 
        startAuthorizationMonitor();
    }
 
 
    /*
     * -----------------------------------------
     * AUTHENTICATED BUT ACCESS NOT AVAILABLE
     * -----------------------------------------
     */
 
    else if (
    startupState.state ===
    'access_denied'
) {

    console.warn(
        'ServiceCall access denied at startup.'
    );


    /*
     * Startup access denial belongs to the
     * authentication flow.
     *
     * Do NOT use the runtime unavailable page
     * here.
     */
    serviceCallAccessUnavailable =
        false;


    await mainWindow.loadFile(
        path.join(
            'auth',
            'auth-gate.html'
        )
    );


    /*
     * Do NOT start the runtime authorization
     * monitor here.
     *
     * The runtime monitor is only for a user
     * who successfully entered ServiceCall
     * and later loses access.
     */
}
 
 
    /*
     * -----------------------------------------
     * LOGIN / AUTHENTICATION GATE
     * -----------------------------------------
     */
 
    else {
 
        /*
         * No usable authenticated ServiceCall
         * session exists.
         */
 
        serviceCallAccessUnavailable =
            false;
 
 
        await mainWindow.loadFile(
            path.join(
                'auth',
                'auth-gate.html'
            )
        );
    }
 
 
    /*
     * -----------------------------------------
     * WINDOW CLOSE
     * -----------------------------------------
     */
 
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
 

/* =====================================================
   SERVICECALL DESKTOP NOTIFICATION POPUP
===================================================== */

function showNotificationPopup(
    notification
) {
 
    if (
        !notification ||
        !notification.sys_id
    ) {
        return;
    }
 
 
    /*
     * If all visible popup slots are occupied,
     * keep this notification waiting.
     */
    if (
        notificationPopupWindows.length >=
        MAX_VISIBLE_NOTIFICATION_POPUPS
    ) {
 
        notificationPopupQueue.push(
            notification
        );
 
        console.log(
            'ServiceCall notification queued:',
            notification.sys_id
        );
 
        return;
    }
 
 
    createNotificationPopupWindow(
        notification
    );
}
 
 
/* =========================================================
   CREATE NOTIFICATION POPUP WINDOW
========================================================= */
 
function createNotificationPopupWindow(
    notification
) {
 
    const {
        screen
    } = require(
        'electron'
    );
 
 
    const display =
        screen.getPrimaryDisplay();
 
 
    const workArea =
        display.workArea;
 
 
    /*
     * Existing visible popups are counted
     * from the bottom upward.
     *
     * 0 = bottom slot
     * 1 = second slot
     * 2 = third slot
     */
    const slotIndex =
        notificationPopupWindows.length;
 
 
    const popupX =
        Math.round(
            workArea.x +
            workArea.width -
            NOTIFICATION_POPUP_WIDTH -
            NOTIFICATION_POPUP_MARGIN
        );
 
 
    const popupY =
        Math.round(
            workArea.y +
            workArea.height -
            NOTIFICATION_POPUP_HEIGHT -
            NOTIFICATION_POPUP_MARGIN -
            (
                slotIndex *
                (
                    NOTIFICATION_POPUP_HEIGHT +
                    NOTIFICATION_POPUP_GAP
                )
            )
        );
 
 
    const popupWindow =
        new BrowserWindow({
 
            width:
                NOTIFICATION_POPUP_WIDTH,
 
            height:
                NOTIFICATION_POPUP_HEIGHT,
 
            x:
                popupX,
 
            y:
                popupY,
 
            frame:
                false,
 
            transparent:
                true,
 
            resizable:
                false,
 
            movable:
                false,
 
            minimizable:
                false,
 
            maximizable:
                false,
 
            fullscreenable:
                false,
 
            skipTaskbar:
                true,
 
            alwaysOnTop:
                true,
 
            show:
                false,
 
            focusable:
                true,
 
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
 
 
    /*
     * Keep the notification identity attached
     * to its own BrowserWindow.
     */
    popupWindow.serviceCallNotification =
        notification;
 
 
    notificationPopupWindows.push(
        popupWindow
    );
 
 
    popupWindow.loadFile(
        'notification-popup.html',
        {
            query: {
 
                notificationSysId:
                    String(
                        notification.sys_id ||
                        ''
                    ),
 
                type:
                    String(
                        notification.type_display ||
                        notification.type ||
                        'Notification'
                    ),
 
                title:
                    String(
                        notification.title ||
                        'ServiceCall'
                    ),
 
                message:
                    String(
                        notification.message ||
                        ''
                    ),
 
                meetingSysId:
                    String(
                        notification.meeting_sys_id ||
                        ''
                    ),
 
                callSysId:
                    String(
                        notification.call_sys_id ||
                        ''
                    )
            }
        }
    );
 
 
    popupWindow.once(
        'ready-to-show',
        () => {
 
            if (
                popupWindow.isDestroyed()
            ) {
                return;
            }
 
 
            popupWindow.showInactive();
        }
    );
 
 
    popupWindow.on(
        'closed',
        () => {
 
            const popupIndex =
                notificationPopupWindows.indexOf(
                    popupWindow
                );
 
 
            if (
                popupIndex !== -1
            ) {
 
                notificationPopupWindows.splice(
                    popupIndex,
                    1
                );
            }
 
 
            /*
             * Reposition whatever remains,
             * then use the newly available slot
             * for the next queued notification.
             */
            repositionNotificationPopups();
 
 
            showNextQueuedNotification();
        }
    );
}
 
 
/* =========================================================
   REPOSITION VISIBLE NOTIFICATIONS
========================================================= */
 
function repositionNotificationPopups() {
 
    const {
        screen
    } = require(
        'electron'
    );
 
 
    const display =
        screen.getPrimaryDisplay();
 
 
    const workArea =
        display.workArea;
 
 
    notificationPopupWindows.forEach(
        (
            popupWindow,
            index
        ) => {
 
            if (
                !popupWindow ||
                popupWindow.isDestroyed()
            ) {
                return;
            }
 
 
            const popupX =
                Math.round(
                    workArea.x +
                    workArea.width -
                    NOTIFICATION_POPUP_WIDTH -
                    NOTIFICATION_POPUP_MARGIN
                );
 
 
            const popupY =
                Math.round(
                    workArea.y +
                    workArea.height -
                    NOTIFICATION_POPUP_HEIGHT -
                    NOTIFICATION_POPUP_MARGIN -
                    (
                        index *
                        (
                            NOTIFICATION_POPUP_HEIGHT +
                            NOTIFICATION_POPUP_GAP
                        )
                    )
                );
 
 
            popupWindow.setPosition(
                popupX,
                popupY,
                true
            );
        }
    );
}
 
 
/* =========================================================
   SHOW NEXT QUEUED NOTIFICATION
========================================================= */
 
function showNextQueuedNotification() {
 
    if (
        notificationPopupWindows.length >=
        MAX_VISIBLE_NOTIFICATION_POPUPS
    ) {
        return;
    }
 
 
    if (
        notificationPopupQueue.length === 0
    ) {
        return;
    }
 
 
    const nextNotification =
        notificationPopupQueue.shift();
 
 
    createNotificationPopupWindow(
        nextNotification
    );
}

/* -------------------------------------------------------
   TEMPORARY POPUP TEST
------------------------------------------------------- */

ipcMain.handle(
    'servicecall-test-notification-popup',

    async () => {

        showNotificationPopup({

            sys_id:
                'test-notification',

            type:
                'meeting started',

            type_display:
                'Meeting Started',

            title:
                'Meeting started',

            message:
                '"ServiceCall Test Meeting" has started.',

            meeting_sys_id:
                '',

            call_sys_id:
                ''
        });


        return {
            success: true
        };
    }
);

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

            const deepLink =
                getServiceCallDeepLink(
                    commandLine
                );


            if (deepLink) {

                handleServiceCallDeepLink(
                    deepLink
                );
            }


            showMainWindow();
        }
    );
}

app.whenReady().then(
    async () => {

        registerServiceCallProtocol();


        /*
         * Was ServiceCall launched by a
         * servicecall:// URL?
         */
        const startupDeepLink =
            getServiceCallDeepLink(
                process.argv
            );


        if (startupDeepLink) {

            handleServiceCallDeepLink(
                startupDeepLink
            );
        }


        await createWindow();

        /* -----------------------------------------
           DEVICE SLEEP / RESUME
        ----------------------------------------- */

        powerMonitor.on(
            'suspend',
            async () => {

                console.log(
                    'ServiceCall detected device suspend.'
                );


                isDeviceSuspended =
                    true;


                /*
                 * Tell ServiceNow this device
                 * intentionally became Away.
                 *
                 * Keep the user's manual presence
                 * untouched.
                 */
                await updateDesktopState(
                    'away'
                );
            }
        );


        /* -----------------------------------------
           DEVICE SLEEP / RESUME
        ----------------------------------------- */

        powerMonitor.on(
            'suspend',
            async () => {

                console.log(
                    'ServiceCall detected device suspend.'
                );


                isDeviceSuspended =
                    true;


                /*
                 * Tell ServiceNow this device
                 * intentionally became Away.
                 *
                 * Keep the user's manual presence
                 * untouched.
                 */
                await updateDesktopState(
                    'away'
                );
            }
        );


        powerMonitor.on(
            'resume',
            async () => {

                console.log(
                    'ServiceCall detected device resume.'
                );


                isDeviceSuspended =
                    false;


                /*
                 * A heartbeat is better than simply
                 * changing the state to Connected:
                 *
                 * - marks registration Connected
                 * - refreshes Last Seen
                 * - proves ServiceNow is reachable
                 */
                try {

                    const result =
                        await sendHeartbeatOnce();


                    console.log(
                        'ServiceCall resume heartbeat:',
                        result
                    );


                } catch (error) {

                    console.error(
                        'ServiceCall resume heartbeat failed:',
                        error
                    );
                }
            }
        );

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


        handleServiceCallDeepLink(
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
                    : 'false',

            /*
             * MEETING CONTEXT
             *
             * Normal calls will simply receive
             * isMeeting=false and empty values.
             */
            isMeeting:
                callData.isMeeting
                    ? 'true'
                    : 'false',

            meetingSysId:
                callData.meetingSysId ||
                '',

            meetingNumber:
                callData.meetingNumber ||
                '',

            meetingTitle:
                callData.meetingTitle ||
                ''
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

ipcMain.handle(
    'servicecall-download-recording',
 
    async (
        event,
        recordingSysId
    ) => {
 
        if (!recordingSysId) {
 
            return {
                success: false,
                code: 'RECORDING_ID_REQUIRED',
                message: 'Recording ID was not provided.'
            };
        }
 
 
        try {
 
            /*
             * -----------------------------------------
             * 1. ASK SERVICENOW IF THIS USER
             *    MAY DOWNLOAD THIS RECORDING
             * -----------------------------------------
             */
 
            const downloadInfo =
                await serviceCallApiRequest(
                    '/recording-download' +
                    '?recording_sys_id=' +
                    encodeURIComponent(
                        recordingSysId
                    ),
                    'GET'
                );
 
 
            if (
                !downloadInfo ||
                downloadInfo.success !== true ||
                !downloadInfo.attachment_sys_id
            ) {
 
                throw new Error(
                    downloadInfo &&
                    downloadInfo.message
                        ? downloadInfo.message
                        : 'Recording download was not authorized.'
                );
            }
 
 
            /*
             * -----------------------------------------
             * 2. GET CURRENT INSTANCE + OAUTH TOKEN
             * -----------------------------------------
             */
 
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
 
 
            let accessToken =
                await ensureValidAccessToken();
 
 
            const attachmentUrl =
                config.instanceUrl
                    .replace(/\/$/, '') +
                '/api/now/attachment/' +
                encodeURIComponent(
                    downloadInfo.attachment_sys_id
                ) +
                '/file';
 
 
            async function performDownload(
                token
            ) {
 
                return await fetch(
                    attachmentUrl,
                    {
                        method: 'GET',
 
                        headers: {
                            'Authorization':
                                'Bearer ' +
                                token
                        }
                    }
                );
            }
 
 
            /*
             * -----------------------------------------
             * 3. DOWNLOAD ATTACHMENT
             * -----------------------------------------
             */
 
            let response =
                await performDownload(
                    accessToken
                );
 
 
            /*
             * Access token could expire between
             * authorization and attachment download.
             */
            if (
                response.status === 401 ||
                response.status === 403
            ) {
 
                accessToken =
                    await refreshAccessToken();
 
 
                response =
                    await performDownload(
                        accessToken
                    );
            }
 
 
            if (!response.ok) {
 
                throw new Error(
                    'Unable to download the recording attachment from ServiceNow.'
                );
            }
 
 
            const arrayBuffer =
                await response.arrayBuffer();
 
 
            const recordingBuffer =
                Buffer.from(
                    arrayBuffer
                );
 
 
            if (
                recordingBuffer.length <= 0
            ) {
 
                throw new Error(
                    'The downloaded recording file is empty.'
                );
            }
 
 
            /*
             * -----------------------------------------
             * 4. DETERMINE SAFE FILE NAME
             * -----------------------------------------
             */
 
            const format =
                String(
                    downloadInfo.format ||
                    'mp3'
                )
                    .trim()
                    .toLowerCase();
 
 
            const extension =
                format === 'mp4'
                    ? '.mp4'
                    : '.mp3';
 
 
            let fileName =
                String(
                    downloadInfo.file_name ||
                    (
                        'servicecall-recording-' +
                        recordingSysId +
                        extension
                    )
                )
                    .replace(
                        /[<>:"/\\|?*\x00-\x1F]/g,
                        '_'
                    );
 
 
            if (
                !fileName
                    .toLowerCase()
                    .endsWith(
                        extension
                    )
            ) {
 
                fileName +=
                    extension;
            }
 
 
            /*
             * -----------------------------------------
             * 5. WINDOWS SAVE AS DIALOG
             * -----------------------------------------
             */
 
            const saveResult =
                await dialog.showSaveDialog(
                    {
                        title:
                            'Save ServiceCall Recording',
 
                        defaultPath:
                            path.join(
                                app.getPath(
                                    'downloads'
                                ),
                                fileName
                            ),
 
                        filters: [
                            {
                                name:
                                    format === 'mp4'
                                        ? 'MP4 Video'
                                        : 'MP3 Audio',
 
                                extensions: [
                                    format === 'mp4'
                                        ? 'mp4'
                                        : 'mp3'
                                ]
                            }
                        ]
                    }
                );
 
 
            /*
             * User pressed Cancel.
             *
             * This is NOT an error.
             */
            if (
                saveResult.canceled ||
                !saveResult.filePath
            ) {
 
                return {
                    success: false,
                    code: 'DOWNLOAD_CANCELLED',
                    message: 'Recording download was cancelled.'
                };
            }
 
 
            /*
             * -----------------------------------------
             * 6. SAVE FILE LOCALLY
             * -----------------------------------------
             */
 
            await fs.promises.writeFile(
                saveResult.filePath,
                recordingBuffer
            );
 
 
            console.log(
                'ServiceCall recording downloaded successfully.',
                recordingSysId
            );
 
 
            return {
                success: true,
                code: 'RECORDING_DOWNLOADED',
                recording_sys_id:
                    recordingSysId,
                file_name:
                    path.basename(
                        saveResult.filePath
                    ),
                file_path:
                    saveResult.filePath,
                format:
                    format,
                file_size:
                    recordingBuffer.length
            };
 
 
        } catch (error) {
 
            console.error(
                'ServiceCall recording download failed:',
                error.message
            );
 
 
            return {
                success: false,
                code:
                    error.code ||
                    'RECORDING_DOWNLOAD_FAILED',
                message:
                    error.message ||
                    'Unable to download recording.'
            };
        }
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
    !intentionallyLeftCallIds.has(
        outgoingCallId
    ) &&
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

console.log(
    'ServiceCall Attachment API response:',
    'HTTP',
    response.status,
    data
);


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

            console.log(
    'ServiceCall attachment upload result:',
    uploadResult
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

ipcMain.handle(
    'servicecall-get-recording-history',
 
    async () => {
 
        try {
 
            const result =
                await serviceCallApiRequest(
                    '/recording-history',
                    'GET'
                );
 
 
            return {
                success: true,
                count:
                    Number(
                        result.count || 0
                    ),
                recordings:
                    Array.isArray(
                        result.recordings
                    )
                        ? result.recordings
                        : []
            };
 
 
        } catch (error) {
 
            console.error(
                'Unable to load ServiceCall recording history:',
                error.message
            );
 
 
            return {
                success: false,
                code:
                    error.code ||
                    'RECORDING_HISTORY_FAILED',
                message:
                    error.message ||
                    'Unable to load recording history.',
                count: 0,
                recordings: []
            };
        }
    }
)

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

    async (
        event,
        options = {}
    ) => {

        try {

            let page =
                parseInt(
                    options.page,
                    10
                ) || 1;


            if (page < 1) {
                page = 1;
            }


            const search =
                String(
                    options.search || ''
                ).trim();

            const status =
    String(
        options.status || ''
    )
        .toLowerCase()
        .trim();


            const query =
    new URLSearchParams({
        page:
            String(page),

        page_size:
            '7',

        search:
            search,

        status:
            status
    });


            const result =
                await serviceCallApiRequest(
                    '/my-meetings?' +
                        query.toString(),
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

                count: 0,

                total_count: 0,

                current_page: 1,

                page_size: 7,

                total_pages: 0,

                has_previous: false,

                has_next: false,

                search: '',

                status: '',

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

     timezone:
        String(
            meetingData.timezone || ''
        ).trim(),

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

/* =====================================================
   UPDATE MEETING
===================================================== */

ipcMain.handle(
    'servicecall-update-meeting',

    async (
        event,
        meetingSysId,
        meetingData
    ) => {

        try {

            /*
             * Meeting sys_id is required.
             */
            if (!meetingSysId) {

                return {
                    success: false,
                    code:
                        'MEETING_SYS_ID_REQUIRED',
                    message:
                        'Meeting sys_id is required.'
                };
            }


            /*
             * Basic meeting data validation.
             */
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


            /*
             * Build the payload sent to
             * ServiceNow.
             */
            const payload = {

                meeting_sys_id:
                    String(
                        meetingSysId
                    ).trim(),

                title:
                    String(
                        meetingData.title
                    ).trim(),

                description:
                    String(
                        meetingData.description ||
                        ''
                    ).trim(),

                scheduled_start:
                    meetingData
                        .scheduled_start,

                scheduled_end:
                    meetingData
                        .scheduled_end,

                timezone:
                    String(
                        meetingData.timezone ||
                        ''
                    ).trim(),

                participants:
                    Array.isArray(
                        meetingData.participants
                    )
                        ? meetingData.participants
                        : []
            };


            console.log(
                'Updating ServiceCall meeting:',
                payload
            );


            /*
             * Send the update to ServiceNow.
             *
             * We will create this REST resource
             * in the next step.
             */
            const result =
                await serviceCallApiRequest(
                    '/update-meeting',
                    'POST',
                    payload
                );


            return result;


        } catch (error) {

            console.error(
                'Unable to update ServiceCall meeting:',
                error.message
            );


            return {
                success: false,
                code:
                    error.code ||
                    'UPDATE_MEETING_FAILED',
                message:
                    error.message ||
                    'Unable to update meeting.'
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


            /*
             * -------------------------------------------------
             * OPEN EXISTING SERVICECALL WINDOW FOR THE MEETING
             * -------------------------------------------------
             *
             * A meeting creates a normal ServiceCall call record.
             *
             * We reuse the existing call window instead of
             * creating a separate meeting window.
             */
            if (
                result &&
                result.success === true &&
                result.call_sys_id
            ) {

                /*
                 * Mark this call as already handled so the
                 * generic outgoing-call polling loop does not
                 * treat the meeting as a normal direct call.
                 */
                activeOutgoingCallId =
                    result.call_sys_id;


                showCallWindow(
                    'connected',

                    {
                        callSysId:
                            result.call_sys_id,

                        callNumber:
                            result.call_number ||
                            '',

                        /*
                         * For a meeting, the main display name
                         * is the meeting title.
                         */
                        name:
                            result.title ||
                            'ServiceCall Meeting',

                        department:
                            'Meeting',

                        /*
                         * Meetings use the existing conference
                         * call functionality.
                         */
                        isConference:
                            true,

                        /*
                         * Keep meeting context available for
                         * the next step.
                         */
                        isMeeting:
                            true,

                        meetingSysId:
                            result.meeting_sys_id ||
                            meetingSysId,

                        meetingNumber:
                            result.meeting_number ||
                            '',

                        meetingTitle:
                            result.title ||
                            'ServiceCall Meeting'
                    }
                );
            }


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

/* =======================================================
   MEETING CHANGE NOTIFICATION
======================================================= */

ipcMain.on(
    'servicecall-meeting-changed',
    (
        event,
        meetingSysId
    ) => {

        /*
         * Forward the meeting change from the
         * call window to the main desktop window.
         */
        if (
            mainWindow &&
            !mainWindow.isDestroyed()
        ) {

            mainWindow.webContents.send(
                'servicecall-meeting-changed',
                {
                    meetingSysId:
                        meetingSysId || ''
                }
            );
        }
    }
);

/* =====================================================
   RENDERER READY FOR DEEP LINKS
===================================================== */

ipcMain.on(
    'servicecall-renderer-ready',
    (event) => {

        /*
         * Only accept this signal from
         * the main ServiceCall window.
         */
        if (
            !mainWindow ||
            mainWindow.isDestroyed() ||
            event.sender !==
                mainWindow.webContents
        ) {
            return;
        }


        if (!pendingDeepLink) {
            return;
        }


        console.log(
            'Renderer ready. Sending pending ServiceCall deep link:',
            pendingDeepLink
        );


        mainWindow.webContents.send(
            'servicecall-deep-link',
            {
                url:
                    pendingDeepLink
            }
        );


        pendingDeepLink =
            null;
    }
);

/* =======================================================
   JOIN MEETING
======================================================= */

ipcMain.handle(
    'servicecall-join-meeting',

    async (
        event,
        meetingSysId
    ) => {

        try {

            if (!meetingSysId) {

                return {
                    success: false,
                    code: 'MEETING_REQUIRED',
                    message:
                        'Meeting sys_id is required.'
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
                    '/join-meeting',
                    'POST',
                    payload
                );


            if (
                result &&
                result.success === true &&
                result.call_sys_id
            ) {

                /*
 * Explicit Join means the user wants
 * this meeting call again.
 */
intentionallyLeftCallIds.delete(
    result.call_sys_id
);

                /*
                 * This user is now connected
                 * to the existing meeting call.
                 */
                activeOutgoingCallId =
                    result.call_sys_id;


                /*
                 * Reuse our existing call window.
                 *
                 * IMPORTANT:
                 * Pass full meeting context so
                 * call-window.js knows this is
                 * a ServiceCall Meeting.
                 */
                showCallWindow(
                    'connected',
                    {
                        callSysId:
                            result.call_sys_id,

                        callNumber:
                            result.call_number ||
                            '',

                        name:
                            result.title ||
                            'ServiceCall Meeting',

                        department:
                            'Meeting',

                        isConference:
                            true,

                        isMeeting:
                            true,

                        meetingSysId:
                            result.meeting_sys_id ||
                            meetingSysId,

                        meetingNumber:
                            result.meeting_number ||
                            '',

                        meetingTitle:
                            result.title ||
                            'ServiceCall Meeting'
                    }
                );
            }


            return result;


        } catch (error) {

            console.error(
                'Unable to join ServiceCall meeting:',
                error.message
            );


            return {
                success: false,
                code:
                    error.code ||
                    'JOIN_MEETING_FAILED',
                message:
                    error.message ||
                    'Unable to join meeting.'
            };
        }
    }
);

/* =======================================================
   LEAVE MEETING
======================================================= */

ipcMain.handle(
    'servicecall-leave-meeting',

    async (
        event,
        meetingSysId
    ) => {

        try {

            if (!meetingSysId) {

                return {
                    success: false,
                    code: 'MEETING_REQUIRED',
                    message:
                        'Meeting sys_id is required.'
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
                    '/leave-meeting',
                    'POST',
                    payload
                );


            if (
                result &&
                result.success === true
            ) {

                /*
 * Remember that THIS user deliberately
 * left this call.
 *
 * The meeting itself may remain In Progress,
 * so polling must not reopen its window.
 */
if (result.call_sys_id) {

    intentionallyLeftCallIds.add(
        result.call_sys_id
    );
}

                /*
                 * The user left this meeting,
                 * so clear this call from the
                 * active desktop state if needed.
                 */
                if (
                    activeOutgoingCallId ===
                    result.call_sys_id
                ) {

                    activeOutgoingCallId =
                        null;
                }


                /*
                 * Refresh the Meetings page.
                 */
                if (
                    mainWindow &&
                    !mainWindow.isDestroyed()
                ) {

                    mainWindow.webContents.send(
                        'servicecall-meeting-changed',
                        {
                            meetingSysId:
                                result.meeting_sys_id ||
                                meetingSysId
                        }
                    );
                }
            }


            return result;


        } catch (error) {

            console.error(
                'Unable to leave ServiceCall meeting:',
                error.message
            );


            return {
                success: false,
                code:
                    error.code ||
                    'LEAVE_MEETING_FAILED',
                message:
                    error.message ||
                    'Unable to leave meeting.'
            };
        }
    }
);

/* =======================================================
   END MEETING
======================================================= */

ipcMain.handle(
    'servicecall-end-meeting',

    async (
        event,
        meetingSysId
    ) => {

        try {

            if (!meetingSysId) {

                return {
                    success: false,
                    code: 'MEETING_REQUIRED',
                    message:
                        'Meeting sys_id is required.'
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
                    '/end-meeting',
                    'POST',
                    payload
                );


            if (
                result &&
                result.success === true
            ) {

                /*
                 * Meeting call has ended.
                 */
                if (
                    activeOutgoingCallId ===
                    result.call_sys_id
                ) {

                    activeOutgoingCallId =
                        null;
                }


                if (
                    activeIncomingCallId ===
                    result.call_sys_id
                ) {

                    activeIncomingCallId =
                        null;
                }


                /*
                 * Tell the main Meetings page
                 * that meeting data changed.
                 */
                if (
                    mainWindow &&
                    !mainWindow.isDestroyed()
                ) {

                    mainWindow.webContents.send(
                        'servicecall-meeting-changed',
                        {
                            meetingSysId:
                                result.meeting_sys_id ||
                                meetingSysId
                        }
                    );
                }
            }


            return result;


        } catch (error) {

            console.error(
                'Unable to end ServiceCall meeting:',
                error.message
            );


            return {
                success: false,
                code:
                    error.code ||
                    'END_MEETING_FAILED',
                message:
                    error.message ||
                    'Unable to end meeting.'
            };
        }
    }
);

/* =======================================================
   CANCEL MEETING
======================================================= */

ipcMain.handle(
    'servicecall-cancel-meeting',

    async (
        event,
        meetingSysId
    ) => {

        try {

            if (!meetingSysId) {

                return {
                    success: false,
                    code: 'MEETING_REQUIRED',
                    message:
                        'Meeting sys_id is required.'
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
                    '/cancel-meeting',
                    'POST',
                    payload
                );


            if (
                result &&
                result.success === true
            ) {

                /*
                 * Tell the main Meetings page
                 * that this meeting changed.
                 */
                if (
                    mainWindow &&
                    !mainWindow.isDestroyed()
                ) {

                    mainWindow.webContents.send(
                        'servicecall-meeting-changed',
                        {
                            meetingSysId:
                                result.meeting_sys_id ||
                                meetingSysId
                        }
                    );
                }
            }


            return result;


        } catch (error) {

            console.error(
                'Unable to cancel ServiceCall meeting:',
                error.message
            );


            return {
                success: false,
                code:
                    error.code ||
                    'CANCEL_MEETING_FAILED',
                message:
                    error.message ||
                    'Unable to cancel meeting.'
            };
        }
    }
);

ipcMain.handle(
    'servicecall-get-meeting-details',

    async (
        event,
        meetingSysId
    ) => {

        try {

            if (!meetingSysId) {

                return {
                    success: false,
                    code: 'MEETING_REQUIRED',
                    message:
                        'Meeting sys_id is required.'
                };
            }


            const query =
                new URLSearchParams({
                    meeting_sys_id:
                        String(
                            meetingSysId
                        ).trim()
                });


            const result =
                await serviceCallApiRequest(
                    '/meeting-details?' +
                        query.toString(),
                    'GET'
                );


            return result;


        } catch (error) {

            console.error(
                'Unable to get ServiceCall meeting details:',
                error.message
            );


            return {
                success: false,
                code:
                    error.code ||
                    'MEETING_DETAILS_FAILED',
                message:
                    error.message ||
                    'Unable to retrieve meeting details.'
            };
        }
    }
);

/* =======================================================
   SERVICECALL NOTIFICATIONS
======================================================= */

/*
 * Get notifications for the currently
 * authenticated ServiceCall user.
 *
 * ServiceNow determines the recipient from
 * the authenticated OAuth user.
 */
ipcMain.handle(
    'servicecall-get-notifications',

    async (
        event,
        options = {}
    ) => {

        try {

            let page =
                parseInt(
                    options.page,
                    10
                ) || 1;


            if (page < 1) {
                page = 1;
            }


            let pageSize =
                parseInt(
                    options.pageSize,
                    10
                ) || 20;


            /*
             * Keep desktop requests reasonable.
             */
            if (pageSize < 1) {
                pageSize = 20;
            }


            if (pageSize > 50) {
                pageSize = 50;
            }

            const search =
    String(
        options.search || ''
    )
        .trim()
        .substring(
            0,
            100
        );


            const query =
    new URLSearchParams({
        page:
            String(page),

        page_size:
            String(pageSize)
    });


if (search) {

    query.set(
        'search',
        search
    );
}


            const result =
                await serviceCallApiRequest(
                    '/notifications?' +
                        query.toString(),
                    'GET'
                );


            return result;


        } catch (error) {

            console.error(
                'Unable to get ServiceCall notifications:',
                error.message
            );


            return {

                success: false,

                code:
                    error.code ||
                    'GET_NOTIFICATIONS_FAILED',

                message:
                    error.message ||
                    'Unable to retrieve notifications.',

                notifications: [],

                unread_count: 0,

                page: 1,

                page_size: 20,

                has_more: false
            };
        }
    }
);

/* =======================================================
   MARK NOTIFICATION READ
======================================================= */

ipcMain.handle(
    'servicecall-mark-notification-read',

    async (
        event,
        notificationSysId
    ) => {

        try {

            const sysId =
                String(
                    notificationSysId || ''
                ).trim();


            if (!sysId) {

                return {
                    success: false,
                    code:
                        'NOTIFICATION_REQUIRED',
                    message:
                        'Notification sys_id is required.'
                };
            }


            return await serviceCallApiRequest(
                '/mark-notification-read',
                'POST',
                {
                    notification_sys_id:
                        sysId
                }
            );


        } catch (error) {

            console.error(
                'Unable to mark ServiceCall notification as read:',
                error.message
            );


            return {
                success: false,

                code:
                    error.code ||
                    'MARK_NOTIFICATION_READ_FAILED',

                message:
                    error.message ||
                    'Unable to mark notification as read.'
            };
        }
    }
);

/* -------------------------------------------------------
   DISMISS NOTIFICATION POPUP
------------------------------------------------------- */

ipcMain.on(
    'servicecall-dismiss-notification-popup',
    (
        event
    ) => {
 
        /*
         * Get the exact BrowserWindow that
         * sent this dismiss request.
         *
         * This is important now because several
         * notification popups may exist at once.
         */
 
        const popupWindow =
            BrowserWindow.fromWebContents(
                event.sender
            );
 
 
        if (
            !popupWindow ||
            popupWindow.isDestroyed()
        ) {
 
            return;
        }
 
 
        popupWindow.destroy();
    }
);

ipcMain.on(
    'servicecall-open-notification',
    (
        event,
        notificationData
    ) => {
 
        console.log(
            'ServiceCall notification OPEN request received:',
            notificationData
        );
 
 
        /*
         * Restore ServiceCall when the user
         * clicks a desktop notification.
         */
 
        showMainWindow();
 
 
        /*
         * Make sure the main ServiceCall
         * window is brought to the front.
         */
 
        if (
            !mainWindow ||
            mainWindow.isDestroyed()
        ) {
            return;
        }
 
 
        if (
            mainWindow.isMinimized()
        ) {
 
            mainWindow.restore();
        }
 
 
        mainWindow.show();
 
        mainWindow.focus();
 
 
        /*
         * -----------------------------------------
         * MEETING NOTIFICATION
         * -----------------------------------------
         *
         * If this notification belongs to a
         * meeting, forward that meeting to the
         * main renderer.
         *
         * The renderer will remain responsible
         * for securely retrieving the meeting
         * details from ServiceNow.
         */
 
        const meetingSysId =
            String(
                notificationData &&
                notificationData.meetingSysId
                    ? notificationData.meetingSysId
                    : ''
            ).trim();
 
 
        if (
            /^[0-9a-f]{32}$/i.test(
                meetingSysId
            )
        ) {
 
            console.log(
                'Opening meeting from ServiceCall notification:',
                meetingSysId
            );
 
 
            mainWindow.webContents.send(
                'servicecall-open-notification-meeting',
                {
                    meetingSysId:
                        meetingSysId,
 
                    notificationSysId:
                        String(
                            notificationData &&
                            notificationData.notificationSysId
                                ? notificationData.notificationSysId
                                : ''
                        ),
 
                    type:
                        String(
                            notificationData &&
                            notificationData.type
                                ? notificationData.type
                                : ''
                        )
                }
            );
        }
    }
);

ipcMain.handle(
    'servicecall-start-call',

    async (
        event,
        targetUserSysId
    ) => {

        try {

            const targetUser =
                String(
                    targetUserSysId || ''
                ).trim();


            if (!targetUser) {

                return {
                    success: false,
                    code:
                        'TARGET_USER_REQUIRED',
                    message:
                        'Target user is required.'
                };
            }


            const result =
                await serviceCallApiRequest(
                    '/start-call',
                    'POST',
                    {
                        target_user_sys_id:
                            targetUser
                    }
                );


            return result;


        } catch (error) {

            console.error(
                'Unable to start ServiceCall:',
                error.message
            );


            return {

                success: false,

                code:
                    error.code ||
                    'START_CALL_FAILED',

                message:
                    error.message ||
                    'Unable to start the ServiceCall.'
            };
        }
    }
);

ipcMain.handle(
    'servicecall-update-presence',

    async (
        event,
        presenceData = {}
    ) => {

        try {

            const status =
                String(
                    presenceData.status || ''
                ).trim();

            const oofReason =
                String(
                    presenceData.oofReason || ''
                ).trim();

            if (!status) {

                return {
                    success: false,
                    code: 'PRESENCE_REQUIRED',
                    message:
                        'Presence status is required.'
                };
            }

            return await serviceCallApiRequest(
                '/presence',
                'POST',
                {
                    status: status,
                    oof_reason: oofReason
                }
            );

        } catch (error) {

            console.error(
                'Unable to update ServiceCall presence:',
                error.message
            );

            return {
                success: false,
                code:
                    error.code ||
                    'UPDATE_PRESENCE_FAILED',
                message:
                    error.message ||
                    'Unable to update presence.'
            };
        }
    }
);

ipcMain.handle(
    'servicecall-get-my-presence',

    async () => {

        try {

            const result =
                await serviceCallApiRequest(
                    '/presence',
                    'GET'
                );

            return result;

        } catch (error) {

            console.error(
                'Unable to get ServiceCall presence:',
                error.message
            );

            return {
                success: false,

                code:
                    error.code ||
                    'GET_PRESENCE_FAILED',

                message:
                    error.message ||
                    'Unable to retrieve presence.'
            };
        }
    }
);

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
 
 
            /*
             * Keep current runtime identity
             * and authorization up to date.
             */
 
            currentServiceCallUser =
                currentUser?.user || null;
 
            currentServiceCallAuthorization =
                authorization;
 
 
            /*
             * ACCESS IS STILL NOT AVAILABLE
             */
 
            if (
                authorization.allowed !== true
            ) {
 
                return {
                    success: true,
                    authenticated: true,
                    authorized: false,
                    state: 'access_denied',
                    user:
                        currentServiceCallUser,
                    authorization:
                        currentServiceCallAuthorization
                };
            }
 
 
            /*
             * ACCESS IS AVAILABLE
             *
             * If ServiceCall was suspended because
             * access had previously been removed,
             * restore the complete runtime and UI.
             */
 
            if (serviceCallAccessUnavailable) {
 
                console.log(
                    'ServiceCall access restored by manual check.'
                );
 
 
                /*
                 * Change the state before restoring.
                 */
 
                serviceCallAccessUnavailable =
                    false;
 
 
                try {
 
                    await resumeServiceCallRuntime();
 
                    await restoreServiceCallApplication();
 
                }
                catch (restoreError) {
 
                    /*
                     * Restoration failed.
                     *
                     * Put ServiceCall back into the
                     * unavailable state so another
                     * check can retry.
                     */
 
                    serviceCallAccessUnavailable =
                        true;
 
 
                    console.error(
                        'Unable to restore ServiceCall after manual access check:',
                        restoreError
                    );
 
 
                    throw restoreError;
                }
 
            }
            else {
 
                /*
                 * Normal access check.
                 *
                 * ServiceCall was not previously
                 * suspended.
                 */
 
                await startHeartbeatLoop();
 
                startIncomingCallLoop();
 
                startOutgoingCallLoop();

                startNotificationLoop();

                startAuthorizationMonitor();
            }
 
 
            return {
                success: true,
                authenticated: true,
                authorized: true,
                state: 'ready',
                user:
                    currentServiceCallUser,
                authorization:
                    currentServiceCallAuthorization
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

ipcMain.handle(
    'servicecall-get-current-account',
    async () => {

        /*
         * No authenticated identity
         * has been resolved yet.
         */

        if (!currentServiceCallUser) {

            return {
                success: false,
                authenticated: false,
                user: null,
                authorization: null
            };
        }


        return {
            success: true,
            authenticated: true,

            user:
                currentServiceCallUser,

            authorization:
                currentServiceCallAuthorization || {
                    allowed: false,
                    is_servicecall_user: false,
                    is_servicecall_admin: false
                }
        };
    }
);

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
     * STOP NOTIFICATION MONITOR
     * -----------------------------------------
     */
 
    stopNotificationLoop();
 
 
    /*
     * -----------------------------------------
     * STOP AUTHORIZATION MONITOR
     * -----------------------------------------
     */
 
    stopAuthorizationMonitor();
 
 
    /*
     * -----------------------------------------
     * RESET NOTIFICATION SESSION STATE
     * -----------------------------------------
     *
     * Desktop notification detection state
     * belongs to the currently authenticated
     * ServiceCall account.
     *
     * The next account must establish its own
     * notification baseline.
     */
 
    surfacedNotificationIds.clear();
 
    console.log('ServiceCall Notification Session state reset')
 
 
    console.log(
        'ServiceCall notification baseline reset.'
    );
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

ipcMain.handle(
    'servicecall-sign-out',
    async () => {

        try {

            console.log(
                'ServiceCall sign out requested.'
            );

            /*
 * Preserve the latest OAuth session
 * inside the currently active saved
 * account before signing out.
 */
let config =
    loadConfig();

config =
    syncActiveAccountTokens(
        config
    );

saveConfig(
    config
);


            /*
             * Stop heartbeat/call monitoring,
             * mark this desktop offline,
             * and clear the active in-memory
             * account identity.
             */
            await stopCurrentServiceCallSession();


            /*
             * IMPORTANT:
             *
             * Do NOT delete accessToken or
             * refreshToken here.
             *
             * Saved-account support will own
             * token storage shortly.
             */


            /*
             * Return the main window to our
             * authentication/account entry page.
             */
            if (
                mainWindow &&
                !mainWindow.isDestroyed()
            ) {

                await mainWindow.loadFile(
                    'auth/auth-gate.html'
                );

                mainWindow.show();

                mainWindow.focus();
            }


            console.log(
                'ServiceCall signed out of active session.'
            );


            return {
                success: true,
                state: 'signed_out'
            };


        } catch (error) {

            console.error(
                'ServiceCall sign out failed:',
                error
            );


            return {
                success: false,
                state: 'error',
                message:
                    error?.message ||
                    'Unable to sign out of ServiceCall.'
            };
        }
    }
);

async function signOutDesktopSession() {

    const deviceId =
        getOrCreateDeviceId();


    const result =
        await serviceCallApiRequest(
            '/desktop-sign-out',
            'POST',
            {
                device_id:
                    deviceId
            }
        );


    if (
        !result ||
        result.success !== true
    ) {

        throw new Error(
            result?.message ||
            'Unable to sign out of ServiceCall Desktop.'
        );
    }


    console.log(
        'ServiceCall desktop sign out:',
        result
    );


    return result;
}

ipcMain.handle(
    'servicecall-get-saved-accounts',
    async () => {

        try {

            const accounts =
                getSavedAccounts();


            return {
                success: true,
                accounts: accounts
            };


        } catch (error) {

            console.error(
                'Unable to load ServiceCall saved accounts:',
                error
            );


            return {
                success: false,
                accounts: [],
                message:
                    error?.message ||
                    'Unable to load saved accounts.'
            };
        }
    }
);

ipcMain.handle(
    'servicecall-remove-saved-account',
    async (
        event,
        accountKey
    ) => {

        try {

            return await removeSavedAccount(
                String(
                    accountKey || ''
                )
            );

        } catch (error) {

            console.error(
                'Unable to remove saved ServiceCall account:',
                error
            );


            return {
                success: false,
                state: 'error',
                message:
                    error?.message ||
                    'Unable to remove the saved account.'
            };
        }
    }
);

async function activateSavedAccount(
    accountKey
) {

    let config =
        loadConfig();

    config =
        ensureSavedAccountStructure(
            config
        );


    const account =
        config.savedAccounts[
            accountKey
        ];


    if (!account) {

        return {
            success: false,
            state: 'account_not_found',
            message:
                'The selected ServiceCall account could not be found.'
        };
    }


    /*
     * Restore this account's instance
     * and OAuth session into the active
     * runtime configuration.
     */

    config.instanceUrl =
        account.instanceUrl || '';

    config.accessToken =
        account.accessToken || '';

    config.refreshToken =
        account.refreshToken || '';

    config.tokenType =
        account.tokenType ||
        'Bearer';

    config.expiresIn =
        account.expiresIn || 0;

    config.tokenObtainedAt =
        account.tokenObtainedAt || 0;

    config.activeAccountKey =
        accountKey;


    saveConfig(
        config
    );


    try {

        /*
         * May automatically refresh the
         * selected account's access token.
         */

        await ensureValidAccessToken();


        /*
         * IMPORTANT:
         * Never trust the role snapshot
         * stored in savedAccounts.
         *
         * Ask ServiceNow again.
         */

        const currentUser =
            await getCurrentServiceCallUser();

        const authorization =
            currentUser?.authorization || {};

        /*
         * SECURITY CHECK:
         *
         * Make sure the OAuth identity still
         * belongs to the account that the
         * user selected.
         */

        if (
            String(
                currentUser?.user?.sys_id ||
                ''
            ) !==
            String(
                account.userSysId || ''
            )
        ) {

            throw new Error(
                'The authenticated ServiceNow identity does not match the selected saved account.'
            );
        }

        /*
 * Restore the authenticated identity
 * into the active Electron runtime.
 *
 * Saved-account activation must establish
 * the same runtime identity state as a
 * fresh OAuth login.
 */

currentServiceCallUser =
    currentUser?.user || null;

currentServiceCallAuthorization =
    authorization;


        /*
         * Refresh cached DISPLAY metadata.
         * These values never grant access.
         */

        config =
            loadConfig();

        config =
            ensureSavedAccountStructure(
                config
            );


        const savedAccount =
            config.savedAccounts[
                accountKey
            ];


        if (savedAccount) {

            savedAccount.name =
                currentUser?.user?.name ||
                savedAccount.name ||
                '';

            savedAccount.userName =
                currentUser?.user?.user_name ||
                savedAccount.userName ||
                '';

            savedAccount.email =
                currentUser?.user?.email ||
                '';

            savedAccount.serviceCallId =
                currentUser?.user
                    ?.servicecall_id ||
                '';

            savedAccount.isServiceCallUser =
                authorization
                    .is_servicecall_user ===
                true;

            savedAccount.isServiceCallAdmin =
                authorization
                    .is_servicecall_admin ===
                true;

            savedAccount.lastUsedAt =
                new Date().toISOString();


            config.savedAccounts[
                accountKey
            ] =
                savedAccount;


            saveConfig(
                config
            );
        }


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


        await startHeartbeatLoop();

        startIncomingCallLoop();

        startOutgoingCallLoop();

        startNotificationLoop();

        startAuthorizationMonitor();
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


    } catch (error) {

        console.error(
            'Unable to activate saved ServiceCall account:',
            error
        );


        return {
            success: false,
            authenticated: false,
            authorized: false,
            state:
                'login_required',

            message:
                error?.message ||
                'This account needs to sign in again.'
        };
    }
}

/* =========================================================
   REMOVE SAVED ACCOUNT
========================================================= */

async function removeSavedAccount(
    accountKey
) {

    let config =
        loadConfig();

    config =
        ensureSavedAccountStructure(
            config
        );


    accountKey =
        String(
            accountKey || ''
        ).trim();


    if (
        !accountKey ||
        !config.savedAccounts[
            accountKey
        ]
    ) {

        return {
            success: false,
            state: 'account_not_found',
            message:
                'The saved ServiceCall account could not be found.'
        };
    }


    const wasActive =
        config.activeAccountKey ===
        accountKey;


    /*
     * If this happens to be the currently
     * active account, stop its runtime
     * session first.
     */
    if (wasActive) {

        try {

            await stopCurrentServiceCallSession();

        } catch (error) {

            console.warn(
                'ServiceCall session cleanup during account removal failed:',
                error
            );
        }
    }


    /*
     * Delete the complete saved entry.
     *
     * Because OAuth credentials live inside
     * this account object, its saved tokens
     * disappear with it.
     */
    delete config.savedAccounts[
        accountKey
    ];


    if (wasActive) {

        config.activeAccountKey =
            '';


        /*
         * Clear the legacy/current runtime
         * OAuth fields too.
         *
         * Other saved accounts remain
         * completely untouched.
         */
        delete config.accessToken;
        delete config.refreshToken;
        delete config.tokenType;
        delete config.expiresIn;
        delete config.tokenObtainedAt;
    }


    saveConfig(
        config
    );


    console.log(
        'ServiceCall saved account removed:',
        accountKey
    );


    return {
        success: true,
        state: 'account_removed',
        removedAccountKey:
            accountKey
    };
}

ipcMain.handle(
    'servicecall-activate-saved-account',
    async (
        event,
        accountKey
    ) => {

        return await activateSavedAccount(
            String(
                accountKey || ''
            )
        );
    }
);



document.addEventListener(
    'DOMContentLoaded',
    async () => {

        /* -------------------------------------------------
           ELEMENTS
        ------------------------------------------------- */

        const form =
            document.getElementById(
                'instanceForm'
            );

        const input =
            document.getElementById(
                'instanceUrl'
            );

        const message =
            document.getElementById(
                'instanceMessage'
            );

        const loginButton =
            document.getElementById(
                'loginButton'
            );

        const signOutButton =
    document.getElementById(
        'signOutButton'
    );

        const openActiveCallButton =
            document.getElementById(
                'openActiveCallButton'
            );

        /* -------------------------------------------------
   PEOPLE ELEMENTS
------------------------------------------------- */

const peopleSearchInput =
    document.getElementById(
        'peopleSearchInput'
    );

const peopleSearchMessage =
    document.getElementById(
        'peopleSearchMessage'
    );

const peopleSearchResults =
    document.getElementById(
        'peopleSearchResults'
    );

/* =====================================================
   TOP BAR PRESENCE
===================================================== */

const presenceButton =
    document.getElementById(
        'presenceButton'
    );

const presenceMenu =
    document.getElementById(
        'presenceMenu'
    );

const presenceText =
    document.getElementById(
        'presenceText'
    );

const presenceDot =
    document.getElementById(
        'presenceDot'
    );

const presenceOptions =
    document.querySelectorAll(
        '.presence-option'
    );

const oofReasonPanel =
    document.getElementById(
        'oofReasonPanel'
    );

const oofReasonInput =
    document.getElementById(
        'oofReasonInput'
    );

const saveOofButton =
    document.getElementById(
        'saveOofButton'
    );

const cancelOofButton =
    document.getElementById(
        'cancelOofButton'
    );

let peopleSearchTimer =
    null;

        const connectionPill =
            document.getElementById(
                'connectionPill'
            );

        const currentPageTitle =
            document.getElementById(
                'currentPageTitle'
            );

        const meetingsContainer =
            document.getElementById(
                'meetingsContainer'
            );

        const scheduleMeetingButton =
            document.getElementById(
                'scheduleMeetingButton'
            );

        const meetingSearchInput =
    document.getElementById(
        'meetingSearchInput'
    );

const meetingSearchClear =
    document.getElementById(
        'meetingSearchClear'
    );

const meetingPagination =
    document.getElementById(
        'meetingPagination'
    );


const meetingStatusFilter =
    document.getElementById(
        'meetingStatusFilter'
    );

/* -------------------------------------------------
   NOTIFICATION ELEMENTS
------------------------------------------------- */

const notificationsContainer =
    document.getElementById(
        'notificationsContainer'
    );


const notificationPagination =
    document.getElementById(
        'notificationPagination'
    );


const notificationUnreadBadge =
    document.getElementById(
        'notificationUnreadBadge'
    );

const notificationUnreadFilterCount =
    document.getElementById(
        'notificationUnreadFilterCount'
    );


const refreshNotificationsButton =
    document.getElementById(
        'refreshNotificationsButton'
    );


const markAllNotificationsReadButton =
    document.getElementById(
        'markAllNotificationsReadButton'
    );


const notificationFilterButtons =
    document.querySelectorAll(
        '.notification-filter'
    );


const notificationsNavButton =
    document.querySelector(
        '[data-view="notificationsView"]'
    );

const notificationSearchInput =
    document.getElementById(
        'notificationSearchInput'
    );


const notificationSearchClear =
    document.getElementById(
        'notificationSearchClear'
    );

/* -------------------------------------------------
   NOTIFICATION DETAIL ELEMENTS
------------------------------------------------- */

const notificationListPanel =
    document.getElementById(
        'notificationListPanel'
    );

const notificationDetailPanel =
    document.getElementById(
        'notificationDetailPanel'
    );

const notificationDetailBackButton =
    document.getElementById(
        'notificationDetailBackButton'
    );

const notificationDetailIcon =
    document.getElementById(
        'notificationDetailIcon'
    );

const notificationDetailType =
    document.getElementById(
        'notificationDetailType'
    );

const notificationDetailTitle =
    document.getElementById(
        'notificationDetailTitle'
    );

const notificationDetailTime =
    document.getElementById(
        'notificationDetailTime'
    );

const notificationDetailMessage =
    document.getElementById(
        'notificationDetailMessage'
    );

const notificationDetailActions =
    document.getElementById(
        'notificationDetailActions'
    );

let currentNotificationPage = 1;

let currentNotificationFilter = 'all';

let notificationAutoRefreshTimer = null;

let currentNotificationSearch = '';

let notificationSearchTimer = null;

let knownNotificationIds =
    new Set();

/*
 * Notifications currently loaded from ServiceNow.
 *
 * Filters can use this immediately without
 * making another API request.
 */
let cachedNotifications = [];

let cachedNotificationResult = null;

let notificationsInitialized =
    false;

let notificationSearchVersion = 0;

let currentMeetingPage = 1;

let currentMeetingSearch = '';

let currentMeetingStatus = '';

let meetingSearchTimer = null;

let schedulePeopleSearchTimer = null;

let selectedMeetingPeople = [];

let currentMeetingTimezone = '';

let meetingsAutoRefreshTimer = null;

let currentMeetingDetails = '';

/*
 * Meeting form mode:
 *
 * create = scheduling a new meeting
 * edit   = modifying an existing meeting
 */
let meetingFormMode = 'create';


/*
 * Stores the meeting currently being edited.
 */
let editingMeetingSysId = '';

const meetingDetailsModal =
    document.getElementById(
        'meetingDetailsModal'
    );

const meetingDetailsCloseButton =
    document.getElementById(
        'meetingDetailsCloseButton'
    );

const meetingDetailsFooterCloseButton =
    document.getElementById(
        'meetingDetailsFooterCloseButton'
    );

const meetingDetailsActionButton =
    document.getElementById(
        'meetingDetailsActionButton'
    );

const meetingDetailsNumber =
    document.getElementById(
        'meetingDetailsNumber'
    );

const meetingDetailsHeading =
    document.getElementById(
        'meetingDetailsHeading'
    );

const meetingDetailsStatus =
    document.getElementById(
        'meetingDetailsStatus'
    );

const meetingDetailsDescription =
    document.getElementById(
        'meetingDetailsDescription'
    );

const meetingDetailsOrganizer =
    document.getElementById(
        'meetingDetailsOrganizer'
    );

const meetingDetailsStart =
    document.getElementById(
        'meetingDetailsStart'
    );

const meetingDetailsEnd =
    document.getElementById(
        'meetingDetailsEnd'
    );

const meetingDetailsStartedBy =
    document.getElementById(
        'meetingDetailsStartedBy'
    );

const meetingDetailsStartedAt =
    document.getElementById(
        'meetingDetailsStartedAt'
    );

const meetingDetailsEndedAt =
    document.getElementById(
        'meetingDetailsEndedAt'
    );

const meetingDetailsStartedByField =
    document.getElementById(
        'meetingDetailsStartedByField'
    );

const meetingDetailsStartedAtField =
    document.getElementById(
        'meetingDetailsStartedAtField'
    );

const meetingDetailsEndedAtField =
    document.getElementById(
        'meetingDetailsEndedAtField'
    );

const meetingDetailsParticipantCount =
    document.getElementById(
        'meetingDetailsParticipantCount'
    );

const meetingDetailsParticipants =
    document.getElementById(
        'meetingDetailsParticipants'
    );

    const scheduleMeetingModal =
    document.getElementById(
        'scheduleMeetingModal'
    );

const scheduleMeetingCloseButton =
    document.getElementById(
        'scheduleMeetingCloseButton'
    );

const scheduleMeetingCancelButton =
    document.getElementById(
        'scheduleMeetingCancelButton'
    );

const scheduleMeetingTitle =
    document.getElementById(
        'scheduleMeetingTitle'
    );

const scheduleMeetingDescription =
    document.getElementById(
        'scheduleMeetingDescription'
    );

const scheduleMeetingStart =
    document.getElementById(
        'scheduleMeetingStart'
    );

const scheduleMeetingEnd =
    document.getElementById(
        'scheduleMeetingEnd'
    );

    const scheduleMeetingTimezone =
    document.getElementById(
        'scheduleMeetingTimezone'
    );

const scheduleMeetingHeading =
    document.getElementById(
        'scheduleMeetingHeading'
    );


const scheduleMeetingSubtitle =
    document.getElementById(
        'scheduleMeetingSubtitle'
    );

const scheduleMeetingPeopleSearch =
    document.getElementById(
        'scheduleMeetingPeopleSearch'
    );

const scheduleMeetingPeopleResults =
    document.getElementById(
        'scheduleMeetingPeopleResults'
    );

const resetPresenceButton =
    document.getElementById(
        'resetPresenceButton'
    );

const scheduleMeetingSelectedPeople =
    document.getElementById(
        'scheduleMeetingSelectedPeople'
    );

const scheduleMeetingMessage =
    document.getElementById(
        'scheduleMeetingMessage'
    );

const scheduleMeetingSubmitButton =
    document.getElementById(
        'scheduleMeetingSubmitButton'
    );

    if (
    scheduleMeetingCloseButton
) {

    scheduleMeetingCloseButton.addEventListener(
        'click',
        closeScheduleMeetingModal
    );
}


if (
    scheduleMeetingCancelButton
) {

    scheduleMeetingCancelButton.addEventListener(
        'click',
        closeScheduleMeetingModal
    );
}


if (
    scheduleMeetingModal
) {

    scheduleMeetingModal.addEventListener(
        'click',
        event => {

            if (
                event.target.hasAttribute(
                    'data-schedule-meeting-close'
                )
            ) {

                closeScheduleMeetingModal();
            }
        }
    );
}

        /* -------------------------------------------------
           CONNECTION STATUS
        ------------------------------------------------- */

        function setConnectionDisplay(
            connected,
            text
        ) {

            if (!connectionPill) {
                return;
            }


            connectionPill.textContent =
                text;


            connectionPill.classList.toggle(
                'connected',
                connected
            );
        }


        /* -------------------------------------------------
           LOAD SAVED INSTANCE
        ------------------------------------------------- */

        try {

            const savedInstance =
                await window.serviceCall
                    .getInstance();


            if (
                input &&
                savedInstance.instanceUrl
            ) {

                input.value =
                    savedInstance.instanceUrl;
            }


        } catch (error) {

            console.error(
                'Unable to load saved ServiceNow instance:',
                error
            );
        }


        /* -------------------------------------------------
           CHECK CONNECTION
        ------------------------------------------------- */

        try {

            const connectionStatus =
                await window.serviceCall
                    .getConnectionStatus();


            if (
                connectionStatus.connected
            ) {

                loginButton.disabled =
                    true;

                loginButton.textContent =
                    'Connected to ServiceNow';


                setConnectionDisplay(
                    true,
                    'Connected'
                );

                await loadMyPresence();

                if (message) {

                    message.textContent =
                        connectionStatus.message ||
                        '';
                }


            } else {

                loginButton.disabled =
                    false;

                loginButton.textContent =
                    'Sign in to ServiceNow';


                setConnectionDisplay(
                    false,
                    'Not connected'
                );
            }


        } catch (error) {

            console.error(
                'Unable to check ServiceCall connection:',
                error
            );


            setConnectionDisplay(
                false,
                'Connection unavailable'
            );
        }


        /* -------------------------------------------------
           SAVE INSTANCE
        ------------------------------------------------- */

        if (form) {

            form.addEventListener(
                'submit',

                async (event) => {

                    event.preventDefault();


                    const instanceUrl =
                        input.value.trim();


                    message.textContent =
                        'Saving ServiceNow instance...';


                    try {

                        const result =
                            await window
                                .serviceCall
                                .saveInstance(
                                    instanceUrl
                                );


                        message.textContent =
                            result.message ||
                            '';


                    } catch (error) {

                        console.error(
                            'Save instance failed:',
                            error
                        );


                        message.textContent =
                            'Unable to save the ServiceNow instance.';
                    }
                }
            );
        }


        /* -------------------------------------------------
           LOGIN
        ------------------------------------------------- */

        if (loginButton) {

            loginButton.addEventListener(
                'click',

                async () => {

                    message.textContent =
                        'Opening ServiceNow sign-in...';


                    try {

                        const result =
                            await window
                                .serviceCall
                                .startLogin();


                        message.textContent =
                            result.message ||
                            '';


                    } catch (error) {

                        console.error(
                            'ServiceNow login failed:',
                            error
                        );


                        message.textContent =
                            'Unable to start ServiceNow sign-in.';
                    }
                }
            );
        }


        /* -------------------------------------------------
           AUTH STATUS EVENTS
        ------------------------------------------------- */

        window.serviceCall.onAuthStatus(
            async(data) => {

                if (message) {

                    message.textContent =
                        data.message ||
                        '';
                }


                if (
                    data.status ===
                    'connected'
                ) {

                    loginButton.disabled =
                        true;

                    loginButton.textContent =
                        'Connected to ServiceNow';


                    setConnectionDisplay(
                        true,
                        'Connected'
                    );
await loadMyPresence();

                } else if (
                    data.status === 'warning' ||
                    data.status === 'error' ||
                    data.status ===
                        'authentication_required'
                ) {

                    loginButton.disabled =
                        false;

                    loginButton.textContent =
                        'Sign in to ServiceNow';


                    setConnectionDisplay(
                        false,
                        'Connection required'
                    );
                }
            }
        );


        /* -------------------------------------------------
           OPEN ACTIVE CALL
        ------------------------------------------------- */

        if (openActiveCallButton) {

            openActiveCallButton.addEventListener(
                'click',

                async () => {

                    try {

                        const result =
                            await window
                                .serviceCall
                                .openActiveCall();


                        if (
                            !result.active_call &&
                            message
                        ) {

                            message.textContent =
                                result.message ||
                                'No active call is available.';
                        }


                    } catch (error) {

                        console.error(
                            'Open active call failed:',
                            error
                        );


                        if (message) {

                            message.textContent =
                                'Unable to open the active call.';
                        }
                    }
                }
            );
        }


        /* -------------------------------------------------
           SIDEBAR NAVIGATION
        ------------------------------------------------- */

        const navigationButtons =
            document.querySelectorAll(
                '.nav-button[data-view]'
            );


        const views =
            document.querySelectorAll(
                '.view'
            );


        navigationButtons.forEach(
            (button) => {

                button.addEventListener(
                    'click',

                    async () => {

                        const targetView =
                            button.dataset.view;


                        /*
                         * Hide all pages.
                         */
                        views.forEach(
                            (view) => {

                                view.classList.remove(
                                    'active'
                                );
                            }
                        );


                        /*
                         * Remove active state
                         * from navigation.
                         */
                        navigationButtons.forEach(
                            (navButton) => {

                                navButton.classList.remove(
                                    'active'
                                );
                            }
                        );


                        /*
                         * Show selected page.
                         */
                        const selectedView =
                            document.getElementById(
                                targetView
                            );


                        if (selectedView) {

                            selectedView.classList.add(
                                'active'
                            );
                        }


                        button.classList.add(
                            'active'
                        );


                        /*
 * Do not use the entire button text because
 * some navigation buttons can contain badges.
 *
 * Example:
 *
 * Notifications + unread badge "3"
 *
 * should still produce the page title:
 *
 * Notifications
 */
let pageName = '';


const explicitPageNames = {

    homeView:
        'Home',

    peopleView:
        'People',

    meetingsView:
        'Meetings',

    notificationsView:
        'Notifications',

    chatView:
        'Chat',

    historyView:
        'History',

    recordingsView:
        'Recordings',

    settingsView:
        'Settings'
};


pageName =
    explicitPageNames[
        targetView
    ] ||
    button.textContent.trim();


if (currentPageTitle) {

    currentPageTitle.textContent =
        pageName;
}


                       if (
    targetView ===
    'meetingsView'
) {

    await loadMeetings();

    startMeetingsAutoRefresh();

} else {

    stopMeetingsAutoRefresh();
}


/*
 * Notifications are different from Meetings.
 *
 * Their background monitor runs globally,
 * but opening the Notifications page performs
 * a normal visible refresh.
 */
if (
    targetView ===
    'notificationsView'
) {

    currentNotificationPage =
        1;

    await loadNotifications(
        false
    );
}
                    }
                );
            }
        );


        /* -------------------------------------------------
           MEETING HELPERS
        ------------------------------------------------- */

        function escapeHtml(
            value
        ) {

            return String(
                value || ''
            )
                .replace(
                    /&/g,
                    '&amp;'
                )
                .replace(
                    /</g,
                    '&lt;'
                )
                .replace(
                    />/g,
                    '&gt;'
                )
                .replace(
                    /"/g,
                    '&quot;'
                )
                .replace(
                    /'/g,
                    '&#039;'
                );
        }


        function formatMeetingDate(
            value
        ) {

            if (!value) {
                return '';
            }


            /*
             * ServiceNow returns:
             *
             * YYYY-MM-DD HH:mm:ss
             *
             * For now we display that authoritative
             * value without applying timezone
             * conversion in the renderer.
             */
            return value;
        }


        function getStatusClass(
            state
        ) {

            const normalized =
                String(
                    state || ''
                )
                    .toLowerCase()
                    .trim();


            if (
                normalized ===
                'in progress'
            ) {

                return 'in-progress';
            }


            if (
                normalized ===
                'scheduled'
            ) {

                return 'scheduled';
            }


            return '';
        }

        function formatMeetingDetailsValue(
    value
) {

    const text =
        String(
            value || ''
        ).trim();

    return text || '—';
}


function formatMeetingDetailsStatus(
    value
) {

    const text =
        String(
            value || ''
        )
            .trim()
            .toLowerCase();

    if (!text) {
        return '—';
    }

    return text
        .split(' ')
        .map(
            word =>
                word
                    ? word.charAt(0).toUpperCase() +
                      word.slice(1)
                    : ''
        )
        .join(' ');
}


function closeMeetingDetailsModal() {

    if (!meetingDetailsModal) {
        return;
    }

    meetingDetailsModal.classList.remove(
        'open'
    );

    meetingDetailsModal.setAttribute(
        'aria-hidden',
        'true'
    );
}

function getDateTimeLocalValueInTimezone(
    date,
    timeZone
) {

    if (!date || !timeZone) {
        return '';
    }

    const parts =
        new Intl.DateTimeFormat(
            'en-CA',
            {
                timeZone: timeZone,
                year: 'numeric',
                month: '2-digit',
                day: '2-digit',
                hour: '2-digit',
                minute: '2-digit',
                hourCycle: 'h23'
            }
        ).formatToParts(date);


    const values = {};

    parts.forEach(
        part => {

            if (part.type !== 'literal') {
                values[part.type] =
                    part.value;
            }
        }
    );


    return (
        values.year +
        '-' +
        values.month +
        '-' +
        values.day +
        'T' +
        values.hour +
        ':' +
        values.minute
    );
}

function openScheduleMeetingModal() {

    if (!scheduleMeetingModal) {
        return;
    }

    /*
 * Normal Schedule Meeting button always
 * opens the form in CREATE mode.
 */
meetingFormMode =
    'create';

editingMeetingSysId =
    '';

    if (scheduleMeetingHeading) {

    scheduleMeetingHeading.textContent =
        'Schedule Meeting';
}


if (scheduleMeetingSubtitle) {

    scheduleMeetingSubtitle.textContent =
        'Create a new ServiceCall meeting.';
}


if (scheduleMeetingSubmitButton) {

    scheduleMeetingSubmitButton.textContent =
        'Schedule Meeting';
}

    if (scheduleMeetingTimezone) {

    scheduleMeetingTimezone.textContent =
        currentMeetingTimezone ||
        'Loading...';
}


    /*
     * Start every new scheduling attempt
     * with a clean form.
     */

    if (scheduleMeetingTitle) {
        scheduleMeetingTitle.value = '';
    }

    if (scheduleMeetingDescription) {
        scheduleMeetingDescription.value = '';
    }

    /*
 * Default meeting times are based on
 * the authenticated ServiceNow user's
 * timezone, NOT the laptop timezone.
 */
if (
    currentMeetingTimezone &&
    scheduleMeetingStart &&
    scheduleMeetingEnd
) {

    const now =
        new Date();

    /*
     * Default start = 5 minutes from now.
     */
    const defaultStart =
        new Date(
            now.getTime() +
            (5 * 60 * 1000)
        );

    /*
     * Default end = 35 minutes from now,
     * giving a 30-minute meeting.
     */
    const defaultEnd =
        new Date(
            now.getTime() +
            (35 * 60 * 1000)
        );


    scheduleMeetingStart.value =
        getDateTimeLocalValueInTimezone(
            defaultStart,
            currentMeetingTimezone
        );


    scheduleMeetingEnd.value =
        getDateTimeLocalValueInTimezone(
            defaultEnd,
            currentMeetingTimezone
        );

} else {

    if (scheduleMeetingStart) {
        scheduleMeetingStart.value = '';
    }

    if (scheduleMeetingEnd) {
        scheduleMeetingEnd.value = '';
    }
}

    selectedMeetingPeople = [];

    if (scheduleMeetingPeopleSearch) {
        scheduleMeetingPeopleSearch.value = '';
    }

    if (scheduleMeetingPeopleResults) {
        scheduleMeetingPeopleResults.innerHTML = '';
        scheduleMeetingPeopleResults.style.display = 'none';
    }

    if (scheduleMeetingSelectedPeople) {

        scheduleMeetingSelectedPeople.innerHTML = `
            <div
                id="scheduleMeetingNoPeople"
                class="schedule-meeting-no-people"
            >
                No people selected.
            </div>
        `;
    }

    if (scheduleMeetingMessage) {
        scheduleMeetingMessage.textContent = '';
    }


    scheduleMeetingModal.classList.add(
        'open'
    );

    scheduleMeetingModal.setAttribute(
        'aria-hidden',
        'false'
    );


    /*
     * Put the cursor directly in Title.
     */

    setTimeout(
        () => {

            if (scheduleMeetingTitle) {
                scheduleMeetingTitle.focus();
            }

        },
        0
    );
}


function closeScheduleMeetingModal() {

    if (!scheduleMeetingModal) {
        return;
    }


    scheduleMeetingModal.classList.remove(
        'open'
    );

    scheduleMeetingModal.setAttribute(
        'aria-hidden',
        'true'
    );


    if (scheduleMeetingPeopleResults) {
        scheduleMeetingPeopleResults.style.display =
            'none';
    }
}


function meetingDisplayValueToDateTimeLocal(
    value
) {

    const text =
        String(
            value || ''
        ).trim();


    if (!text) {
        return '';
    }

    return text
        .replace(
            ' ',
            'T'
        )
        .substring(
            0,
            16
        );
}

async function openEditMeetingModal(
    meetingSysId
) {

    if (
        !meetingSysId ||
        !scheduleMeetingModal
    ) {
        return;
    }


    /*
     * EDIT mode.
     */
    meetingFormMode =
        'edit';

    editingMeetingSysId =
        meetingSysId;


    /*
     * Change the existing modal UI.
     */
    if (scheduleMeetingHeading) {

        scheduleMeetingHeading.textContent =
            'Edit Meeting';
    }


    if (scheduleMeetingSubtitle) {

        scheduleMeetingSubtitle.textContent =
            'Update this ServiceCall meeting.';
    }


    if (scheduleMeetingSubmitButton) {

        scheduleMeetingSubmitButton.textContent =
            'Loading...';

        scheduleMeetingSubmitButton.disabled =
            true;
    }


    if (scheduleMeetingMessage) {

        scheduleMeetingMessage.textContent =
            '';
    }


    /*
     * Clear old participant search results.
     */
    if (scheduleMeetingPeopleSearch) {

        scheduleMeetingPeopleSearch.value =
            '';
    }


    if (scheduleMeetingPeopleResults) {

        scheduleMeetingPeopleResults.innerHTML =
            '';

        scheduleMeetingPeopleResults.style.display =
            'none';
    }


    /*
     * Open modal immediately while details load.
     */
    scheduleMeetingModal.classList.add(
        'open'
    );

    scheduleMeetingModal.setAttribute(
        'aria-hidden',
        'false'
    );


    try {

        const result =
            await window
                .serviceCall
                .getMeetingDetails(
                    meetingSysId
                );


        if (
            !result ||
            result.success !== true
        ) {

            throw new Error(
                result &&
                result.message
                    ? result.message
                    : 'Unable to load meeting.'
            );
        }


        console.log(
            'Editing meeting:',
            result
        );


        /*
         * Title + Description
         */
        if (scheduleMeetingTitle) {

            scheduleMeetingTitle.value =
                result.title || '';
        }


        if (scheduleMeetingDescription) {

            scheduleMeetingDescription.value =
                result.description || '';
        }


        /*
         * Meeting Details API currently returns:
         *
         * YYYY-MM-DD HH:mm:ss
         *
         * datetime-local requires:
         *
         * YYYY-MM-DDTHH:mm
         */
        if (scheduleMeetingStart) {

            scheduleMeetingStart.value =
                meetingDisplayValueToDateTimeLocal(
                    result.scheduled_start
                );
        }


        if (scheduleMeetingEnd) {

            scheduleMeetingEnd.value =
                meetingDisplayValueToDateTimeLocal(
                    result.scheduled_end
                );
        }


        /*
         * Display authenticated user's
         * ServiceNow timezone.
         */
        if (scheduleMeetingTimezone) {

            scheduleMeetingTimezone.textContent =
                currentMeetingTimezone ||
                'Unavailable';
        }


        /*
         * Populate existing attendees.
         *
         * Do not add the organizer as a
         * selectable attendee.
         */
        selectedMeetingPeople =
            Array.isArray(
                result.participants
            )
                ? result.participants
                    .filter(
                        participant =>
                            participant.role !==
                            'organizer'
                    )
                    .map(
                        participant => ({
                            sys_id:
                                participant.user_sys_id,

                            name:
                                participant.user_name
                        })
                    )
                : [];


        /*
         * Re-render selected participant chips.
         */
        renderSelectedMeetingPeople();


        if (scheduleMeetingSubmitButton) {

            scheduleMeetingSubmitButton.disabled =
                false;

            scheduleMeetingSubmitButton.textContent =
                'Save Changes';
        }


        if (scheduleMeetingTitle) {

            scheduleMeetingTitle.focus();
        }


    } catch (error) {

        console.error(
            'Unable to open Edit Meeting:',
            error
        );


        if (scheduleMeetingMessage) {

            scheduleMeetingMessage.textContent =
                error.message ||
                'Unable to load meeting.';
        }


        if (scheduleMeetingSubmitButton) {

            scheduleMeetingSubmitButton.disabled =
                true;

            scheduleMeetingSubmitButton.textContent =
                'Save Changes';
        }
    }
}


function openMeetingDetailsModal(
    details
) {

    if (!meetingDetailsModal) {
        return;
    }

    /*
     * Remember the meeting currently
     * displayed in the Details modal.
     */
    currentMeetingDetails =
        details;

    console.log('ServiceCall Meeting Details:', details);


    /*
     * Reset the contextual action button.
     *
     * We will determine its exact action
     * from the meeting state/permissions next.
     */
    if (meetingDetailsActionButton) {

        meetingDetailsActionButton.style.display =
            'none';

        meetingDetailsActionButton.disabled =
            false;

        meetingDetailsActionButton.textContent =
            'Join Meeting';
    }

    /* -------------------------
   PRIMARY MEETING ACTION
------------------------- */

if (meetingDetailsActionButton) {

    if (
        details.can_start === true
    ) {

        meetingDetailsActionButton.textContent =
            'Start Meeting';

        meetingDetailsActionButton.style.display =
            '';

    }
    else if (
        details.can_join === true
    ) {

        meetingDetailsActionButton.textContent =
            'Join Meeting';

        meetingDetailsActionButton.style.display =
            '';

    }
}

    /* -------------------------
       BASIC INFORMATION
    ------------------------- */

    meetingDetailsNumber.textContent =
        formatMeetingDetailsValue(
            details.meeting_number
        );


    meetingDetailsHeading.textContent =
        formatMeetingDetailsValue(
            details.title
        );


    meetingDetailsStatus.textContent =
        formatMeetingDetailsStatus(
            details.state
        );


    meetingDetailsDescription.textContent =
        String(
            details.description || ''
        ).trim() ||
        'No description.';


    meetingDetailsOrganizer.textContent =
        formatMeetingDetailsValue(
            details.organizer_name
        );

    meetingDetailsStart.textContent =
        formatMeetingDetailsValue(
            details.scheduled_start
        );


    meetingDetailsEnd.textContent =
        formatMeetingDetailsValue(
            details.scheduled_end
        );


    /* -------------------------
       STARTED INFORMATION
    ------------------------- */

    const hasStartedBy =
        Boolean(
            String(
                details.started_by_name ||
                ''
            ).trim()
        );


    const hasStartedAt =
        Boolean(
            String(
                details.started_at ||
                ''
            ).trim()
        );


    const hasEndedAt =
        Boolean(
            String(
                details.ended_at ||
                ''
            ).trim()
        );


    meetingDetailsStartedByField.style.display =
        hasStartedBy
            ? ''
            : 'none';


    meetingDetailsStartedAtField.style.display =
        hasStartedAt
            ? ''
            : 'none';


    meetingDetailsEndedAtField.style.display =
        hasEndedAt
            ? ''
            : 'none';


    meetingDetailsStartedBy.textContent =
        formatMeetingDetailsValue(
            details.started_by_name
        );


    meetingDetailsStartedAt.textContent =
        formatMeetingDetailsValue(
            details.started_at
        );


    meetingDetailsEndedAt.textContent =
        formatMeetingDetailsValue(
            details.ended_at
        );


    /* -------------------------
       PARTICIPANTS
    ------------------------- */

    const participants =
        Array.isArray(
            details.participants
        )
            ? details.participants
            : [];


    meetingDetailsParticipantCount.textContent =
        String(
            participants.length
        );


    meetingDetailsParticipants.innerHTML =
        '';


    if (
        participants.length === 0
    ) {

        const empty =
            document.createElement(
                'div'
            );

        empty.className =
            'meeting-details-empty';

        empty.textContent =
            'No participants.';

        meetingDetailsParticipants.appendChild(
            empty
        );

    } else {

        participants.forEach(
            participant => {

                const row =
                    document.createElement(
                        'div'
                    );

                row.className =
                    'meeting-details-participant';


                /* -----------------
                   PERSON
                ----------------- */

                const main =
                    document.createElement(
                        'div'
                    );

                main.className =
                    'meeting-details-participant-main';


                const name =
                    document.createElement(
                        'div'
                    );

                name.className =
                    'meeting-details-participant-name';

                name.textContent =
                    formatMeetingDetailsValue(
                        participant.user_name
                    );


                const role =
                    document.createElement(
                        'div'
                    );

                role.className =
                    'meeting-details-participant-role';

                role.textContent =
                    formatMeetingDetailsStatus(
                        participant.role
                    );


                main.appendChild(
                    name
                );

                main.appendChild(
                    role
                );


                /* -----------------
                   STATUSES
                ----------------- */

                const statuses =
                    document.createElement(
                        'div'
                    );

                statuses.className =
                    'meeting-details-participant-statuses';


                if (
                    participant.invitation_status
                ) {

                    const invitationBadge =
                        document.createElement(
                            'span'
                        );

                    invitationBadge.className =
                        'meeting-details-badge';

                    invitationBadge.textContent =
                        formatMeetingDetailsStatus(
                            participant.invitation_status
                        );

                    statuses.appendChild(
                        invitationBadge
                    );
                }


                if (
                    participant.join_status
                ) {

                    const joinBadge =
                        document.createElement(
                            'span'
                        );

                    joinBadge.className =
                        'meeting-details-badge';

                    joinBadge.textContent =
                        formatMeetingDetailsStatus(
                            participant.join_status
                        );

                    statuses.appendChild(
                        joinBadge
                    );
                }


                row.appendChild(
                    main
                );

                row.appendChild(
                    statuses
                );


                meetingDetailsParticipants.appendChild(
                    row
                );
            }
        );
    }


    /* -------------------------
       OPEN
    ------------------------- */

    meetingDetailsModal.classList.add(
        'open'
    );

    meetingDetailsModal.setAttribute(
        'aria-hidden',
        'false'
    );
}

/* =========================================
   MEETING DETAILS PRIMARY ACTION
========================================= */

if (meetingDetailsActionButton) {

    meetingDetailsActionButton.addEventListener(
        'click',
        async () => {

            if (
                !currentMeetingDetails ||
                !currentMeetingDetails.meeting_sys_id
            ) {
                return;
            }


            const meetingSysId =
                currentMeetingDetails.meeting_sys_id;


            meetingDetailsActionButton.disabled =
                true;


            try {

                /* -------------------------
                   START MEETING
                ------------------------- */

                if (
                    currentMeetingDetails.can_start ===
                    true
                ) {

                    meetingDetailsActionButton.textContent =
                        'Starting...';


                    const result =
                        await window.serviceCall
                            .startMeeting(
                                meetingSysId
                            );


                    if (
                        !result ||
                        result.success !== true
                    ) {

                        throw new Error(
                            result &&
                            result.message
                                ? result.message
                                : 'Unable to start meeting.'
                        );
                    }


                    /*
                     * main.js already opens the
                     * connected meeting call window.
                     */

                    meetingDetailsModal.classList.remove(
                        'open'
                    );

                    meetingDetailsModal.setAttribute(
                        'aria-hidden',
                        'true'
                    );


                    currentMeetingDetails =
                        null;


                    await loadMeetings(true);

                    return;
                }


                /* -------------------------
                   JOIN MEETING
                ------------------------- */

                if (
                    currentMeetingDetails.can_join ===
                    true
                ) {

                    meetingDetailsActionButton.textContent =
                        'Joining...';


                    const result =
                        await window.serviceCall
                            .joinMeeting(
                                meetingSysId
                            );


                    if (
                        !result ||
                        result.success !== true
                    ) {

                        throw new Error(
                            result &&
                            result.message
                                ? result.message
                                : 'Unable to join meeting.'
                        );
                    }


                    meetingDetailsModal.classList.remove(
                        'open'
                    );

                    meetingDetailsModal.setAttribute(
                        'aria-hidden',
                        'true'
                    );


                    currentMeetingDetails =
                        null;


                    await loadMeetings(true);

                    return;
                }

            } catch (error) {

                console.error(
                    'Meeting details action failed:',
                    error
                );


                /*
                 * Re-fetch the meeting because its
                 * state may have changed on the server.
                 */
                try {

                    const refreshed =
                        await window.serviceCall
                            .getMeetingDetails(
                                meetingSysId
                            );


                    if (
                        refreshed &&
                        refreshed.success === true
                    ) {

                        openMeetingDetailsModal(
                            refreshed
                        );

                        return;
                    }

                } catch (refreshError) {

                    console.error(
                        'Unable to refresh meeting details:',
                        refreshError
                    );
                }


                meetingDetailsActionButton.textContent =
                    'Try Again';

            } finally {

                meetingDetailsActionButton.disabled =
                    false;
            }
        }
    );
}

/* =================================================
   COPY MEETING LINK
================================================= */

async function copyMeetingLink(
    meetingSysId
) {

    if (!meetingSysId) {
        return false;
    }


    const meetingLink =
        'servicecall://meeting/' +
        encodeURIComponent(
            meetingSysId
        );


    try {

        await navigator.clipboard.writeText(
            meetingLink
        );


        console.log(
            'Meeting link copied:',
            meetingLink
        );


        return true;

    } catch (error) {

        console.error(
            'Unable to copy meeting link:',
            error
        );


        return false;
    }
}

/* =================================================
   OPEN MEETING FROM DEEP LINK
================================================= */

async function openMeetingFromDeepLink(
    meetingSysId
) {

    const cleanMeetingSysId =
        String(
            meetingSysId || ''
        ).trim();


    /*
     * ServiceNow sys_id must be
     * exactly 32 hexadecimal characters.
     */
    if (
        !/^[0-9a-f]{32}$/i.test(
            cleanMeetingSysId
        )
    ) {

        console.error(
            'Invalid ServiceCall meeting link.'
        );

        return;
    }


    try {

        /*
         * ServiceNow remains the security
         * authority.
         *
         * Knowing a meeting sys_id does NOT
         * automatically grant access.
         */
        const result =
            await window
                .serviceCall
                .getMeetingDetails(
                    cleanMeetingSysId
                );


        if (
            !result ||
            result.success !== true
        ) {

            throw new Error(
                result &&
                result.message
                    ? result.message
                    : 'Unable to retrieve meeting details.'
            );
        }


        /*
         * Reuse the exact same Details UI
         * used by the View Details button.
         */
        openMeetingDetailsModal(
            result
        );


    } catch (error) {

        console.error(
            'Unable to open meeting link:',
            error
        );


        if (message) {

            message.textContent =
                error.message ||
                'Unable to open this meeting.';
        }
    }
}
        /* -------------------------------------------------
           RENDER MEETING
        ------------------------------------------------- */

        function createMeetingCard(
            meeting
        ) {

            const card =
                document.createElement(
                    'div'
                );


            card.className =
                'meeting-card';


            const statusClass =
                getStatusClass(
                    meeting.state
                );


            const organizer =
                meeting.organizer_name ||
                'Unknown organizer';


            card.innerHTML = `

                <div class="meeting-card-left">

                    <div class="meeting-title">
                        ${escapeHtml(
                            meeting.title ||
                            'Untitled meeting'
                        )}
                    </div>

                    <div class="meeting-meta">

                        ${escapeHtml(
                            meeting.meeting_number ||
                            ''
                        )}

                        <br>

                        ${escapeHtml(
                            formatMeetingDate(
                                meeting.scheduled_start
                            )
                        )}

                        ${
                            meeting.scheduled_end
                                ? ' – ' +
                                  escapeHtml(
                                      formatMeetingDate(
                                          meeting.scheduled_end
                                      )
                                  )
                                : ''
                        }

                        <br>

                        Organizer:
                        ${escapeHtml(
                            organizer
                        )}

                    </div>

                </div>


                <div class="meeting-actions">

                    <span
                        class="
                            meeting-status
                            ${statusClass}
                        "
                    >
                        ${escapeHtml(
                            meeting.state ||
                            'Unknown'
                        )}
                    </span>

                </div>
            `;


            /*
             * IMPORTANT:
             *
             * These buttons are based entirely
             * on permissions/actions returned
             * by ServiceNow.
             *
             * We are NOT deciding authorization
             * in the Desktop application.
             */

            const actions =
                card.querySelector(
                    '.meeting-actions'
                );

            /* =========================================
   VIEW MEETING DETAILS
========================================= */

const detailsButton =
    document.createElement(
        'button'
    );


detailsButton.type =
    'button';

detailsButton.className =
    'secondary-button';

detailsButton.textContent =
    'View Details';


detailsButton.addEventListener(
    'click',

    async () => {

        detailsButton.disabled =
            true;

        detailsButton.textContent =
            'Loading...';


        try {

            const result =
                await window
                    .serviceCall
                    .getMeetingDetails(
                        meeting.meeting_sys_id
                    );


            if (
                !result ||
                result.success !== true
            ) {

                throw new Error(
                    result &&
                    result.message
                        ? result.message
                        : 'Unable to retrieve meeting details.'
                );
            }


            /*
             * Temporary runtime test.
             *
             * Once confirmed, we'll replace
             * this with the actual Details UI.
             */
            openMeetingDetailsModal(result);


        } catch (error) {

            console.error(
                'Meeting details failed:',
                error
            );


            if (message) {

                message.textContent =
                    error.message ||
                    'Unable to retrieve meeting details.';
            }


        } finally {

            detailsButton.disabled =
                false;

            detailsButton.textContent =
                'View Details';
        }
    }
);


actions.appendChild(
    detailsButton
);

/* -----------------------------------------
   COPY MEETING LINK
----------------------------------------- */

const copyLinkButton =
    document.createElement(
        'button'
    );


copyLinkButton.className =
    'secondary-button';


copyLinkButton.textContent =
    'Copy Link';


copyLinkButton.addEventListener(
    'click',
    async () => {

        const copied =
            await copyMeetingLink(
                meeting.meeting_sys_id
            );


        if (!copied) {

            copyLinkButton.textContent =
                'Copy failed';


            setTimeout(
                () => {

                    copyLinkButton.textContent =
                        'Copy Link';

                },
                1500
            );


            return;
        }


        copyLinkButton.textContent =
            'Copied!';


        setTimeout(
            () => {

                copyLinkButton.textContent =
                    'Copy Link';

            },
            1500
        );
    }
);


actions.appendChild(
    copyLinkButton
);

/* =========================================
   EDIT MEETING
========================================= */

if (
    meeting.can_edit === true
) {

    const button =
        document.createElement(
            'button'
        );

    button.className =
        'secondary-button';

    button.textContent =
        'Edit';


    button.addEventListener(
    'click',

    async () => {

        await openEditMeetingModal(
            meeting.meeting_sys_id
        );
    }
);


    actions.appendChild(
        button
    );
}


            if (
    meeting.can_start === true
) {

    const button =
        document.createElement(
            'button'
        );

    button.className =
        'primary-button';

    button.textContent =
        'Start';


    button.addEventListener(
        'click',

        async () => {

            /*
             * Prevent double-clicking Start.
             */
            button.disabled = true;

            button.textContent =
                'Starting...';


            try {

                const result =
                    await window
                        .serviceCall
                        .startMeeting(
                            meeting.meeting_sys_id
                        );


                if (
                    !result ||
                    result.success !== true
                ) {

                    throw new Error(
                        result &&
                        result.message
                            ? result.message
                            : 'Unable to start meeting.'
                    );
                }


                console.log(
                    'Meeting started:',
                    result
                );


                /*
                 * Refresh Meetings so the card
                 * changes from Scheduled to
                 * In Progress.
                 */
                await loadMeetings(true);


            } catch (error) {

                console.error(
                    'Start meeting failed:',
                    error
                );


                button.disabled = false;

                button.textContent =
                    'Start';


                if (message) {

                    message.textContent =
                        error.message ||
                        'Unable to start meeting.';
                }
            }
        }
    );


    actions.appendChild(
        button
    );
}


            if (
    meeting.can_join === true
) {

    const button =
        document.createElement(
            'button'
        );

    button.className =
        'primary-button';

    button.textContent =
        'Join';


    button.addEventListener(
        'click',

        async () => {

            button.disabled =
                true;

            button.textContent =
                'Joining...';


            try {

                const result =
                    await window
                        .serviceCall
                        .joinMeeting(
                            meeting.meeting_sys_id
                        );


                if (
                    !result ||
                    result.success !== true
                ) {

                    throw new Error(
                        result &&
                        result.message
                            ? result.message
                            : 'Unable to join meeting.'
                    );
                }


                console.log(
                    'Meeting joined:',
                    result
                );


                /*
                 * Refresh the card because
                 * can_join / can_leave may
                 * now have changed.
                 */
                await loadMeetings(true);


            } catch (error) {

                console.error(
                    'Join meeting failed:',
                    error
                );


                button.disabled =
                    false;

                button.textContent =
                    'Join';


                if (message) {

                    message.textContent =
                        error.message ||
                        'Unable to join meeting.';
                }
            }
        }
    );


    actions.appendChild(
        button
    );
}


            if (
    meeting.can_leave === true
) {

    const button =
        document.createElement(
            'button'
        );

    button.className =
        'secondary-button';

    button.textContent =
        'Leave';


    button.addEventListener(
        'click',

        async () => {

            button.disabled =
                true;

            button.textContent =
                'Leaving...';


            try {

                const result =
                    await window
                        .serviceCall
                        .leaveMeeting(
                            meeting.meeting_sys_id
                        );


                if (
                    !result ||
                    result.success !== true
                ) {

                    throw new Error(
                        result &&
                        result.message
                            ? result.message
                            : 'Unable to leave meeting.'
                    );
                }


                console.log(
                    'Meeting left:',
                    result
                );


                await loadMeetings(true);


            } catch (error) {

                console.error(
                    'Leave meeting failed:',
                    error
                );


                button.disabled =
                    false;

                button.textContent =
                    'Leave';


                if (message) {

                    message.textContent =
                        error.message ||
                        'Unable to leave meeting.';
                }
            }
        }
    );


    actions.appendChild(
        button
    );
}


            if (
    meeting.can_end === true
) {

    const button =
        document.createElement(
            'button'
        );

    button.className =
        'danger-button';

    button.textContent =
        'End';


    button.addEventListener(
        'click',

        async () => {

            /*
             * Prevent accidental double-clicks.
             */
            button.disabled =
                true;

            button.textContent =
                'Ending...';


            try {

                const result =
                    await window
                        .serviceCall
                        .endMeeting(
                            meeting.meeting_sys_id
                        );


                if (
                    !result ||
                    result.success !== true
                ) {

                    throw new Error(
                        result &&
                        result.message
                            ? result.message
                            : 'Unable to end meeting.'
                    );
                }


                console.log(
                    'Meeting ended:',
                    result
                );


                /*
                 * Refresh meeting cards.
                 * The ended meeting should now
                 * display as Ended and its live
                 * action buttons disappear.
                 */
                await loadMeetings(true);


            } catch (error) {

                console.error(
                    'End meeting failed:',
                    error
                );


                button.disabled =
                    false;

                button.textContent =
                    'End';


                if (message) {

                    message.textContent =
                        error.message ||
                        'Unable to end meeting.';
                }
            }
        }
    );


    actions.appendChild(
        button
    );
}

if (
    meeting.can_cancel === true
) {

    const button =
        document.createElement(
            'button'
        );

    button.className =
        'danger-button';

    button.textContent =
        'Cancel';


    button.addEventListener(
        'click',

        async () => {

            button.disabled =
                true;

            button.textContent =
                'Cancelling...';


            try {

                const result =
                    await window
                        .serviceCall
                        .cancelMeeting(
                            meeting.meeting_sys_id
                        );


                if (
                    !result ||
                    result.success !== true
                ) {

                    throw new Error(
                        result &&
                        result.message
                            ? result.message
                            : 'Unable to cancel meeting.'
                    );
                }


                console.log(
                    'Meeting cancelled:',
                    result
                );


                /*
                 * Refresh meeting cards.
                 *
                 * State should now be Cancelled
                 * and Start / Cancel disappear.
                 */
                await loadMeetings(true);


            } catch (error) {

                console.error(
                    'Cancel meeting failed:',
                    error
                );


                button.disabled =
                    false;

                button.textContent =
                    'Cancel';


                if (message) {

                    message.textContent =
                        error.message ||
                        'Unable to cancel meeting.';
                }
            }
        }
    );


    actions.appendChild(
        button
    );
}


            return card;
        }

        if (
    meetingDetailsCloseButton
) {

    meetingDetailsCloseButton.addEventListener(
        'click',
        closeMeetingDetailsModal
    );
}


if (
    meetingDetailsFooterCloseButton
) {

    meetingDetailsFooterCloseButton.addEventListener(
        'click',
        closeMeetingDetailsModal
    );
}


if (
    meetingDetailsModal
) {

    meetingDetailsModal.addEventListener(
        'click',
        event => {

            if (
                event.target.hasAttribute(
                    'data-meeting-details-close'
                )
            ) {

                closeMeetingDetailsModal();
            }
        }
    );
}

document.addEventListener(
    'keydown',
    event => {

        if (
            event.key !== 'Escape'
        ) {
            return;
        }


        if (
            scheduleMeetingModal &&
            scheduleMeetingModal.classList.contains(
                'open'
            )
        ) {

            closeScheduleMeetingModal();

            return;
        }


        if (
            meetingDetailsModal &&
            meetingDetailsModal.classList.contains(
                'open'
            )
        ) {

            closeMeetingDetailsModal();
        }
    }
);
        /* -------------------------------------------------
   RENDER MEETING PAGINATION
------------------------------------------------- */

function renderMeetingPagination(
    result
) {

    if (!meetingPagination) {
        return;
    }


    meetingPagination.innerHTML =
        '';


    const totalPages =
        parseInt(
            result.total_pages,
            10
        ) || 0;


    const currentPage =
        parseInt(
            result.current_page,
            10
        ) || 1;


    /*
     * No pagination is necessary when
     * there is only one page.
     */
    if (totalPages <= 1) {
        return;
    }


    /* -----------------------------
       PREVIOUS
    ----------------------------- */

    const previousButton =
        document.createElement(
            'button'
        );


    previousButton.type =
        'button';

    previousButton.className =
        'meeting-page-button';

    previousButton.textContent =
        '‹ Previous';

    previousButton.disabled =
        currentPage <= 1;


    previousButton.addEventListener(
        'click',
        async () => {

            if (currentPage <= 1) {
                return;
            }


            currentMeetingPage =
                currentPage - 1;


            await loadMeetings();
        }
    );


    meetingPagination.appendChild(
        previousButton
    );


   /* -----------------------------
   PAGE NUMBERS
----------------------------- */

function addPageButton(
    pageNumber
) {

    const pageButton =
        document.createElement(
            'button'
        );


    pageButton.type =
        'button';

    pageButton.className =
        'meeting-page-button';

    pageButton.textContent =
        String(
            pageNumber
        );


    if (
        pageNumber ===
        currentPage
    ) {

        pageButton.classList.add(
            'active'
        );

        pageButton.disabled =
            true;
    }


    pageButton.addEventListener(
        'click',
        async () => {

            if (
                pageNumber ===
                currentPage
            ) {
                return;
            }


            currentMeetingPage =
                pageNumber;


            await loadMeetings();
        }
    );


    meetingPagination.appendChild(
        pageButton
    );
}


function addEllipsis() {

    const ellipsis =
        document.createElement(
            'span'
        );


    ellipsis.className =
        'meeting-page-info';

    ellipsis.textContent =
        '…';


    meetingPagination.appendChild(
        ellipsis
    );
}


/*
 * Small number of pages:
 *
 * 1 2 3 4 5
 */
if (
    totalPages <= 5
) {

    for (
        let pageNumber = 1;
        pageNumber <= totalPages;
        pageNumber++
    ) {

        addPageButton(
            pageNumber
        );
    }

}


/*
 * Near the beginning:
 *
 * 1 2 3 … 15
 */
else if (
    currentPage <= 3
) {

    addPageButton(1);
    addPageButton(2);
    addPageButton(3);

    addEllipsis();

    addPageButton(
        totalPages
    );

}


/*
 * Near the end:
 *
 * 1 … 13 14 15
 */
else if (
    currentPage >=
    totalPages - 2
) {

    addPageButton(1);

    addEllipsis();

    addPageButton(
        totalPages - 2
    );

    addPageButton(
        totalPages - 1
    );

    addPageButton(
        totalPages
    );

}


/*
 * Somewhere in the middle:
 *
 * 1 … 7 8 9 … 15
 */
else {

    addPageButton(1);

    addEllipsis();

    addPageButton(
        currentPage - 1
    );

    addPageButton(
        currentPage
    );

    addPageButton(
        currentPage + 1
    );

    addEllipsis();

    addPageButton(
        totalPages
    );
}

    /* -----------------------------
       NEXT
    ----------------------------- */

    const nextButton =
        document.createElement(
            'button'
        );


    nextButton.type =
        'button';

    nextButton.className =
        'meeting-page-button';

    nextButton.textContent =
        'Next ›';

    nextButton.disabled =
        currentPage >=
        totalPages;


    nextButton.addEventListener(
        'click',
        async () => {

            if (
                currentPage >=
                totalPages
            ) {
                return;
            }


            currentMeetingPage =
                currentPage + 1;


            await loadMeetings();
        }
    );


    meetingPagination.appendChild(
        nextButton
    );


    /* -----------------------------
       PAGE INFORMATION
    ----------------------------- */

    const pageInfo =
        document.createElement(
            'span'
        );


    pageInfo.className =
        'meeting-page-info';


    pageInfo.textContent =
        'Page ' +
        currentPage +
        ' of ' +
        totalPages;


    meetingPagination.appendChild(
        pageInfo
    );
}

/* -------------------------------------------------
   MEETINGS AUTO REFRESH
------------------------------------------------- */

function startMeetingsAutoRefresh() {

    /*
     * Prevent multiple timers from
     * being created.
     */
    if (meetingsAutoRefreshTimer) {
        return;
    }


    meetingsAutoRefreshTimer =
        setInterval(
            async () => {

                /*
                 * Refresh only when the
                 * Meetings page is actually open.
                 */
                const meetingsView =
                    document.getElementById(
                        'meetingsView'
                    );


                if (
                    !meetingsView ||
                    !meetingsView.classList.contains(
                        'active'
                    )
                ) {
                    return;
                }


                /*
                 * Don't disturb the user while
                 * the Schedule Meeting modal
                 * is open.
                 */
                if (
                    scheduleMeetingModal &&
                    scheduleMeetingModal.classList.contains(
                        'open'
                    )
                ) {
                    return;
                }


                try {

                    console.log(
                        'Auto-refreshing meetings...'
                    );

                    await loadMeetings(true);

                } catch (error) {

                    console.error(
                        'Meeting auto-refresh failed:',
                        error
                    );
                }

            },
            15000
        );
}


function stopMeetingsAutoRefresh() {

    if (!meetingsAutoRefreshTimer) {
        return;
    }


    clearInterval(
        meetingsAutoRefreshTimer
    );


    meetingsAutoRefreshTimer = null;
}


        /* -------------------------------------------------
           LOAD MEETINGS
        ------------------------------------------------- */

        async function loadMeetings(
    silent = false
) {

    if (!meetingsContainer) {
        return;
    }


    /*
     * Normal/manual load:
     * show loading state.
     *
     * Background refresh:
     * keep the existing UI visible.
     */
    if (!silent) {

        if (meetingPagination) {

            meetingPagination.innerHTML =
                '';
        }


        meetingsContainer.innerHTML = `
            <div class="loading">
                Loading your meetings...
            </div>
        `;
    }


            try {

                const result =
                    await window
                        .serviceCall
                        .getMyMeetings(currentMeetingPage, currentMeetingSearch, currentMeetingStatus);

                if (
    !result ||
    result.success !== true
) {

    throw new Error(
        result &&
        result.message
            ? result.message
            : 'Unable to retrieve meetings.'
    );
}


/*
 * Save the authenticated user's
 * ServiceNow timezone.
 */
currentMeetingTimezone =
    String(
        result.user_timezone ||
        ''
    ).trim();

    console.log(
    'ServiceNow user timezone:',
    currentMeetingTimezone
);


if (scheduleMeetingTimezone) {

    scheduleMeetingTimezone.textContent =
        currentMeetingTimezone ||
        'Unavailable';
}


const meetings =
    Array.isArray(
        result.meetings
    )
        ? result.meetings
        : [];

                currentMeetingPage =
    parseInt(
        result.current_page,
        10
    ) || 1;


renderMeetingPagination(
    result
);


                meetingsContainer.innerHTML =
                    '';


               if (
    meetings.length === 0
) {

    /*
     * Search and/or status filter is active.
     */
    if (
        currentMeetingSearch ||
        currentMeetingStatus
    ) {

        meetingsContainer.innerHTML = `
            <div class="empty-state">
                No meetings found.
            </div>
        `;

    } else {

        meetingsContainer.innerHTML = `
            <div class="empty-state">
                You don't have any ServiceCall meetings yet.
            </div>
        `;
    }


    return;
}


                meetings.forEach(
                    (meeting) => {

                        meetingsContainer.appendChild(
                            createMeetingCard(
                                meeting
                            )
                        );
                    }
                );


            } catch (error) {

    console.error(
        'Unable to load meetings:',
        error
    );


    /*
     * During a silent background refresh,
     * keep the existing meeting cards visible.
     */
    if (!silent) {

        meetingsContainer.innerHTML = `
            <div class="empty-state">
                Unable to load your meetings.
            </div>
        `;
    }
}
        }


/* -------------------------------------------------
   MEETING SEARCH
------------------------------------------------- */

if (meetingSearchInput) {

    meetingSearchInput.addEventListener(
        'input',
        () => {

            const searchValue =
                meetingSearchInput
                    .value
                    .trim();


            /*
             * Show the clear button whenever
             * something has been entered.
             */
            if (meetingSearchClear) {

                meetingSearchClear.style.display =
                    searchValue
                        ? 'flex'
                        : 'none';
            }


            /*
             * Cancel the previous pending search.
             *
             * This prevents an API request for
             * every individual keystroke.
             */
            if (meetingSearchTimer) {

                clearTimeout(
                    meetingSearchTimer
                );
            }


            meetingSearchTimer =
                setTimeout(
                    async () => {

                        currentMeetingSearch =
                            searchValue;


                        /*
                         * Every new search begins
                         * from page 1.
                         */
                        currentMeetingPage =
                            1;


                        await loadMeetings();

                    },
                    300
                );
        }
    );
}

if (meetingSearchClear) {

    meetingSearchClear.addEventListener(
        'click',
        async () => {

            if (meetingSearchTimer) {

                clearTimeout(
                    meetingSearchTimer
                );

                meetingSearchTimer =
                    null;
            }


            if (meetingSearchInput) {

                meetingSearchInput.value =
                    '';

                meetingSearchInput.focus();
            }


            meetingSearchClear.style.display =
                'none';


            currentMeetingSearch =
                '';

            currentMeetingPage =
                1;


            await loadMeetings();
        }
    );
}

/* -------------------------------------------------
   MEETING STATUS FILTER
------------------------------------------------- */

if (meetingStatusFilter) {

    meetingStatusFilter.addEventListener(
        'change',
        async () => {

            /*
             * Values come directly from the
             * dropdown:
             *
             * ''            = All
             * scheduled     = Scheduled
             * in progress   = In Progress
             * ended         = Ended
             * cancelled     = Cancelled
             */
            currentMeetingStatus =
                String(
                    meetingStatusFilter.value ||
                    ''
                )
                    .toLowerCase()
                    .trim();


            /*
             * A new filter always begins
             * from page 1.
             */
            currentMeetingPage =
                1;


            await loadMeetings();
        }
    );
}
        
/* =======================================================
   MEETING CHANGE REFRESH
======================================================= */

if (
    window.serviceCall &&
    typeof window.serviceCall
        .onMeetingChanged ===
        'function'
) {

    window.serviceCall
        .onMeetingChanged(
            async (data) => {

                console.log(
                    'ServiceCall meeting changed:',
                    data
                );

                await loadMeetings(true);
            }
        );
}


        /* -------------------------------------------------
           SCHEDULE MEETING
        ------------------------------------------------- */

        if (scheduleMeetingButton) {

    scheduleMeetingButton.addEventListener(
        'click',
        () => {

            openScheduleMeetingModal();
        }
    );
}


/* =================================================
   OPEN MEETING FROM DESKTOP NOTIFICATION
================================================= */
 
if (
    window.serviceCall &&
    typeof window.serviceCall
        .onNotificationMeetingOpen ===
        'function'
) {
 
    window.serviceCall
        .onNotificationMeetingOpen(
            async (data) => {
 
                try {
 
                    const meetingSysId =
                        String(
                            data &&
                            data.meetingSysId
                                ? data.meetingSysId
                                : ''
                        ).trim();
 
 
                    console.log(
                        'ServiceCall meeting notification received:',
                        data
                    );
 
 
                    /*
                     * Validate the meeting sys_id
                     * before doing anything.
                     */
                    if (
                        !/^[0-9a-f]{32}$/i.test(
                            meetingSysId
                        )
                    ) {
 
                        console.error(
                            'Invalid meeting sys_id from notification.'
                        );
 
                        return;
                    }
 
 
                    /*
                     * Use the existing Meetings
                     * navigation button.
                     *
                     * This means we reuse the exact
                     * same navigation logic already
                     * used by the sidebar.
                     */
                    const meetingsNavigationButton =
                        document.querySelector(
                            '.nav-button[data-view="meetingsView"]'
                        );
 
 
                    if (
                        meetingsNavigationButton
                    ) {
 
                        meetingsNavigationButton.click();
                    }
 
 
                    /*
                     * Open the exact meeting using
                     * the existing secure flow.
                     *
                     * This calls ServiceNow again
                     * to retrieve/authorize the
                     * meeting before displaying it.
                     */
                    await openMeetingFromDeepLink(
                        meetingSysId
                    );
 
 
                } catch (error) {
 
                    console.error(
                        'Unable to open meeting from notification:',
                        error
                    );
                }
            }
        );
}
   /* =================================================
   SERVICECALL DEEP LINK
================================================= */

if (
    window.serviceCall &&
    window.serviceCall.onDeepLink
) {

    window.serviceCall.onDeepLink(
        async (data) => {

            try {

                const deepLink =
                    String(
                        data &&
                        data.url
                            ? data.url
                            : ''
                    ).trim();


                if (!deepLink) {
                    return;
                }


                console.log(
                    'ServiceCall deep link received in renderer:',
                    deepLink
                );


/*
 * Expected formats:
 *
 * servicecall://meeting/<meeting_sys_id>
 * servicecall:///meeting/<meeting_sys_id>
 */
const meetingLinkMatch =
    String(
        deepLink || ''
    )
        .trim()
        .match(
            /^servicecall:\/\/\/?meeting\/([0-9a-f]{32})$/i
        );


if (!meetingLinkMatch) {

    console.error(
        'Unsupported ServiceCall deep link:',
        deepLink
    );

    return;
}


const meetingSysId =
    meetingLinkMatch[1];


console.log(
    'ServiceCall meeting sys_id from link:',
    meetingSysId
);



                if (
                    !/^[0-9a-f]{32}$/i.test(
                        meetingSysId
                    )
                ) {

                    console.error(
                        'Invalid meeting sys_id in ServiceCall link.'
                    );

                    return;
                }


                /*
                 * Open the meeting using our
                 * existing secure details flow.
                 */
                await openMeetingFromDeepLink(
                    meetingSysId
                );


            } catch (error) {

                console.error(
                    'Unable to process ServiceCall meeting link:',
                    error
                );
            }
        }
    );
}


/*
 * Tell the main process that the renderer
 * has installed its deep-link listener.
 */
if (
    window.serviceCall &&
    window.serviceCall.rendererReady
) {

    window.serviceCall.rendererReady();
}

function renderSelectedMeetingPeople() {

    if (!scheduleMeetingSelectedPeople) {
        return;
    }


    /*
     * Clear the current display.
     */
    scheduleMeetingSelectedPeople.innerHTML =
        '';


    /*
     * No selected people.
     */
    if (
        selectedMeetingPeople.length === 0
    ) {

        scheduleMeetingSelectedPeople.innerHTML = `
            <div
                id="scheduleMeetingNoPeople"
                class="schedule-meeting-no-people"
            >
                No people selected.
            </div>
        `;

        return;
    }


    /*
     * Render every selected person.
     */
    selectedMeetingPeople.forEach(
        person => {

            const selectedPerson =
                document.createElement(
                    'div'
                );


            selectedPerson.className =
                'schedule-meeting-selected-person';


            const name =
                document.createElement(
                    'span'
                );


            name.textContent =
                person.name ||
                'Unknown User';


            const removeButton =
                document.createElement(
                    'button'
                );


            removeButton.type =
                'button';

            removeButton.textContent =
                '×';

            removeButton.title =
                'Remove';


            removeButton.addEventListener(
                'click',
                () => {

                    /*
                     * Remove the person from
                     * our selected array.
                     */
                    selectedMeetingPeople =
                        selectedMeetingPeople.filter(
                            selected =>
                                selected.sys_id !==
                                person.sys_id
                        );


                    /*
                     * Re-render everything.
                     */
                    renderSelectedMeetingPeople();
                }
            );


            selectedPerson.appendChild(
                name
            );


            selectedPerson.appendChild(
                removeButton
            );


            scheduleMeetingSelectedPeople.appendChild(
                selectedPerson
            );
        }
    );
}

/* -------------------------------------------------
   SCHEDULE MEETING - PEOPLE SEARCH
------------------------------------------------- */

if (
    scheduleMeetingPeopleSearch
) {

    scheduleMeetingPeopleSearch.addEventListener(
        'input',
        () => {

            const searchText =
                scheduleMeetingPeopleSearch
                    .value
                    .trim();


            if (
                schedulePeopleSearchTimer
            ) {

                clearTimeout(
                    schedulePeopleSearchTimer
                );
            }


            /*
             * Our existing /users API requires
             * at least 2 characters.
             */
            if (
                searchText.length < 2
            ) {

                scheduleMeetingPeopleResults.innerHTML =
                    '';

                scheduleMeetingPeopleResults.style.display =
                    'none';

                return;
            }


            schedulePeopleSearchTimer =
                setTimeout(
                    async () => {

                        try {

                            const result =
                                await window
                                    .serviceCall
                                    .searchUsers(
                                        searchText
                                    );


                            console.log(
                                'Schedule meeting user search:',
                                result
                            );

                            if (
    !result ||
    result.success !== true
) {
    return;
}

const users =
    Array.isArray(result.users)
        ? result.users.filter(
            user =>
                !selectedMeetingPeople.some(
                    person =>
                        person.sys_id ===
                        user.sys_id
                )
        )
        : [];


scheduleMeetingPeopleResults.innerHTML =
    '';


users.forEach(
    user => {

        const item =
            document.createElement(
                'div'
            );


        item.className =
            'schedule-meeting-person-result';


        item.textContent =
            user.name ||
            'Unknown User';

item.addEventListener(
    'click',
    () => {

        /*
         * Don't add the same person twice.
         */
        const alreadySelected =
            selectedMeetingPeople.some(
                person =>
                    person.sys_id ===
                    user.sys_id
            );


        if (!alreadySelected) {

            selectedMeetingPeople.push(
                {
                    sys_id: user.sys_id,
                    name:
                        user.name ||
                        'Unknown User'
                }
            );
        }


       renderSelectedMeetingPeople();
        /*
         * Clear search and hide results.
         */
        scheduleMeetingPeopleSearch.value =
            '';

        scheduleMeetingPeopleResults.innerHTML =
            '';

        scheduleMeetingPeopleResults.style.display =
            'none';
    }
);


        scheduleMeetingPeopleResults.appendChild(
            item
        );
    }
);


scheduleMeetingPeopleResults.style.display =
    users.length > 0
        ? 'block'
        : 'none';

                        } catch (error) {

                            console.error(
                                'Schedule meeting user search failed:',
                                error
                            );
                        }

                    },
                    300
                );
        }
    );
}

/* -------------------------------------------------
   SCHEDULE MEETING - VALIDATION
------------------------------------------------- */

if (scheduleMeetingSubmitButton) {

    scheduleMeetingSubmitButton.addEventListener(
        'click',
        async () => {

            const title =
                scheduleMeetingTitle.value.trim();

            const description =
                scheduleMeetingDescription.value.trim();

            const start =
                scheduleMeetingStart.value;

            const end =
                scheduleMeetingEnd.value;


            /*
             * Clear previous message.
             */
            scheduleMeetingMessage.textContent =
                '';


            if (!title) {

                scheduleMeetingMessage.textContent =
                    'Please enter a meeting title.';

                scheduleMeetingTitle.focus();

                return;
            }


            if (!start) {

                scheduleMeetingMessage.textContent =
                    'Please select a start date and time.';

                scheduleMeetingStart.focus();

                return;
            }


            if (!end) {

                scheduleMeetingMessage.textContent =
                    'Please select an end date and time.';

                scheduleMeetingEnd.focus();

                return;
            }


            /*
 * datetime-local produces:
 * YYYY-MM-DDTHH:mm
 *
 * Start and end represent wall-clock values
 * in the SAME ServiceNow user timezone,
 * so compare them directly.
 *
 * Do NOT use new Date() here because that
 * would interpret them using the computer's
 * local timezone.
 */
if (end <= start) {

    scheduleMeetingMessage.textContent =
        'End time must be after the start time.';

    scheduleMeetingEnd.focus();

    return;
}

if (!currentMeetingTimezone) {

    scheduleMeetingMessage.textContent =
        'Unable to determine your ServiceNow time zone. Please refresh Meetings and try again.';

    return;
}


            if (
                selectedMeetingPeople.length === 0
            ) {

                scheduleMeetingMessage.textContent =
                    'Please select at least one person.';

                scheduleMeetingPeopleSearch.focus();

                return;
            }


            /*
             * Build participant sys_id array.
             */
            const participants =
                selectedMeetingPeople.map(
                    person => person.sys_id
                );


            /*
             * Build meeting payload.
             */
            const meetingData = {

    title:
        title,

    description:
        description,

    scheduled_start:
        start,

    scheduled_end:
        end,

    timezone:
        currentMeetingTimezone,

    participants:
        participants
};


            console.log(
    meetingFormMode === 'edit'
        ? 'Updating ServiceCall meeting:'
        : 'Creating ServiceCall meeting:',
    meetingData
);


            /*
             * Prevent duplicate clicks while
             * the meeting is being created.
             */
            scheduleMeetingSubmitButton.disabled =
                true;


            scheduleMeetingMessage.textContent =
                'Scheduling meeting...';


            try {

                console.log(
    'Meeting timezone test:',
    {
        start: start,
        end: end,
        timezone:
            currentMeetingTimezone,
        browserTimezone:
            Intl.DateTimeFormat()
                .resolvedOptions()
                .timeZone
    }
);

                let result;


/*
 * CREATE MODE
 */
if (
    meetingFormMode === 'create'
) {

    result =
        await window.serviceCall
            .createMeeting(
                meetingData
            );

}


/*
 * EDIT MODE
 */
else if (
    meetingFormMode === 'edit'
) {

    if (!editingMeetingSysId) {

        throw new Error(
            'Meeting sys_id is missing.'
        );
    }


    result =
    await window.serviceCall
        .updateMeeting(
            editingMeetingSysId,
            meetingData
        );
}


                console.log(
                    'Create meeting result:',
                    result
                );


                if (
                    !result ||
                    result.success !== true
                ) {

                    scheduleMeetingMessage.textContent =
                        result &&
                        result.message
                            ? result.message
                            : 'Unable to schedule meeting.';

                    return;
                }


                /*
                 * Meeting created successfully.
                 */
                scheduleMeetingMessage.textContent =
    meetingFormMode === 'edit'
        ? 'Meeting updated successfully.'
        : 'Meeting scheduled successfully.';


                /*
                 * Refresh Meetings list.
                 */
                await loadMeetings();


                /*
                 * Close modal shortly after
                 * successful creation.
                 */
                setTimeout(
                    () => {

                        closeScheduleMeetingModal();

                    },
                    700
                );


            } catch (error) {

                console.error(
                    'Schedule meeting failed:',
                    error
                );


                scheduleMeetingMessage.textContent =
                    error &&
                    error.message
                        ? error.message
                        : 'Unable to schedule meeting.';


            } finally {

                scheduleMeetingSubmitButton.disabled =
                    false;
            }
        }
    );
}

/* =======================================================
   SERVICECALL PEOPLE
======================================================= */

if (
    peopleSearchInput &&
    peopleSearchResults
) {

    peopleSearchInput.addEventListener(
        'input',
        () => {

            const searchText =
                peopleSearchInput
                    .value
                    .trim();


            /*
             * Cancel previous pending search.
             */
            if (peopleSearchTimer) {

                clearTimeout(
                    peopleSearchTimer
                );

                peopleSearchTimer =
                    null;
            }


            /*
             * Existing /users API requires
             * at least two characters.
             */
            if (
                searchText.length < 2
            ) {

                peopleSearchResults.innerHTML =
                    '';

                if (peopleSearchMessage) {

                    peopleSearchMessage.textContent =
                        searchText.length === 1
                            ? 'Type at least 2 characters.'
                            : '';
                }

                return;
            }


            if (peopleSearchMessage) {

                peopleSearchMessage.textContent =
                    'Searching...';
            }


            peopleSearchTimer =
                setTimeout(
                    async () => {

                        try {

                            const result =
                                await window
                                    .serviceCall
                                    .searchUsers(
                                        searchText
                                    );


                            /*
                             * User may have typed something
                             * different while this request
                             * was running.
                             */
                            if (
                                peopleSearchInput
                                    .value
                                    .trim() !==
                                searchText
                            ) {

                                return;
                            }


                            if (
                                !result ||
                                result.success !== true
                            ) {

                                throw new Error(
                                    result &&
                                    result.message
                                        ? result.message
                                        : 'Unable to search users.'
                                );
                            }


                            const users =
                                Array.isArray(
                                    result.users
                                )
                                    ? result.users
                                    : [];


                            peopleSearchResults.innerHTML =
                                '';


                            if (
                                users.length === 0
                            ) {

                                if (
                                    peopleSearchMessage
                                ) {

                                    peopleSearchMessage.textContent =
                                        'No users found.';
                                }

                                return;
                            }


                            if (
                                peopleSearchMessage
                            ) {

                                peopleSearchMessage.textContent =
                                    '';
                            }


                            users.forEach(
                                user => {

                                    const row =
                                        document.createElement(
                                            'div'
                                        );


                                    row.className =
                                        'people-result';


                                    /* -------------------------
                                       USER INFORMATION
                                    ------------------------- */

                                    const userInfo =
                                        document.createElement(
                                            'div'
                                        );


                                    const userName =
                                        document.createElement(
                                            'div'
                                        );


                                    userName.textContent =
                                        user.name ||
                                        'Unknown User';


                                    userName.style.fontWeight =
                                        '600';


                                    const userDetails =
                                        document.createElement(
                                            'div'
                                        );


                                    userDetails.style.fontSize =
                                        '13px';


                                    userDetails.style.marginTop =
                                        '4px';


                                    const identityParts =
                                        [];


                                    if (
                                        user.user_name
                                    ) {

                                        identityParts.push(
                                            user.user_name
                                        );
                                    }


                                    if (
                                        user.email
                                    ) {

                                        identityParts.push(
                                            user.email
                                        );
                                    }


                                    userDetails.textContent =
                                        identityParts.join(
                                            ' • '
                                        );


                                    /* -------------------------
                                       CURRENT STATUS
                                    ------------------------- */

                                    const userStatus =
                                        document.createElement(
                                            'div'
                                        );


                                    userStatus.style.fontSize =
                                        '13px';


                                    userStatus.style.marginTop =
                                        '5px';


                                    userStatus.textContent =
                                        user.display_status ||
                                        'Offline';


                                    userInfo.appendChild(
                                        userName
                                    );


                                    if (
                                        identityParts.length >
                                        0
                                    ) {

                                        userInfo.appendChild(
                                            userDetails
                                        );
                                    }


                                    userInfo.appendChild(
                                        userStatus
                                    );


                                    /* -------------------------
                                       CALL BUTTON
                                    ------------------------- */

                                    const callButton =
                                        document.createElement(
                                            'button'
                                        );


                                    callButton.type =
                                        'button';


                                    callButton.className =
                                        'primary-button';


                                    callButton.textContent =
                                        'Call';


                                    callButton.addEventListener(
                                        'click',

                                        async () => {

                                            callButton.disabled =
                                                true;


                                            callButton.textContent =
                                                'Calling...';


                                            if (
                                                peopleSearchMessage
                                            ) {

                                                peopleSearchMessage.textContent =
                                                    'Calling ' +
                                                    (
                                                        user.name ||
                                                        'user'
                                                    ) +
                                                    '...';
                                            }


                                            try {

                                                const callResult =
                                                    await window
                                                        .serviceCall
                                                        .startCall(
                                                            user.sys_id
                                                        );


                                                console.log(
                                                    'Start call result:',
                                                    callResult
                                                );


                                                if (
                                                    !callResult ||
                                                    callResult.success !==
                                                        true
                                                ) {

                                                    throw new Error(
                                                        callResult &&
                                                        callResult.message
                                                            ? callResult.message
                                                            : 'Unable to start call.'
                                                    );
                                                }


                                                if (
                                                    peopleSearchMessage
                                                ) {

                                                    peopleSearchMessage.textContent =
                                                        'Calling ' +
                                                        (
                                                            callResult
                                                                .target_user_name ||
                                                            user.name ||
                                                            'user'
                                                        ) +
                                                        '...';
                                                }


                                                /*
                                                 * Do NOT manually open the
                                                 * call window here.
                                                 *
                                                 * main.js already monitors
                                                 * /outgoing-call and will
                                                 * open the existing call
                                                 * window for this call.
                                                 */


                                            } catch (error) {

                                                console.error(
                                                    'Start ServiceCall failed:',
                                                    error
                                                );


                                                if (
                                                    peopleSearchMessage
                                                ) {

                                                    peopleSearchMessage.textContent =
                                                        error.message ||
                                                        'Unable to start call.';
                                                }


                                                callButton.disabled =
                                                    false;


                                                callButton.textContent =
                                                    'Call';
                                            }
                                        }
                                    );


                                    /* -------------------------
                                       RESULT ROW
                                    ------------------------- */

                                    row.appendChild(
                                        userInfo
                                    );


                                    row.appendChild(
                                        callButton
                                    );


                                    /*
                                     * Temporary functional layout.
                                     * Final UI comes later.
                                     */
                                    row.style.display =
                                        'flex';

                                    row.style.alignItems =
                                        'center';

                                    row.style.justifyContent =
                                        'space-between';

                                    row.style.gap =
                                        '16px';

                                    row.style.padding =
                                        '12px 0';

                                    row.style.borderBottom =
                                        '1px solid #e5e5e5';


                                    peopleSearchResults.appendChild(
                                        row
                                    );
                                }
                            );


                        } catch (error) {

                            console.error(
                                'People search failed:',
                                error
                            );


                            peopleSearchResults.innerHTML =
                                '';


                            if (
                                peopleSearchMessage
                            ) {

                                peopleSearchMessage.textContent =
                                    error.message ||
                                    'Unable to search users.';
                            }
                        }

                    },
                    300
                );
        }
    );
}

/* =======================================================
   SERVICECALL NOTIFICATIONS
======================================================= */


/* -------------------------------------------------
   NOTIFICATION TYPE HELPERS
------------------------------------------------- */

function getNotificationCategory(
    notification
) {

    const type =
        String(
            notification &&
            notification.type
                ? notification.type
                : ''
        )
            .toLowerCase()
            .trim();


    if (
        type.includes(
            'meeting'
        )
    ) {

        return 'meeting';
    }


    if (
        type.includes(
            'chat'
        ) ||
        type.includes(
            'message'
        )
    ) {

        return 'chat';
    }


    if (
        type.includes(
            'call'
        ) ||
        type.includes(
            'recording'
        )
    ) {

        return 'call';
    }


    return 'system';
}


/* -------------------------------------------------
   NOTIFICATION ICON
------------------------------------------------- */

function getNotificationIcon(
    notification
) {

    const category =
        getNotificationCategory(
            notification
        );


    switch (
        category
    ) {

        case 'meeting':
            return '📅';

        case 'chat':
            return '💬';

        case 'call':
            return '☎';

        default:
            return '🔔';
    }
}


/* -------------------------------------------------
   NOTIFICATION TIME
------------------------------------------------- */

function formatNotificationTime(
    notification
) {

    if (!notification) {
        return '';
    }


    return (
        notification.created_at_display ||
        notification.created_at ||
        ''
    );
}


/* -------------------------------------------------
   UPDATE UNREAD BADGE
------------------------------------------------- */

function updateNotificationUnreadBadge(
    unreadCount
) {

    const count =
        Math.max(
            0,
            parseInt(
                unreadCount,
                10
            ) || 0
        );


    /*
     * -----------------------------------------
     * SIDEBAR BADGE
     * -----------------------------------------
     */

    if (
        notificationUnreadBadge
    ) {

        notificationUnreadBadge.textContent =
            count > 99
                ? '99+'
                : String(
                    count
                );


        notificationUnreadBadge.style.display =
            count > 0
                ? 'flex'
                : 'none';
    }


    /*
     * -----------------------------------------
     * UNREAD FILTER COUNT
     * -----------------------------------------
     */

    if (
        notificationUnreadFilterCount
    ) {

        notificationUnreadFilterCount.textContent =
            count > 99
                ? '99+'
                : String(
                    count
                );


        notificationUnreadFilterCount.style.display =
            count > 0
                ? 'inline-flex'
                : 'none';
    }


    /*
     * -----------------------------------------
     * MARK ALL AS READ
     * -----------------------------------------
     *
     * Keep hidden until the Mark All backend
     * functionality is implemented.
     */

    if (
        markAllNotificationsReadButton
    ) {

        markAllNotificationsReadButton.style.display =
            'none';
    }
}

/* -------------------------------------------------
   BRIEF NEW-NOTIFICATION PULSE
------------------------------------------------- */

function pulseNotificationsNavigation() {

    if (
        !notificationsNavButton
    ) {
        return;
    }


    notificationsNavButton.classList.remove(
        'notification-arrived'
    );


    /*
     * Force a reflow so the animation can
     * restart even if another notification
     * arrives shortly afterwards.
     */
    void notificationsNavButton.offsetWidth;


    notificationsNavButton.classList.add(
        'notification-arrived'
    );


    setTimeout(
        () => {

            notificationsNavButton.classList.remove(
                'notification-arrived'
            );

        },
        2200
    );
}

/* -------------------------------------------------
   HIGHLIGHT NOTIFICATION SEARCH MATCH
------------------------------------------------- */

function highlightNotificationSearch(
    text,
    search
) {

    const value =
        String(
            text || ''
        );


    const searchValue =
        String(
            search || ''
        ).trim();


    /*
     * No active search.
     */
    if (!searchValue) {

        return escapeHtml(
            value
        );
    }


    /*
     * Escape the original text first
     * so notification content cannot
     * inject HTML.
     */
    const safeText =
        escapeHtml(
            value
        );


    /*
     * Escape special RegExp characters
     * entered by the user.
     */
    const safeSearch =
        searchValue.replace(
            /[.*+?^${}()|[\]\\]/g,
            '\\$&'
        );


    const expression =
        new RegExp(
            '(' + safeSearch + ')',
            'gi'
        );


    return safeText.replace(
        expression,
        '<mark class="notification-search-highlight">$1</mark>'
    );
}

/* -------------------------------------------------
   OPEN NOTIFICATION DETAIL
------------------------------------------------- */

function openNotificationDetail(
    notification
) {

    if (
        !notification ||
        !notificationListPanel ||
        !notificationDetailPanel
    ) {
        return;
    }


    /*
     * Populate detail information.
     */
    if (notificationDetailIcon) {

        notificationDetailIcon.textContent =
            getNotificationIcon(
                notification
            );
    }


    if (notificationDetailType) {

        notificationDetailType.textContent =
            notification.type_display ||
            notification.type ||
            'Notification';
    }


    if (notificationDetailTitle) {

        /*
         * Detail view deliberately does not
         * use search highlighting.
         */
        notificationDetailTitle.textContent =
            notification.title ||
            notification.type_display ||
            'ServiceCall notification';
    }


    if (notificationDetailTime) {

        notificationDetailTime.textContent =
            formatNotificationTime(
                notification
            );
    }


    if (notificationDetailMessage) {

        notificationDetailMessage.textContent =
            notification.message ||
            'No additional information.';
    }


    /*
     * Clear actions left by the previously
     * opened notification.
     */
    if (notificationDetailActions) {

        notificationDetailActions.innerHTML =
            '';


        /*
         * Meeting notification.
         */
        if (
            notification.action_type ===
                'open_meeting' &&
            notification.meeting_sys_id
        ) {

            const openMeetingButton =
                document.createElement(
                    'button'
                );


            openMeetingButton.type =
                'button';

            openMeetingButton.className =
                'primary-button';

            openMeetingButton.textContent =
                'Open Meeting';


            openMeetingButton.addEventListener(
                'click',

                async (event) => {

                    event.stopPropagation();

                    openMeetingButton.disabled =
                        true;

                    openMeetingButton.textContent =
                        'Opening...';


                    try {

                        await openMeetingFromDeepLink(
                            notification.meeting_sys_id
                        );

                    } catch (error) {

                        console.error(
                            'Unable to open notification meeting:',
                            error
                        );

                    } finally {

                        openMeetingButton.disabled =
                            false;

                        openMeetingButton.textContent =
                            'Open Meeting';
                    }
                }
            );


            notificationDetailActions.appendChild(
                openMeetingButton
            );
        }
    }


    /*
     * Move from list → detail.
     */
    notificationListPanel.classList.add(
        'detail-open'
    );


    notificationDetailPanel.classList.add(
        'active'
    );


    notificationDetailPanel.setAttribute(
        'aria-hidden',
        'false'
    );

    /*
 * Mark this specific notification as read
 * after it has already opened.
 *
 * Do not delay the detail UI while waiting
 * for ServiceNow.
 */
/*
 * Mark only THIS notification as read.
 *
 * The detail view is already open, so
 * ServiceNow does not delay the UI.
 */
if (
    notification.read !== true &&
    notification.sys_id
) {

    window.serviceCall
        .markNotificationRead(
            notification.sys_id
        )
        .then(
            result => {

                console.log(
                    'Mark notification read result:',
                    result
                );


                if (
                    !result ||
                    result.success !== true
                ) {

                    console.error(
                        'ServiceNow did not mark notification as read:',
                        result
                    );

                    return;
                }


                /*
                 * -----------------------------------------
                 * UPDATE THIS NOTIFICATION LOCALLY
                 * -----------------------------------------
                 */

                notification.read =
                    true;

                notification.read_at =
                    result.read_at || '';


                /*
                 * cachedNotifications normally contains
                 * this same object, but update by sys_id
                 * as well so we are explicit.
                 */
                const cachedNotification =
                    cachedNotifications.find(
                        item =>
                            String(
                                item.sys_id || ''
                            ) ===
                            String(
                                notification.sys_id
                            )
                    );


                if (cachedNotification) {

                    cachedNotification.read =
                        true;

                    cachedNotification.read_at =
                        result.read_at || '';
                }


                /*
                 * -----------------------------------------
                 * UPDATE AUTHORITATIVE UNREAD COUNT
                 * -----------------------------------------
                 */

                const unreadCount =
                    Number(
                        result.unread_count || 0
                    );


                if (
                    cachedNotificationResult
                ) {

                    cachedNotificationResult.unread_count =
                        unreadCount;
                }


                /*
                 * Sidebar Notifications badge.
                 */
                updateNotificationUnreadBadge(
                    unreadCount
                );


                /*
                 * Do NOT redraw the list while the
                 * detail screen is open.
                 *
                 * Back will render the correct state.
                 */
            }
        )
        .catch(
            error => {

                console.error(
                    'Unable to mark notification as read:',
                    error
                );
            }
        );
}
}

/* -------------------------------------------------
   CREATE NOTIFICATION CARD
------------------------------------------------- */

function createNotificationCard(
    notification,
    animateArrival = false
) {

    const card =
        document.createElement(
            'div'
        );


    card.className =
        'notification-card';


    if (
        notification.read === true
    ) {

        card.classList.add(
            'read'
        );

    } else {

        card.classList.add(
            'unread'
        );
    }


    if (
        animateArrival
    ) {

        card.classList.add(
            'notification-card-arriving'
        );
    }


    /*
     * Unread indicator.
     */
    if (
        notification.read !== true
    ) {

        const unreadDot =
            document.createElement(
                'div'
            );


        unreadDot.className =
            'notification-unread-dot';


        card.appendChild(
            unreadDot
        );
    }


    /*
     * Icon.
     */
    const icon =
        document.createElement(
            'div'
        );


    icon.className =
        'notification-icon';


    icon.textContent =
        getNotificationIcon(
            notification
        );


    card.appendChild(
        icon
    );


    /*
     * Main body.
     */
    const body =
        document.createElement(
            'div'
        );


    body.className =
        'notification-body';


    const titleRow =
        document.createElement(
            'div'
        );


    titleRow.className =
        'notification-title-row';


    const title =
        document.createElement(
            'div'
        );


    title.className =
        'notification-title';


    title.innerHTML =
    highlightNotificationSearch(
        notification.title ||
        notification.type_display ||
        'ServiceCall notification',

        currentNotificationSearch
    );


    const time =
        document.createElement(
            'div'
        );


    time.className =
        'notification-time';


    time.textContent =
        formatNotificationTime(
            notification
        );


    titleRow.appendChild(
        title
    );


    titleRow.appendChild(
        time
    );


    body.appendChild(
        titleRow
    );


    /*
     * Message.
     */
    if (
        notification.message
    ) {

        const notificationMessage =
            document.createElement(
                'div'
            );


        notificationMessage.className =
            'notification-message';


        notificationMessage.innerHTML =
    highlightNotificationSearch(
        notification.message,
        currentNotificationSearch
    );


        body.appendChild(
            notificationMessage
        );
    }


    /*
     * Actions.
     */
    const actions =
        document.createElement(
            'div'
        );


    actions.className =
        'notification-actions';


    /*
     * OPEN MEETING
     *
     * Reuses the existing secure meeting
     * details flow.
     */
    if (
        notification.action_type ===
            'open_meeting' &&
        notification.meeting_sys_id
    ) {

        const openMeetingButton =
            document.createElement(
                'button'
            );


        openMeetingButton.type =
            'button';


        openMeetingButton.className =
            'notification-action-button primary';


        openMeetingButton.textContent =
            'Open Meeting';


        openMeetingButton.addEventListener(
            'click',

            async () => {

                openMeetingButton.disabled =
                    true;


                openMeetingButton.textContent =
                    'Opening...';


                try {

                    await openMeetingFromDeepLink(
                        notification
                            .meeting_sys_id
                    );


                } catch (error) {

                    console.error(
                        'Unable to open notification meeting:',
                        error
                    );

                } finally {

                    openMeetingButton.disabled =
                        false;


                    openMeetingButton.textContent =
                        'Open Meeting';
                }
            }
        );


        actions.appendChild(
            openMeetingButton
        );
    }


    /*
     * Only append the action row if
     * something was actually added.
     */
    if (
        actions.children.length > 0
    ) {

        body.appendChild(
            actions
        );
    }


    card.appendChild(
    body
);


/*
 * Open the notification in the
 * same-page detail view.
 */
card.addEventListener(
    'click',

    () => {

        openNotificationDetail(
            notification
        );
    }
);


card.setAttribute(
    'role',
    'button'
);

card.setAttribute(
    'tabindex',
    '0'
);


return card;
}

/* -------------------------------------------------
   CLOSE NOTIFICATION DETAIL
------------------------------------------------- */

function closeNotificationDetail() {

    if (
        !notificationListPanel ||
        !notificationDetailPanel
    ) {
        return;
    }


    /*
     * Animate the detail screen out.
     */
    notificationDetailPanel.classList.remove(
        'active'
    );

    notificationDetailPanel.classList.add(
        'closing'
    );


    /*
     * Wait for the exit animation before
     * restoring the notification list.
     */
    setTimeout(
        () => {

            notificationDetailPanel.classList.remove(
                'closing'
            );

            notificationDetailPanel.setAttribute(
                'aria-hidden',
                'true'
            );


            /*
             * Restore list.
             */
            notificationListPanel.classList.remove(
                'detail-open'
            );

            notificationListPanel.classList.add(
                'returning'
            );


            /*
             * Clean temporary animation class.
             */
            setTimeout(
    () => {

        notificationListPanel.classList.remove(
            'returning'
        );


        /*
         * Re-render from our local cache.
         *
         * No ServiceNow request is required.
         */
        renderNotifications(
            cachedNotifications
        );


        if (
            cachedNotificationResult
        ) {

            renderNotificationPagination(
                cachedNotificationResult
            );
        }

    },
    220
);

        },
        180
    );
}


if (
    notificationDetailBackButton
) {

    notificationDetailBackButton.addEventListener(
        'click',
        closeNotificationDetail
    );
}


/* -------------------------------------------------
   RENDER NOTIFICATIONS
------------------------------------------------- */

function renderNotifications(
    notifications,
    newlyArrivedIds = new Set()
) {

    if (
        !notificationsContainer
    ) {
        return;
    }


    notificationsContainer.innerHTML =
        '';


    /*
     * -----------------------------------------
     * FILTER BY READ STATE
     * -----------------------------------------
     *
     * all
     *     Every notification.
     *
     * unread
     *     Only notifications that have not
     *     been opened/read.
     *
     * read
     *     Only notifications already read.
     */

    const filteredNotifications =
        notifications.filter(
            notification => {

                /*
                 * ALL
                 */
                if (
                    currentNotificationFilter ===
                    'all'
                ) {

                    return true;
                }


                /*
                 * UNREAD
                 */
                if (
                    currentNotificationFilter ===
                    'unread'
                ) {

                    return (
                        notification.read !==
                        true
                    );
                }


                /*
                 * READ
                 */
                if (
                    currentNotificationFilter ===
                    'read'
                ) {

                    return (
                        notification.read ===
                        true
                    );
                }


                return true;
            }
        );


    /*
     * -----------------------------------------
     * EMPTY STATE
     * -----------------------------------------
     */

    if (
        filteredNotifications.length ===
        0
    ) {

        let emptyMessage =
            'You don\'t have any ServiceCall notifications yet.';


        if (
            currentNotificationFilter ===
            'unread'
        ) {

            emptyMessage =
                'You have no unread notifications.';
        }


        if (
            currentNotificationFilter ===
            'read'
        ) {

            emptyMessage =
                'You have no read notifications.';
        }


        notificationsContainer.innerHTML = `
            <div class="notification-empty">
                ${emptyMessage}
            </div>
        `;


        return;
    }


    /*
     * -----------------------------------------
     * RENDER
     * -----------------------------------------
     */

    filteredNotifications.forEach(
        notification => {

            notificationsContainer.appendChild(
                createNotificationCard(
                    notification,

                    newlyArrivedIds.has(
                        notification.sys_id
                    )
                )
            );
        }
    );
}

/* -------------------------------------------------
   NOTIFICATION PAGINATION
------------------------------------------------- */

function renderNotificationPagination(
    result
) {

    if (
        !notificationPagination
    ) {
        return;
    }


    notificationPagination.innerHTML =
        '';


    const page =
        parseInt(
            result.page,
            10
        ) || 1;


    const hasMore =
        result.has_more === true;


    /*
     * With the current notification API,
     * we know the current page and whether
     * another page exists.
     *
     * Therefore use simple Previous/Next
     * pagination instead of pretending we
     * know a total page count.
     */
    if (
        page <= 1 &&
        !hasMore
    ) {

        return;
    }


    const previousButton =
        document.createElement(
            'button'
        );


    previousButton.type =
        'button';


    previousButton.className =
        'meeting-page-button';


    previousButton.textContent =
        '‹ Previous';


    previousButton.disabled =
        page <= 1;


    previousButton.addEventListener(
        'click',

        async () => {

            if (
                page <= 1
            ) {
                return;
            }


            currentNotificationPage =
                page - 1;


            await loadNotifications(
                false
            );
        }
    );


    notificationPagination.appendChild(
        previousButton
    );


    const pageInfo =
        document.createElement(
            'span'
        );


    pageInfo.className =
        'meeting-page-info';


    pageInfo.textContent =
        'Page ' +
        page;


    notificationPagination.appendChild(
        pageInfo
    );


    const nextButton =
        document.createElement(
            'button'
        );


    nextButton.type =
        'button';


    nextButton.className =
        'meeting-page-button';


    nextButton.textContent =
        'Next ›';


    nextButton.disabled =
        !hasMore;


    nextButton.addEventListener(
        'click',

        async () => {

            if (
                !hasMore
            ) {
                return;
            }


            currentNotificationPage =
                page + 1;


            await loadNotifications(
                false
            );
        }
    );


    notificationPagination.appendChild(
        nextButton
    );
}


/* -------------------------------------------------
   LOAD NOTIFICATIONS
------------------------------------------------- */

async function loadNotifications(
    silent = false
) {

    const requestSearch =
    currentNotificationSearch;

const requestSearchVersion =
    notificationSearchVersion;

    if (
        !notificationsContainer
    ) {
        return;
    }


    /*
     * Manual/open-page refresh:
     * show a loading state.
     *
     * Background refresh:
     * leave the existing UI untouched
     * until fresh data arrives.
     */
    if (
        !silent
    ) {

        notificationsContainer.innerHTML = `
            <div class="loading">
                Loading notifications...
            </div>
        `;


        if (
            notificationPagination
        ) {

            notificationPagination.innerHTML =
                '';
        }
    }


    try {

        const result =
    await window.serviceCall
        .getNotifications(
            currentNotificationPage,
            20,
            requestSearch
        );

        /*
 * The user typed something else while
 * this request was running.
 *
 * Ignore this old response completely.
 */
if (
    requestSearchVersion !==
        notificationSearchVersion ||
    requestSearch !==
        currentNotificationSearch
) {

    return;
}

        if (
            !result ||
            result.success !== true
        ) {

            throw new Error(
                result &&
                result.message
                    ? result.message
                    : 'Unable to retrieve notifications.'
            );
        }


        const notifications =
            Array.isArray(
                result.notifications
            )
                ? result.notifications
                : [];

                /*
 * Keep the latest ServiceNow result in memory.
 *
 * All / Unread / Read can now switch instantly.
 */
cachedNotifications =
    notifications;

cachedNotificationResult =
    result;

        /*
         * Update unread count globally,
         * regardless of which page the
         * user currently has open.
         */
        updateNotificationUnreadBadge(
            result.unread_count
        );


        const newlyArrivedIds =
            new Set();


        /*
         * IMPORTANT:
         *
         * The first successful load establishes
         * our baseline.
         *
         * Existing notifications must NOT all
         * pulse as though they just arrived.
         */
        if (
            notificationsInitialized
        ) {

            notifications.forEach(
                notification => {

                    const sysId =
                        String(
                            notification.sys_id ||
                            ''
                        ).trim();


                    if (
                        sysId &&
                        !knownNotificationIds.has(
                            sysId
                        )
                    ) {

                        newlyArrivedIds.add(
                            sysId
                        );
                    }
                }
            );
        }


        /*
         * Remember everything returned by
         * this API response.
         */
        notifications.forEach(
            notification => {

                const sysId =
                    String(
                        notification.sys_id ||
                        ''
                    ).trim();


                if (
                    sysId
                ) {

                    knownNotificationIds.add(
                        sysId
                    );
                }
            }
        );


        notificationsInitialized =
            true;


        if (
    newlyArrivedIds.size > 0
) {

    pulseNotificationsNavigation();


    /*
     * Show a desktop popup only for
     * genuinely new notifications.
     */
    notifications.forEach(
        notification => {

            const sysId =
                String(
                    notification.sys_id ||
                    ''
                ).trim();


            if (
                sysId &&
                newlyArrivedIds.has(
                    sysId
                )
            ) {

                window.serviceCall
    .showNotificationPopup(
        {
            notificationSysId:
                sysId,
 
            type:
                notification.type_display ||
                notification.type ||
                'Notification',
 
            title:
                notification.title ||
                'ServiceCall',
 
            message:
                notification.message ||
                '',
 
            meetingSysId:
                notification.meeting_sys_id ||
                ''
        }
    )
                    .catch(
                        error => {

                            console.error(
                                'Unable to show ServiceCall notification popup:',
                                error
                            );
                        }
                    );
            }
        }
    );
}


        /*
         * Only render the notification list
         * when the Notifications page is open
         * OR when this was an explicit load.
         *
         * Background polling while on Home,
         * Meetings, Chat, etc. therefore only
         * updates the badge/pulse.
         */
        const notificationsView =
            document.getElementById(
                'notificationsView'
            );


        if (
            !silent ||
            (
                notificationsView &&
                notificationsView.classList.contains(
                    'active'
                )
            )
        ) {

            renderNotifications(
                notifications,
                newlyArrivedIds
            );


            renderNotificationPagination(
                result
            );
        }


    } catch (error) {

        console.error(
            'Unable to load notifications:',
            error
        );


        /*
         * Never destroy existing cards because
         * a silent background refresh failed.
         */
        if (
            !silent
        ) {

            notificationsContainer.innerHTML = `
                <div class="notification-empty">
                    Unable to load your notifications.
                </div>
            `;
        }
    }
}


/* -------------------------------------------------
   MANUAL REFRESH
------------------------------------------------- */

if (
    refreshNotificationsButton
) {

    refreshNotificationsButton.addEventListener(
        'click',

        async () => {

            refreshNotificationsButton.disabled =
                true;


            const originalText =
                refreshNotificationsButton
                    .textContent;


            refreshNotificationsButton.textContent =
                'Refreshing...';


            try {

                await loadNotifications(
                    true
                );

            } finally {

                refreshNotificationsButton.disabled =
                    false;


                refreshNotificationsButton.textContent =
                    originalText;
            }
        }
    );
}

/* -------------------------------------------------
   NOTIFICATION SEARCH
------------------------------------------------- */

if (notificationSearchInput) {

    notificationSearchInput.addEventListener(
        'input',
        () => {

            const searchValue =
                notificationSearchInput
                    .value
                    .trim();

            /*
 * Update the active search immediately.
 *
 * This lets the already-loaded cards respond
 * instantly while the full ServiceNow search
 * is waiting for the debounce timer.
 */
currentNotificationSearch =
    searchValue;

notificationSearchVersion++;


            /*
             * Show / hide clear button.
             */
            if (notificationSearchClear) {

                notificationSearchClear.style.display =
                    searchValue
                        ? 'flex'
                        : 'none';
            }


            /*
             * Cancel previous pending search.
             */
            if (notificationSearchTimer) {

                clearTimeout(
                    notificationSearchTimer
                );
            }


            /*
             * Wait briefly before searching
             * so we don't call ServiceNow on
             * every keystroke.
             */
            notificationSearchTimer =
                setTimeout(
                    async () => {

                        /*
                         * New search always begins
                         * from page 1.
                         */
                        currentNotificationPage =
                            1;


                        await loadNotifications(
                            true
                        );

                    },
                    180
                );
        }
    );
}


/* -------------------------------------------------
   CLEAR NOTIFICATION SEARCH
------------------------------------------------- */

if (notificationSearchClear) {

    notificationSearchClear.addEventListener(
        'click',

        async () => {

            if (notificationSearchTimer) {

                clearTimeout(
                    notificationSearchTimer
                );

                notificationSearchTimer =
                    null;
            }


            if (notificationSearchInput) {

                notificationSearchInput.value =
                    '';

                notificationSearchInput.focus();
            }


            notificationSearchClear.style.display =
                'none';


            currentNotificationSearch =
                '';

            notificationSearchVersion++;

            currentNotificationPage =
                1;


            await loadNotifications(
                true
            );
        }
    );
}


/* -------------------------------------------------
   NOTIFICATION FILTERS
------------------------------------------------- */

notificationFilterButtons.forEach(
    button => {

        button.addEventListener(
            'click',

            () => {

                /*
                 * Update selected filter visually.
                 */
                notificationFilterButtons.forEach(
                    filterButton => {

                        filterButton.classList.remove(
                            'active'
                        );
                    }
                );


                button.classList.add(
                    'active'
                );


                currentNotificationFilter =
                    String(
                        button.dataset
                            .notificationFilter ||
                        'all'
                    )
                        .toLowerCase()
                        .trim();


                /*
                 * IMPORTANT:
                 *
                 * Do NOT contact ServiceNow here.
                 *
                 * The notifications are already
                 * available in memory, so switching
                 * All / Unread / Read should feel
                 * immediate.
                 */
                renderNotifications(
                    cachedNotifications
                );


                /*
                 * Pagination still belongs to the
                 * server result currently loaded.
                 */
                if (
                    cachedNotificationResult
                ) {

                    renderNotificationPagination(
                        cachedNotificationResult
                    );
                }
            }
        );
    }
);


/* -------------------------------------------------
   GLOBAL NOTIFICATION AUTO REFRESH
------------------------------------------------- */

function startNotificationsAutoRefresh() {

    if (
        notificationAutoRefreshTimer
    ) {
        return;
    }


    /*
     * Establish baseline immediately.
     *
     * This is silent because ServiceCall may
     * currently be displaying Home/Meetings/etc.
     */
    loadNotifications(
        true
    );


    notificationAutoRefreshTimer =
        setInterval(
            async () => {

                try {

                    /*
                     * Background monitoring always
                     * checks page 1 because that's
                     * where newly created notifications
                     * appear.
                     *
                     * Preserve the user's pagination.
                     */
                    const originalPage =
                        currentNotificationPage;


                    currentNotificationPage =
                        1;


                    await loadNotifications(
                        true
                    );


                    currentNotificationPage =
                        originalPage;


                } catch (error) {

                    console.error(
                        'Notification auto-refresh failed:',
                        error
                    );
                }

            },
            15000
        );
}


/*
 * Start notification monitoring for the
 * lifetime of the ServiceCall renderer.
 */
startNotificationsAutoRefresh();

const refreshRecordingsButton =
    document.getElementById(
        'refreshRecordingsButton'
    );
 
 
const recordingsMessage =
    document.getElementById(
        'recordingsMessage'
    );
 
 
const recordingsList =
    document.getElementById(
        'recordingsList'
    );

window.addEventListener(
    'scroll',
    () => {

        if (
            scheduleMeetingPeopleResults
        ) {

            scheduleMeetingPeopleResults.style.display =
                'none';
        }
    },
    true
);

document.addEventListener(
    'click',
    event => {

        if (
            !scheduleMeetingPeopleSearch ||
            !scheduleMeetingPeopleResults
        ) {
            return;
        }


        const clickedSearch =
            scheduleMeetingPeopleSearch.contains(
                event.target
            );


        const clickedResults =
            scheduleMeetingPeopleResults.contains(
                event.target
            );


        /*
         * Click anywhere outside the
         * People search/results → close dropdown.
         */
        if (
            !clickedSearch &&
            !clickedResults
        ) {

            scheduleMeetingPeopleResults.style.display =
                'none';
        }
    }
);
 
 
async function loadRecordingHistory() {
 
    if (
        !recordingsMessage ||
        !recordingsList
    ) {
        return;
    }
 
 
    recordingsMessage.textContent =
        'Loading recordings...';
 
 
    recordingsList.innerHTML =
        '';
 
 
    try {
 
        const result =
            await window.serviceCall
                .getRecordingHistory();
 
 
        if (
            !result ||
            result.success !== true
        ) {
 
            recordingsMessage.textContent =
                result &&
                result.message
                    ? result.message
                    : 'Unable to load recordings.';
 
            return;
        }
 
 
        const recordings =
            Array.isArray(
                result.recordings
            )
                ? result.recordings
                : [];
 
 
        if (
            recordings.length === 0
        ) {
 
            recordingsMessage.textContent =
                'No recordings found.';
 
            return;
        }
 
 
        recordingsMessage.textContent =
            recordings.length +
            (
                recordings.length === 1
                    ? ' recording'
                    : ' recordings'
            );
 
 
        recordings.forEach(
            (recording) => {
 
                const card =
                    document.createElement(
                        'div'
                    );
 
 
                card.style.cssText = `
                    border:1px solid #dfe8e5;
                    border-radius:10px;
                    padding:16px;
                    margin-bottom:12px;
                    background:#ffffff;
                `;
 
 
                /*
                 * ---------------------------------
                 * HEADER
                 * ---------------------------------
                 */
 
                const title =
                    document.createElement(
                        'div'
                    );
 
 
                title.style.cssText = `
                    font-weight:600;
                    font-size:15px;
                    margin-bottom:8px;
                `;
 
 
                title.textContent =
                    recording.number ||
                    'ServiceCall Recording';
 
 
                card.appendChild(
                    title
                );
 
 
                /*
                 * ---------------------------------
                 * DETAILS
                 * ---------------------------------
                 */
 
                const details =
                    document.createElement(
                        'div'
                    );
 
 
                details.style.cssText = `
                    font-size:13px;
                    line-height:1.7;
                    color:#52635f;
                `;
 
 
                const participants =
                    Array.isArray(
                        recording.participants
                    )
                        ? recording.participants
                            .join(', ')
                        : '';
 
 
                details.textContent =
                    'Call: ' +
                    (
                        recording.call_number ||
                        '-'
                    ) +
                    '\n' +
 
                    'Participants: ' +
                    (
                        participants ||
                        '-'
                    ) +
                    '\n' +
 
                    'Status: ' +
                    (
                        recording.status ||
                        '-'
                    ) +
                    '\n' +
 
                    'Format: ' +
                    (
                        recording.format ||
                        '-'
                    ) +
                    '\n' +
 
                    'Started: ' +
                    (
                        recording.started_at ||
                        '-'
                    );
 
 
                details.style.whiteSpace =
                    'pre-line';
 
 
                card.appendChild(
                    details
                );
 
 
                /*
                 * ---------------------------------
                 * AVAILABLE → DOWNLOAD
                 * ---------------------------------
                 */
 
                if (
                    recording.status ===
                    'available'
                ) {
 
                    const downloadButton =
                        document.createElement(
                            'button'
                        );
 
 
                    downloadButton.type =
                        'button';
 
 
                    downloadButton.className =
                        'button';
 
 
                    downloadButton.textContent =
                        'Download';
 
 
                    downloadButton.style.marginTop =
                        '12px';
 
 
                    downloadButton.addEventListener(
                        'click',
 
                        async () => {
 
                            downloadButton.disabled =
                                true;
 
 
                            downloadButton.textContent =
                                'Downloading...';
 
 
                            try {
 
                                const downloadResult =
                                    await window
                                        .serviceCall
                                        .downloadRecording(
                                            recording
                                                .recording_sys_id
                                        );
 
 
                                if (
                                    downloadResult &&
                                    downloadResult.success
                                ) {
 
                                    recordingsMessage
                                        .textContent =
                                        'Recording downloaded successfully.';
 
                                } else if (
                                    downloadResult &&
                                    downloadResult.code ===
                                        'DOWNLOAD_CANCELLED'
                                ) {
 
                                    recordingsMessage
                                        .textContent =
                                        'Download cancelled.';
 
                                } else {
 
                                    recordingsMessage
                                        .textContent =
                                        downloadResult &&
                                        downloadResult.message
                                            ? downloadResult.message
                                            : 'Unable to download recording.';
                                }
 
 
                            } catch (error) {
 
                                recordingsMessage
                                    .textContent =
                                    error.message ||
                                    'Unable to download recording.';
 
                            } finally {
 
                                downloadButton.disabled =
                                    false;
 
 
                                downloadButton.textContent =
                                    'Download';
                            }
                        }
                    );
 
 
                    card.appendChild(
                        downloadButton
                    );
                }
 
 
                /*
                 * ---------------------------------
                 * PROCESSING
                 * ---------------------------------
                 */
 
                if (
                    recording.status ===
                    'processing'
                ) {
 
                    const state =
                        document.createElement(
                            'div'
                        );
 
 
                    state.style.cssText = `
                        margin-top:12px;
                        font-size:13px;
                        color:#667773;
                    `;
 
 
                    state.textContent =
                        'Recording is being processed...';
 
 
                    card.appendChild(
                        state
                    );
                }
 
 
                /*
                 * ---------------------------------
                 * EXPIRED
                 * ---------------------------------
                 */
 
                if (
                    recording.status ===
                    'expired'
                ) {
 
                    const state =
                        document.createElement(
                            'div'
                        );
 
 
                    state.style.cssText = `
                        margin-top:12px;
                        font-size:13px;
                        color:#667773;
                    `;
 
 
                    state.textContent =
                        'Recording expired';
 
 
                    card.appendChild(
                        state
                    );
                }
 
 
                recordingsList.appendChild(
                    card
                );
            }
        );
 
 
    } catch (error) {
 
        console.error(
            'Unable to load recording history:',
            error
        );
 
 
        recordingsMessage.textContent =
            error.message ||
            'Unable to load recordings.';
    }
}
 
 
if (
    refreshRecordingsButton
) {
 
    refreshRecordingsButton.addEventListener(
        'click',
        loadRecordingHistory
    );
}

function updatePresenceDisplay(
    status
) {

    const normalized =
        String(
            status || 'available'
        )
            .trim()
            .toLowerCase();


    let label =
        'Available';

    let cssClass =
        'available';


    if (
        normalized === 'busy'
    ) {

        label = 'Busy';
        cssClass = 'busy';

    } else if (
        normalized === 'away'
    ) {

        label = 'Away';
        cssClass = 'away';

    } else if (
        normalized === 'out of office'
    ) {

        label = 'Out of Office';
        cssClass = 'out-of-office';

    } else if (
        normalized === 'in call'
    ) {

        label = 'In a Call';
        cssClass = 'in-call';

    } else if (
        normalized === 'offline'
    ) {

        label = 'Offline';
        cssClass = 'offline';
    }


    if (
        presenceText
    ) {

        presenceText.textContent =
            label;
    }


    if (
        presenceDot
    ) {

        presenceDot.className =
            'presence-dot ' +
            cssClass;
    }
}

async function loadMyPresence() {

    try {

        const result =
            await window.serviceCall
                .getMyPresence();


        if (
            !result ||
            result.success !== true
        ) {

            console.warn(
                'Unable to load presence:',
                result
            );

            return;
        }


        /*
         * Display the effective status.
         *
         * This respects:
         * Offline → In Call → Ringing →
         * user-selected presence.
         */
        updatePresenceDisplay(
            result.effective_status ||
            result.presence_status ||
            'available'
        );


        /*
         * Keep the saved OOF reason ready
         * for editing/reuse.
         */
        if (
            oofReasonInput
        ) {

            oofReasonInput.value =
                result.oof_reason || '';
        }


        console.log(
            'ServiceCall presence loaded:',
            result
        );

    } catch (error) {

        console.error(
            'Unable to load ServiceCall presence:',
            error
        );
    }
}

/* =====================================================
   PRESENCE MENU
===================================================== */

if (
    presenceButton &&
    presenceMenu
) {

    presenceButton.addEventListener(
        'click',
        event => {

            event.stopPropagation();

            const isOpen =
                presenceMenu.style.display ===
                'block';


            presenceMenu.style.display =
                isOpen
                    ? 'none'
                    : 'block';
        }
    );
}


/*
 * Available / Busy / Away / OOF
 */

presenceOptions.forEach(
    option => {

        option.addEventListener(
            'click',

            async event => {

                event.stopPropagation();


                const status =
                    String(
                        option.dataset.presence ||
                        ''
                    )
                        .trim()
                        .toLowerCase();


                if (!status) {
                    return;
                }


                /*
                 * OOF needs a reason before
                 * being saved.
                 */

                if (
                    status ===
                    'out of office'
                ) {

                    if (
                        oofReasonPanel
                    ) {

                        oofReasonPanel.style.display =
                            'block';
                    }


                    if (
                        oofReasonInput
                    ) {

                        oofReasonInput.focus();
                    }


                    return;
                }


                /*
                 * Available / Busy / Away
                 */

                try {

                    const result =
                        await window.serviceCall
                            .updatePresence(
                                status,
                                ''
                            );


                    if (
                        !result ||
                        result.success !== true
                    ) {

                        throw new Error(
                            result &&
                            result.message
                                ? result.message
                                : 'Unable to update presence.'
                        );
                    }


                    updatePresenceDisplay(
                        result.effective_status ||
                        status
                    );


                    if (
                        oofReasonPanel
                    ) {

                        oofReasonPanel.style.display =
                            'none';
                    }


                    if (
                        presenceMenu
                    ) {

                        presenceMenu.style.display =
                            'none';
                    }


                } catch (error) {

                    console.error(
                        'Unable to update presence:',
                        error
                    );
                }
            }
        );
    }
);

/* =====================================================
   OUT OF OFFICE
===================================================== */

if (
    saveOofButton
) {

    saveOofButton.addEventListener(
        'click',

        async event => {

            event.stopPropagation();


            const reason =
                String(
                    oofReasonInput
                        ? oofReasonInput.value
                        : ''
                ).trim();


            if (!reason) {

                if (
                    oofReasonInput
                ) {

                    oofReasonInput.focus();
                }

                return;
            }


            try {

                const result =
                    await window.serviceCall
                        .updatePresence(
                            'out of office',
                            reason
                        );


                if (
                    !result ||
                    result.success !== true
                ) {

                    throw new Error(
                        result &&
                        result.message
                            ? result.message
                            : 'Unable to update Out of Office.'
                    );
                }


                updatePresenceDisplay(
                    result.effective_status ||
                    'out of office'
                );


                if (
                    oofReasonPanel
                ) {

                    oofReasonPanel.style.display =
                        'none';
                }


                if (
                    presenceMenu
                ) {

                    presenceMenu.style.display =
                        'none';
                }


            } catch (error) {

                console.error(
                    'Unable to update Out of Office:',
                    error
                );
            }
        }
    );
}

if (
    cancelOofButton
) {

    cancelOofButton.addEventListener(
        'click',
        event => {

            event.stopPropagation();


            if (
                oofReasonPanel
            ) {

                oofReasonPanel.style.display =
                    'none';
            }

        }
    );
}

document.addEventListener(
    'click',
    event => {

        if (
            presenceMenu &&
            presenceButton &&
            !presenceMenu.contains(
                event.target
            ) &&
            !presenceButton.contains(
                event.target
            )
        ) {

            presenceMenu.style.display =
                'none';


            if (
                oofReasonPanel
            ) {

                oofReasonPanel.style.display =
                    'none';
            }
        }
    }
);

if (resetPresenceButton) {

    resetPresenceButton.addEventListener(
        'click',
        async () => {

            try {

                resetPresenceButton.disabled = true;

                const result =
                    await window.serviceCall
                        .updatePresence(
                            'available',
                            ''
                        );

                if (
                    !result ||
                    result.success !== true
                ) {

                    console.error(
                        'Unable to reset presence:',
                        result
                    );

                    return;
                }


                updatePresenceDisplay(
                    result.effective_status ||
                    result.presence_status ||
                    'available'
                );


                if (oofReasonPanel) {
                    oofReasonPanel.style.display =
                        'none';
                }


                if (presenceMenu) {
                    presenceMenu.style.display =
                        'none';
                }


                console.log(
                    'ServiceCall presence reset:',
                    result
                );

            } catch (error) {

                console.error(
                    'Unable to reset ServiceCall presence:',
                    error
                );

            } finally {

                resetPresenceButton.disabled =
                    false;
            }
        }
    );
}

async function loadCurrentAccount() {

    const nameElement =
        document.getElementById(
            'currentAccountName'
        );

    const usernameElement =
        document.getElementById(
            'currentAccountUsername'
        );

    const serviceCallIdElement =
        document.getElementById(
            'currentAccountServiceCallId'
        );

    const accountMessage =
        document.getElementById(
            'accountMessage'
        );


    try {

        const result =
            await window.serviceCall
                .getCurrentAccount();


        console.log(
            'Current ServiceCall account:',
            result
        );


        if (
            !result?.success ||
            !result?.user
        ) {

            throw new Error(
                'Current ServiceCall account could not be loaded.'
            );
        }


        const user =
            result.user;

        const authorization =
            result.authorization || {};


        /*
         * NAME
         */

        if (nameElement) {

            nameElement.textContent =
                user.name ||
                'ServiceCall User';
        }


        /*
         * USERNAME
         */

        if (usernameElement) {

            usernameElement.textContent =
                user.user_name
                    ? `@${user.user_name}`
                    : '';
        }


        /*
         * SERVICECALL ID
         */

        if (serviceCallIdElement) {

            serviceCallIdElement.textContent =
                user.servicecall_id
                    ? `ServiceCall ID: ${user.servicecall_id}`
                    : '';
        }


        /*
         * ACCESS LEVEL
         */

        if (accountMessage) {

            if (
                authorization
                    .is_servicecall_admin ===
                true
            ) {

                accountMessage.textContent =
                    'ServiceCall Administrator';
            }

            else {

                accountMessage.textContent =
                    'ServiceCall User';
            }
        }

    }
    catch (error) {

        console.error(
            'Failed to load current account:',
            error
        );


        if (accountMessage) {

            accountMessage.textContent =
                'Unable to load account information.';
        }
    }
}


/* =====================================================
   SIGN OUT
===================================================== */

if (signOutButton) {

    signOutButton.addEventListener(
        'click',
        async () => {

            /*
             * Prevent duplicate sign-out requests.
             */
            signOutButton.disabled =
                true;

            const originalText =
                signOutButton.textContent;

            signOutButton.textContent =
                'Signing out...';


            const signOutMessage =
    document.getElementById(
        'accountMessage'
    );


            try {

                const result =
                    await window.serviceCall
                        .signOut();


                console.log(
                    'ServiceCall sign out result:',
                    result
                );


                if (
                    !result ||
                    result.success !== true
                ) {

                    throw new Error(
                        result?.message ||
                        'Unable to sign out.'
                    );
                }


                /*
                 * main.js owns navigation.
                 *
                 * It will load:
                 *
                 * auth/auth-gate.html
                 *
                 * after the current runtime
                 * session has been stopped.
                 */

            } catch (error) {

                console.error(
                    'ServiceCall sign out failed:',
                    error
                );


                if (accountMessage) {

                    accountMessage.textContent =
                        error?.message ||
                        'Unable to sign out.';
                }


                signOutButton.disabled =
                    false;

                signOutButton.textContent =
                    originalText;
            }
        }
    );
}

await loadCurrentAccount();
});

