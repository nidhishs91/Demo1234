main.js

const {
  app,
  BrowserWindow,
  ipcMain,
  shell,
  Tray,
  Menu,
  desktopCapturer,
  dialog,
  powerMonitor,
} = require("electron");

const path = require("path");
const fs = require("fs");
const crypto = require("crypto");
const http = require("http");
const os = require("os");

const { convertWebmToMp3, convertWebmToMp4 } = require("./media-converter");

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
const CALLBACK_HOST = "127.0.0.1";
const CALLBACK_PORT = 42813;
const SERVICECALL_PROTOCOL = "servicecall";

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

const notificationPopupWindows = [];

const notificationPopupQueue = [];

const MAX_VISIBLE_NOTIFICATION_POPUPS = 3;

const NOTIFICATION_POPUP_WIDTH = 380;

const NOTIFICATION_POPUP_HEIGHT = 150;

const NOTIFICATION_POPUP_MARGIN = 18;

const NOTIFICATION_POPUP_GAP = 10;

/* -------------------------------------------------------
   CONFIG
------------------------------------------------------- */

function getConfigPath() {
  return path.join(app.getPath("userData"), "servicecall-config.json");
}

function saveConfig(config) {
  fs.writeFileSync(getConfigPath(), JSON.stringify(config, null, 2), "utf8");
}

function loadConfig() {
  const configPath = getConfigPath();

  if (!fs.existsSync(configPath)) {
    return {};
  }

  return JSON.parse(fs.readFileSync(configPath, "utf8"));
}

function ensureAccountStore(config) {
  if (!config || typeof config !== "object") {
    config = {};
  }

  /*
   * Saved ServiceCall accounts.
   */
  if (!Array.isArray(config.accounts)) {
    config.accounts = [];
  }

  /*
   * Currently selected saved account.
   */
  if (typeof config.activeAccountId !== "string") {
    config.activeAccountId = "";
  }

  return config;
}

/* =========================================================
   SAVED ACCOUNT HELPERS
========================================================= */

function getSavedAccountKey(instanceUrl, userSysId) {
  const normalizedInstance = String(instanceUrl || "")
    .trim()
    .replace(/\/+$/, "")
    .toLowerCase();

  const normalizedUser = String(userSysId || "")
    .trim()
    .toLowerCase();

  if (!normalizedInstance || !normalizedUser) {
    return "";
  }

  return normalizedInstance + "::" + normalizedUser;
}

function ensureSavedAccountStructure(config) {
  if (!config || typeof config !== "object") {
    config = {};
  }

  if (
    !config.savedAccounts ||
    typeof config.savedAccounts !== "object" ||
    Array.isArray(config.savedAccounts)
  ) {
    config.savedAccounts = {};
  }

  if (typeof config.activeAccountKey !== "string") {
    config.activeAccountKey = "";
  }

  return config;
}

/* =========================================================
   SAVE AUTHENTICATED ACCOUNT
========================================================= */

function saveAuthenticatedAccount(config, user, authorization) {
  config = ensureSavedAccountStructure(config);

  if (!user || !user.sys_id) {
    throw new Error("Authenticated ServiceCall user is missing.");
  }

  const accountKey = getSavedAccountKey(config.instanceUrl, user.sys_id);

  if (!accountKey) {
    throw new Error("Unable to create the ServiceCall account key.");
  }

  /*
   * Preserve anything already stored
   * for this account.
   */

  const existingAccount = config.savedAccounts[accountKey] || {};

  const now = new Date().toISOString();

  config.savedAccounts[accountKey] = {
    /*
     * -----------------------------------------
     * STABLE ACCOUNT IDENTITY
     * -----------------------------------------
     */

    accountKey: accountKey,

    instanceUrl: config.instanceUrl || "",

    userSysId: user.sys_id,

    name: user.name || "",

    userName: user.user_name || "",

    email: user.email || "",

    serviceCallId: user.servicecall_id || "",

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

    isServiceCallUser: authorization?.is_servicecall_user === true,

    isServiceCallAdmin: authorization?.is_servicecall_admin === true,

    /*
     * -----------------------------------------
     * OAUTH CREDENTIALS
     * -----------------------------------------
     *
     * Each saved account owns its own
     * OAuth credentials.
     */

    accessToken: config.accessToken || existingAccount.accessToken || "",

    refreshToken: config.refreshToken || existingAccount.refreshToken || "",

    tokenType: config.tokenType || existingAccount.tokenType || "Bearer",

    expiresIn: config.expiresIn || existingAccount.expiresIn || 0,

    tokenObtainedAt:
      config.tokenObtainedAt || existingAccount.tokenObtainedAt || 0,

    /*
     * -----------------------------------------
     * ACCOUNT TIMESTAMPS
     * -----------------------------------------
     */

    addedAt: existingAccount.addedAt || now,

    lastUsedAt: now,

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

    notificationCheckpoint: existingAccount.notificationCheckpoint || {
      createdAt: "",
      sysId: "",
    },
  };

  /*
   * This account becomes the currently
   * selected ServiceCall account.
   */

  config.activeAccountKey = accountKey;

  return {
    config: config,

    accountKey: accountKey,

    account: config.savedAccounts[accountKey],
  };
}

function getActiveNotificationCheckpoint() {
  try {
    const config = ensureSavedAccountStructure(loadConfig());

    const activeAccountKey = String(config.activeAccountKey || "").trim();

    if (!activeAccountKey || !config.savedAccounts[activeAccountKey]) {
      return {
        createdAt: "",
        sysId: "",
      };
    }

    const account = config.savedAccounts[activeAccountKey];

    const checkpoint = account.notificationCheckpoint || {};

    return {
      createdAt: String(checkpoint.createdAt || "").trim(),

      sysId: String(checkpoint.sysId || "").trim(),
    };
  } catch (error) {
    console.error("Unable to read ServiceCall notification checkpoint:", error);

    return {
      createdAt: "",
      sysId: "",
    };
  }
}

function saveActiveNotificationCheckpoint(notification) {
  try {
    if (!notification) {
      return false;
    }

    const notificationSysId = String(notification.sys_id || "").trim();

    const createdAt = String(notification.created_at || "").trim();

    if (!notificationSysId || !createdAt) {
      console.warn(
        "ServiceCall notification checkpoint not saved because notification identity is incomplete:",
        notification,
      );

      return false;
    }

    const config = ensureSavedAccountStructure(loadConfig());

    const activeAccountKey = String(config.activeAccountKey || "").trim();

    if (!activeAccountKey || !config.savedAccounts[activeAccountKey]) {
      console.warn(
        "ServiceCall notification checkpoint not saved because no active saved account exists.",
      );

      return false;
    }

    config.savedAccounts[activeAccountKey].notificationCheckpoint = {
      createdAt: createdAt,

      sysId: notificationSysId,
    };

    saveConfig(config);

    console.log("ServiceCall notification checkpoint saved:", {
      accountKey: activeAccountKey,

      createdAt: createdAt,

      sysId: notificationSysId,
    });

    return true;
  } catch (error) {
    console.error("Unable to save ServiceCall notification checkpoint:", error);

    return false;
  }
}

/* =========================================================
   SYNC ACTIVE ACCOUNT OAUTH TOKENS
========================================================= */

function syncActiveAccountTokens(config) {
  config = ensureSavedAccountStructure(config);

  const accountKey = String(config.activeAccountKey || "").trim();

  /*
   * No active saved account yet.
   *
   * This is expected during our migration
   * from the old single-account config.
   */
  if (!accountKey) {
    return config;
  }

  const account = config.savedAccounts[accountKey];

  /*
   * Never create an account here.
   *
   * Account creation happens only after
   * /me tells us who authenticated.
   */
  if (!account) {
    console.warn("ServiceCall active saved account was not found:", accountKey);

    return config;
  }

  /* -----------------------------------------
       COPY CURRENT OAUTH SESSION
       INTO THE ACTIVE ACCOUNT
    ----------------------------------------- */

  account.accessToken = config.accessToken || "";

  account.refreshToken = config.refreshToken || "";

  account.tokenType = config.tokenType || "Bearer";

  account.expiresIn = config.expiresIn || 0;

  account.tokenObtainedAt = config.tokenObtainedAt || 0;

  /*
   * Keep instance information synchronized.
   */
  account.instanceUrl = config.instanceUrl || account.instanceUrl || "";

  /*
   * Token refresh counts as account activity.
   */
  account.lastUsedAt = new Date().toISOString();

  config.savedAccounts[accountKey] = account;

  return config;
}

/* =========================================================
   GET SAVED ACCOUNTS
========================================================= */

function getSavedAccounts() {
  let config = loadConfig();

  config = ensureSavedAccountStructure(config);

  const accounts = Object.values(config.savedAccounts);

  /*
   * Most recently used accounts first.
   */
  accounts.sort((a, b) => {
    const aTime = new Date(a.lastUsedAt || a.addedAt || 0).getTime();

    const bTime = new Date(b.lastUsedAt || b.addedAt || 0).getTime();

    return bTime - aTime;
  });

  /*
   * SECURITY:
   *
   * Never send OAuth tokens to the renderer.
   */
  return accounts.map((account) => ({
    accountKey: account.accountKey || "",

    instanceUrl: account.instanceUrl || "",

    userSysId: account.userSysId || "",

    name: account.name || "",

    userName: account.userName || "",

    email: account.email || "",

    serviceCallId: account.serviceCallId || "",

    isServiceCallUser: account.isServiceCallUser === true,

    isServiceCallAdmin: account.isServiceCallAdmin === true,

    addedAt: account.addedAt || "",

    lastUsedAt: account.lastUsedAt || "",

    isActive: account.accountKey === config.activeAccountKey,
  }));
}

function updateActiveSavedAccountAuthorization(authorization) {
  let config = loadConfig();

  config = ensureSavedAccountStructure(config);

  const accountKey = config.activeAccountKey;

  if (!accountKey || !config.savedAccounts[accountKey]) {
    return;
  }

  const account = config.savedAccounts[accountKey];

  /*
   * DISPLAY METADATA ONLY.
   *
   * These cached values must never be
   * used to authorize ServiceCall.
   */

  account.isServiceCallUser = authorization?.is_servicecall_user === true;

  account.isServiceCallAdmin = authorization?.is_servicecall_admin === true;

  config.savedAccounts[accountKey] = account;

  saveConfig(config);
}

function getOrCreateDeviceId() {
  const config = loadConfig();

  if (config.deviceId) {
    return config.deviceId;
  }

  const deviceId = crypto.randomUUID();

  config.deviceId = deviceId;

  saveConfig(config);

  return deviceId;
}

async function sendHeartbeatOnce() {
  if (isDeviceSuspended) {
    console.log("ServiceCall heartbeat skipped because device is suspended.");

    return {
      success: true,
      skipped: true,
      reason: "device_suspended",
    };
  }

  const config = loadConfig();

  if (!config.instanceUrl) {
    throw new Error("ServiceNow instance is not configured.");
  }

  if (!config.accessToken) {
    throw new Error("ServiceNow access token was not found.");
  }

  const validAccessToken = await ensureValidAccessToken();

  const heartbeatUrl = config.instanceUrl + config.heartbeatPath;

  const payload = {
    device_id: getOrCreateDeviceId(),

    device_name: os.hostname(),

    platform: process.platform === "win32" ? "Windows" : process.platform,

    app_version: app.getVersion(),
  };

  let response = await fetch(heartbeatUrl, {
    method: "POST",

    headers: {
      Authorization: "Bearer " + validAccessToken,

      "Content-Type": "application/json",

      Accept: "application/json",
    },

    body: JSON.stringify(payload),
  });

  /*
   * If the short-lived access token expired,
   * automatically renew it and retry heartbeat once.
   */
  if (response.status === 401 || response.status === 403) {
    console.log(
      "Heartbeat authorization expired. Attempting automatic renewal...",
    );

    try {
      const newAccessToken = await refreshAccessToken();

      response = await fetch(heartbeatUrl, {
        method: "POST",

        headers: {
          Authorization: "Bearer " + newAccessToken,

          "Content-Type": "application/json",

          Accept: "application/json",
        },

        body: JSON.stringify(payload),
      });
    } catch (refreshError) {
      const authError = new Error(
        "Your ServiceCall authorization has expired. Please sign in again.",
      );

      authError.code = "AUTHENTICATION_REQUIRED";

      throw authError;
    }
  }

  const responseText = await response.text();

  let data;

  try {
    data = JSON.parse(responseText);
  } catch (error) {
    throw new Error("Heartbeat API returned an invalid response.");
  }

  if (!response.ok) {
    console.error("Heartbeat HTTP status:", response.status);

    console.error("Heartbeat response:", JSON.stringify(data, null, 2));

    if (response.status === 401 || response.status === 403) {
      const authError = new Error(
        "Your ServiceNow session has expired. Please sign in again.",
      );

      authError.code = "AUTHENTICATION_REQUIRED";

      throw authError;
    }

    var errorMessage = "Heartbeat failed with HTTP " + response.status;

    if (data && typeof data.error === "object" && data.error) {
      errorMessage = data.error.message || data.error.detail || errorMessage;
    } else if (data && typeof data.error === "string") {
      errorMessage = data.error;
    } else if (data && typeof data.message === "string") {
      errorMessage = data.message;
    }

    throw new Error(errorMessage);
  }

  return data;
}

async function updateDesktopState(state) {
  try {
    const config = loadConfig();

    if (!config.instanceUrl || !config.accessToken) {
      return {
        success: false,
        code: "NOT_CONNECTED",
      };
    }

    const result = await serviceCallApiRequest("/desktop-state", "POST", {
      device_id: getOrCreateDeviceId(),

      state: state,
    });

    console.log("ServiceCall desktop state:", result);

    return result;
  } catch (error) {
    console.error("Unable to update ServiceCall desktop state:", error.message);

    return {
      success: false,
      code: error.code || "DESKTOP_STATE_FAILED",
      message: error.message,
    };
  }
}

async function checkIncomingCallOnce() {
  try {
    const result = await serviceCallApiRequest("/incoming-call", "GET");

    if (result && result.success && result.incoming_call) {
      const incomingCallId = result.call_sys_id;

      if (
        incomingCallId &&
        !intentionallyLeftCallIds.has(incomingCallId) &&
        incomingCallId !== activeIncomingCallId
      ) {
        activeIncomingCallId = incomingCallId;

        console.log("Incoming ServiceCall:", result);

        const isConference =
          result.is_conference === true || result.call_type === "conference";

        const displayName = isConference
          ? result.conference_display || "Conference Call"
          : result.caller_name || "Unknown User";

        const displayDepartment = isConference
          ? "Conference Call"
          : result.caller_department || "";

        showCallWindow("incoming", {
          callSysId: result.call_sys_id,

          callNumber: result.call_number,

          name: displayName,

          department: displayDepartment,

          isConference: isConference,
        });
      }

      return;
    }

    /*
     * No incoming ringing invitation.
     */
    activeIncomingCallId = null;
  } catch (error) {
    console.error("Incoming call check failed:", error);

    if (error.code === "AUTHENTICATION_REQUIRED") {
      stopIncomingCallLoop();

      sendAuthStatus(
        "authentication_required",
        "Your ServiceCall authorization has expired. Please sign in to ServiceNow again.",
      );
    }
  }
}

function stopIncomingCallLoop() {
  if (incomingCallTimer) {
    clearInterval(incomingCallTimer);

    incomingCallTimer = null;
  }
}

/* =========================================================
   SERVICECALL NOTIFICATION MONITOR
========================================================= */

async function checkNotificationsOnce() {
  try {
    const result = await serviceCallApiRequest(
      "/notifications?page=1&page_size=20&filter=unread",
      "GET",
    );

    if (!result || result.success !== true) {
      console.warn(
        "ServiceCall notification check returned no valid result:",
        result,
      );

      return;
    }

    const notifications = Array.isArray(result.notifications)
      ? result.notifications
      : [];

    /*
     * -----------------------------------------
     * NO UNREAD NOTIFICATIONS
     * -----------------------------------------
     */

    if (notifications.length === 0) {
      console.log("ServiceCall notification check: no unread notifications.");

      return;
    }

    /*
     * -----------------------------------------
     * CURRENT ACCOUNT CHECKPOINT
     * -----------------------------------------
     */

    const checkpoint = getActiveNotificationCheckpoint();

    console.log("ServiceCall notification checkpoint:", checkpoint);

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

    if (!checkpoint.createdAt || !checkpoint.sysId) {
      const newestNotification = notifications[0];

      if (
        newestNotification &&
        newestNotification.sys_id &&
        newestNotification.created_at
      ) {
        saveActiveNotificationCheckpoint(newestNotification);

        /*
         * Also remember all notifications
         * returned by this poll for the
         * current Electron session.
         */

        notifications.forEach((notification) => {
          const sysId = String(notification.sys_id || "").trim();

          if (sysId) {
            surfacedNotificationIds.add(sysId);
          }
        });

        console.log(
          "ServiceCall initial notification checkpoint established:",
          {
            createdAt: newestNotification.created_at,

            sysId: newestNotification.sys_id,
          },
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

    for (const notification of notifications) {
      const notificationSysId = String(notification.sys_id || "").trim();

      const createdAt = String(notification.created_at || "").trim();

      if (!notificationSysId || !createdAt) {
        continue;
      }

      /*
       * Once we reach the exact checkpoint,
       * everything after it is older because
       * the API is newest-first.
       */

      if (notificationSysId === checkpoint.sysId) {
        break;
      }

      /*
       * Anything created after the saved
       * checkpoint is new.
       */

      if (createdAt > checkpoint.createdAt) {
        newNotifications.push(notification);

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
        createdAt === checkpoint.createdAt &&
        notificationSysId !== checkpoint.sysId
      ) {
        newNotifications.push(notification);
      }
    }

    if (newNotifications.length === 0) {
      return;
    }

    console.log(
      "ServiceCall genuinely new notifications detected:",
      newNotifications.length,
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

    const notificationsToSurface = newNotifications.reverse();

    notificationsToSurface.forEach((notification) => {
      const notificationSysId = String(notification.sys_id || "").trim();

      if (!notificationSysId) {
        return;
      }

      /*
       * Same-session duplicate protection.
       */

      if (surfacedNotificationIds.has(notificationSysId)) {
        return;
      }

      surfacedNotificationIds.add(notificationSysId);

      console.log("ServiceCall NEW notification detected:", {
        sys_id: notificationSysId,

        type: notification.type || "",

        title: notification.title || "",

        message: notification.message || "",

        meeting_sys_id: notification.meeting_sys_id || "",

        created_at: notification.created_at || "",
      });

      showNotificationPopup(notification);
    });

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
      notificationsToSurface[notificationsToSurface.length - 1];

    if (newestProcessedNotification) {
      saveActiveNotificationCheckpoint(newestProcessedNotification);
    }
  } catch (error) {
    console.error("ServiceCall notification check failed:", error);

    if (error.code === "AUTHENTICATION_REQUIRED") {
      stopNotificationLoop();

      sendAuthStatus(
        "authentication_required",
        "Your ServiceCall authorization has expired. Please sign in again.",
      );
    }
  }
}

function stopNotificationLoop() {
  if (notificationTimer) {
    clearInterval(notificationTimer);

    notificationTimer = null;
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
  console.log(">>> SERVICECALL NOTIFICATION LOOP STARTED <<<");

  stopNotificationLoop();

  checkNotificationsOnce();

  notificationTimer = setInterval(checkNotificationsOnce, 5000);
}

function stopHeartbeatLoop() {
  if (heartbeatTimer) {
    clearInterval(heartbeatTimer);

    heartbeatTimer = null;
  }
}

async function startHeartbeatLoop() {
  stopHeartbeatLoop();

  try {
    const result = await sendHeartbeatOnce();

    console.log("ServiceCall heartbeat:", result);

    sendAuthStatus("connected", "ServiceCall Desktop is connected.");
  } catch (error) {
    console.error("Initial heartbeat failed:", error);

    if (error.code === "AUTHENTICATION_REQUIRED") {
      stopHeartbeatLoop();

      sendAuthStatus(
        "authentication_required",
        "Your ServiceNow session has expired. Please sign in again.",
      );

      return;
    }

    sendAuthStatus(
      "warning",
      "ServiceCall Desktop heartbeat failed: " + error.message,
    );

    return;
  }

  heartbeatTimer = setInterval(async () => {
    try {
      const result = await sendHeartbeatOnce();

      console.log("ServiceCall heartbeat:", result);
    } catch (error) {
      console.error("Heartbeat failed:", error);

      if (error.code === "AUTHENTICATION_REQUIRED") {
        stopHeartbeatLoop();

        sendAuthStatus(
          "authentication_required",
          "Your ServiceNow session has expired. Please sign in again.",
        );

        return;
      }

      sendAuthStatus(
        "warning",
        "ServiceCall Desktop heartbeat failed: " + error.message,
      );
    }
  }, 30000);
}

function startIncomingCallLoop() {
  stopIncomingCallLoop();

  checkIncomingCallOnce();

  incomingCallTimer = setInterval(checkIncomingCallOnce, 3000);
}

async function restoreSavedConnection() {
  const config = loadConfig();

  if (!config.instanceUrl || !config.accessToken) {
    console.log("No saved ServiceCall connection found.");

    return;
  }

  console.log("Saved ServiceCall connection found. Restoring...");

  try {
    await startHeartbeatLoop();

    startIncomingCallLoop();

    startOutgoingCallLoop();

    console.log("ServiceCall connection restored.");
  } catch (error) {
    console.error("Unable to restore ServiceCall connection:", error);

    sendAuthStatus(
      "warning",
      "Saved ServiceNow connection could not be restored: " + error.message,
    );
  }
}

/* -------------------------------------------------------
   INSTANCE URL
------------------------------------------------------- */

function normalizeInstanceUrl(value) {
  if (!value) {
    return "";
  }

  value = value.trim();

  if (!value.startsWith("https://")) {
    value = "https://" + value;
  }

  value = value.replace(/\/+$/, "");

  return value;
}

function isValidServiceNowUrl(value) {
  try {
    const parsed = new URL(value);

    return (
      parsed.protocol === "https:" &&
      parsed.hostname.endsWith(".service-now.com")
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
    .toString("base64")
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");
}

function generateCodeVerifier() {
  return base64UrlEncode(crypto.randomBytes(64));
}

function generateCodeChallenge(verifier) {
  const hash = crypto.createHash("sha256").update(verifier).digest();

  return base64UrlEncode(hash);
}

function generateState() {
  return base64UrlEncode(crypto.randomBytes(32));
}

/* -------------------------------------------------------
   SEND STATUS TO DESKTOP WINDOW
------------------------------------------------------- */

function sendAuthStatus(status, message) {
  if (mainWindow && !mainWindow.isDestroyed()) {
    mainWindow.webContents.send("servicecall-auth-status", {
      status: status,
      message: message,
    });
  }
}

/* =========================================================
   RUNTIME SERVICECALL AUTHORIZATION MONITOR
========================================================= */

function stopAuthorizationMonitor() {
  if (authorizationMonitorTimer) {
    clearInterval(authorizationMonitorTimer);

    authorizationMonitorTimer = null;
  }
}

/* =========================================================
   SUSPEND / RESUME SERVICECALL RUNTIME
========================================================= */

function suspendServiceCallRuntime() {
  console.warn("Suspending ServiceCall runtime...");

  /*
   * STOP HEARTBEAT
   */
  if (heartbeatTimer) {
    clearInterval(heartbeatTimer);

    heartbeatTimer = null;
  }

  /*
   * STOP INCOMING CALL MONITOR
   */
  if (incomingCallTimer) {
    clearInterval(incomingCallTimer);

    incomingCallTimer = null;
  }

  /*
   * STOP OUTGOING CALL MONITOR
   */
  if (outgoingCallTimer) {
    clearInterval(outgoingCallTimer);

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

  console.log("ServiceCall runtime suspended.");
}

async function terminateActiveSessionForAccessLoss() {
  console.warn(
    "Terminating active ServiceCall desktop session because access was removed...",
  );

  /*
   * Remember the active call before clearing
   * the desktop state.
   */

  const callSysId =
    activeCallWindowId || activeOutgoingCallId || activeIncomingCallId || null;

  /*
   * Prevent polling from reopening this call
   * if ServiceCall access is restored later.
   */

  if (callSysId) {
    intentionallyLeftCallIds.add(callSysId);

    console.log("ServiceCall call blocked from automatic reopen:", callSysId);
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

  if (callWindow && !callWindow.isDestroyed()) {
    callWindowClosing = true;

    try {
      callWindow.close();
    } catch (error) {
      console.error(
        "Unable to close ServiceCall call window during access loss:",
        error,
      );
    }
  }

  /*
   * Clear desktop call tracking.
   *
   * Restoring ServiceCall authorization must
   * NOT restore the previous call automatically.
   */

  activeCallWindowId = null;

  activeIncomingCallId = null;

  activeOutgoingCallId = null;

  console.log("Active ServiceCall desktop session cleared.");
}

async function showServiceCallUnavailablePage() {
  if (!mainWindow || mainWindow.isDestroyed()) {
    return;
  }

  try {
    await mainWindow.loadFile("access-unavailable.html");

    mainWindow.show();

    mainWindow.focus();

    console.log("ServiceCall unavailable page displayed.");
  } catch (error) {
    console.error("Unable to display ServiceCall unavailable page:", error);
  }
}

async function restoreServiceCallApplication() {
  if (!mainWindow || mainWindow.isDestroyed()) {
    return;
  }

  try {
    await mainWindow.loadFile("index.html");

    mainWindow.show();

    mainWindow.focus();

    console.log("ServiceCall application restored.");
  } catch (error) {
    console.error("Unable to restore ServiceCall application:", error);
  }
}

async function resumeServiceCallRuntime() {
  console.log("Resuming ServiceCall runtime...");

  /*
   * Restart operational background services.
   */

  await startHeartbeatLoop();

  startIncomingCallLoop();

  startOutgoingCallLoop();

  startNotificationLoop();

  console.log("ServiceCall runtime resumed.");
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

    const currentUser = await getCurrentServiceCallUser();

    const authorization = currentUser?.authorization || {};

    /*
     * Keep current identity and authorization
     * information synchronized.
     */

    currentServiceCallUser = currentUser?.user || null;

    currentServiceCallAuthorization = authorization;

    updateActiveSavedAccountAuthorization(authorization);

    /*
     * -----------------------------------------
     * SERVICECALL ACCESS REMOVED
     * -----------------------------------------
     */

    if (authorization.allowed !== true) {
      /*
       * Only perform the suspension/page
       * transition once.
       */

      if (!serviceCallAccessUnavailable) {
        serviceCallAccessUnavailable = true;

        console.warn("ServiceCall runtime access removed.", {
          user: currentServiceCallUser,

          authorization: authorization,
        });

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
          "access_removed",
          "ServiceCall access is currently unavailable for this account.",
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
      console.log("ServiceCall runtime access restored.", {
        user: currentServiceCallUser,

        authorization: authorization,
      });

      try {
        /*
         * Tell the CURRENT unavailable page
         * that access has been restored
         * BEFORE replacing that page.
         */

        mainWindow?.webContents.send("servicecall-authorization-status", {
          status: "access_restored",

          message: "ServiceCall access has been restored.",
        });

        /*
         * Keep the success/reconnecting
         * message visible for 3.5 seconds.
         */

        await new Promise((resolve) => {
          setTimeout(resolve, 3500);
        });

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

        serviceCallAccessUnavailable = false;

        console.log("ServiceCall runtime restored successfully.");
      } catch (restoreError) {
        /*
         * Keep ServiceCall unavailable so
         * the authorization monitor can
         * retry restoration later.
         */

        serviceCallAccessUnavailable = true;

        console.error("Unable to restore ServiceCall runtime:", restoreError);
      }
    }
  } catch (error) {
    console.error("ServiceCall runtime authorization check failed:", error);
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
  authorizationMonitorTimer = setInterval(checkRuntimeAuthorizationOnce, 30000);
}

/* -------------------------------------------------------
   TOKEN EXCHANGE
------------------------------------------------------- */

async function exchangeAuthorizationCode(code, returnedState) {
  const config = loadConfig();

  if (!config.oauthState || returnedState !== config.oauthState) {
    throw new Error("OAuth state validation failed.");
  }

  if (!config.pkceCodeVerifier) {
    throw new Error("PKCE code verifier was not found.");
  }

  const tokenUrl = config.instanceUrl + "/oauth_token.do";

  const body = new URLSearchParams();

  body.set("grant_type", "authorization_code");

  body.set("code", code);

  body.set("redirect_uri", config.redirectUri);

  body.set("client_id", config.oauthClientId);

  body.set("code_verifier", config.pkceCodeVerifier);

  body.set("state", returnedState);

  const response = await fetch(tokenUrl, {
    method: "POST",

    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
    },

    body: body.toString(),
  });

  const responseText = await response.text();

  let tokenData;

  try {
    tokenData = JSON.parse(responseText);
  } catch (error) {
    throw new Error("ServiceNow returned an invalid token response.");
  }

  if (!response.ok || !tokenData.access_token) {
    throw new Error(
      tokenData.error_description ||
        tokenData.error ||
        "Unable to obtain OAuth access token.",
    );
  }

  config.accessToken = tokenData.access_token;

  config.tokenType = tokenData.token_type || "Bearer";

  config.expiresIn = tokenData.expires_in || 0;

  config.tokenObtainedAt = Date.now();

  if (tokenData.refresh_token) {
    config.refreshToken = tokenData.refresh_token;

    console.log("ServiceCall refresh token received successfully.");
  } else {
    console.log("ServiceCall refresh token was NOT returned.");
  }

  /*
       We no longer need these after
       successful authentication.
    */

  delete config.pkceCodeVerifier;
  delete config.oauthState;

  saveConfig(config);

  /*
   * -----------------------------------------
   * VERIFY SERVICECALL AUTHORIZATION
   * -----------------------------------------
   */

  const currentUser = await getCurrentServiceCallUser();

  const authorization = currentUser?.authorization || {};

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

  currentServiceCallUser = currentUser?.user || null;

  currentServiceCallAuthorization = authorization;

  /*
   * -----------------------------------------
   * SAVE AUTHENTICATED ACCOUNT
   * -----------------------------------------
   */

  const savedAccountResult = saveAuthenticatedAccount(
    config,
    currentUser?.user || null,
    authorization,
  );

  saveConfig(savedAccountResult.config);

  console.log("ServiceCall authenticated account saved:", {
    accountKey: savedAccountResult.accountKey,

    name: savedAccountResult.account?.name,

    userName: savedAccountResult.account?.userName,
  });

  /*
   * -----------------------------------------
   * ACCESS DENIED
   * -----------------------------------------
   */

  if (authorization.allowed !== true) {
    console.warn("ServiceCall access denied:", currentUser);

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
      state: "access_denied",
      user: currentUser?.user || null,
      authorization: authorization,
    };
  }

  /*
   * -----------------------------------------
   * AUTHORIZED
   * -----------------------------------------
   */

  console.log("ServiceCall authorization successful:", {
    user: currentUser?.user,

    authorization: authorization,
  });

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
    state: "ready",
    user: currentUser?.user || null,
    authorization: authorization,
    tokenData: tokenData,
  };
}

async function ensureValidAccessToken() {
  const config = loadConfig();

  if (!config.accessToken) {
    const error = new Error("ServiceCall is not authenticated.");

    error.code = "AUTHENTICATION_REQUIRED";

    throw error;
  }

  /*
   * If expiry information is unavailable,
   * keep using the current token.
   *
   * The normal 401/403 refresh mechanism
   * remains our fallback.
   */
  if (!config.expiresIn || !config.tokenObtainedAt) {
    return config.accessToken;
  }

  const expiresAt = config.tokenObtainedAt + Number(config.expiresIn) * 1000;

  /*
   * Refresh 60 seconds before actual expiry.
   */
  const refreshAt = expiresAt - 60000;

  if (Date.now() >= refreshAt) {
    console.log(
      "ServiceCall access token is close to expiry. Renewing automatically...",
    );

    return await refreshAccessToken();
  }

  return config.accessToken;
}

async function refreshAccessToken() {
  const config = loadConfig();

  if (!config.instanceUrl || !config.oauthClientId || !config.refreshToken) {
    const error = new Error("A ServiceCall refresh token is not available.");

    error.code = "REAUTHENTICATION_REQUIRED";

    throw error;
  }

  console.log(
    "ServiceCall access token expired. Attempting automatic renewal...",
  );

  const tokenUrl = config.instanceUrl.replace(/\/$/, "") + "/oauth_token.do";

  const body = new URLSearchParams();

  body.set("grant_type", "refresh_token");

  body.set("refresh_token", config.refreshToken);

  body.set("client_id", config.oauthClientId);

  const response = await fetch(tokenUrl, {
    method: "POST",

    headers: {
      "Content-Type": "application/x-www-form-urlencoded",

      Accept: "application/json",
    },

    body: body.toString(),
  });

  const responseText = await response.text();

  let tokenData;

  try {
    tokenData = JSON.parse(responseText);
  } catch (error) {
    const refreshError = new Error(
      "ServiceNow returned an invalid token renewal response.",
    );

    refreshError.code = "TOKEN_REFRESH_FAILED";

    throw refreshError;
  }

  if (!response.ok || !tokenData.access_token) {
    console.error("ServiceCall automatic token renewal failed.");

    const refreshError = new Error(
      tokenData.error_description ||
        tokenData.error ||
        "ServiceNow authorization must be renewed.",
    );

    refreshError.code = "REAUTHENTICATION_REQUIRED";

    throw refreshError;
  }

  /*
   * Store the NEW short-lived access token.
   */
  config.accessToken = tokenData.access_token;

  config.tokenType = tokenData.token_type || "Bearer";

  config.expiresIn = tokenData.expires_in || 0;

  config.tokenObtainedAt = Date.now();

  /*
   * Some OAuth servers rotate refresh tokens.
   * If ServiceNow gives us a new one,
   * replace the previous refresh token.
   */
  if (tokenData.refresh_token) {
    config.refreshToken = tokenData.refresh_token;
  }

  saveConfig(config);

  console.log("ServiceCall access token renewed automatically.");

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
  return new Promise((resolve, reject) => {
    stopCallbackServer();

    callbackServer = http.createServer(async (request, response) => {
      try {
        const callbackUrl = new URL(
          request.url,
          `http://${CALLBACK_HOST}:${CALLBACK_PORT}`,
        );

        /*
         * -----------------------------------------
         * VALIDATE CALLBACK PATH
         * -----------------------------------------
         */

        if (callbackUrl.pathname !== "/callback") {
          response.writeHead(404, {
            "Content-Type": "text/plain",
          });

          response.end("Not found.");

          return;
        }

        /*
         * -----------------------------------------
         * OAUTH ERROR
         * -----------------------------------------
         */

        const oauthError = callbackUrl.searchParams.get("error");

        if (oauthError) {
          const description =
            callbackUrl.searchParams.get("error_description") || oauthError;

          response.writeHead(400, {
            "Content-Type": "text/html; charset=utf-8",
          });

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

          sendAuthStatus("error", description);

          stopCallbackServer();

          return;
        }

        /*
         * -----------------------------------------
         * READ AUTHORIZATION CODE
         * -----------------------------------------
         */

        const code = callbackUrl.searchParams.get("code");

        const state = callbackUrl.searchParams.get("state");

        if (!code || !state) {
          throw new Error("Authorization code or state was missing.");
        }

        /*
         * -----------------------------------------
         * EXCHANGE CODE + CHECK SERVICECALL ACCESS
         * -----------------------------------------
         */

        const authResult = await exchangeAuthorizationCode(code, state);

        /*
         * -----------------------------------------
         * AUTHENTICATED BUT ACCESS DENIED
         * -----------------------------------------
         */

        if (authResult?.state === "access_denied") {
          console.warn(
            "ServiceCall OAuth succeeded but access was denied:",
            authResult,
          );

          response.writeHead(200, {
            "Content-Type": "text/html; charset=utf-8",
          });

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
            "access_denied",
            "ServiceCall access has not been assigned to this account.",
          );

          if (mainWindow && !mainWindow.isDestroyed()) {
            mainWindow.show();

            mainWindow.focus();
          }

          setTimeout(stopCallbackServer, 1000);

          return;
        }

        /*
         * -----------------------------------------
         * AUTHORIZED
         * -----------------------------------------
         */

        if (authResult?.state === "ready") {
          console.log("ServiceCall login authorized:", authResult.user);

          response.writeHead(200, {
            "Content-Type": "text/html; charset=utf-8",
          });

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

          sendAuthStatus("connected", "Connected to ServiceCall.");

          /*
           * OAuth + /me authorization passed.
           *
           * exchangeAuthorizationCode()
           * has already started the background
           * ServiceCall services.
           *
           * Now enter the application.
           */

          if (mainWindow && !mainWindow.isDestroyed()) {
            await mainWindow.loadFile("index.html");

            mainWindow.show();

            mainWindow.focus();
          }

          setTimeout(stopCallbackServer, 1000);

          return;
        }

        /*
         * -----------------------------------------
         * UNEXPECTED AUTH RESULT
         * -----------------------------------------
         */

        throw new Error(
          "ServiceCall authentication completed with an unexpected result.",
        );
      } catch (error) {
        console.error("ServiceCall OAuth callback failed:", error);

        response.writeHead(500, {
          "Content-Type": "text/html; charset=utf-8",
        });

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
          "error",
          error?.message || "ServiceCall connection failed.",
        );

        stopCallbackServer();
      }
    });

    /*
     * -----------------------------------------
     * CALLBACK SERVER ERROR
     * -----------------------------------------
     */

    callbackServer.on("error", (error) => {
      callbackServer = null;

      reject(error);
    });

    /*
     * -----------------------------------------
     * START CALLBACK SERVER
     * -----------------------------------------
     */

    callbackServer.listen(CALLBACK_PORT, CALLBACK_HOST, () => {
      resolve();
    });
  });
}

/* -------------------------------------------------------
   WINDOW
------------------------------------------------------- */

function registerServiceCallProtocol() {
  if (process.defaultApp) {
    if (process.argv.length >= 2) {
      app.setAsDefaultProtocolClient(SERVICECALL_PROTOCOL, process.execPath, [
        path.resolve(process.argv[1]),
      ]);
    }
  } else {
    app.setAsDefaultProtocolClient(SERVICECALL_PROTOCOL);
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
function getServiceCallDeepLink(args) {
  if (!Array.isArray(args)) {
    return null;
  }

  const deepLink = args.find(
    (arg) => typeof arg === "string" && arg.startsWith("servicecall://"),
  );

  return deepLink || null;
}

/*
 * Handle the received deep link.
 *
 * For now we only store and log it.
 * Renderer navigation comes next.
 */
function handleServiceCallDeepLink(deepLink) {
  if (!deepLink || !deepLink.startsWith(SERVICECALL_PROTOCOL + "://")) {
    return;
  }

  pendingDeepLink = deepLink;

  console.log("ServiceCall deep link received:", deepLink);

  /*
   * If the renderer is already loaded,
   * send the link immediately.
   */
  if (
    mainWindow &&
    !mainWindow.isDestroyed() &&
    !mainWindow.webContents.isLoading()
  ) {
    mainWindow.webContents.send("servicecall-deep-link", {
      url: deepLink,
    });

    /*
     * The renderer now owns this link,
     * so it is no longer pending.
     */
    pendingDeepLink = null;
  }
}

function showMainWindow() {
  if (mainWindow && !mainWindow.isDestroyed()) {
    if (mainWindow.isMinimized()) {
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
    title: "ServiceCall Desktop",

    webPreferences: {
      preload: path.join(__dirname, "preload.js"),

      contextIsolation: true,
      nodeIntegration: false,
    },
  });

  /*
   * -----------------------------------------
   * RESOLVE STARTUP AUTHENTICATION
   * -----------------------------------------
   */

  const startupState = await resolveStartupAuthentication();

  console.log("ServiceCall startup state:", startupState);

  /*
   * -----------------------------------------
   * AUTHORIZED USER
   * -----------------------------------------
   */

  if (startupState.state === "ready") {
    /*
     * Make sure we begin in the
     * available state.
     */

    serviceCallAccessUnavailable = false;

    await mainWindow.loadFile("index.html");

    /*
     * Start background ServiceCall services
     * ONLY after authorization succeeds.
     */

    await startHeartbeatLoop();

    startIncomingCallLoop();

    startOutgoingCallLoop();

    startNotificationLoop();

    startAuthorizationMonitor();
  } else if (startupState.state === "access_denied") {
    /*
     * -----------------------------------------
     * AUTHENTICATED BUT ACCESS NOT AVAILABLE
     * -----------------------------------------
     */
    console.warn("ServiceCall access denied at startup.");

    /*
     * Startup access denial belongs to the
     * authentication flow.
     *
     * Do NOT use the runtime unavailable page
     * here.
     */
    serviceCallAccessUnavailable = false;

    await mainWindow.loadFile(path.join("auth", "auth-gate.html"));

    /*
     * Do NOT start the runtime authorization
     * monitor here.
     *
     * The runtime monitor is only for a user
     * who successfully entered ServiceCall
     * and later loses access.
     */
  } else {
    /*
     * -----------------------------------------
     * LOGIN / AUTHENTICATION GATE
     * -----------------------------------------
     */
    /*
     * No usable authenticated ServiceCall
     * session exists.
     */

    serviceCallAccessUnavailable = false;

    await mainWindow.loadFile(path.join("auth", "auth-gate.html"));
  }

  /*
   * -----------------------------------------
   * WINDOW CLOSE
   * -----------------------------------------
   */

  mainWindow.on("close", (event) => {
    if (!isQuitting) {
      event.preventDefault();

      mainWindow.hide();
    }
  });
}

/* =====================================================
   SERVICECALL DESKTOP NOTIFICATION POPUP
===================================================== */

function showNotificationPopup(notification) {
  if (!notification || !notification.sys_id) {
    return;
  }

  /*
   * If all visible popup slots are occupied,
   * keep this notification waiting.
   */
  if (notificationPopupWindows.length >= MAX_VISIBLE_NOTIFICATION_POPUPS) {
    notificationPopupQueue.push(notification);

    console.log("ServiceCall notification queued:", notification.sys_id);

    return;
  }

  createNotificationPopupWindow(notification);
}

/* =========================================================
   CREATE NOTIFICATION POPUP WINDOW
========================================================= */

function createNotificationPopupWindow(notification) {
  const { screen } = require("electron");

  const display = screen.getPrimaryDisplay();

  const workArea = display.workArea;

  /*
   * Existing visible popups are counted
   * from the bottom upward.
   *
   * 0 = bottom slot
   * 1 = second slot
   * 2 = third slot
   */
  const slotIndex = notificationPopupWindows.length;

  const popupX = Math.round(
    workArea.x +
      workArea.width -
      NOTIFICATION_POPUP_WIDTH -
      NOTIFICATION_POPUP_MARGIN,
  );

  const popupY = Math.round(
    workArea.y +
      workArea.height -
      NOTIFICATION_POPUP_HEIGHT -
      NOTIFICATION_POPUP_MARGIN -
      slotIndex * (NOTIFICATION_POPUP_HEIGHT + NOTIFICATION_POPUP_GAP),
  );

  const popupWindow = new BrowserWindow({
    width: NOTIFICATION_POPUP_WIDTH,

    height: NOTIFICATION_POPUP_HEIGHT,

    x: popupX,

    y: popupY,

    frame: false,

    transparent: true,

    resizable: false,

    movable: false,

    minimizable: false,

    maximizable: false,

    fullscreenable: false,

    skipTaskbar: true,

    alwaysOnTop: true,

    show: false,

    focusable: true,

    webPreferences: {
      preload: path.join(__dirname, "preload.js"),

      contextIsolation: true,

      nodeIntegration: false,
    },
  });

  /*
   * Keep the notification identity attached
   * to its own BrowserWindow.
   */
  popupWindow.serviceCallNotification = notification;

  notificationPopupWindows.push(popupWindow);

  popupWindow.loadFile("notification-popup.html", {
    query: {
      notificationSysId: String(notification.sys_id || ""),

      type: String(
        notification.type_display || notification.type || "Notification",
      ),

      title: String(notification.title || "ServiceCall"),

      message: String(notification.message || ""),

      meetingSysId: String(notification.meeting_sys_id || ""),

      callSysId: String(notification.call_sys_id || ""),
    },
  });

  popupWindow.once("ready-to-show", () => {
    if (popupWindow.isDestroyed()) {
      return;
    }

    popupWindow.showInactive();
  });

  popupWindow.on("closed", () => {
    const popupIndex = notificationPopupWindows.indexOf(popupWindow);

    if (popupIndex !== -1) {
      notificationPopupWindows.splice(popupIndex, 1);
    }

    /*
     * Reposition whatever remains,
     * then use the newly available slot
     * for the next queued notification.
     */
    repositionNotificationPopups();

    showNextQueuedNotification();
  });
}

/* =========================================================
   REPOSITION VISIBLE NOTIFICATIONS
========================================================= */

function repositionNotificationPopups() {
  const { screen } = require("electron");

  const display = screen.getPrimaryDisplay();

  const workArea = display.workArea;

  notificationPopupWindows.forEach((popupWindow, index) => {
    if (!popupWindow || popupWindow.isDestroyed()) {
      return;
    }

    const popupX = Math.round(
      workArea.x +
        workArea.width -
        NOTIFICATION_POPUP_WIDTH -
        NOTIFICATION_POPUP_MARGIN,
    );

    const popupY = Math.round(
      workArea.y +
        workArea.height -
        NOTIFICATION_POPUP_HEIGHT -
        NOTIFICATION_POPUP_MARGIN -
        index * (NOTIFICATION_POPUP_HEIGHT + NOTIFICATION_POPUP_GAP),
    );

    popupWindow.setPosition(popupX, popupY, true);
  });
}

/* =========================================================
   SHOW NEXT QUEUED NOTIFICATION
========================================================= */

function showNextQueuedNotification() {
  if (notificationPopupWindows.length >= MAX_VISIBLE_NOTIFICATION_POPUPS) {
    return;
  }

  if (notificationPopupQueue.length === 0) {
    return;
  }

  const nextNotification = notificationPopupQueue.shift();

  createNotificationPopupWindow(nextNotification);
}

/* -------------------------------------------------------
   TEMPORARY POPUP TEST
------------------------------------------------------- */

ipcMain.handle(
  "servicecall-test-notification-popup",

  async () => {
    showNotificationPopup({
      sys_id: "test-notification",

      type: "meeting started",

      type_display: "Meeting Started",

      title: "Meeting started",

      message: '"ServiceCall Test Meeting" has started.',

      meeting_sys_id: "",

      call_sys_id: "",
    });

    return {
      success: true,
    };
  },
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

  const trayMenu = Menu.buildFromTemplate([
    {
      label: "Open ServiceCall",
      click: () => {
        if (mainWindow && !mainWindow.isDestroyed()) {
          mainWindow.show();
          mainWindow.focus();
        }
      },
    },

    {
      type: "separator",
    },

    {
      label: "Quit ServiceCall",
      click: () => {
        isQuitting = true;

        stopHeartbeatLoop();
        stopCallbackServer();

        app.quit();
      },
    },
  ]);

  return trayMenu;
}

/* -------------------------------------------------------
   SAVE INSTANCE
------------------------------------------------------- */

ipcMain.handle(
  "servicecall-get-connection-status",

  async () => {
    const config = loadConfig();

    if (config.instanceUrl && config.accessToken && heartbeatTimer) {
      return {
        connected: true,

        message: "ServiceCall Desktop is connected.",
      };
    }

    return {
      connected: false,

      message: "Sign in to ServiceNow to connect ServiceCall Desktop.",
    };
  },
);

ipcMain.handle(
  "servicecall-save-instance",

  async (event, instanceUrl) => {
    const normalizedUrl = normalizeInstanceUrl(instanceUrl);

    if (!isValidServiceNowUrl(normalizedUrl)) {
      return {
        success: false,

        message:
          "Please enter a valid ServiceNow instance URL, for example https://dev12345.service-now.com",
      };
    }

    const config = loadConfig();

    config.instanceUrl = normalizedUrl;

    const configUrl =
      normalizedUrl + "/api/x_1806573_servic_0/servicecall_desktop_api/config";

    const configResponse = await fetch(configUrl, {
      method: "GET",

      headers: {
        Accept: "application/json",
      },
    });

    const configText = await configResponse.text();

    let serviceCallConfig;

    try {
      serviceCallConfig = JSON.parse(configText);
    } catch (error) {
      return {
        success: false,
        message:
          "The ServiceNow instance returned an invalid ServiceCall configuration.",
      };
    }

    const result = serviceCallConfig.result || serviceCallConfig;

    if (!configResponse.ok || !result.success) {
      return {
        success: false,
        message:
          result.message ||
          "Unable to retrieve ServiceCall configuration from this instance.",
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
          "ServiceCall Desktop is not fully configured on this ServiceNow instance.",
      };
    }

    config.oauthClientId = result.oauth_client_id;

    config.redirectUri = result.redirect_uri;

    config.oauthScope = result.oauth_scope;

    config.heartbeatPath = result.heartbeat_path;

    saveConfig(config);

    return {
      success: true,

      message: "ServiceNow instance saved successfully.",

      instanceUrl: normalizedUrl,
    };
  },
);

/* -------------------------------------------------------
   GET INSTANCE
------------------------------------------------------- */

ipcMain.handle(
  "servicecall-get-instance",

  async () => {
    const config = loadConfig();

    return {
      success: true,

      instanceUrl: config.instanceUrl || "",
    };
  },
);

/* -------------------------------------------------------
   START OAUTH LOGIN
------------------------------------------------------- */

ipcMain.handle(
  "servicecall-start-login",

  async () => {
    try {
      const config = loadConfig();

      if (!config.instanceUrl || !config.oauthClientId) {
        return {
          success: false,

          message: "Please save your ServiceNow instance first.",
        };
      }

      /*
               Start localhost listener BEFORE
               opening the browser.
            */

      await startCallbackServer();

      const codeVerifier = generateCodeVerifier();

      const codeChallenge = generateCodeChallenge(codeVerifier);

      const state = generateState();

      config.pkceCodeVerifier = codeVerifier;

      config.oauthState = state;

      saveConfig(config);

      const authorizeUrl = new URL(config.instanceUrl + "/oauth_auth.do");

      authorizeUrl.searchParams.set("response_type", "code");

      authorizeUrl.searchParams.set("client_id", config.oauthClientId);

      authorizeUrl.searchParams.set("redirect_uri", config.redirectUri);

      authorizeUrl.searchParams.set("scope", config.oauthScope);

      authorizeUrl.searchParams.set("code_challenge", codeChallenge);

      authorizeUrl.searchParams.set("code_challenge_method", "S256");

      authorizeUrl.searchParams.set("state", state);

      await shell.openExternal(authorizeUrl.toString());

      return {
        success: true,

        message: "ServiceNow sign-in opened in your browser.",
      };
    } catch (error) {
      stopCallbackServer();

      return {
        success: false,

        message: error.message,
      };
    }
  },
);

const gotSingleInstanceLock = app.requestSingleInstanceLock();

if (!gotSingleInstanceLock) {
  app.quit();
} else {
  app.on("second-instance", (event, commandLine) => {
    const deepLink = getServiceCallDeepLink(commandLine);

    if (deepLink) {
      handleServiceCallDeepLink(deepLink);
    }

    showMainWindow();
  });
}

app.whenReady().then(async () => {
  registerServiceCallProtocol();

  /*
   * Was ServiceCall launched by a
   * servicecall:// URL?
   */
  const startupDeepLink = getServiceCallDeepLink(process.argv);

  if (startupDeepLink) {
    handleServiceCallDeepLink(startupDeepLink);
  }

  await createWindow();

  /* -----------------------------------------
           DEVICE SLEEP / RESUME
        ----------------------------------------- */

  powerMonitor.on("suspend", async () => {
    console.log("ServiceCall detected device suspend.");

    isDeviceSuspended = true;

    /*
     * Tell ServiceNow this device
     * intentionally became Away.
     *
     * Keep the user's manual presence
     * untouched.
     */
    await updateDesktopState("away");
  });

  /* -----------------------------------------
           DEVICE SLEEP / RESUME
        ----------------------------------------- */

  powerMonitor.on("suspend", async () => {
    console.log("ServiceCall detected device suspend.");

    isDeviceSuspended = true;

    /*
     * Tell ServiceNow this device
     * intentionally became Away.
     *
     * Keep the user's manual presence
     * untouched.
     */
    await updateDesktopState("away");
  });

  powerMonitor.on("resume", async () => {
    console.log("ServiceCall detected device resume.");

    isDeviceSuspended = false;

    /*
     * A heartbeat is better than simply
     * changing the state to Connected:
     *
     * - marks registration Connected
     * - refreshes Last Seen
     * - proves ServiceNow is reachable
     */
    try {
      const result = await sendHeartbeatOnce();

      console.log("ServiceCall resume heartbeat:", result);
    } catch (error) {
      console.error("ServiceCall resume heartbeat failed:", error);
    }
  });

  app.on(
    "activate",

    () => {
      if (BrowserWindow.getAllWindows().length === 0) {
        createWindow();
      }
    },
  );
});

app.on(
  "window-all-closed",

  () => {
    // stopHeartbeatLoop();
    // stopCallbackServer();
    // if (
    //     process.platform !==
    //     'darwin'
    // ) {
    //     app.quit();
    // }
  },
);

app.on("before-quit", () => {
  isQuitting = true;

  stopHeartbeatLoop();
  stopIncomingCallLoop();
  stopCallbackServer();
  stopOutgoingCallLoop();
});

app.on("open-url", (event, url) => {
  event.preventDefault();

  handleServiceCallDeepLink(url);

  showMainWindow();
});

let callWindow = null;

function showCallWindow(mode, callData) {
  const callSysId = callData.callSysId || "";

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
      "Another ServiceCall window is already active:",
      activeCallWindowId,
    );

    return;
  }

  activeCallWindowId = callSysId;

  callWindow = new BrowserWindow({
    width: 440,
    height: 560,

    minWidth: 440,
    minHeight: 560,

    resizable: false,

    title: "ServiceCall",

    autoHideMenuBar: true,

    show: false,

    webPreferences: {
      preload: path.join(__dirname, "preload.js"),

      contextIsolation: true,

      nodeIntegration: false,
    },
  });

  callWindow.loadFile("call-window.html", {
    query: {
      mode: mode,

      callSysId: callSysId,

      callNumber: callData.callNumber || "",

      name: callData.name || "Unknown User",

      department: callData.department || "",

      isConference: callData.isConference ? "true" : "false",

      /*
       * MEETING CONTEXT
       *
       * Normal calls will simply receive
       * isMeeting=false and empty values.
       */
      isMeeting: callData.isMeeting ? "true" : "false",

      meetingSysId: callData.meetingSysId || "",

      meetingNumber: callData.meetingNumber || "",

      meetingTitle: callData.meetingTitle || "",
    },
  });

  callWindow.once("ready-to-show", () => {
    if (callWindow && !callWindow.isDestroyed()) {
      callWindow.show();
      callWindow.focus();
    }
  });

  callWindow.on("close", async (event) => {
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
      const result = await serviceCallApiRequest(
        "/call-status?call_sys_id=" + encodeURIComponent(callSysId),
        "GET",
      );

      const state = result.state || "";

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
      const participantStatus = result.participant_status || "";

      if (participantStatus === "ringing" || participantStatus === "invited") {
        /*
         * Original caller cancelling
         * an unanswered direct call.
         */
        if (mode === "calling" && state === "ringing") {
          await serviceCallApiRequest("/cancel-call", "POST", {
            call_sys_id: callSysId,
          });
        } else {
          /*
           * Direct incoming receiver
           * OR conference invite receiver.
           */
          await serviceCallApiRequest("/decline-call", "POST", {
            call_sys_id: callSysId,
          });
        }

        callWindowClosing = true;

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
      if (state === "connected" && participantStatus === "connected") {
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
        state === "completed" ||
        state === "cancelled" ||
        state === "declined" ||
        state === "failed"
      ) {
        callWindowClosing = true;

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
      console.error("ServiceCall window close handling failed:", error);

      /*
       * If ServiceNow cannot be reached,
       * do NOT accidentally terminate
       * a live call.
       */
      if (callWindow && !callWindow.isDestroyed()) {
        callWindow.hide();
      }
    }
  });

  callWindow.on("closed", () => {
    callWindow = null;

    callWindowClosing = false;

    activeCallWindowId = null;

    if (activeIncomingCallId === callSysId) {
      activeIncomingCallId = null;
    }

    if (activeOutgoingCallId === callSysId) {
      activeOutgoingCallId = null;
    }
  });
}

async function serviceCallApiRequest(
  pathName,
  method = "GET",
  body = null,
  allowRefresh = true,
) {
  let config = loadConfig();

  const validAccessToken = await ensureValidAccessToken();

  if (!config || !config.instanceUrl || !config.accessToken) {
    throw new Error("ServiceCall Desktop is not connected to ServiceNow.");
  }

  const url =
    config.instanceUrl.replace(/\/$/, "") +
    "/api/x_1806573_servic_0/servicecall_desktop_api" +
    pathName;

  const options = {
    method: method,

    headers: {
      Accept: "application/json",

      Authorization: "Bearer " + validAccessToken,
    },
  };

  if (body) {
    options.headers["Content-Type"] = "application/json";

    options.body = JSON.stringify(body);
  }

  let response = await fetch(url, options);

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
  if (response.status === 401 && allowRefresh) {
    console.log(
      "ServiceCall API authorization expired. Trying automatic renewal...",
    );

    try {
      const newAccessToken = await refreshAccessToken();

      /*
       * Retry the ORIGINAL request using
       * the newly issued access token.
       */
      options.headers["Authorization"] = "Bearer " + newAccessToken;

      response = await fetch(url, options);
    } catch (refreshError) {
      console.error(
        "Automatic ServiceCall authorization renewal failed:",
        refreshError.message,
      );

      const error = new Error(
        "Your ServiceCall authorization has expired. Please sign in to ServiceNow again.",
      );

      error.code = "AUTHENTICATION_REQUIRED";

      throw error;
    }
  }

  let data = {};

  try {
    data = await response.json();
  } catch (error) {
    data = {};
  }

  const result = data.result || data;

  /*
   * If we're STILL unauthorized after refreshing,
   * the long-lived authorization is no longer usable.
   */
  if (response.status === 401) {
    const error = new Error(
      "Your ServiceCall authorization has expired. Please sign in to ServiceNow again.",
    );

    error.code = "AUTHENTICATION_REQUIRED";

    throw error;
  }

  if (!response.ok) {
    const error = new Error(result.message || "ServiceCall request failed.");

    error.code = result.code || "SERVICECALL_API_ERROR";

    throw error;
  }

  return result;
}

async function serviceCallBinaryApiRequest(
  pathName,
  binaryData,
  allowRefresh = true,
) {
  const config = loadConfig();

  const validAccessToken = await ensureValidAccessToken();

  if (!config || !config.instanceUrl || !config.accessToken) {
    throw new Error("ServiceCall Desktop is not connected to ServiceNow.");
  }

  /*
   * Binary data must be a Buffer.
   */
  if (!Buffer.isBuffer(binaryData)) {
    const error = new Error("Attachment data must be binary.");

    error.code = "INVALID_BINARY_DATA";

    throw error;
  }

  if (binaryData.length === 0) {
    const error = new Error("Attachment file is empty.");

    error.code = "EMPTY_ATTACHMENT";

    throw error;
  }

  const url =
    config.instanceUrl.replace(/\/$/, "") +
    "/api/x_1806573_servic_0/servicecall_desktop_api" +
    pathName;

  const options = {
    method: "POST",

    headers: {
      Accept: "application/json",

      Authorization: "Bearer " + validAccessToken,

      "Content-Type": "application/octet-stream",

      /*
       * Gives ServiceNow the actual byte count
       * of the request body.
       */
      "Content-Length": String(binaryData.length),
    },

    body: binaryData,
  };

  let response = await fetch(url, options);

  /*
   * ---------------------------------------------
   * ACCESS TOKEN EXPIRED
   * ---------------------------------------------
   *
   * Same behavior as our existing JSON helper.
   */
  if (response.status === 401 && allowRefresh) {
    console.log(
      "ServiceCall binary API authorization expired. Trying automatic renewal...",
    );

    try {
      const newAccessToken = await refreshAccessToken();

      options.headers["Authorization"] = "Bearer " + newAccessToken;

      /*
       * A Buffer can safely be reused for
       * this single retry.
       */
      response = await fetch(url, options);
    } catch (refreshError) {
      console.error(
        "Automatic ServiceCall authorization renewal failed:",
        refreshError.message,
      );

      const error = new Error(
        "Your ServiceCall authorization has expired. Please sign in to ServiceNow again.",
      );

      error.code = "AUTHENTICATION_REQUIRED";

      throw error;
    }
  }

  /* -------------------------
     RESPONSE
  ------------------------- */

  let data = {};

  try {
    data = await response.json();
  } catch (error) {
    data = {};
  }

  const result = data.result || data;

  if (response.status === 401) {
    const error = new Error(
      "Your ServiceCall authorization has expired. Please sign in to ServiceNow again.",
    );

    error.code = "AUTHENTICATION_REQUIRED";

    throw error;
  }

  if (!response.ok) {
    console.error("ServiceCall binary API failed:", {
      status: response.status,
      statusText: response.statusText,
      url: url,
      response: result,
    });

    const error = new Error(
      result.message ||
        "ServiceCall binary request failed. HTTP " +
          response.status +
          " " +
          response.statusText,
    );

    error.code = result.code || "SERVICECALL_BINARY_API_ERROR";

    throw error;
  }
  return result;
}

async function serviceCallBinaryDownloadRequest(pathName, allowRefresh = true) {
  const config = loadConfig();

  const validAccessToken = await ensureValidAccessToken();

  if (!config || !config.instanceUrl || !config.accessToken) {
    throw new Error("ServiceCall Desktop is not connected to ServiceNow.");
  }

  const url =
    config.instanceUrl.replace(/\/$/, "") +
    "/api/x_1806573_servic_0/servicecall_desktop_api" +
    pathName;

  const options = {
    method: "GET",

    headers: {
      Accept: "*/*",

      Authorization: "Bearer " + validAccessToken,
    },
  };

  let response = await fetch(url, options);

  /*
   * ---------------------------------------------
   * ACCESS TOKEN EXPIRED
   * ---------------------------------------------
   */

  if (response.status === 401 && allowRefresh) {
    console.log(
      "ServiceCall binary download authorization expired. Trying automatic renewal...",
    );

    try {
      const newAccessToken = await refreshAccessToken();

      options.headers["Authorization"] = "Bearer " + newAccessToken;

      response = await fetch(url, options);
    } catch (refreshError) {
      console.error(
        "Automatic ServiceCall authorization renewal failed:",
        refreshError.message,
      );

      const error = new Error(
        "Your ServiceCall authorization has expired. Please sign in to ServiceNow again.",
      );

      error.code = "AUTHENTICATION_REQUIRED";

      throw error;
    }
  }

  /*
   * ---------------------------------------------
   * AUTHENTICATION FAILURE
   * ---------------------------------------------
   */

  if (response.status === 401) {
    const error = new Error(
      "Your ServiceCall authorization has expired. Please sign in to ServiceNow again.",
    );

    error.code = "AUTHENTICATION_REQUIRED";

    throw error;
  }

  /*
   * ---------------------------------------------
   * ERROR RESPONSE
   * ---------------------------------------------
   */

  if (!response.ok) {
    let result = {};

    try {
      const data = await response.json();

      result = data.result || data;
    } catch (error) {
      result = {};
    }

    console.error("ServiceCall binary download failed:", {
      status: response.status,
      statusText: response.statusText,
      url: url,
      response: result,
    });

    const error = new Error(
      result.message ||
        "ServiceCall attachment download failed. HTTP " +
          response.status +
          " " +
          response.statusText,
    );

    error.code = result.code || "SERVICECALL_BINARY_DOWNLOAD_ERROR";

    throw error;
  }

  /*
   * ---------------------------------------------
   * BINARY RESPONSE
   * ---------------------------------------------
   */

  const arrayBuffer = await response.arrayBuffer();

  const buffer = Buffer.from(arrayBuffer);

  if (buffer.length === 0) {
    const error = new Error("Downloaded attachment is empty.");

    error.code = "EMPTY_DOWNLOADED_ATTACHMENT";

    throw error;
  }

  const contentType =
    response.headers.get("content-type") || "application/octet-stream";

  const contentDisposition = response.headers.get("content-disposition") || "";

  return {
    buffer,
    contentType,
    contentDisposition,
  };
}

async function getCurrentServiceCallUser() {
  const result = await serviceCallApiRequest("/me", "GET");

  if (!result || result.success !== true) {
    throw new Error(
      result?.message || "Unable to retrieve the current ServiceCall user.",
    );
  }

  return result;
}

async function resolveStartupAuthentication() {
  const config = loadConfig();

  /*
   * -----------------------------------------
   * 1. INSTANCE NOT CONFIGURED
   * -----------------------------------------
   */

  if (!config || !config.instanceUrl) {
    return {
      authenticated: false,
      authorized: false,
      state: "instance_required",
    };
  }

  /*
   * -----------------------------------------
   * 2. NO SAVED SESSION
   * -----------------------------------------
   */

  if (!config.accessToken && !config.refreshToken) {
    return {
      authenticated: false,
      authorized: false,
      state: "login_required",
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

    if (!config.accessToken && config.refreshToken) {
      await refreshAccessToken();
    } else {
      await ensureValidAccessToken();
    }

    /*
     * -------------------------------------
     * 4. CHECK SERVICENOW IDENTITY + ROLE
     * -------------------------------------
     */

    const currentUser = await getCurrentServiceCallUser();

    const authorization = currentUser?.authorization || {};

    const user = currentUser?.user || null;

    /*
     * -----------------------------------------
     * STORE CURRENT ACCOUNT IDENTITY
     * -----------------------------------------
     */

    currentServiceCallUser = user;

    currentServiceCallAuthorization = authorization;

    /*
     * -------------------------------------
     * 5. AUTHENTICATED BUT NOT AUTHORIZED
     * -------------------------------------
     */

    if (authorization.allowed !== true) {
      return {
        authenticated: true,
        authorized: false,
        state: "access_denied",
        user: user,
        authorization: authorization,
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
      state: "ready",
      user: user,
      authorization: authorization,
    };
  } catch (error) {
    console.error("ServiceCall startup authentication failed:", error);

    return {
      authenticated: false,
      authorized: false,
      state: "login_required",
      error: error?.message || "Authentication could not be restored.",
    };
  }
}

ipcMain.handle("servicecall-accept-call", async (event, callSysId) => {
  return await serviceCallApiRequest("/accept-call", "POST", {
    call_sys_id: callSysId,
  });
});

ipcMain.handle("servicecall-decline-call", async (event, callSysId) => {
  return await serviceCallApiRequest("/decline-call", "POST", {
    call_sys_id: callSysId,
  });
});

ipcMain.handle("servicecall-cancel-call", async (event, callSysId) => {
  return await serviceCallApiRequest("/cancel-call", "POST", {
    call_sys_id: callSysId,
  });
});

ipcMain.handle("servicecall-end-call", async (event, callSysId) => {
  return await serviceCallApiRequest("/end-call", "POST", {
    call_sys_id: callSysId,
  });
});

ipcMain.handle("servicecall-get-call-status", async (event, callSysId) => {
  return await serviceCallApiRequest(
    "/call-status?call_sys_id=" + encodeURIComponent(callSysId),
    "GET",
  );
});

ipcMain.handle(
  "servicecall-download-recording",

  async (event, recordingSysId) => {
    if (!recordingSysId) {
      return {
        success: false,
        code: "RECORDING_ID_REQUIRED",
        message: "Recording ID was not provided.",
      };
    }

    try {
      /*
       * -----------------------------------------
       * 1. ASK SERVICENOW IF THIS USER
       *    MAY DOWNLOAD THIS RECORDING
       * -----------------------------------------
       */

      const downloadInfo = await serviceCallApiRequest(
        "/recording-download" +
          "?recording_sys_id=" +
          encodeURIComponent(recordingSysId),
        "GET",
      );

      if (
        !downloadInfo ||
        downloadInfo.success !== true ||
        !downloadInfo.attachment_sys_id
      ) {
        throw new Error(
          downloadInfo && downloadInfo.message
            ? downloadInfo.message
            : "Recording download was not authorized.",
        );
      }

      /*
       * -----------------------------------------
       * 2. GET CURRENT INSTANCE + OAUTH TOKEN
       * -----------------------------------------
       */

      const config = loadConfig();

      if (!config || !config.instanceUrl) {
        throw new Error("ServiceCall Desktop is not connected to ServiceNow.");
      }

      let accessToken = await ensureValidAccessToken();

      const attachmentUrl =
        config.instanceUrl.replace(/\/$/, "") +
        "/api/now/attachment/" +
        encodeURIComponent(downloadInfo.attachment_sys_id) +
        "/file";

      async function performDownload(token) {
        return await fetch(attachmentUrl, {
          method: "GET",

          headers: {
            Authorization: "Bearer " + token,
          },
        });
      }

      /*
       * -----------------------------------------
       * 3. DOWNLOAD ATTACHMENT
       * -----------------------------------------
       */

      let response = await performDownload(accessToken);

      /*
       * Access token could expire between
       * authorization and attachment download.
       */
      if (response.status === 401 || response.status === 403) {
        accessToken = await refreshAccessToken();

        response = await performDownload(accessToken);
      }

      if (!response.ok) {
        throw new Error(
          "Unable to download the recording attachment from ServiceNow.",
        );
      }

      const arrayBuffer = await response.arrayBuffer();

      const recordingBuffer = Buffer.from(arrayBuffer);

      if (recordingBuffer.length <= 0) {
        throw new Error("The downloaded recording file is empty.");
      }

      /*
       * -----------------------------------------
       * 4. DETERMINE SAFE FILE NAME
       * -----------------------------------------
       */

      const format = String(downloadInfo.format || "mp3")
        .trim()
        .toLowerCase();

      const extension = format === "mp4" ? ".mp4" : ".mp3";

      let fileName = String(
        downloadInfo.file_name ||
          "servicecall-recording-" + recordingSysId + extension,
      ).replace(/[<>:"/\\|?*\x00-\x1F]/g, "_");

      if (!fileName.toLowerCase().endsWith(extension)) {
        fileName += extension;
      }

      /*
       * -----------------------------------------
       * 5. WINDOWS SAVE AS DIALOG
       * -----------------------------------------
       */

      const saveResult = await dialog.showSaveDialog({
        title: "Save ServiceCall Recording",

        defaultPath: path.join(app.getPath("downloads"), fileName),

        filters: [
          {
            name: format === "mp4" ? "MP4 Video" : "MP3 Audio",

            extensions: [format === "mp4" ? "mp4" : "mp3"],
          },
        ],
      });

      /*
       * User pressed Cancel.
       *
       * This is NOT an error.
       */
      if (saveResult.canceled || !saveResult.filePath) {
        return {
          success: false,
          code: "DOWNLOAD_CANCELLED",
          message: "Recording download was cancelled.",
        };
      }

      /*
       * -----------------------------------------
       * 6. SAVE FILE LOCALLY
       * -----------------------------------------
       */

      await fs.promises.writeFile(saveResult.filePath, recordingBuffer);

      console.log(
        "ServiceCall recording downloaded successfully.",
        recordingSysId,
      );

      return {
        success: true,
        code: "RECORDING_DOWNLOADED",
        recording_sys_id: recordingSysId,
        file_name: path.basename(saveResult.filePath),
        file_path: saveResult.filePath,
        format: format,
        file_size: recordingBuffer.length,
      };
    } catch (error) {
      console.error("ServiceCall recording download failed:", error.message);

      return {
        success: false,
        code: error.code || "RECORDING_DOWNLOAD_FAILED",
        message: error.message || "Unable to download recording.",
      };
    }
  },
);

async function checkOutgoingCallOnce() {
  try {
    const result = await serviceCallApiRequest("/outgoing-call", "GET");

    if (result && result.success && result.outgoing_call) {
      const outgoingCallId = result.call_sys_id;

      if (
        outgoingCallId &&
        !intentionallyLeftCallIds.has(outgoingCallId) &&
        outgoingCallId !== activeOutgoingCallId
      ) {
        activeOutgoingCallId = outgoingCallId;

        console.log("Outgoing ServiceCall:", result);

        showCallWindow(
          result.state === "connected" ? "connected" : "calling",

          //console.warn('DEBUG: GENERIC OUTGOING ROLLER OPENING CALL WINDOW'),
          {
            callSysId: result.call_sys_id,

            callNumber: result.call_number,

            name: result.target_user_name || "Unknown User",

            department: result.target_department || "",
          },
        );
      }

      return;
    }

    /*
     * No outgoing ringing/connected call.
     */
    activeOutgoingCallId = null;
  } catch (error) {
    console.error("Outgoing call check failed:", error);

    if (error.code === "AUTHENTICATION_REQUIRED") {
      stopOutgoingCallLoop();

      sendAuthStatus(
        "authentication_required",
        "Your ServiceCall authorization has expired. Please sign in to ServiceNow again.",
      );
    }
  }
}

function startOutgoingCallLoop() {
  stopOutgoingCallLoop();

  checkOutgoingCallOnce();

  outgoingCallTimer = setInterval(checkOutgoingCallOnce, 3000);
}

function stopOutgoingCallLoop() {
  if (outgoingCallTimer) {
    clearInterval(outgoingCallTimer);

    outgoingCallTimer = null;
  }
}

ipcMain.handle("servicecall-open-active-call", async () => {
  if (callWindow && !callWindow.isDestroyed()) {
    if (callWindow.isMinimized()) {
      callWindow.restore();
    }

    callWindow.show();
    callWindow.focus();

    return {
      success: true,
      active_call: true,
    };
  }

  return {
    success: true,
    active_call: false,
    message: "No active call window is currently available.",
  };
});

/* -------------------------------------------------------
   DYNAMIC MEDIA CREDENTIALS
------------------------------------------------------- */

ipcMain.handle(
  "servicecall-get-media-credentials",

  async (event, callSysId) => {
    if (!callSysId) {
      return {
        success: false,
        code: "CALL_ID_REQUIRED",
        message: "Call ID was not provided.",
      };
    }

    try {
      const result = await serviceCallApiRequest(
        "/media-credentials?call_sys_id=" + encodeURIComponent(callSysId),
        "GET",
      );

      return result;
    } catch (error) {
      console.error(
        "Unable to get ServiceCall media credentials:",
        error.message,
      );

      return {
        success: false,

        code: error.code || "MEDIA_CREDENTIALS_FAILED",

        message: error.message || "Unable to obtain media credentials.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-invite-participant",

  async (event, callSysId, userSysId) => {
    if (!callSysId || !userSysId) {
      return {
        success: false,
        code: "INVITE_DATA_REQUIRED",
        message: "Call ID and user ID are required.",
      };
    }

    try {
      const result = await serviceCallApiRequest(
        "/invite-participant",
        "POST",
        {
          call_sys_id: callSysId,

          user_sys_id: userSysId,
        },
      );

      return result;
    } catch (error) {
      console.error("Unable to invite ServiceCall participant:", error.message);

      return {
        success: false,
        code: error.code || "INVITE_PARTICIPANT_FAILED",
        message: error.message || "Unable to invite participant.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-search-users",

  async (event, searchText) => {
    const search = String(searchText || "").trim();

    if (search.length < 2) {
      return {
        success: true,
        users: [],
      };
    }

    try {
      return await serviceCallApiRequest(
        "/users?search=" + encodeURIComponent(search),
        "GET",
      );
    } catch (error) {
      console.error("Unable to search ServiceCall users:", error.message);

      return {
        success: false,

        code: error.code || "USER_SEARCH_FAILED",

        message: error.message || "Unable to search users.",

        users: [],
      };
    }
  },
);

/* =====================================================
   SERVICECALL CHAT - GET CONVERSATIONS
===================================================== */

ipcMain.handle(
  "servicecall-get-conversations",

  async () => {
    try {
      const result = await serviceCallApiRequest("/conversations", "GET");

      return result;
    } catch (error) {
      console.error("Unable to get ServiceCall conversations:", error.message);

      return {
        success: false,

        code: error.code || "GET_CONVERSATIONS_FAILED",

        message: error.message || "Unable to retrieve conversations.",

        conversations: [],
      };
    }
  },
);

ipcMain.handle(
  "servicecall-leave-call",

  async (event, callSysId) => {
    if (!callSysId) {
      return {
        success: false,
        code: "CALL_ID_REQUIRED",
        message: "Call ID was not provided.",
      };
    }

    try {
      return await serviceCallApiRequest("/leave-call", "POST", {
        call_sys_id: callSysId,
      });
    } catch (error) {
      console.error("Unable to leave ServiceCall:", error.message);

      return {
        success: false,

        code: error.code || "LEAVE_CALL_FAILED",

        message: error.message || "Unable to leave the call.",
      };
    }
  },
);

/* -------------------------------------------------------
   UPLOAD RECORDING ATTACHMENT
------------------------------------------------------- */

async function uploadRecordingAttachment(
  recordingSysId,
  fileData,
  fileName,
  format,
) {
  if (!recordingSysId) {
    throw new Error("Recording ID was not provided.");
  }

  if (!fileData) {
    throw new Error("Recording file data was not provided.");
  }

  const normalizedFormat = String(format || "")
    .trim()
    .toLowerCase();

  if (normalizedFormat !== "mp3" && normalizedFormat !== "mp4") {
    throw new Error("Recording format must be mp3 or mp4.");
  }

  const expectedExtension = "." + normalizedFormat;

  let safeFileName = String(fileName || "")
    .trim()
    .replace(/[^a-zA-Z0-9._-]/g, "_");

  if (!safeFileName) {
    safeFileName = "servicecall-recording-" + Date.now() + expectedExtension;
  }

  /*
   * Ensure the filename agrees with the
   * recording format.
   */
  if (!safeFileName.toLowerCase().endsWith(expectedExtension)) {
    safeFileName += expectedExtension;
  }

  const config = loadConfig();

  if (!config || !config.instanceUrl) {
    throw new Error("ServiceCall Desktop is not connected to ServiceNow.");
  }

  /*
   * Get a valid OAuth access token.
   *
   * The renderer never receives this token.
   */
  let accessToken = await ensureValidAccessToken();

  /*
   * IPC can give us a Uint8Array rather
   * than a Node Buffer.
   */
  const recordingBuffer = Buffer.isBuffer(fileData)
    ? fileData
    : Buffer.from(fileData);

  if (recordingBuffer.length <= 0) {
    throw new Error("Recording file is empty.");
  }

  const contentType = normalizedFormat === "mp3" ? "audio/mpeg" : "video/mp4";

  const tableName = "x_1806573_servic_0_servicecall_recording";

  const uploadUrl =
    config.instanceUrl.replace(/\/$/, "") +
    "/api/now/attachment/file" +
    "?table_name=" +
    encodeURIComponent(tableName) +
    "&table_sys_id=" +
    encodeURIComponent(recordingSysId) +
    "&file_name=" +
    encodeURIComponent(safeFileName);

  async function performUpload(token) {
    return await fetch(uploadUrl, {
      method: "POST",

      headers: {
        Authorization: "Bearer " + token,

        Accept: "application/json",

        "Content-Type": contentType,
      },

      /*
       * IMPORTANT:
       *
       * Raw binary.
       * No JSON.
       * No Base64.
       */
      body: recordingBuffer,
    });
  }

  let response = await performUpload(accessToken);

  /*
   * If OAuth expired between obtaining the
   * token and uploading the file, refresh
   * once and retry the same binary upload.
   */
  if (response.status === 401 || response.status === 403) {
    console.log(
      "Recording upload authorization expired. Attempting automatic renewal...",
    );

    try {
      accessToken = await refreshAccessToken();

      response = await performUpload(accessToken);
    } catch (refreshError) {
      const authError = new Error(
        "Your ServiceCall authorization has expired. Please sign in again.",
      );

      authError.code = "AUTHENTICATION_REQUIRED";

      throw authError;
    }
  }

  const responseText = await response.text();

  let data = {};

  try {
    data = responseText ? JSON.parse(responseText) : {};
  } catch (error) {
    throw new Error(
      "ServiceNow returned an invalid attachment upload response.",
    );
  }

  const result = data.result || data;

  console.log(
    "ServiceCall Attachment API response:",
    "HTTP",
    response.status,
    data,
  );

  if (!response.ok) {
    const uploadError = new Error(
      (result &&
        result.error &&
        (result.error.message || result.error.detail)) ||
        (result && result.message) ||
        (data && data.error && (data.error.message || data.error.detail)) ||
        "ServiceNow recording upload failed.",
    );

    uploadError.code = "RECORDING_UPLOAD_FAILED";

    throw uploadError;
  }

  if (!result || !result.sys_id) {
    throw new Error("ServiceNow did not return an attachment ID.");
  }

  /*
   * Extra verification using the metadata
   * ServiceNow returned from Attachment API.
   */
  if (result.table_sys_id && result.table_sys_id !== recordingSysId) {
    throw new Error(
      "Uploaded attachment was associated with the wrong recording.",
    );
  }

  if (result.table_name && result.table_name !== tableName) {
    throw new Error("Uploaded attachment was associated with the wrong table.");
  }

  console.log(
    "ServiceCall recording uploaded successfully.",
    "Attachment:",
    result.sys_id,
    "Size:",
    result.size_bytes || recordingBuffer.length,
  );

  /*
   * Never return the OAuth token.
   */
  return {
    success: true,

    attachment_sys_id: result.sys_id,

    file_name: result.file_name || safeFileName,

    file_size: Number(result.size_bytes || recordingBuffer.length),

    content_type: result.content_type || contentType,
  };
}

ipcMain.handle(
  "servicecall-upload-recording",

  async (event, recordingSysId, fileData, fileName, format) => {
    try {
      return await uploadRecordingAttachment(
        recordingSysId,
        fileData,
        fileName,
        format,
      );
    } catch (error) {
      console.error("Unable to upload ServiceCall recording:", error.message);

      return {
        success: false,

        code: error.code || "RECORDING_UPLOAD_FAILED",

        message: error.message || "Unable to upload recording.",
      };
    }
  },
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
  "servicecall-start-recording",

  async (event, callSysId) => {
    if (!callSysId) {
      return {
        success: false,
        code: "CALL_ID_REQUIRED",
        message: "Call ID was not provided.",
      };
    }

    try {
      return await serviceCallApiRequest("/start-recording", "POST", {
        call_sys_id: callSysId,
      });
    } catch (error) {
      console.error("Unable to start ServiceCall recording:", error.message);

      return {
        success: false,

        code: error.code || "START_RECORDING_FAILED",

        message: error.message || "Unable to start recording.",
      };
    }
  },
);

/*
 * Tell ServiceNow that media capture has stopped
 * and the recording is now being processed.
 */
ipcMain.handle(
  "servicecall-finish-recording",

  async (event, recordingSysId) => {
    if (!recordingSysId) {
      return {
        success: false,
        code: "RECORDING_ID_REQUIRED",
        message: "Recording ID was not provided.",
      };
    }

    try {
      return await serviceCallApiRequest("/finish-recording", "POST", {
        recording_sys_id: recordingSysId,
      });
    } catch (error) {
      console.error("Unable to finish ServiceCall recording:", error.message);

      return {
        success: false,

        code: error.code || "FINISH_RECORDING_FAILED",

        message: error.message || "Unable to finish recording.",
      };
    }
  },
);

/*
 * After the binary file has been successfully
 * uploaded to sys_attachment, this finalizes the
 * ServiceCall Recording record.
 */
ipcMain.handle(
  "servicecall-complete-recording",

  async (event, recordingSysId, attachmentSysId, format) => {
    if (!recordingSysId || !attachmentSysId || !format) {
      return {
        success: false,
        code: "RECORDING_DATA_REQUIRED",
        message: "Recording ID, attachment ID and format are required.",
      };
    }

    try {
      return await serviceCallApiRequest("/complete-recording", "POST", {
        recording_sys_id: recordingSysId,

        attachment_sys_id: attachmentSysId,

        format: format,
      });
    } catch (error) {
      console.error("Unable to complete ServiceCall recording:", error.message);

      return {
        success: false,

        code: error.code || "COMPLETE_RECORDING_FAILED",

        message: error.message || "Unable to complete recording.",
      };
    }
  },
);

/* -------------------------------------------------------
   FINALIZE VOICE RECORDING
------------------------------------------------------- */

ipcMain.handle(
  "servicecall-finalize-voice-recording",

  async (event, recordingSysId, webmData) => {
    if (!recordingSysId) {
      return {
        success: false,
        code: "RECORDING_ID_REQUIRED",
        message: "Recording ID was not provided.",
      };
    }

    if (!webmData) {
      return {
        success: false,
        code: "RECORDING_DATA_REQUIRED",
        message: "Recording data was not provided.",
      };
    }

    try {
      /*
       * IPC structured cloning commonly
       * gives the main process a Uint8Array.
       */
      const webmBuffer = Buffer.isBuffer(webmData)
        ? webmData
        : Buffer.from(webmData);

      if (webmBuffer.length <= 0) {
        throw new Error("Recording data is empty.");
      }

      console.log(
        "ServiceCall voice recording received.",
        "WebM size:",
        webmBuffer.length,
      );

      /* -----------------------------------------
               1. WEBM -> REAL MP3
            ----------------------------------------- */

      const converted = await convertWebmToMp3(webmBuffer);

      if (!converted || !converted.success || !converted.buffer) {
        throw new Error("Unable to convert ServiceCall recording to MP3.");
      }

      /* -----------------------------------------
               2. MARK RECORDING PROCESSING
            ----------------------------------------- */

      const finishResult = await serviceCallApiRequest(
        "/finish-recording",
        "POST",
        {
          recording_sys_id: recordingSysId,
        },
      );

      if (!finishResult || finishResult.success !== true) {
        throw new Error(
          finishResult && finishResult.message
            ? finishResult.message
            : "Unable to finish ServiceCall recording.",
        );
      }

      /* -----------------------------------------
               3. UPLOAD FINAL MP3
            ----------------------------------------- */

      const fileName = "servicecall-recording-" + recordingSysId + ".mp3";

      const uploadResult = await uploadRecordingAttachment(
        recordingSysId,
        converted.buffer,
        fileName,
        "mp3",
      );

      console.log("ServiceCall attachment upload result:", uploadResult);

      if (
        !uploadResult ||
        !uploadResult.success ||
        !uploadResult.attachment_sys_id
      ) {
        throw new Error(
          uploadResult && uploadResult.message
            ? uploadResult.message
            : "Unable to upload ServiceCall recording.",
        );
      }

      /* -----------------------------------------
               4. COMPLETE RECORDING
            ----------------------------------------- */

      const completeResult = await serviceCallApiRequest(
        "/complete-recording",
        "POST",
        {
          recording_sys_id: recordingSysId,

          attachment_sys_id: uploadResult.attachment_sys_id,

          format: "mp3",
        },
      );

      if (!completeResult || completeResult.success !== true) {
        throw new Error(
          completeResult && completeResult.message
            ? completeResult.message
            : "Unable to complete ServiceCall recording.",
        );
      }

      console.log(
        "ServiceCall voice recording finalized successfully.",
        recordingSysId,
      );

      return {
        success: true,

        code: "VOICE_RECORDING_AVAILABLE",

        recording_sys_id: recordingSysId,

        attachment_sys_id: uploadResult.attachment_sys_id,

        format: "mp3",

        file_name: uploadResult.file_name,

        file_size: uploadResult.file_size,

        status: completeResult.status || "available",

        expires_at: completeResult.expires_at || "",
      };
    } catch (error) {
      console.error("ServiceCall voice recording finalization failed:", error);

      return {
        success: false,

        code: error.code || "VOICE_RECORDING_FINALIZATION_FAILED",

        message: error.message || "Unable to finalize ServiceCall recording.",
      };
    }
  },
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
  "servicecall-finalize-screen-recording",

  async (event, recordingSysId, webmData) => {
    if (!recordingSysId) {
      return {
        success: false,

        code: "RECORDING_ID_REQUIRED",

        message: "Recording ID was not provided.",
      };
    }

    if (!webmData) {
      return {
        success: false,

        code: "RECORDING_DATA_REQUIRED",

        message: "Recording data was not provided.",
      };
    }

    try {
      /*
       * IPC structured cloning normally
       * gives the main process a Uint8Array.
       */
      const webmBuffer = Buffer.isBuffer(webmData)
        ? webmData
        : Buffer.from(webmData);

      if (webmBuffer.length <= 0) {
        throw new Error("Recording data is empty.");
      }

      console.log(
        "ServiceCall screen recording received.",
        "WebM size:",
        webmBuffer.length,
      );

      /* -----------------------------------------
               1. WEBM -> REAL MP4
            ----------------------------------------- */

      const converted = await convertWebmToMp4(webmBuffer);

      if (!converted || !converted.success || !converted.buffer) {
        throw new Error("Unable to convert ServiceCall recording to MP4.");
      }

      console.log(
        "ServiceCall screen recording converted.",
        "MP4 size:",
        converted.size,
      );

      /* -----------------------------------------
               2. MARK RECORDING PROCESSING
            ----------------------------------------- */

      const finishResult = await serviceCallApiRequest(
        "/finish-recording",
        "POST",
        {
          recording_sys_id: recordingSysId,
        },
      );

      if (!finishResult || finishResult.success !== true) {
        throw new Error(
          finishResult && finishResult.message
            ? finishResult.message
            : "Unable to finish ServiceCall recording.",
        );
      }

      /* -----------------------------------------
               3. UPLOAD FINAL MP4
            ----------------------------------------- */

      const fileName = "servicecall-recording-" + recordingSysId + ".mp4";

      const uploadResult = await uploadRecordingAttachment(
        recordingSysId,
        converted.buffer,
        fileName,
        "mp4",
      );

      if (
        !uploadResult ||
        !uploadResult.success ||
        !uploadResult.attachment_sys_id
      ) {
        throw new Error(
          uploadResult && uploadResult.message
            ? uploadResult.message
            : "Unable to upload ServiceCall screen recording.",
        );
      }

      /* -----------------------------------------
               4. COMPLETE RECORDING
            ----------------------------------------- */

      const completeResult = await serviceCallApiRequest(
        "/complete-recording",
        "POST",
        {
          recording_sys_id: recordingSysId,

          attachment_sys_id: uploadResult.attachment_sys_id,

          format: "mp4",
        },
      );

      if (!completeResult || completeResult.success !== true) {
        throw new Error(
          completeResult && completeResult.message
            ? completeResult.message
            : "Unable to complete ServiceCall screen recording.",
        );
      }

      console.log(
        "ServiceCall screen recording finalized successfully.",
        recordingSysId,
      );

      return {
        success: true,

        code: "SCREEN_RECORDING_AVAILABLE",

        recording_sys_id: recordingSysId,

        attachment_sys_id: uploadResult.attachment_sys_id,

        format: "mp4",

        file_name: uploadResult.file_name,

        file_size: uploadResult.file_size,

        status: completeResult.status || "available",

        expires_at: completeResult.expires_at || "",
      };
    } catch (error) {
      console.error("ServiceCall screen recording finalization failed:", error);

      return {
        success: false,

        code: error.code || "SCREEN_RECORDING_FINALIZATION_FAILED",

        message:
          error.message || "Unable to finalize ServiceCall screen recording.",
      };
    }
  },
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
  "servicecall-get-screen-sources",

  async () => {
    try {
      const sources = await desktopCapturer.getSources({
        types: ["screen", "window"],

        thumbnailSize: {
          width: 320,
          height: 180,
        },

        fetchWindowIcons: true,
      });

      const safeSources = sources.map((source) => {
        return {
          id: source.id,

          name: source.name,

          thumbnail:
            source.thumbnail && !source.thumbnail.isEmpty()
              ? source.thumbnail.toDataURL()
              : "",

          appIcon:
            source.appIcon && !source.appIcon.isEmpty()
              ? source.appIcon.toDataURL()
              : "",
        };
      });

      console.log("ServiceCall screen sources available:", safeSources.length);

      return {
        success: true,

        sources: safeSources,
      };
    } catch (error) {
      console.error("Unable to retrieve ServiceCall screen sources:", error);

      return {
        success: false,

        code: "SCREEN_SOURCE_FAILED",

        message: error.message || "Unable to retrieve screens and windows.",

        sources: [],
      };
    }
  },
);

/* -------------------------------------------------------
   SERVICECALL CALL WINDOW LAYOUT
------------------------------------------------------- */

ipcMain.handle(
  "servicecall-set-call-window-layout",

  async (event, layout) => {
    try {
      const senderWindow = BrowserWindow.fromWebContents(event.sender);

      if (
        !senderWindow ||
        senderWindow.isDestroyed() ||
        senderWindow !== callWindow
      ) {
        return {
          success: false,
          message: "ServiceCall call window is unavailable.",
        };
      }

      if (layout === "screen") {
        /*
         * Allow the existing call window
         * to become a larger screen-share
         * experience.
         */
        senderWindow.setResizable(true);

        senderWindow.setMinimumSize(760, 620);

        senderWindow.setSize(1000, 760, true);

        senderWindow.center();

        return {
          success: true,
          layout: "screen",
        };
      }

      /*
       * Return to compact voice-call mode.
       */
      senderWindow.setMinimumSize(440, 560);

      senderWindow.setSize(440, 560, true);

      senderWindow.setResizable(false);

      senderWindow.center();

      return {
        success: true,
        layout: "compact",
      };
    } catch (error) {
      console.error("Unable to change ServiceCall call window layout:", error);

      return {
        success: false,
        message: error.message || "Unable to change call window layout.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-get-recording-history",

  async () => {
    try {
      const result = await serviceCallApiRequest("/recording-history", "GET");

      return {
        success: true,
        count: Number(result.count || 0),
        recordings: Array.isArray(result.recordings) ? result.recordings : [],
      };
    } catch (error) {
      console.error(
        "Unable to load ServiceCall recording history:",
        error.message,
      );

      return {
        success: false,
        code: error.code || "RECORDING_HISTORY_FAILED",
        message: error.message || "Unable to load recording history.",
        count: 0,
        recordings: [],
      };
    }
  },
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
  "servicecall-get-my-meetings",

  async (event, options = {}) => {
    try {
      let page = parseInt(options.page, 10) || 1;

      if (page < 1) {
        page = 1;
      }

      const search = String(options.search || "").trim();

      const status = String(options.status || "")
        .toLowerCase()
        .trim();

      const query = new URLSearchParams({
        page: String(page),

        page_size: "7",

        search: search,

        status: status,
      });

      const result = await serviceCallApiRequest(
        "/my-meetings?" + query.toString(),
        "GET",
      );

      return result;
    } catch (error) {
      console.error("Unable to get ServiceCall meetings:", error.message);

      return {
        success: false,

        code: error.code || "GET_MEETINGS_FAILED",

        message: error.message || "Unable to retrieve meetings.",

        count: 0,

        total_count: 0,

        current_page: 1,

        page_size: 7,

        total_pages: 0,

        has_previous: false,

        has_next: false,

        search: "",

        status: "",

        meetings: [],
      };
    }
  },
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
  "servicecall-create-meeting",

  async (event, meetingData) => {
    try {
      if (
        !meetingData ||
        !meetingData.title ||
        !meetingData.scheduled_start ||
        !meetingData.scheduled_end
      ) {
        return {
          success: false,

          code: "MEETING_DATA_REQUIRED",

          message: "Title, start time and end time are required.",
        };
      }

      const payload = {
        title: String(meetingData.title).trim(),

        description: String(meetingData.description || "").trim(),

        scheduled_start: meetingData.scheduled_start,

        scheduled_end: meetingData.scheduled_end,

        timezone: String(meetingData.timezone || "").trim(),

        participants: Array.isArray(meetingData.participants)
          ? meetingData.participants
          : [],
      };

      const result = await serviceCallApiRequest(
        "/create-meeting",
        "POST",
        payload,
      );

      return result;
    } catch (error) {
      console.error("Unable to create ServiceCall meeting:", error.message);

      return {
        success: false,

        code: error.code || "CREATE_MEETING_FAILED",

        message: error.message || "Unable to create meeting.",
      };
    }
  },
);

/* =====================================================
   UPDATE MEETING
===================================================== */

ipcMain.handle(
  "servicecall-update-meeting",

  async (event, meetingSysId, meetingData) => {
    try {
      /*
       * Meeting sys_id is required.
       */
      if (!meetingSysId) {
        return {
          success: false,
          code: "MEETING_SYS_ID_REQUIRED",
          message: "Meeting sys_id is required.",
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
          code: "MEETING_DATA_REQUIRED",
          message: "Title, start time and end time are required.",
        };
      }

      /*
       * Build the payload sent to
       * ServiceNow.
       */
      const payload = {
        meeting_sys_id: String(meetingSysId).trim(),

        title: String(meetingData.title).trim(),

        description: String(meetingData.description || "").trim(),

        scheduled_start: meetingData.scheduled_start,

        scheduled_end: meetingData.scheduled_end,

        timezone: String(meetingData.timezone || "").trim(),

        participants: Array.isArray(meetingData.participants)
          ? meetingData.participants
          : [],
      };

      console.log("Updating ServiceCall meeting:", payload);

      /*
       * Send the update to ServiceNow.
       *
       * We will create this REST resource
       * in the next step.
       */
      const result = await serviceCallApiRequest(
        "/update-meeting",
        "POST",
        payload,
      );

      return result;
    } catch (error) {
      console.error("Unable to update ServiceCall meeting:", error.message);

      return {
        success: false,
        code: error.code || "UPDATE_MEETING_FAILED",
        message: error.message || "Unable to update meeting.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-start-meeting",

  async (event, meetingSysId) => {
    try {
      if (!meetingSysId) {
        return {
          success: false,
          code: "MEETING_REQUIRED",
          message: "Meeting sys_id is required.",
        };
      }

      const payload = {
        meeting_sys_id: String(meetingSysId).trim(),
      };

      const result = await serviceCallApiRequest(
        "/start-meeting",
        "POST",
        payload,
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
      if (result && result.success === true && result.call_sys_id) {
        /*
         * Mark this call as already handled so the
         * generic outgoing-call polling loop does not
         * treat the meeting as a normal direct call.
         */
        activeOutgoingCallId = result.call_sys_id;

        showCallWindow(
          "connected",

          {
            callSysId: result.call_sys_id,

            callNumber: result.call_number || "",

            /*
             * For a meeting, the main display name
             * is the meeting title.
             */
            name: result.title || "ServiceCall Meeting",

            department: "Meeting",

            /*
             * Meetings use the existing conference
             * call functionality.
             */
            isConference: true,

            /*
             * Keep meeting context available for
             * the next step.
             */
            isMeeting: true,

            meetingSysId: result.meeting_sys_id || meetingSysId,

            meetingNumber: result.meeting_number || "",

            meetingTitle: result.title || "ServiceCall Meeting",
          },
        );
      }

      return result;
    } catch (error) {
      console.error("Unable to start ServiceCall meeting:", error.message);

      return {
        success: false,

        code: error.code || "START_MEETING_FAILED",

        message: error.message || "Unable to start meeting.",
      };
    }
  },
);

/* =======================================================
   MEETING CHANGE NOTIFICATION
======================================================= */

ipcMain.on("servicecall-meeting-changed", (event, meetingSysId) => {
  /*
   * Forward the meeting change from the
   * call window to the main desktop window.
   */
  if (mainWindow && !mainWindow.isDestroyed()) {
    mainWindow.webContents.send("servicecall-meeting-changed", {
      meetingSysId: meetingSysId || "",
    });
  }
});

/* =====================================================
   RENDERER READY FOR DEEP LINKS
===================================================== */

ipcMain.on("servicecall-renderer-ready", (event) => {
  /*
   * Only accept this signal from
   * the main ServiceCall window.
   */
  if (
    !mainWindow ||
    mainWindow.isDestroyed() ||
    event.sender !== mainWindow.webContents
  ) {
    return;
  }

  if (!pendingDeepLink) {
    return;
  }

  console.log(
    "Renderer ready. Sending pending ServiceCall deep link:",
    pendingDeepLink,
  );

  mainWindow.webContents.send("servicecall-deep-link", {
    url: pendingDeepLink,
  });

  pendingDeepLink = null;
});

/* =======================================================
   JOIN MEETING
======================================================= */

ipcMain.handle(
  "servicecall-join-meeting",

  async (event, meetingSysId) => {
    try {
      if (!meetingSysId) {
        return {
          success: false,
          code: "MEETING_REQUIRED",
          message: "Meeting sys_id is required.",
        };
      }

      const payload = {
        meeting_sys_id: String(meetingSysId).trim(),
      };

      const result = await serviceCallApiRequest(
        "/join-meeting",
        "POST",
        payload,
      );

      if (result && result.success === true && result.call_sys_id) {
        /*
         * Explicit Join means the user wants
         * this meeting call again.
         */
        intentionallyLeftCallIds.delete(result.call_sys_id);

        /*
         * This user is now connected
         * to the existing meeting call.
         */
        activeOutgoingCallId = result.call_sys_id;

        /*
         * Reuse our existing call window.
         *
         * IMPORTANT:
         * Pass full meeting context so
         * call-window.js knows this is
         * a ServiceCall Meeting.
         */
        //console.warn('DEBUG: GENERIC OUTGOING ROLLER OPENING CALL WINDOW'),
        showCallWindow("connected", {
          callSysId: result.call_sys_id,

          callNumber: result.call_number || "",

          name: result.title || "ServiceCall Meeting",

          department: "Meeting",

          isConference: true,

          isMeeting: true,

          meetingSysId: result.meeting_sys_id || meetingSysId,

          meetingNumber: result.meeting_number || "",

          meetingTitle: result.title || "ServiceCall Meeting",
        });
      }

      return result;
    } catch (error) {
      console.error("Unable to join ServiceCall meeting:", error.message);

      return {
        success: false,
        code: error.code || "JOIN_MEETING_FAILED",
        message: error.message || "Unable to join meeting.",
      };
    }
  },
);

/* =======================================================
   LEAVE MEETING
======================================================= */

ipcMain.handle(
  "servicecall-leave-meeting",

  async (event, meetingSysId) => {
    try {
      if (!meetingSysId) {
        return {
          success: false,
          code: "MEETING_REQUIRED",
          message: "Meeting sys_id is required.",
        };
      }

      const payload = {
        meeting_sys_id: String(meetingSysId).trim(),
      };

      const result = await serviceCallApiRequest(
        "/leave-meeting",
        "POST",
        payload,
      );

      if (result && result.success === true) {
        /*
         * Remember that THIS user deliberately
         * left this call.
         *
         * The meeting itself may remain In Progress,
         * so polling must not reopen its window.
         */
        if (result.call_sys_id) {
          intentionallyLeftCallIds.add(result.call_sys_id);
        }

        /*
         * The user left this meeting,
         * so clear this call from the
         * active desktop state if needed.
         */
        if (activeOutgoingCallId === result.call_sys_id) {
          activeOutgoingCallId = null;
        }

        /*
         * Refresh the Meetings page.
         */
        if (mainWindow && !mainWindow.isDestroyed()) {
          mainWindow.webContents.send("servicecall-meeting-changed", {
            meetingSysId: result.meeting_sys_id || meetingSysId,
          });
        }
      }

      return result;
    } catch (error) {
      console.error("Unable to leave ServiceCall meeting:", error.message);

      return {
        success: false,
        code: error.code || "LEAVE_MEETING_FAILED",
        message: error.message || "Unable to leave meeting.",
      };
    }
  },
);

/* =======================================================
   END MEETING
======================================================= */

ipcMain.handle(
  "servicecall-end-meeting",

  async (event, meetingSysId) => {
    try {
      if (!meetingSysId) {
        return {
          success: false,
          code: "MEETING_REQUIRED",
          message: "Meeting sys_id is required.",
        };
      }

      const payload = {
        meeting_sys_id: String(meetingSysId).trim(),
      };

      const result = await serviceCallApiRequest(
        "/end-meeting",
        "POST",
        payload,
      );

      if (result && result.success === true) {
        /*
         * Meeting call has ended.
         */
        if (activeOutgoingCallId === result.call_sys_id) {
          activeOutgoingCallId = null;
        }

        if (activeIncomingCallId === result.call_sys_id) {
          activeIncomingCallId = null;
        }

        /*
         * Tell the main Meetings page
         * that meeting data changed.
         */
        if (mainWindow && !mainWindow.isDestroyed()) {
          mainWindow.webContents.send("servicecall-meeting-changed", {
            meetingSysId: result.meeting_sys_id || meetingSysId,
          });
        }
      }

      return result;
    } catch (error) {
      console.error("Unable to end ServiceCall meeting:", error.message);

      return {
        success: false,
        code: error.code || "END_MEETING_FAILED",
        message: error.message || "Unable to end meeting.",
      };
    }
  },
);

/* =======================================================
   CANCEL MEETING
======================================================= */

ipcMain.handle(
  "servicecall-cancel-meeting",

  async (event, meetingSysId) => {
    try {
      if (!meetingSysId) {
        return {
          success: false,
          code: "MEETING_REQUIRED",
          message: "Meeting sys_id is required.",
        };
      }

      const payload = {
        meeting_sys_id: String(meetingSysId).trim(),
      };

      const result = await serviceCallApiRequest(
        "/cancel-meeting",
        "POST",
        payload,
      );

      if (result && result.success === true) {
        /*
         * Tell the main Meetings page
         * that this meeting changed.
         */
        if (mainWindow && !mainWindow.isDestroyed()) {
          mainWindow.webContents.send("servicecall-meeting-changed", {
            meetingSysId: result.meeting_sys_id || meetingSysId,
          });
        }
      }

      return result;
    } catch (error) {
      console.error("Unable to cancel ServiceCall meeting:", error.message);

      return {
        success: false,
        code: error.code || "CANCEL_MEETING_FAILED",
        message: error.message || "Unable to cancel meeting.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-get-meeting-details",

  async (event, meetingSysId) => {
    try {
      if (!meetingSysId) {
        return {
          success: false,
          code: "MEETING_REQUIRED",
          message: "Meeting sys_id is required.",
        };
      }

      const query = new URLSearchParams({
        meeting_sys_id: String(meetingSysId).trim(),
      });

      const result = await serviceCallApiRequest(
        "/meeting-details?" + query.toString(),
        "GET",
      );

      return result;
    } catch (error) {
      console.error(
        "Unable to get ServiceCall meeting details:",
        error.message,
      );

      return {
        success: false,
        code: error.code || "MEETING_DETAILS_FAILED",
        message: error.message || "Unable to retrieve meeting details.",
      };
    }
  },
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
  "servicecall-get-notifications",

  async (event, options = {}) => {
    try {
      let page = parseInt(options.page, 10) || 1;

      if (page < 1) {
        page = 1;
      }

      let pageSize = parseInt(options.pageSize, 10) || 20;

      /*
       * Keep desktop requests reasonable.
       */
      if (pageSize < 1) {
        pageSize = 20;
      }

      if (pageSize > 50) {
        pageSize = 50;
      }

      const search = String(options.search || "")
        .trim()
        .substring(0, 100);

      const query = new URLSearchParams({
        page: String(page),

        page_size: String(pageSize),
      });

      if (search) {
        query.set("search", search);
      }

      const result = await serviceCallApiRequest(
        "/notifications?" + query.toString(),
        "GET",
      );

      return result;
    } catch (error) {
      console.error("Unable to get ServiceCall notifications:", error.message);

      return {
        success: false,

        code: error.code || "GET_NOTIFICATIONS_FAILED",

        message: error.message || "Unable to retrieve notifications.",

        notifications: [],

        unread_count: 0,

        page: 1,

        page_size: 20,

        has_more: false,
      };
    }
  },
);

/* =======================================================
   MARK NOTIFICATION READ
======================================================= */

ipcMain.handle(
  "servicecall-mark-notification-read",

  async (event, notificationSysId) => {
    try {
      const sysId = String(notificationSysId || "").trim();

      if (!sysId) {
        return {
          success: false,
          code: "NOTIFICATION_REQUIRED",
          message: "Notification sys_id is required.",
        };
      }

      return await serviceCallApiRequest("/mark-notification-read", "POST", {
        notification_sys_id: sysId,
      });
    } catch (error) {
      console.error(
        "Unable to mark ServiceCall notification as read:",
        error.message,
      );

      return {
        success: false,

        code: error.code || "MARK_NOTIFICATION_READ_FAILED",

        message: error.message || "Unable to mark notification as read.",
      };
    }
  },
);

/* -------------------------------------------------------
   DISMISS NOTIFICATION POPUP
------------------------------------------------------- */

ipcMain.on("servicecall-dismiss-notification-popup", (event) => {
  /*
   * Get the exact BrowserWindow that
   * sent this dismiss request.
   *
   * This is important now because several
   * notification popups may exist at once.
   */

  const popupWindow = BrowserWindow.fromWebContents(event.sender);

  if (!popupWindow || popupWindow.isDestroyed()) {
    return;
  }

  popupWindow.destroy();
});

ipcMain.on("servicecall-open-notification", (event, notificationData) => {
  console.log(
    "ServiceCall notification OPEN request received:",
    notificationData,
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

  if (!mainWindow || mainWindow.isDestroyed()) {
    return;
  }

  if (mainWindow.isMinimized()) {
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

  const meetingSysId = String(
    notificationData && notificationData.meetingSysId
      ? notificationData.meetingSysId
      : "",
  ).trim();

  if (/^[0-9a-f]{32}$/i.test(meetingSysId)) {
    console.log("Opening meeting from ServiceCall notification:", meetingSysId);

    mainWindow.webContents.send("servicecall-open-notification-meeting", {
      meetingSysId: meetingSysId,

      notificationSysId: String(
        notificationData && notificationData.notificationSysId
          ? notificationData.notificationSysId
          : "",
      ),

      type: String(
        notificationData && notificationData.type ? notificationData.type : "",
      ),
    });
  }
});

ipcMain.handle(
  "servicecall-start-call",

  async (event, targetUserSysId) => {
    try {
      const targetUser = String(targetUserSysId || "").trim();

      if (!targetUser) {
        return {
          success: false,
          code: "TARGET_USER_REQUIRED",
          message: "Target user is required.",
        };
      }

      const result = await serviceCallApiRequest("/start-call", "POST", {
        target_user_sys_id: targetUser,
      });

      return result;
    } catch (error) {
      console.error("Unable to start ServiceCall:", error.message);

      return {
        success: false,

        code: error.code || "START_CALL_FAILED",

        message: error.message || "Unable to start the ServiceCall.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-update-presence",

  async (event, presenceData = {}) => {
    try {
      const status = String(presenceData.status || "").trim();

      const oofReason = String(presenceData.oofReason || "").trim();

      if (!status) {
        return {
          success: false,
          code: "PRESENCE_REQUIRED",
          message: "Presence status is required.",
        };
      }

      return await serviceCallApiRequest("/presence", "POST", {
        status: status,
        oof_reason: oofReason,
      });
    } catch (error) {
      console.error("Unable to update ServiceCall presence:", error.message);

      return {
        success: false,
        code: error.code || "UPDATE_PRESENCE_FAILED",
        message: error.message || "Unable to update presence.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-get-my-presence",

  async () => {
    try {
      const result = await serviceCallApiRequest("/presence", "GET");

      return result;
    } catch (error) {
      console.error("Unable to get ServiceCall presence:", error.message);

      return {
        success: false,

        code: error.code || "GET_PRESENCE_FAILED",

        message: error.message || "Unable to retrieve presence.",
      };
    }
  },
);

ipcMain.handle("servicecall-check-access", async () => {
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

    const currentUser = await getCurrentServiceCallUser();

    const authorization = currentUser?.authorization || {};

    /*
     * Keep current runtime identity
     * and authorization up to date.
     */

    currentServiceCallUser = currentUser?.user || null;

    currentServiceCallAuthorization = authorization;

    /*
     * ACCESS IS STILL NOT AVAILABLE
     */

    if (authorization.allowed !== true) {
      return {
        success: true,
        authenticated: true,
        authorized: false,
        state: "access_denied",
        user: currentServiceCallUser,
        authorization: currentServiceCallAuthorization,
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
      console.log("ServiceCall access restored by manual check.");

      /*
       * Change the state before restoring.
       */

      serviceCallAccessUnavailable = false;

      try {
        await resumeServiceCallRuntime();

        await restoreServiceCallApplication();
      } catch (restoreError) {
        /*
         * Restoration failed.
         *
         * Put ServiceCall back into the
         * unavailable state so another
         * check can retry.
         */

        serviceCallAccessUnavailable = true;

        console.error(
          "Unable to restore ServiceCall after manual access check:",
          restoreError,
        );

        throw restoreError;
      }
    } else {
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
      state: "ready",
      user: currentServiceCallUser,
      authorization: currentServiceCallAuthorization,
    };
  } catch (error) {
    console.error("ServiceCall access check failed:", error);

    return {
      success: false,
      authenticated: false,
      authorized: false,
      state: "login_required",
      message: error?.message || "Unable to check ServiceCall access.",
    };
  }
});

ipcMain.handle("servicecall-get-current-account", async () => {
  /*
   * No authenticated identity
   * has been resolved yet.
   */

  if (!currentServiceCallUser) {
    return {
      success: false,
      authenticated: false,
      user: null,
      authorization: null,
    };
  }

  return {
    success: true,
    authenticated: true,

    user: currentServiceCallUser,

    authorization: currentServiceCallAuthorization || {
      allowed: false,
      is_servicecall_user: false,
      is_servicecall_admin: false,
    },
  };
});

async function stopCurrentServiceCallSession() {
  /*
   * -----------------------------------------
   * STOP HEARTBEAT
   * -----------------------------------------
   */

  if (heartbeatTimer) {
    clearInterval(heartbeatTimer);

    heartbeatTimer = null;
  }

  /*
   * -----------------------------------------
   * STOP INCOMING CALL MONITOR
   * -----------------------------------------
   */

  if (incomingCallTimer) {
    clearInterval(incomingCallTimer);

    incomingCallTimer = null;
  }

  /*
   * -----------------------------------------
   * STOP OUTGOING CALL MONITOR
   * -----------------------------------------
   */

  if (outgoingCallTimer) {
    clearInterval(outgoingCallTimer);

    outgoingCallTimer = null;
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

  console.log("ServiceCall Notification Session state reset");

  console.log("ServiceCall notification baseline reset.");
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
    console.error("ServiceCall explicit sign out failed:", error);

    /*
     * Fallback:
     *
     * If the dedicated sign-out endpoint
     * fails, still try to mark this device
     * registration offline.
     */

    try {
      await updateDesktopState("offline");
    } catch (fallbackError) {
      console.error("ServiceCall offline fallback failed:", fallbackError);
    }
  }

  /*
   * -----------------------------------------
   * CLEAR ACTIVE IN-MEMORY IDENTITY
   * -----------------------------------------
   */

  currentServiceCallUser = null;

  currentServiceCallAuthorization = null;

  activeIncomingCallId = null;

  activeOutgoingCallId = null;

  console.log("Current ServiceCall session stopped.");

  return {
    success: true,
  };
}

ipcMain.handle("servicecall-sign-out", async () => {
  try {
    console.log("ServiceCall sign out requested.");

    /*
     * Preserve the latest OAuth session
     * inside the currently active saved
     * account before signing out.
     */
    let config = loadConfig();

    config = syncActiveAccountTokens(config);

    saveConfig(config);

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
    if (mainWindow && !mainWindow.isDestroyed()) {
      await mainWindow.loadFile("auth/auth-gate.html");

      mainWindow.show();

      mainWindow.focus();
    }

    console.log("ServiceCall signed out of active session.");

    return {
      success: true,
      state: "signed_out",
    };
  } catch (error) {
    console.error("ServiceCall sign out failed:", error);

    return {
      success: false,
      state: "error",
      message: error?.message || "Unable to sign out of ServiceCall.",
    };
  }
});

async function signOutDesktopSession() {
  const deviceId = getOrCreateDeviceId();

  const result = await serviceCallApiRequest("/desktop-sign-out", "POST", {
    device_id: deviceId,
  });

  if (!result || result.success !== true) {
    throw new Error(
      result?.message || "Unable to sign out of ServiceCall Desktop.",
    );
  }

  console.log("ServiceCall desktop sign out:", result);

  return result;
}

ipcMain.handle("servicecall-get-saved-accounts", async () => {
  try {
    const accounts = getSavedAccounts();

    return {
      success: true,
      accounts: accounts,
    };
  } catch (error) {
    console.error("Unable to load ServiceCall saved accounts:", error);

    return {
      success: false,
      accounts: [],
      message: error?.message || "Unable to load saved accounts.",
    };
  }
});

ipcMain.handle(
  "servicecall-remove-saved-account",
  async (event, accountKey) => {
    try {
      return await removeSavedAccount(String(accountKey || ""));
    } catch (error) {
      console.error("Unable to remove saved ServiceCall account:", error);

      return {
        success: false,
        state: "error",
        message: error?.message || "Unable to remove the saved account.",
      };
    }
  },
);

async function activateSavedAccount(accountKey) {
  let config = loadConfig();

  config = ensureSavedAccountStructure(config);

  const account = config.savedAccounts[accountKey];

  if (!account) {
    return {
      success: false,
      state: "account_not_found",
      message: "The selected ServiceCall account could not be found.",
    };
  }

  /*
   * Restore this account's instance
   * and OAuth session into the active
   * runtime configuration.
   */

  config.instanceUrl = account.instanceUrl || "";

  config.accessToken = account.accessToken || "";

  config.refreshToken = account.refreshToken || "";

  config.tokenType = account.tokenType || "Bearer";

  config.expiresIn = account.expiresIn || 0;

  config.tokenObtainedAt = account.tokenObtainedAt || 0;

  config.activeAccountKey = accountKey;

  saveConfig(config);

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

    const currentUser = await getCurrentServiceCallUser();

    const authorization = currentUser?.authorization || {};

    /*
     * SECURITY CHECK:
     *
     * Make sure the OAuth identity still
     * belongs to the account that the
     * user selected.
     */

    if (
      String(currentUser?.user?.sys_id || "") !==
      String(account.userSysId || "")
    ) {
      throw new Error(
        "The authenticated ServiceNow identity does not match the selected saved account.",
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

    currentServiceCallUser = currentUser?.user || null;

    currentServiceCallAuthorization = authorization;

    /*
     * Refresh cached DISPLAY metadata.
     * These values never grant access.
     */

    config = loadConfig();

    config = ensureSavedAccountStructure(config);

    const savedAccount = config.savedAccounts[accountKey];

    if (savedAccount) {
      savedAccount.name = currentUser?.user?.name || savedAccount.name || "";

      savedAccount.userName =
        currentUser?.user?.user_name || savedAccount.userName || "";

      savedAccount.email = currentUser?.user?.email || "";

      savedAccount.serviceCallId = currentUser?.user?.servicecall_id || "";

      savedAccount.isServiceCallUser =
        authorization.is_servicecall_user === true;

      savedAccount.isServiceCallAdmin =
        authorization.is_servicecall_admin === true;

      savedAccount.lastUsedAt = new Date().toISOString();

      config.savedAccounts[accountKey] = savedAccount;

      saveConfig(config);
    }

    if (authorization.allowed !== true) {
      return {
        success: true,
        authenticated: true,
        authorized: false,
        state: "access_denied",
        user: currentUser?.user || null,
        authorization: authorization,
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
      state: "ready",
      user: currentUser?.user || null,
      authorization: authorization,
    };
  } catch (error) {
    console.error("Unable to activate saved ServiceCall account:", error);

    return {
      success: false,
      authenticated: false,
      authorized: false,
      state: "login_required",

      message: error?.message || "This account needs to sign in again.",
    };
  }
}

/* =========================================================
   REMOVE SAVED ACCOUNT
========================================================= */

async function removeSavedAccount(accountKey) {
  let config = loadConfig();

  config = ensureSavedAccountStructure(config);

  accountKey = String(accountKey || "").trim();

  if (!accountKey || !config.savedAccounts[accountKey]) {
    return {
      success: false,
      state: "account_not_found",
      message: "The saved ServiceCall account could not be found.",
    };
  }

  const wasActive = config.activeAccountKey === accountKey;

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
        "ServiceCall session cleanup during account removal failed:",
        error,
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
  delete config.savedAccounts[accountKey];

  if (wasActive) {
    config.activeAccountKey = "";

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

  saveConfig(config);

  console.log("ServiceCall saved account removed:", accountKey);

  return {
    success: true,
    state: "account_removed",
    removedAccountKey: accountKey,
  };
}

ipcMain.handle(
  "servicecall-activate-saved-account",
  async (event, accountKey) => {
    return await activateSavedAccount(String(accountKey || ""));
  },
);

/* =====================================================
   SERVICECALL CHAT - GET MESSAGES
===================================================== */

ipcMain.handle(
  "servicecall-get-messages",

  async (event, payload = {}) => {
    const conversationId = String(payload.conversationSysId || "").trim();

    const afterMessageSysId = String(payload.afterMessageSysId || "").trim();

    const beforeMessageSysId = String(payload.beforeMessageSysId || "").trim();

    const editedAfter = String(payload.editedAfter || "").trim();

    let limit = parseInt(payload.limit, 10);

    /*
     * Keep the Electron-side value bounded
     * exactly like the ServiceNow API.
     */
    if (!Number.isFinite(limit) || limit < 1) {
      limit = 50;
    }

    if (limit > 100) {
      limit = 100;
    }

    if (!conversationId) {
      return {
        success: false,
        code: "CONVERSATION_REQUIRED",
        message: "Conversation is required.",
        messages: [],
      };
    }

    /*
     * A request must never ask for both
     * newer and older messages at once.
     */
    if (afterMessageSysId && beforeMessageSysId) {
      return {
        success: false,
        code: "INVALID_MESSAGE_CURSOR",
        message:
          "after and before message checkpoints cannot be used together.",
        messages: [],
      };
    }

    try {
      /*
       * Base request.
       */
      let endpoint =
        "/messages?conversation_id=" + encodeURIComponent(conversationId);

      /*
       * Silent incremental refresh:
       *
       * Give me messages NEWER than
       * this checkpoint.
       */
      if (afterMessageSysId) {
        endpoint += "&after=" + encodeURIComponent(afterMessageSysId);
      }

      /*
       * Edited-message live synchronization.
       */
      if (editedAfter) {
        endpoint += "&edited_after=" + encodeURIComponent(editedAfter);
      }
      /*
       * Older history:
       *
       * Give me messages OLDER than
       * this checkpoint.
       */
      if (beforeMessageSysId) {
        endpoint += "&before=" + encodeURIComponent(beforeMessageSysId);
      }

      /*
       * Initial/history page size.
       *
       * The backend currently ignores this
       * for an "after" live-sync request,
       * which is intentional.
       */
      endpoint += "&limit=" + encodeURIComponent(String(limit));

      const result = await serviceCallApiRequest(endpoint, "GET");

      return result;
    } catch (error) {
      console.error("Unable to get ServiceCall messages:", error.message);

      return {
        success: false,
        code: error.code || "GET_MESSAGES_FAILED",
        message: error.message || "Unable to retrieve messages.",
        messages: [],
      };
    }
  },
);

/* =====================================================
   SERVICECALL CHAT - SEND MESSAGE
===================================================== */

ipcMain.handle(
  "servicecall-send-message",

  async (event, payload = {}) => {
    const recipientSysId = String(payload.recipientSysId || "").trim();

    const conversationSysId = String(payload.conversationSysId || "").trim();

    const message = String(payload.message || "").trim();

    const replyToMessageSysId = String(
      payload.replyToMessageSysId || "",
    ).trim();

    /* -------------------------
       VALIDATION
    ------------------------- */

    if (!recipientSysId && !conversationSysId) {
      return {
        success: false,
        code: "MESSAGE_TARGET_REQUIRED",
        message: "A message recipient or conversation is required.",
      };
    }

    if (!message) {
      return {
        success: false,
        code: "MESSAGE_REQUIRED",
        message: "Message cannot be empty.",
      };
    }

    if (message.length > 10000) {
      return {
        success: false,
        code: "MESSAGE_TOO_LONG",
        message: "Message cannot exceed 10000 characters.",
      };
    }

    /*
     * Reply is optional.
     *
     * When supplied, it must look like a
     * ServiceNow sys_id before we send it
     * to the server.
     */
    if (replyToMessageSysId && !/^[0-9a-f]{32}$/i.test(replyToMessageSysId)) {
      return {
        success: false,
        code: "INVALID_REPLY_MESSAGE",
        message: "Invalid reply message.",
      };
    }

    try {
      const requestBody = {
        message: message,
      };

      /*
       * Existing direct-chat path.
       */
      if (recipientSysId) {
        requestBody.recipient_sys_id = recipientSysId;
      }

      /*
       * Existing group-conversation path.
       */
      if (conversationSysId) {
        requestBody.conversation_id = conversationSysId;
      }

      /*
       * Optional reply.
       *
       * Do not send the property at all for
       * an ordinary message.
       */
      if (replyToMessageSysId) {
        requestBody.reply_to_message_sys_id = replyToMessageSysId;
      }

      const result = await serviceCallApiRequest(
        "/send-message",
        "POST",
        requestBody,
      );

      return result;
    } catch (error) {
      console.error("Unable to send ServiceCall message:", error.message);

      return {
        success: false,

        code: error.code || "SEND_MESSAGE_FAILED",

        message: error.message || "Unable to send message.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-prepare-chat-attachment",

  async (event, payload = {}) => {
    const conversationSysId = String(payload.conversationSysId || "").trim();

    const fileName = String(payload.fileName || "").trim();

    const mimeType = String(
      payload.mimeType || "application/octet-stream",
    ).trim();

    const fileSize = Number(payload.fileSize || 0);

    /* -------------------------
       VALIDATION
    ------------------------- */

    if (!conversationSysId || !/^[0-9a-f]{32}$/i.test(conversationSysId)) {
      return {
        success: false,
        code: "INVALID_CONVERSATION",
        message: "A valid conversation is required.",
      };
    }

    if (!fileName) {
      return {
        success: false,
        code: "FILE_NAME_REQUIRED",
        message: "File name is required.",
      };
    }

    if (!Number.isFinite(fileSize) || fileSize <= 0) {
      return {
        success: false,
        code: "INVALID_FILE_SIZE",
        message: "File size must be greater than zero.",
      };
    }

    const MAX_FILE_SIZE = 25 * 1024 * 1024;

    if (fileSize > MAX_FILE_SIZE) {
      return {
        success: false,
        code: "FILE_TOO_LARGE",
        message: "File cannot exceed 25 MB.",
      };
    }

    try {
      const result = await serviceCallApiRequest("/upload-attachment", "POST", {
        conversation_id: conversationSysId,

        file_name: fileName,

        mime_type: mimeType,

        file_size: fileSize,
      });

      return result;
    } catch (error) {
      console.error("Unable to prepare ServiceCall attachment:", error.message);

      return {
        success: false,

        code: error.code || "PREPARE_ATTACHMENT_FAILED",

        message: error.message || "Unable to prepare attachment.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-forward-messages",

  async (event, payload = {}) => {
    /*
     * Multiple source messages.
     */
    const messageSysIds = Array.isArray(payload.messageSysIds)
      ? payload.messageSysIds
          .map((value) => String(value || "").trim())
          .filter(Boolean)
      : [];

    /*
     * Multiple destination conversations.
     */
    const destinationConversationIds = Array.isArray(
      payload.destinationConversationIds,
    )
      ? payload.destinationConversationIds
          .map((value) => String(value || "").trim())
          .filter(Boolean)
      : [];

    /* -------------------------
       VALIDATION
    ------------------------- */

    if (messageSysIds.length === 0) {
      return {
        success: false,
        code: "MESSAGES_REQUIRED",
        message: "Select at least one message to forward.",
      };
    }

    if (destinationConversationIds.length === 0) {
      return {
        success: false,
        code: "DESTINATIONS_REQUIRED",
        message: "Select at least one destination.",
      };
    }

    /*
     * Match the server-side safety limits.
     */
    if (messageSysIds.length > 50) {
      return {
        success: false,
        code: "TOO_MANY_MESSAGES",
        message: "Too many messages were selected.",
      };
    }

    if (destinationConversationIds.length > 50) {
      return {
        success: false,
        code: "TOO_MANY_DESTINATIONS",
        message: "Too many destinations were selected.",
      };
    }

    /*
     * Validate every source message sys_id.
     */
    for (const messageSysId of messageSysIds) {
      if (!/^[0-9a-f]{32}$/i.test(messageSysId)) {
        return {
          success: false,
          code: "INVALID_MESSAGE",
          message: "One or more selected messages are invalid.",
        };
      }
    }

    /*
     * Validate every destination conversation sys_id.
     */
    for (const conversationSysId of destinationConversationIds) {
      if (!/^[0-9a-f]{32}$/i.test(conversationSysId)) {
        return {
          success: false,
          code: "INVALID_DESTINATION",
          message: "One or more destinations are invalid.",
        };
      }
    }

    try {
      const requestBody = {
        message_sys_ids: messageSysIds,

        destination_conversation_ids: destinationConversationIds,
      };

      const result = await serviceCallApiRequest(
        "/forward-message",
        "POST",
        requestBody,
      );

      return result;
    } catch (error) {
      console.error("Unable to forward ServiceCall messages:", error.message);

      return {
        success: false,

        code: error.code || "FORWARD_MESSAGE_FAILED",

        message: error.message || "Unable to forward the selected messages.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-get-reaction-updates",

  async (event, payload = {}) => {
    const conversationId = String(payload.conversationSysId || "").trim();

    const afterCheckpoint = String(payload.afterCheckpoint || "").trim();

    if (!conversationId) {
      return {
        success: false,
        code: "CONVERSATION_REQUIRED",
        message: "Conversation is required.",
        updates: [],
      };
    }

    try {
      let endpoint =
        "/reaction-updates?conversation_id=" +
        encodeURIComponent(conversationId);

      /*
       * If there is no checkpoint,
       * ServiceNow establishes one.
       *
       * Otherwise only reaction changes
       * after that checkpoint are returned.
       */
      if (afterCheckpoint) {
        endpoint += "&after=" + encodeURIComponent(afterCheckpoint);
      }

      const result = await serviceCallApiRequest(endpoint, "GET");

      return result;
    } catch (error) {
      console.error(
        "Unable to get ServiceCall reaction updates:",
        error.message,
      );

      return {
        success: false,
        code: error.code || "GET_REACTION_UPDATES_FAILED",

        message: error.message || "Unable to retrieve reaction updates.",

        updates: [],
      };
    }
  },
);

ipcMain.handle(
  "servicecall-set-message-reaction",

  async (event, payload = {}) => {
    const messageSysId = String(payload.messageSysId || "").trim();

    const reaction = String(payload.reaction || "").trim();

    if (!messageSysId) {
      return {
        success: false,
        code: "MESSAGE_REQUIRED",
        message: "Message is required.",
      };
    }

    if (!reaction) {
      return {
        success: false,
        code: "REACTION_REQUIRED",
        message: "Reaction is required.",
      };
    }

    try {
      const result = await serviceCallApiRequest("/message-reaction", "POST", {
        message_sys_id: messageSysId,

        reaction: reaction,
      });

      return result;
    } catch (error) {
      console.error(
        "Unable to update ServiceCall message reaction:",
        error.message,
      );

      return {
        success: false,
        code: error.code || "MESSAGE_REACTION_FAILED",

        message: error.message || "Unable to update message reaction.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-mark-conversation-read",

  async (event, conversationSysId) => {
    conversationSysId = String(conversationSysId || "").trim();

    if (!conversationSysId) {
      return {
        success: false,
        code: "CONVERSATION_REQUIRED",
        message: "Conversation is required.",
      };
    }

    try {
      const result = await serviceCallApiRequest(
        "/mark-conversation-read",
        "POST",
        {
          conversation_id: conversationSysId,
        },
      );

      return result;
    } catch (error) {
      console.error(
        "Unable to mark ServiceCall conversation as read:",
        error.message,
      );

      return {
        success: false,

        code: error.code || "MARK_CONVERSATION_READ_FAILED",

        message: error.message || "Unable to mark conversation as read.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-create-group",

  async (event, payload = {}) => {
    const title = String(payload.title || "").trim();

    const participantSysIds = Array.isArray(payload.participantSysIds)
      ? payload.participantSysIds
      : [];

    /* =========================================
           VALIDATION
        ========================================= */

    if (!title) {
      return {
        success: false,
        code: "GROUP_TITLE_REQUIRED",
        message: "Group title is required.",
      };
    }

    if (participantSysIds.length === 0) {
      return {
        success: false,
        code: "PARTICIPANTS_REQUIRED",
        message: "Select at least one participant.",
      };
    }

    try {
      /*
       * IMPORTANT:
       *
       * serviceCallApiRequest uses:
       *
       * path,
       * method,
       * body
       *
       * This is the same convention as
       * send-message and message-reaction.
       */

      const result = await serviceCallApiRequest("/create-group", "POST", {
        title: title,

        participant_sys_ids: participantSysIds,
      });

      return result;
    } catch (error) {
      console.error("Unable to create ServiceCall group:", error);

      return {
        success: false,
        code: error.code || "CREATE_GROUP_FAILED",

        message: error.message || "Unable to create group.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-get-group-details",
  async (event, conversationSysId) => {
    const cleanConversationSysId = String(conversationSysId || "").trim();

    if (!cleanConversationSysId) {
      return {
        success: false,
        code: "CONVERSATION_REQUIRED",
        message: "Conversation is required.",
      };
    }

    try {
      return await serviceCallApiRequest(
        "/group-details" +
          "?conversation_id=" +
          encodeURIComponent(cleanConversationSysId),
        "GET",
      );
    } catch (error) {
      console.error("Unable to load ServiceCall group details:", error);

      return {
        success: false,
        code: error.code || "GROUP_DETAILS_FAILED",
        message: error.message || "Unable to load group details.",
      };
    }
  },
);

ipcMain.handle("servicecall-rename-group", async (event, payload = {}) => {
  const conversationSysId = String(payload.conversationSysId || "").trim();

  const title = String(payload.title || "").trim();

  if (!conversationSysId) {
    return {
      success: false,
      code: "CONVERSATION_REQUIRED",
      message: "Conversation is required.",
    };
  }

  if (!title) {
    return {
      success: false,
      code: "GROUP_TITLE_REQUIRED",
      message: "Group title is required.",
    };
  }

  try {
    return await serviceCallApiRequest("/rename-group", "POST", {
      conversation_id: conversationSysId,

      title: title,
    });
  } catch (error) {
    console.error("Unable to rename ServiceCall group:", error);

    return {
      success: false,
      code: error.code || "RENAME_GROUP_FAILED",

      message: error.message || "Unable to rename group.",
    };
  }
});

/* =========================================================
   SERVICECALL CHAT - ADD GROUP MEMBERS
========================================================= */

ipcMain.handle(
  "servicecall-add-group-members",

  async (event, data = {}) => {
    try {
      const conversationSysId = String(data.conversationSysId || "").trim();

      const participantSysIds = Array.isArray(data.participantSysIds)
        ? data.participantSysIds
        : [];

      if (!conversationSysId) {
        return {
          success: false,
          code: "CONVERSATION_REQUIRED",
          message: "Conversation is required.",
        };
      }

      if (participantSysIds.length === 0) {
        return {
          success: false,
          code: "PARTICIPANTS_REQUIRED",
          message: "Select at least one person.",
        };
      }

      const result = await serviceCallApiRequest("/add-group-members", "POST", {
        conversation_id: conversationSysId,

        participant_sys_ids: participantSysIds,
      });

      return result;
    } catch (error) {
      console.error("Unable to add group members:", error);

      return {
        success: false,

        code: error.code || "ADD_GROUP_MEMBERS_FAILED",

        message: error.message || "Unable to add people to the group.",
      };
    }
  },
);

/* =========================================================
   SERVICECALL - SET GROUP MEMBER ROLE
========================================================= */

ipcMain.handle(
  "servicecall-set-group-member-role",
  async (event, { conversationSysId, memberUserSysId, role } = {}) => {
    try {
      const cleanConversationSysId = String(conversationSysId || "").trim();

      const cleanMemberUserSysId = String(memberUserSysId || "").trim();

      const cleanRole = String(role || "")
        .trim()
        .toLowerCase();

      if (!cleanConversationSysId) {
        return {
          success: false,
          code: "MISSING_CONVERSATION_ID",
          message: "Conversation ID is required.",
        };
      }

      if (!cleanMemberUserSysId) {
        return {
          success: false,
          code: "MISSING_MEMBER_USER_ID",
          message: "Member user ID is required.",
        };
      }

      if (cleanRole !== "owner" && cleanRole !== "member") {
        throw new Error("Role must be owner or member.");
      }

      return await serviceCallApiRequest("/set-group-member-role", "POST", {
        conversation_id: cleanConversationSysId,

        member_user_sys_id: cleanMemberUserSysId,

        role: cleanRole,
      });
    } catch (error) {
      console.error("Set group member role failed:", error);

      return {
        success: false,
        code: "SET_GROUP_MEMBER_ROLE_FAILED",
        message:
          error && error.message
            ? error.message
            : "Unable to change group member role.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-remove-group-member",
  async (event, { conversationSysId, memberUserSysId } = {}) => {
    const cleanConversationSysId = String(conversationSysId || "").trim();

    const cleanMemberUserSysId = String(memberUserSysId || "").trim();

    /* -------------------------
       VALIDATION
    ------------------------- */

    if (
      !cleanConversationSysId ||
      !/^[0-9a-f]{32}$/i.test(cleanConversationSysId)
    ) {
      throw new Error("A valid conversation sys_id is required.");
    }

    if (
      !cleanMemberUserSysId ||
      !/^[0-9a-f]{32}$/i.test(cleanMemberUserSysId)
    ) {
      throw new Error("A valid member user sys_id is required.");
    }

    /* -------------------------
       SERVICENOW REQUEST
    ------------------------- */

    return await serviceCallApiRequest("/remove-group-member", "POST", {
      conversation_id: cleanConversationSysId,

      member_user_sys_id: cleanMemberUserSysId,
    });
  },
);

ipcMain.handle(
  "servicecall-leave-group",
  async (event, { conversationSysId }) => {
    const cleanConversationSysId = String(conversationSysId || "").trim();

    if (!/^[0-9a-f]{32}$/i.test(cleanConversationSysId)) {
      return {
        success: false,
        code: "INVALID_CONVERSATION_ID",
        message: "A valid conversation ID is required.",
      };
    }

    try {
      return await serviceCallApiRequest("/leave-group", "POST", {
        conversation_id: cleanConversationSysId,
      });
    } catch (error) {
      console.error("ServiceCall leave group failed:", error);

      return {
        success: false,
        code: "LEAVE_GROUP_FAILED",
        message:
          error && error.message ? error.message : "Unable to leave group.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-delete-messages",

  async (event, payload = {}) => {
    const conversationSysId = String(payload.conversationSysId || "").trim();

    const mode = String(payload.mode || "")
      .trim()
      .toLowerCase();

    const rawMessageSysIds = payload.messageSysIds;

    /* -------------------------
       VALIDATION
    ------------------------- */

    if (!conversationSysId || !/^[0-9a-f]{32}$/i.test(conversationSysId)) {
      return {
        success: false,
        code: "INVALID_CONVERSATION",
        message: "Invalid conversation.",
      };
    }

    /*
     * Supported modes:
     *
     * everyone
     * me
     */
    if (mode !== "everyone" && mode !== "me") {
      return {
        success: false,
        code: "INVALID_DELETE_MODE",
        message: "Delete mode must be everyone or me.",
      };
    }

    if (!Array.isArray(rawMessageSysIds) || rawMessageSysIds.length === 0) {
      return {
        success: false,
        code: "MESSAGES_REQUIRED",
        message: "At least one message is required.",
      };
    }

    if (rawMessageSysIds.length > 100) {
      return {
        success: false,
        code: "TOO_MANY_MESSAGES",
        message: "You can delete up to 100 messages at a time.",
      };
    }

    /*
     * Normalize and validate every message sys_id
     * before sending anything to ServiceNow.
     */
    const messageSysIds = [];

    const seenMessageSysIds = new Set();

    for (const rawMessageSysId of rawMessageSysIds) {
      const messageSysId = String(rawMessageSysId || "").trim();

      if (!/^[0-9a-f]{32}$/i.test(messageSysId)) {
        return {
          success: false,
          code: "INVALID_MESSAGE",
          message: "One or more selected messages are invalid.",
        };
      }

      if (seenMessageSysIds.has(messageSysId)) {
        return {
          success: false,
          code: "DUPLICATE_MESSAGE",
          message: "The same message was selected more than once.",
        };
      }

      seenMessageSysIds.add(messageSysId);

      messageSysIds.push(messageSysId);
    }

    /* -------------------------
       SERVICENOW REQUEST
    ------------------------- */

    try {
      const result = await serviceCallApiRequest("/delete-messages", "POST", {
        conversation_id: conversationSysId,

        message_sys_ids: messageSysIds,

        mode: mode,
      });

      return result;
    } catch (error) {
      console.error("Unable to delete ServiceCall messages:", error.message);

      return {
        success: false,

        code: error.code || "DELETE_MESSAGES_FAILED",

        message: error.message || "Unable to delete selected messages.",
      };
    }
  },
);

/* =========================================================
   SERVICECALL CHAT - DELETE CHAT
========================================================= */

ipcMain.handle(
  "servicecall-delete-chat",

  async (event, conversationSysId) => {
    try {
      const normalizedConversationSysId = String(
        conversationSysId || "",
      ).trim();

      /* -------------------------------------------------
         VALIDATION
      ------------------------------------------------- */

      if (
        !normalizedConversationSysId ||
        !/^[0-9a-f]{32}$/i.test(normalizedConversationSysId)
      ) {
        return {
          success: false,
          code: "INVALID_CONVERSATION",
          message: "A valid conversation is required.",
        };
      }

      /* -------------------------------------------------
         SERVICECALL API
      ------------------------------------------------- */

      const result = await serviceCallApiRequest("/delete-chat", "POST", {
        conversation_id: normalizedConversationSysId,
      });

      return result;
    } catch (error) {
      console.error("ServiceCall delete chat failed:", error);

      return {
        success: false,
        code: "DELETE_CHAT_FAILED",
        message:
          error && error.message ? error.message : "Unable to delete chat.",
      };
    }
  },
);

/* =====================================================
   SERVICECALL CHAT - EDIT MESSAGE
===================================================== */

ipcMain.handle(
  "servicecall-edit-message",

  async (event, payload = {}) => {
    const conversationSysId = String(payload.conversationSysId || "").trim();

    const messageSysId = String(payload.messageSysId || "").trim();

    const message = String(payload.message || "").trim();

    /* -------------------------
       VALIDATION
    ------------------------- */

    if (!conversationSysId || !/^[0-9a-f]{32}$/i.test(conversationSysId)) {
      return {
        success: false,
        code: "INVALID_CONVERSATION",
        message: "Invalid conversation.",
      };
    }

    if (!messageSysId || !/^[0-9a-f]{32}$/i.test(messageSysId)) {
      return {
        success: false,
        code: "INVALID_MESSAGE",
        message: "Invalid message.",
      };
    }

    if (message.length > 10000) {
      return {
        success: false,
        code: "MESSAGE_TOO_LONG",
        message: "Message cannot exceed 10000 characters.",
      };
    }

    /* -------------------------
       SERVICENOW REQUEST
    ------------------------- */

    try {
      const result = await serviceCallApiRequest("/edit-message", "POST", {
        conversation_id: conversationSysId,

        message_sys_id: messageSysId,

        message: message,
      });

      return result;
    } catch (error) {
      console.error("Unable to edit ServiceCall message:", error.message);

      return {
        success: false,

        code: error.code || "EDIT_MESSAGE_FAILED",

        message: error.message || "Unable to edit message.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-upload-chat-attachment-binary",

  async (event, payload = {}) => {
    const attachmentSysId = String(payload.attachmentSysId || "").trim();

    const fileBytes = payload.fileBytes;

    /* -------------------------
       VALIDATION
    ------------------------- */

    if (!attachmentSysId || !/^[0-9a-f]{32}$/i.test(attachmentSysId)) {
      return {
        success: false,
        code: "INVALID_ATTACHMENT",
        message: "A valid attachment is required.",
      };
    }

    if (!fileBytes) {
      return {
        success: false,
        code: "FILE_DATA_REQUIRED",
        message: "Attachment file data is required.",
      };
    }

    try {
      /*
       * Electron IPC may give us an
       * ArrayBuffer / Uint8Array rather
       * than a Node Buffer.
       *
       * Convert it here in main.js.
       */
      let binaryData;

      if (Buffer.isBuffer(fileBytes)) {
        binaryData = fileBytes;
      } else if (fileBytes instanceof ArrayBuffer) {
        binaryData = Buffer.from(fileBytes);
      } else if (ArrayBuffer.isView(fileBytes)) {
        binaryData = Buffer.from(
          fileBytes.buffer,
          fileBytes.byteOffset,
          fileBytes.byteLength,
        );
      } else {
        /*
         * Electron structured-clone can
         * sometimes give Buffer-like data.
         */
        binaryData = Buffer.from(fileBytes);
      }

      if (!binaryData.length) {
        return {
          success: false,
          code: "EMPTY_ATTACHMENT",
          message: "Attachment file is empty.",
        };
      }

      const MAX_FILE_SIZE = 25 * 1024 * 1024;

      if (binaryData.length > MAX_FILE_SIZE) {
        return {
          success: false,
          code: "FILE_TOO_LARGE",
          message: "File cannot exceed 25 MB.",
        };
      }

      /* -------------------------
         BINARY UPLOAD
      ------------------------- */

      const result = await serviceCallBinaryApiRequest(
        "/upload-attachment/" + encodeURIComponent(attachmentSysId) + "/binary",

        binaryData,
      );

      return result;
    } catch (error) {
      console.error(
        "Unable to upload ServiceCall attachment binary:",
        error.message,
      );

      return {
        success: false,

        code: error.code || "UPLOAD_ATTACHMENT_BINARY_FAILED",

        message: error.message || "Unable to upload attachment.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-send-attachment-message",

  async (event, payload = {}) => {
    const attachmentSysId = String(payload.attachmentSysId || "").trim();

    /* -------------------------
       VALIDATION
    ------------------------- */

    if (!attachmentSysId || !/^[0-9a-f]{32}$/i.test(attachmentSysId)) {
      return {
        success: false,
        code: "INVALID_ATTACHMENT",
        message: "A valid attachment is required.",
      };
    }

    try {
      const result = await serviceCallApiRequest(
        "/send-attachment-message",
        "POST",
        {
          attachment_id: attachmentSysId,
        },
      );

      return result;
    } catch (error) {
      console.error(
        "Unable to send ServiceCall attachment message:",
        error.message,
      );

      return {
        success: false,

        code: error.code || "SEND_ATTACHMENT_MESSAGE_FAILED",

        message: error.message || "Unable to send attachment message.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-download-chat-attachment",
  async (event, payload = {}) => {
    const attachmentSysId = String(payload.attachmentSysId || "").trim();

    /*
     * Validate ServiceCall attachment sys_id.
     */
    if (!attachmentSysId || !/^[0-9a-f]{32}$/i.test(attachmentSysId)) {
      return {
        success: false,
        code: "INVALID_ATTACHMENT",
        message: "A valid attachment is required.",
      };
    }

    try {
      /*
       * ServiceNow performs the real
       * authorization:
       *
       * user
       *   -> ServiceCall access
       *   -> conversation
       *   -> membership
       *   -> attachment
       *   -> physical file
       */
      const result = await serviceCallBinaryDownloadRequest(
        "/download-attachment/" + encodeURIComponent(attachmentSysId),
      );

      return {
        success: true,

        /*
         * Return a Uint8Array instead of
         * exposing any ServiceNow file URL.
         *
         * Electron structured cloning can
         * safely transfer this to preload /
         * renderer.
         */
        fileBytes: new Uint8Array(result.buffer),

        contentType: result.contentType,

        contentDisposition: result.contentDisposition,
      };
    } catch (error) {
      console.error("ServiceCall attachment download failed:", error);

      return {
        success: false,
        code: error.code || "DOWNLOAD_ATTACHMENT_FAILED",
        message: error.message || "Unable to download attachment.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-open-chat-attachment",
  async (event, payload = {}) => {
    const attachmentSysId = String(payload.attachmentSysId || "").trim();

    const requestedFileName = String(payload.fileName || "attachment").trim();

    if (!attachmentSysId || !/^[0-9a-f]{32}$/i.test(attachmentSysId)) {
      return {
        success: false,
        code: "INVALID_ATTACHMENT",
        message: "A valid attachment is required.",
      };
    }

    try {
      /*
       * Download through our secure
       * ServiceCall endpoint.
       */
      const result = await serviceCallBinaryDownloadRequest(
        "/download-attachment/" + encodeURIComponent(attachmentSysId),
      );

      /*
       * Never trust the original filename
       * as a filesystem path.
       */
      const safeFileName = path
        .basename(requestedFileName || "attachment")
        .replace(/[<>:"/\\|?*\x00-\x1F]/g, "_");

      /*
       * Use a ServiceCall-owned temporary
       * directory.
       */
      const tempDirectory = path.join(
        os.tmpdir(),
        "ServiceCall",
        "attachments",
      );

      fs.mkdirSync(tempDirectory, {
        recursive: true,
      });

      /*
       * Prefix with attachment sys_id so
       * files with identical names do not
       * overwrite each other.
       */
      const tempFilePath = path.join(
        tempDirectory,
        attachmentSysId + "-" + safeFileName,
      );

      fs.writeFileSync(tempFilePath, result.buffer);

      /*
       * Ask the operating system to open
       * the file with its default app.
       */
      const openError = await shell.openPath(tempFilePath);

      if (openError) {
        const error = new Error(openError);

        error.code = "OPEN_ATTACHMENT_FAILED";

        throw error;
      }

      return {
        success: true,
      };
    } catch (error) {
      console.error("ServiceCall open attachment failed:", error);

      return {
        success: false,

        code: error.code || "OPEN_ATTACHMENT_FAILED",

        message: error.message || "Unable to open attachment.",
      };
    }
  },
);

ipcMain.handle(
  "servicecall-cancel-chat-attachment",
  async (event, payload = {}) => {
    const attachmentSysId = String(payload.attachmentSysId || "").trim();

    if (!attachmentSysId || !/^[0-9a-f]{32}$/i.test(attachmentSysId)) {
      return {
        success: false,
        code: "INVALID_ATTACHMENT",
        message: "A valid attachment is required.",
      };
    }

    try {
      const result = await serviceCallApiRequest("/cancel-attachment", "POST", {
        attachment_id: attachmentSysId,
      });

      console.log("ServiceCall attachment cancelled:", result);

      return result;
    } catch (error) {
      console.error("ServiceCall cancel attachment failed:", error);

      return {
        success: false,

        code: error.code || "CANCEL_ATTACHMENT_FAILED",

        message: error.message || "Unable to cancel attachment.",
      };
    }
  },
);
ipcMain.handle("servicecall-send-chat-content", async (event, payload = {}) => {
  const conversationSysId = String(payload.conversationSysId || "").trim();

  const message = String(payload.message || "").trim();

  const replyToMessageSysId = String(payload.replyToMessageSysId || "").trim();

  const attachmentSysIds = Array.isArray(payload.attachmentSysIds)
    ? payload.attachmentSysIds
        .map((id) => String(id || "").trim())
        .filter(Boolean)
    : [];

  /* =========================================
       CONVERSATION
    ========================================= */

  if (!conversationSysId || !/^[0-9a-f]{32}$/i.test(conversationSysId)) {
    return {
      success: false,
      code: "INVALID_CONVERSATION",
      message: "A valid conversation is required.",
    };
  }

  /* =========================================
       CONTENT
    ========================================= */

  if (!message && attachmentSysIds.length === 0) {
    return {
      success: false,
      code: "EMPTY_MESSAGE",
      message: "Message text or an attachment is required.",
    };
  }

  if (message.length > 10000) {
    return {
      success: false,
      code: "MESSAGE_TOO_LONG",
      message: "Message cannot exceed 10000 characters.",
    };
  }

  /* =========================================
       ATTACHMENTS
    ========================================= */

  if (attachmentSysIds.length > 10) {
    return {
      success: false,
      code: "TOO_MANY_ATTACHMENTS",
      message: "A maximum of 10 attachments is allowed per message.",
    };
  }

  for (const attachmentSysId of attachmentSysIds) {
    if (!/^[0-9a-f]{32}$/i.test(attachmentSysId)) {
      return {
        success: false,
        code: "INVALID_ATTACHMENT",
        message: "One or more attachments are invalid.",
      };
    }
  }

  /*
   * Prevent the same attachment ID from
   * being submitted more than once.
   */
  if (new Set(attachmentSysIds).size !== attachmentSysIds.length) {
    return {
      success: false,
      code: "DUPLICATE_ATTACHMENT",
      message: "The same attachment cannot be included more than once.",
    };
  }

  /* =========================================
       REPLY
    ========================================= */

  if (replyToMessageSysId && !/^[0-9a-f]{32}$/i.test(replyToMessageSysId)) {
    return {
      success: false,
      code: "INVALID_REPLY_MESSAGE",
      message: "Reply message is invalid.",
    };
  }

  /* =========================================
       SERVICENOW
    ========================================= */

  try {
    const result = await serviceCallApiRequest("/send-chat-content", "POST", {
      conversation_id: conversationSysId,

      message: message,

      attachment_ids: attachmentSysIds,

      reply_to_message_sys_id: replyToMessageSysId,
    });

    console.log("ServiceCall chat content sent:", result);

    return result;
  } catch (error) {
    console.error("ServiceCall send chat content failed:", error);

    return {
      success: false,

      code: error.code || "SEND_CHAT_CONTENT_FAILED",

      message: error.message || "Unable to send chat content.",
    };

    renderer.js

    document.addEventListener("DOMContentLoaded", async () => {
  /* -------------------------------------------------
           ELEMENTS
        ------------------------------------------------- */

  const form = document.getElementById("instanceForm");

  const input = document.getElementById("instanceUrl");

  const message = document.getElementById("instanceMessage");

  let chatAttachmentUploadsInProgress = 0;

  const loginButton = document.getElementById("loginButton");

  const signOutButton = document.getElementById("signOutButton");

  const openActiveCallButton = document.getElementById("openActiveCallButton");

  const chatAttachButton = document.getElementById("chatAttachButton");

  const chatAttachmentInput = document.createElement("input");

  let pendingChatAttachments = [];

  chatAttachmentInput.multiple = true;

  chatAttachmentInput.type = "file";

  chatAttachmentInput.style.display = "none";

  document.body.appendChild(chatAttachmentInput);

  chatAttachButton?.addEventListener("click", () => {
    /*
     * Clear the previous value so selecting
     * the same file twice still fires change.
     */
    chatAttachmentInput.value = "";

    chatAttachmentInput.click();
  });

  chatAttachmentInput.addEventListener(
    "change",

    async () => {
      /*
       * Convert FileList to a normal array.
       */
      const selectedFiles = Array.from(chatAttachmentInput.files || []);

      if (selectedFiles.length === 0) {
        return;
      }

      /* =========================================
       CONVERSATION
    ========================================= */

      const conversationSysId = String(
        activeChatConversation?.sys_id || "",
      ).trim();

      const isTemporaryConversation =
        activeChatConversation?.temporary === true;

      /*
       * Attachments currently require an
       * existing ServiceNow conversation.
       */
      if (isTemporaryConversation) {
        console.warn(
          "Attachment upload requires the direct conversation to exist first.",
        );

        chatAttachmentInput.value = "";

        return;
      }

      if (!conversationSysId || !/^[0-9a-f]{32}$/i.test(conversationSysId)) {
        console.error("No valid ServiceCall conversation is currently open.");

        chatAttachmentInput.value = "";

        return;
      }

      /* =========================================
       LIMITS
    ========================================= */

      const MAX_FILE_SIZE = 25 * 1024 * 1024;

      const MAX_ATTACHMENTS = 10;

      /*
       * Existing pending files +
       * newly selected files cannot
       * exceed our backend limit.
       */
      if (
        pendingChatAttachments.length + selectedFiles.length >
        MAX_ATTACHMENTS
      ) {
        console.error("A maximum of 10 attachments is allowed per message.");

        chatAttachmentInput.value = "";

        return;
      }

      /* =========================================
       VALIDATE ALL SELECTED FILES FIRST
    ========================================= */

      for (const file of selectedFiles) {
        if (file.size <= 0) {
          console.error("Selected attachment is empty:", file.name);

          chatAttachmentInput.value = "";

          return;
        }

        if (file.size > MAX_FILE_SIZE) {
          console.error("Attachment exceeds 25 MB:", file.name);

          chatAttachmentInput.value = "";

          return;
        }
      }

      /*
       * These files are now entering the
       * upload pipeline.
       *
       * Send must remain disabled until every
       * selected file has either completed or
       * failed.
       */
      chatAttachmentUploadsInProgress += selectedFiles.length;

      if (chatSendButton) {
        chatSendButton.disabled = true;
      }

      /*
       * Show selected files immediately while
       * their uploads are still in progress.
       */
      selectedFiles.forEach((file) => {
        pendingChatAttachments.push({
          sysId: "",
          fileName: file.name,
          mimeType: file.type || "application/octet-stream",
          fileSize: file.size,
          status: "uploading",
        });
      });

      renderPendingChatAttachment();

      /* =========================================
       UPLOAD EACH FILE
    ========================================= */

      for (const file of selectedFiles) {
        let preparedAttachmentSysId = "";

        try {
          console.log("Preparing ServiceCall attachment:", file.name);

          /* -----------------------------------------
           CREATE ATTACHMENT METADATA
        ----------------------------------------- */

          const result = await window.serviceCall.prepareChatAttachment({
            conversationSysId: conversationSysId,

            fileName: file.name,

            mimeType: file.type || "application/octet-stream",

            fileSize: file.size,
          });

          console.log("Prepare attachment result:", result);

          if (!result?.success) {
            throw new Error(result?.message || "Unable to prepare attachment.");
          }

          preparedAttachmentSysId = String(
            result.attachment?.sys_id || "",
          ).trim();

          if (
            !preparedAttachmentSysId ||
            !/^[0-9a-f]{32}$/i.test(preparedAttachmentSysId)
          ) {
            throw new Error(
              "ServiceCall did not return a valid attachment ID.",
            );
          }

          /* -----------------------------------------
           READ FILE
        ----------------------------------------- */

          const arrayBuffer = await file.arrayBuffer();

          console.log("Uploading attachment binary:", {
            attachmentSysId: preparedAttachmentSysId,

            fileName: file.name,

            bytes: arrayBuffer.byteLength,
          });

          /* -----------------------------------------
           UPLOAD BINARY
        ----------------------------------------- */

          const uploadResult =
            await window.serviceCall.uploadChatAttachmentBinary(
              preparedAttachmentSysId,
              arrayBuffer,
            );

          console.log("Attachment binary upload result:", uploadResult);

          if (!uploadResult?.success) {
            throw new Error(
              uploadResult?.message || "Unable to upload attachment binary.",
            );
          }

          /* -----------------------------------------
           ADD TO PENDING COMPOSER
        ----------------------------------------- */

          /*
           * Convert the existing uploading item
           * into a ready attachment.
           */
          const uploadingAttachment = pendingChatAttachments.find(
            (attachment) =>
              !attachment.sysId &&
              attachment.status === "uploading" &&
              attachment.fileName === file.name &&
              attachment.fileSize === file.size,
          );

          if (uploadingAttachment) {
            uploadingAttachment.sysId = preparedAttachmentSysId;

            uploadingAttachment.status = "ready";
          }

          /*
           * Re-render so the composer reflects
           * the completed upload.
           */
          renderPendingChatAttachment();

          console.log("🔥 ServiceCall attachment ready:", file.name);
        } catch (error) {
          console.error("Attachment preparation failed:", file.name, error);

          if (preparedAttachmentSysId) {
            try {
              await window.serviceCall.cancelChatAttachment(
                preparedAttachmentSysId,
              );
            } catch (cleanupError) {
              console.error(
                "Unable to clean up failed attachment:",
                cleanupError,
              );
            }
          }
        } finally {
          /*
           * This individual file has finished
           * its upload attempt.
           */
          chatAttachmentUploadsInProgress = Math.max(
            0,
            chatAttachmentUploadsInProgress - 1,
          );

          /*
           * Send stays locked while ANY file
           * is still uploading.
           */
          if (chatSendButton) {
            const hasCurrentText = !!String(
              chatMessageInput?.value || "",
            ).trim();

            const hasCurrentAttachments = pendingChatAttachments.some(
              (attachment) =>
                attachment &&
                attachment.status !== "failed" &&
                !!String(attachment.sysId || "").trim(),
            );
          }
        }
        /*
         * Keep the failed file visible so the
         * user knows that its upload did not succeed.
         */
        const failedAttachment = pendingChatAttachments.find(
          (attachment) =>
            attachment.status === "uploading" &&
            attachment.fileName === file.name &&
            attachment.fileSize === file.size,
        );

        if (failedAttachment) {
          failedAttachment.sysId = "";
          failedAttachment.status = "failed";
        }

        renderPendingChatAttachment();
      }

      /* =========================================
       RESET FILE PICKER

       This allows selecting the same file
       again later if needed.
    ========================================= */

      chatAttachmentInput.value = "";

      /* =========================================
       SEND BUTTON
    ========================================= */

      if (chatSendButton) {
        const hasCurrentText = !!String(chatMessageInput?.value || "").trim();

        const hasCurrentAttachments = pendingChatAttachments.some(
          (attachment) =>
            attachment &&
            attachment.status !== "failed" &&
            !!String(attachment.sysId || "").trim(),
        );

        const uploadsStillRunning = chatAttachmentUploadsInProgress > 0;

        chatSendButton.disabled =
          uploadsStillRunning || (!hasCurrentText && !hasCurrentAttachments);
      }
    },
  );
  /* -------------------------------------------------
   PEOPLE ELEMENTS
------------------------------------------------- */

  const peopleSearchInput = document.getElementById("peopleSearchInput");

  /* -------------------------------------------------
   GROUP CHAT STATE
------------------------------------------------- */

  let chatCreateGroupSearchTimer = null;

  /*
   * Map:
   *
   * user sys_id -> user object
   *
   * A Map is important because selections
   * must survive when the search text changes.
   */
  const chatCreateGroupSelectedUsers = new Map();

  const peopleSearchMessage = document.getElementById("peopleSearchMessage");

  const peopleSearchResults = document.getElementById("peopleSearchResults");

  const chatMessages = document.getElementById("chatMessages");

  const chatGroupLeaveButton = document.getElementById("chatGroupLeaveButton");

  const chatGroupLeaveMessage = document.getElementById(
    "chatGroupLeaveMessage",
  );

  const chatMessageInput = document.getElementById("chatMessageInput");

  const chatEmojiButton = document.getElementById("chatEmojiButton");

  /* =====================================================
   CHAT COMPOSER EMOJI PICKER
===================================================== */

  function toggleChatEmojiPicker() {
    /*
     * Only allow the picker when there is an
     * active conversation and the composer can be used.
     */
    if (
      !activeChatConversation ||
      !chatMessageInput ||
      chatMessageInput.disabled
    ) {
      return;
    }

    /*
     * If already open, clicking 😊 closes it.
     */
    const existingPicker = document.getElementById("chatComposerEmojiPicker");

    if (existingPicker) {
      existingPicker.remove();
      return;
    }

    const picker = document.createElement("div");

    picker.id = "chatComposerEmojiPicker";

    picker.style.cssText = `
    position:fixed;
    z-index:5000;

    width:380px;
    height:420px;

    display:flex;
    flex-direction:column;

    overflow:hidden;

    background:#ffffff;

    border:
      1px solid #dce5e2;

    border-radius:16px;

    box-shadow:
      0 18px 50px
      rgba(16,47,43,0.20);
  `;

    /* =====================================================
   EMOJI DATA
===================================================== */

    const emojiCategories = [
      {
        id: "smileys",
        icon: "😀",
        title: "Smileys & Emotion",
        emojis: [
          "😀",
          "😃",
          "😄",
          "😁",
          "😆",
          "😅",
          "😂",
          "🤣",
          "🥲",
          "😊",
          "😇",
          "🙂",
          "🙃",
          "😉",
          "😌",
          "😍",
          "🥰",
          "😘",
          "😗",
          "😙",
          "😚",
          "😋",
          "😛",
          "😝",
          "😜",
          "🤪",
          "🤨",
          "🧐",
          "🤓",
          "😎",
          "🥸",
          "🤩",
          "🥳",
          "🙂‍↕️",
          "😏",
          "😒",
          "🙂‍↔️",
          "😞",
          "😔",
          "😟",
          "😕",
          "🙁",
          "☹️",
          "😣",
          "😖",
          "😫",
          "😩",
          "🥺",
          "😢",
          "😭",
          "😮‍💨",
          "😤",
          "😠",
          "😡",
          "🤬",
          "🤯",
          "😳",
          "🥵",
          "🥶",
          "😱",
          "😨",
          "😰",
          "😥",
          "😓",
          "🫣",
          "🤗",
          "🫡",
          "🤔",
          "🫢",
          "🤭",
          "🤫",
          "🤥",
          "😶",
          "😶‍🌫️",
          "😐",
          "😑",
          "😬",
          "🫨",
          "🫠",
          "🙄",
          "😯",
          "😦",
          "😧",
          "😮",
          "😲",
          "🥱",
          "😴",
          "🤤",
          "😪",
          "😵",
          "😵‍💫",
          "🤐",
          "🥴",
          "🤢",
          "🤮",
          "🤧",
          "😷",
          "🤒",
          "🤕",
          "🤑",
          "🤠",
          "😈",
          "👿",
          "👹",
          "👺",
          "🤡",
          "💩",
          "👻",
          "💀",
          "☠️",
          "👽",
          "👾",
          "🤖",
          "🎃",

          "😺",
          "😸",
          "😹",
          "😻",
          "😼",
          "😽",
          "🙀",
          "😿",
          "😾",

          "❤️",
          "🩷",
          "🧡",
          "💛",
          "💚",
          "💙",
          "🩵",
          "💜",
          "🤎",
          "🖤",
          "🩶",
          "🤍",
          "💔",
          "❤️‍🔥",
          "❤️‍🩹",
          "❣️",
          "💕",
          "💞",
          "💓",
          "💗",
          "💖",
          "💘",
          "💝",
          "💟",
          "♥️",

          "💋",
          "💯",
          "💢",
          "💥",
          "💫",
          "💦",
          "💨",
          "🕳️",
          "💬",
          "👁️‍🗨️",
          "🗨️",
          "🗯️",
          "💭",
          "💤",
        ],
      },

      {
        id: "people",
        icon: "👋",
        title: "People & Gestures",
        emojis: [
          "👋",
          "🤚",
          "🖐️",
          "✋",
          "🖖",
          "🫱",
          "🫲",
          "🫳",
          "🫴",
          "🫷",
          "🫸",
          "👌",
          "🤌",
          "🤏",
          "✌️",
          "🤞",
          "🫰",
          "🤟",
          "🤘",
          "🤙",
          "👈",
          "👉",
          "👆",
          "🖕",
          "👇",
          "☝️",
          "🫵",
          "👍",
          "👎",
          "✊",
          "👊",
          "🤛",
          "🤜",
          "👏",
          "🙌",
          "🫶",
          "👐",
          "🤲",
          "🤝",
          "🙏",
          "✍️",
          "💅",
          "🤳",
          "💪",
          "🦾",
          "🦿",
          "🦵",
          "🦶",
          "👂",
          "🦻",
          "👃",
          "🧠",
          "🫀",
          "🫁",
          "🦷",
          "🦴",
          "👀",
          "👁️",
          "👅",
          "👄",
          "🫦",

          "👶",
          "🧒",
          "👦",
          "👧",
          "🧑",
          "👱",
          "👨",
          "🧔",
          "👩",
          "🧓",
          "👴",
          "👵",

          "🙍",
          "🙎",
          "🙅",
          "🙆",
          "💁",
          "🙋",
          "🧏",
          "🙇",
          "🤦",
          "🤷",
          "🫂",

          "👮",
          "👷",
          "💂",
          "🕵️",
          "👩‍⚕️",
          "👨‍⚕️",
          "👩‍🎓",
          "👨‍🎓",
          "👩‍🏫",
          "👨‍🏫",
          "👩‍⚖️",
          "👨‍⚖️",
          "👩‍🌾",
          "👨‍🌾",
          "👩‍🍳",
          "👨‍🍳",
          "👩‍🔧",
          "👨‍🔧",
          "👩‍💻",
          "👨‍💻",
          "👩‍🎤",
          "👨‍🎤",
          "👩‍🎨",
          "👨‍🎨",
          "👩‍✈️",
          "👨‍✈️",
          "👩‍🚀",
          "👨‍🚀",
          "👩‍🚒",
          "👨‍🚒",

          "👼",
          "🎅",
          "🤶",
          "🦸",
          "🦹",
          "🧙",
          "🧚",
          "🧛",
          "🧜",
          "🧝",
          "🧞",
          "🧟",

          "🚶",
          "🧍",
          "🧎",
          "🏃",
          "💃",
          "🕺",
          "🕴️",
          "👯",
          "🧖",
          "🧘",
          "🛀",
          "🛌",
        ],
      },

      {
        id: "animals",
        icon: "🐶",
        title: "Animals & Nature",
        emojis: [
          "🐶",
          "🐱",
          "🐭",
          "🐹",
          "🐰",
          "🦊",
          "🐻",
          "🐼",
          "🐻‍❄️",
          "🐨",
          "🐯",
          "🦁",
          "🐮",
          "🐷",
          "🐽",
          "🐸",
          "🐵",
          "🙈",
          "🙉",
          "🙊",
          "🐒",
          "🐔",
          "🐧",
          "🐦",
          "🐤",
          "🐣",
          "🐥",
          "🦆",
          "🦅",
          "🦉",
          "🦇",
          "🐺",
          "🐗",
          "🐴",
          "🦄",
          "🐝",
          "🪱",
          "🐛",
          "🦋",
          "🐌",
          "🐞",
          "🐜",
          "🪰",
          "🪲",
          "🪳",
          "🦟",
          "🦗",
          "🕷️",
          "🦂",
          "🐢",
          "🐍",
          "🦎",
          "🦖",
          "🦕",
          "🐙",
          "🦑",
          "🦐",
          "🦞",
          "🦀",
          "🐡",
          "🐠",
          "🐟",
          "🐬",
          "🐳",
          "🐋",
          "🦈",
          "🦭",
          "🐊",
          "🐅",
          "🐆",
          "🦓",
          "🦍",
          "🦧",
          "🐘",
          "🦛",
          "🦏",
          "🐪",
          "🐫",
          "🦒",
          "🦘",
          "🦬",
          "🐃",
          "🐂",
          "🐄",
          "🐎",
          "🐖",
          "🐏",
          "🐑",
          "🦙",
          "🐐",
          "🦌",
          "🐕",
          "🐩",
          "🦮",
          "🐕‍🦺",
          "🐈",
          "🐈‍⬛",
          "🪶",
          "🐓",
          "🦃",
          "🦚",
          "🦜",
          "🪽",
          "🐇",
          "🦝",
          "🦨",
          "🦡",
          "🦫",
          "🦦",
          "🦥",
          "🐁",
          "🐀",
          "🐿️",
          "🦔",

          "🌵",
          "🎄",
          "🌲",
          "🌳",
          "🌴",
          "🪵",
          "🌱",
          "🌿",
          "☘️",
          "🍀",
          "🎍",
          "🪴",
          "🎋",
          "🍃",
          "🍂",
          "🍁",
          "🍄",
          "🐚",
          "🪨",
          "🌾",
          "💐",
          "🌷",
          "🌹",
          "🥀",
          "🪻",
          "🌺",
          "🌸",
          "🌼",
          "🌻",

          "🌞",
          "🌝",
          "🌛",
          "🌜",
          "🌚",
          "🌕",
          "🌖",
          "🌗",
          "🌘",
          "🌑",
          "🌒",
          "🌓",
          "🌔",
          "🌙",
          "🌎",
          "🌍",
          "🌏",
          "🪐",
          "💫",
          "⭐",
          "🌟",
          "✨",
          "⚡",
          "☄️",
          "💥",
          "🔥",
          "🌪️",
          "🌈",
          "☀️",
          "🌤️",
          "⛅",
          "🌥️",
          "☁️",
          "🌦️",
          "🌧️",
          "⛈️",
          "🌩️",
          "🌨️",
          "❄️",
          "☃️",
          "⛄",
          "🌬️",
          "💨",
          "💧",
          "💦",
          "☔",
          "☂️",
        ],
      },

      {
        id: "food",
        icon: "🍕",
        title: "Food & Drink",
        emojis: [
          "🍏",
          "🍎",
          "🍐",
          "🍊",
          "🍋",
          "🍋‍🟩",
          "🍌",
          "🍉",
          "🍇",
          "🍓",
          "🫐",
          "🍈",
          "🍒",
          "🍑",
          "🥭",
          "🍍",
          "🥥",
          "🥝",
          "🍅",
          "🍆",
          "🥑",
          "🥦",
          "🫛",
          "🥬",
          "🥒",
          "🌶️",
          "🫑",
          "🌽",
          "🥕",
          "🫒",
          "🧄",
          "🧅",
          "🥔",
          "🍠",
          "🫚",
          "🫘",
          "🥐",
          "🥯",
          "🍞",
          "🥖",
          "🥨",
          "🧀",
          "🥚",
          "🍳",
          "🧈",
          "🥞",
          "🧇",
          "🥓",
          "🥩",
          "🍗",
          "🍖",
          "🌭",
          "🍔",
          "🍟",
          "🍕",
          "🫓",
          "🥪",
          "🥙",
          "🧆",
          "🌮",
          "🌯",
          "🫔",
          "🥗",
          "🥘",
          "🫕",
          "🥫",
          "🍝",
          "🍜",
          "🍲",
          "🍛",
          "🍣",
          "🍱",
          "🥟",
          "🦪",
          "🍤",
          "🍙",
          "🍚",
          "🍘",
          "🍥",
          "🥠",
          "🥮",
          "🍢",
          "🍡",
          "🍧",
          "🍨",
          "🍦",
          "🥧",
          "🧁",
          "🍰",
          "🎂",
          "🍮",
          "🍭",
          "🍬",
          "🍫",
          "🍿",
          "🍩",
          "🍪",
          "🌰",
          "🥜",
          "🍯",

          "🥛",
          "🍼",
          "☕",
          "🫖",
          "🍵",
          "🧃",
          "🥤",
          "🧋",
          "🫙",
          "🍶",
          "🍺",
          "🍻",
          "🥂",
          "🍷",
          "🥃",
          "🍸",
          "🍹",
          "🧉",
          "🍾",
          "🧊",

          "🥄",
          "🍴",
          "🍽️",
          "🥣",
          "🥡",
          "🥢",
          "🧂",
        ],
      },

      {
        id: "activities",
        icon: "⚽",
        title: "Activities",
        emojis: [
          "⚽",
          "🏀",
          "🏈",
          "⚾",
          "🥎",
          "🎾",
          "🏐",
          "🏉",
          "🥏",
          "🎱",
          "🪀",
          "🏓",
          "🏸",
          "🏒",
          "🏑",
          "🥍",
          "🏏",
          "🪃",
          "🥅",
          "⛳",
          "🪁",
          "🏹",
          "🎣",
          "🤿",
          "🥊",
          "🥋",
          "🎽",
          "🛹",
          "🛼",
          "🛷",
          "⛸️",
          "🥌",
          "🎿",
          "⛷️",
          "🏂",
          "🪂",
          "🏋️",
          "🤼",
          "🤸",
          "⛹️",
          "🤺",
          "🤾",
          "🏌️",
          "🏇",
          "🧘",
          "🏄",
          "🏊",
          "🤽",
          "🚣",
          "🧗",
          "🚵",
          "🚴",

          "🏆",
          "🥇",
          "🥈",
          "🥉",
          "🏅",
          "🎖️",
          "🏵️",
          "🎗️",
          "🎫",
          "🎟️",
          "🎪",
          "🤹",
          "🎭",
          "🩰",
          "🎨",
          "🎬",
          "🎤",
          "🎧",
          "🎼",
          "🎹",
          "🥁",
          "🪘",
          "🎷",
          "🎺",
          "🪗",
          "🎸",
          "🪕",
          "🎻",
          "🪈",

          "🎲",
          "♟️",
          "🎯",
          "🎳",
          "🎮",
          "🎰",
          "🧩",
        ],
      },

      {
        id: "travel",
        icon: "🚗",
        title: "Travel & Places",
        emojis: [
          "🚗",
          "🚕",
          "🚙",
          "🚌",
          "🚎",
          "🏎️",
          "🚓",
          "🚑",
          "🚒",
          "🚐",
          "🛻",
          "🚚",
          "🚛",
          "🚜",
          "🏍️",
          "🛵",
          "🚲",
          "🛴",
          "🛹",
          "🛼",
          "🚨",
          "🚔",
          "🚍",
          "🚘",
          "🚖",
          "🚡",
          "🚠",
          "🚟",
          "🚃",
          "🚋",
          "🚞",
          "🚝",
          "🚄",
          "🚅",
          "🚈",
          "🚂",
          "🚆",
          "🚇",
          "🚊",
          "🚉",
          "✈️",
          "🛫",
          "🛬",
          "🛩️",
          "💺",
          "🛰️",
          "🚀",
          "🛸",
          "🚁",
          "🛶",
          "⛵",
          "🚤",
          "🛥️",
          "🛳️",
          "⛴️",
          "🚢",
          "⚓",
          "🛟",
          "⛽",
          "🚧",
          "🚦",
          "🚥",
          "🗺️",
          "🗿",
          "🗽",
          "🗼",
          "🏰",
          "🏯",
          "🏟️",
          "🎡",
          "🎢",
          "🎠",
          "⛲",
          "⛱️",
          "🏖️",
          "🏝️",
          "🏜️",
          "🌋",
          "⛰️",
          "🏔️",
          "🗻",
          "🏕️",
          "⛺",
          "🛖",
          "🏠",
          "🏡",
          "🏢",
          "🏥",
          "🏦",
          "🏨",
          "🏪",
          "🏫",
          "🏭",
          "🏛️",
          "⛪",
          "🕌",
          "🛕",
          "🕍",
          "⛩️",
          "🕋",
          "🌅",
          "🌄",
          "🌠",
          "🎇",
          "🎆",
          "🌇",
          "🌆",
          "🏙️",
          "🌃",
          "🌌",
          "🌉",
          "🌁",
        ],
      },

      {
        id: "objects",
        icon: "💡",
        title: "Objects",
        emojis: [
          "⌚",
          "📱",
          "📲",
          "💻",
          "⌨️",
          "🖥️",
          "🖨️",
          "🖱️",
          "🖲️",
          "🕹️",
          "🗜️",
          "💽",
          "💾",
          "💿",
          "📀",
          "📼",
          "📷",
          "📸",
          "📹",
          "🎥",
          "📽️",
          "🎞️",
          "📞",
          "☎️",
          "📟",
          "📠",
          "📺",
          "📻",
          "🎙️",
          "🎚️",
          "🎛️",
          "🧭",
          "⏱️",
          "⏲️",
          "⏰",
          "🕰️",
          "⌛",
          "⏳",
          "📡",
          "🔋",
          "🪫",
          "🔌",
          "💡",
          "🔦",
          "🕯️",
          "🪔",
          "🧯",
          "🛢️",
          "💸",
          "💵",
          "💴",
          "💶",
          "💷",
          "🪙",
          "💰",
          "💳",
          "💎",
          "⚖️",
          "🪜",
          "🧰",
          "🪛",
          "🔧",
          "🔨",
          "⚒️",
          "🛠️",
          "⛏️",
          "🪚",
          "🔩",
          "⚙️",
          "🧱",
          "⛓️",
          "🧲",
          "🔫",
          "💣",
          "🧨",
          "🪓",
          "🔪",
          "🗡️",
          "⚔️",
          "🛡️",
          "🚬",
          "⚰️",
          "🪦",
          "⚱️",
          "🏺",
          "🔮",
          "📿",
          "🧿",
          "🪬",
          "💈",
          "⚗️",
          "🔭",
          "🔬",
          "🕳️",
          "🩹",
          "🩺",
          "💊",
          "💉",
          "🩸",
          "🧬",
          "🦠",
          "🧫",
          "🧪",
          "🌡️",
          "🧹",
          "🪠",
          "🧺",
          "🧻",
          "🚽",
          "🚿",
          "🛁",
          "🪥",
          "🪒",
          "🧴",
          "🧼",
          "🫧",
          "🧽",
          "🧯",
          "🛒",
          "🎁",
          "🎈",
          "🎏",
          "🎀",
          "🪄",
          "🪅",
          "🎊",
          "🎉",
          "🎎",
          "🏮",
          "🎐",
          "🧧",
          "✉️",
          "📩",
          "📨",
          "📧",
          "💌",
          "📥",
          "📤",
          "📦",
          "🏷️",
          "🪧",
          "📪",
          "📫",
          "📬",
          "📭",
          "📮",
          "📯",
          "📜",
          "📃",
          "📄",
          "📑",
          "🧾",
          "📊",
          "📈",
          "📉",
          "🗒️",
          "🗓️",
          "📆",
          "📅",
          "🗑️",
          "📇",
          "🗃️",
          "🗳️",
          "🗄️",
          "📋",
          "📁",
          "📂",
          "🗂️",
          "🗞️",
          "📰",
          "📓",
          "📔",
          "📒",
          "📕",
          "📗",
          "📘",
          "📙",
          "📚",
          "📖",
          "🔖",
          "🧷",
          "🔗",
          "📎",
          "🖇️",
          "📐",
          "📏",
          "🧮",
          "📌",
          "📍",
          "✂️",
          "🖊️",
          "🖋️",
          "✒️",
          "🖌️",
          "🖍️",
          "📝",
          "✏️",
          "🔍",
          "🔎",
          "🔏",
          "🔐",
          "🔒",
          "🔓",
          "🔑",
          "🗝️",
          "🔨",
        ],
      },

      {
        id: "symbols",
        icon: "❤️",
        title: "Symbols",
        emojis: [
          "❤️",
          "🩷",
          "🧡",
          "💛",
          "💚",
          "💙",
          "🩵",
          "💜",
          "🖤",
          "🩶",
          "🤍",
          "🤎",
          "💔",
          "❣️",
          "💕",
          "💞",
          "💓",
          "💗",
          "💖",
          "💘",
          "💝",
          "💟",

          "☮️",
          "✝️",
          "☪️",
          "🕉️",
          "☸️",
          "✡️",
          "🔯",
          "🕎",
          "☯️",
          "☦️",
          "🛐",
          "⛎",

          "♈",
          "♉",
          "♊",
          "♋",
          "♌",
          "♍",
          "♎",
          "♏",
          "♐",
          "♑",
          "♒",
          "♓",

          "🆔",
          "⚛️",
          "🉑",
          "☢️",
          "☣️",
          "📴",
          "📳",
          "🈶",
          "🈚",
          "🈸",
          "🈺",
          "🈷️",
          "✴️",
          "🆚",
          "💮",
          "🉐",
          "㊙️",
          "㊗️",
          "🈴",
          "🈵",
          "🈹",
          "🈲",
          "🅰️",
          "🅱️",
          "🆎",
          "🆑",
          "🅾️",
          "🆘",
          "❌",
          "⭕",
          "🛑",
          "⛔",
          "📛",
          "🚫",
          "💯",
          "💢",
          "♨️",
          "🚷",
          "🚯",
          "🚳",
          "🚱",
          "🔞",
          "📵",
          "🚭",
          "❗",
          "❕",
          "❓",
          "❔",
          "‼️",
          "⁉️",
          "🔅",
          "🔆",
          "〽️",
          "⚠️",
          "🚸",
          "🔱",
          "⚜️",
          "🔰",
          "♻️",
          "✅",
          "🈯",
          "💹",
          "❇️",
          "✳️",
          "❎",
          "🌐",
          "💠",
          "Ⓜ️",
          "🌀",
          "💤",
          "🏧",
          "🚾",
          "♿",
          "🅿️",
          "🛗",
          "🈳",
          "🈂️",
          "🛂",
          "🛃",
          "🛄",
          "🛅",
          "🚹",
          "🚺",
          "🚼",
          "⚧️",
          "🚻",
          "🚮",
          "🎦",
          "📶",
          "🈁",
          "🔣",
          "ℹ️",
          "🔤",
          "🔡",
          "🔠",
          "🆖",
          "🆗",
          "🆙",
          "🆒",
          "🆕",
          "🆓",
          "0️⃣",
          "1️⃣",
          "2️⃣",
          "3️⃣",
          "4️⃣",
          "5️⃣",
          "6️⃣",
          "7️⃣",
          "8️⃣",
          "9️⃣",
          "🔟",
          "#️⃣",
          "*️⃣",
          "▶️",
          "⏸️",
          "⏯️",
          "⏹️",
          "⏺️",
          "⏭️",
          "⏮️",
          "⏩",
          "⏪",
          "🔀",
          "🔁",
          "🔂",
          "◀️",
          "🔼",
          "🔽",
          "➡️",
          "⬅️",
          "⬆️",
          "⬇️",
          "↗️",
          "↘️",
          "↙️",
          "↖️",
          "↕️",
          "↔️",
          "↪️",
          "↩️",
          "⤴️",
          "⤵️",
          "🔃",
          "🔄",
          "🔙",
          "🔚",
          "🔛",
          "🔜",
          "🔝",
          "🛐",
          "⚛️",
        ],
      },

      // {
      //   id: "flags",
      //   icon: "🏳️",
      //   title: "Flags",
      //   emojis: [
      //     "🏳️",
      //     "🏴",
      //     "🏁",
      //     "🚩",
      //     "🏳️‍🌈",
      //     "🏳️‍⚧️",
      //     "🏴‍☠️",
      //     "🇮🇳",
      //     "🇺🇸",
      //     "🇬🇧",
      //     "🇨🇦",
      //     "🇦🇺",
      //     "🇳🇿",
      //     "🇯🇵",
      //     "🇰🇷",
      //     "🇨🇳",
      //     "🇸🇬",
      //     "🇦🇪",
      //     "🇸🇦",
      //     "🇶🇦",
      //     "🇴🇲",
      //     "🇧🇭",
      //     "🇰🇼",
      //     "🇫🇷",
      //     "🇩🇪",
      //     "🇮🇹",
      //     "🇪🇸",
      //     "🇵🇹",
      //     "🇳🇱",
      //     "🇧🇪",
      //     "🇨🇭",
      //     "🇦🇹",
      //     "🇸🇪",
      //     "🇳🇴",
      //     "🇩🇰",
      //     "🇫🇮",
      //     "🇮🇸",
      //     "🇮🇪",
      //     "🇵🇱",
      //     "🇬🇷",
      //     "🇹🇷",
      //     "🇧🇷",
      //     "🇦🇷",
      //     "🇲🇽",
      //     "🇿🇦",
      //     "🇪🇬",
      //     "🇳🇵",
      //     "🇧🇹",
      //     "🇧🇩",
      //     "🇱🇰",
      //     "🇵🇰",
      //     "🇮🇩",
      //     "🇲🇾",
      //     "🇹🇭",
      //     "🇻🇳",
      //     "🇵🇭",
      //   ],
      // },
    ];

    /* =====================================================
   EMOJI SEARCH KEYWORDS
===================================================== */

    const emojiSearchKeywords = {
      "😀": "grinning smile happy face",
      "😃": "smile happy joy face",
      "😄": "smile happy laugh joy",
      "😁": "grin happy teeth smile",
      "😆": "laugh laughing happy",
      "😅": "sweat nervous laugh relief",
      "😂": "laugh laughing tears joy funny lol",
      "🤣": "rolling laugh laughing funny lol",
      "🥲": "smile tear emotional sad happy",
      "😊": "smile happy blush cute",
      "😇": "angel innocent halo",
      "🙂": "smile slightly happy",
      "🙃": "upside down smile sarcastic",
      "😉": "wink flirty",
      "😍": "love heart eyes crush",
      "🥰": "love hearts affection cute",
      "😘": "kiss love heart",
      "😋": "yum tasty delicious food",
      "😜": "tongue wink playful",
      "🤪": "crazy silly goofy",
      "🤓": "nerd glasses smart",
      "😎": "cool sunglasses",
      "🤩": "star eyes excited amazing",
      "🥳": "party celebration birthday",
      "😏": "smirk",
      "😒": "annoyed unimpressed",
      "😔": "sad disappointed",
      "😕": "confused",
      "☹️": "sad frown",
      "🥺": "pleading puppy eyes cute",
      "😢": "cry sad tear",
      "😭": "cry crying tears sad",
      "😤": "angry frustrated steam",
      "😠": "angry mad",
      "😡": "angry rage mad",
      "🤬": "swearing angry curse",
      "🤯": "mind blown shocked explosion",
      "😳": "embarrassed shocked blush",
      "🥵": "hot heat",
      "🥶": "cold freezing",
      "😱": "scream scared shocked horror",
      "😨": "fear scared",
      "🤗": "hug hugging",
      "🫡": "salute respect",
      "🤔": "thinking think question",
      "🤭": "giggle laugh hand mouth",
      "🤫": "quiet shh secret",
      "🙄": "eye roll annoyed",
      "😴": "sleep sleeping tired",
      "🤤": "drool hungry",
      "🤢": "sick nauseous",
      "🤮": "vomit sick",
      "🤧": "sneeze sick",
      "😷": "mask sick doctor",
      "🤒": "fever sick",
      "🤕": "hurt injury",
      "🤑": "money rich",
      "🤠": "cowboy",
      "😈": "devil evil",
      "🤡": "clown",
      "💩": "poop funny",
      "👻": "ghost halloween",
      "💀": "skull dead death lol",
      "👽": "alien space",
      "🤖": "robot bot ai",
      "🎃": "pumpkin halloween",

      "❤️": "heart red love romance",
      "🩷": "heart pink love",
      "🧡": "heart orange love",
      "💛": "heart yellow love",
      "💚": "heart green love",
      "💙": "heart blue love",
      "🩵": "heart light blue love",
      "💜": "heart purple love",
      "🖤": "heart black love",
      "🤍": "heart white love",
      "🤎": "heart brown love",
      "💔": "broken heart breakup sad",
      "❤️‍🔥": "heart fire passion love",
      "❤️‍🩹": "healing heart recovery",
      "💕": "hearts love",
      "💞": "hearts love romance",
      "💖": "sparkling heart love",
      "💘": "cupid heart love",
      "💋": "kiss lips love",

      "👋": "wave hello hi bye hand",
      "👌": "ok okay perfect hand",
      "✌️": "peace victory two",
      "🤞": "fingers crossed luck",
      "🤟": "love you hand",
      "🤘": "rock metal hand",
      "🤙": "call me hand",
      "👉": "right point",
      "👈": "left point",
      "👆": "up point",
      "👇": "down point",
      "👍": "thumbs up like yes good",
      "👎": "thumbs down dislike no bad",
      "👏": "clap applause congratulations",
      "🙌": "celebrate raised hands hooray",
      "🫶": "heart hands love",
      "🙏": "pray please thanks namaste",
      "💪": "muscle strong strength gym",
      "👀": "eyes look watch see",
      "🧠": "brain smart think",

      "🐶": "dog puppy animal",
      "🐱": "cat kitten animal",
      "🐭": "mouse animal",
      "🐰": "rabbit bunny animal",
      "🦊": "fox animal",
      "🐻": "bear animal",
      "🐼": "panda animal",
      "🐯": "tiger animal",
      "🦁": "lion animal",
      "🐮": "cow animal",
      "🐷": "pig animal",
      "🐸": "frog animal",
      "🐵": "monkey animal",
      "🙈": "monkey see no evil",
      "🙉": "monkey hear no evil",
      "🙊": "monkey speak no evil",
      "🐔": "chicken animal",
      "🐧": "penguin animal",
      "🦄": "unicorn magical",
      "🐝": "bee insect honey",
      "🦋": "butterfly insect",
      "🐢": "turtle animal",
      "🐍": "snake animal",
      "🐙": "octopus sea",
      "🐬": "dolphin sea",
      "🐳": "whale sea",
      "🦈": "shark sea",
      "🌹": "rose flower love",
      "🌸": "cherry blossom flower",
      "🌻": "sunflower flower",
      "🍀": "clover luck",
      "⭐": "star favorite",
      "🌟": "star glowing",
      "✨": "sparkles magic shine",
      "⚡": "lightning electric fast",
      "🔥": "fire flame hot lit",
      "🌈": "rainbow",
      "☀️": "sun sunny weather",
      "☁️": "cloud weather",
      "🌧️": "rain weather",
      "❄️": "snow cold winter",
      "💧": "water drop",

      "🍎": "apple fruit",
      "🍌": "banana fruit",
      "🍉": "watermelon fruit",
      "🍇": "grapes fruit",
      "🍓": "strawberry fruit",
      "🍒": "cherry fruit",
      "🥭": "mango fruit",
      "🍍": "pineapple fruit",
      "🥑": "avocado food",
      "🌶️": "chilli pepper spicy hot",
      "🍞": "bread food",
      "🧀": "cheese food",
      "🍳": "egg breakfast food",
      "🍗": "chicken food",
      "🍔": "burger hamburger food",
      "🍟": "fries chips food",
      "🍕": "pizza food",
      "🌮": "taco food",
      "🍜": "noodles ramen food",
      "🍣": "sushi food",
      "🍦": "ice cream dessert",
      "🎂": "cake birthday",
      "🍫": "chocolate sweet",
      "🍿": "popcorn movie",
      "🍩": "donut doughnut",
      "🍪": "cookie biscuit",
      "☕": "coffee tea drink",
      "🍵": "tea drink",
      "🍺": "beer drink",
      "🍻": "beer cheers",
      "🥂": "cheers celebration",
      "🍷": "wine drink",

      "⚽": "football soccer sport",
      "🏀": "basketball sport",
      "🏈": "american football sport",
      "⚾": "baseball sport",
      "🎾": "tennis sport",
      "🏏": "cricket sport",
      "🏆": "trophy winner champion",
      "🥇": "gold medal first winner",
      "🎯": "target bullseye",
      "🎮": "game gaming controller",
      "🎤": "microphone singing music",
      "🎧": "headphones music",
      "🎹": "piano music",
      "🥁": "drum music",
      "🎸": "guitar music",
      "🎻": "violin music",
      "🎨": "art paint",
      "🎬": "movie film cinema",

      "🚗": "car vehicle travel",
      "🚕": "taxi cab vehicle",
      "🚌": "bus vehicle",
      "🚑": "ambulance emergency",
      "🚒": "fire truck emergency",
      "🚲": "bicycle bike",
      "✈️": "plane airplane flight travel",
      "🚀": "rocket space launch",
      "🚁": "helicopter",
      "🚢": "ship boat",
      "🏠": "house home",
      "🏥": "hospital medical",
      "🏫": "school education",
      "🏖️": "beach vacation",
      "⛰️": "mountain",
      "🌅": "sunrise",
      "🌃": "night city",

      "📱": "phone mobile",
      "💻": "laptop computer work",
      "⌨️": "keyboard computer",
      "🖥️": "desktop computer monitor",
      "📷": "camera photo",
      "📞": "phone call telephone",
      "🔋": "battery power",
      "💡": "bulb idea light",
      "💰": "money bag rich",
      "💳": "card credit payment",
      "💎": "diamond gem",
      "🔧": "wrench tool",
      "🔨": "hammer tool",
      "⚙️": "gear settings",
      "🎁": "gift present",
      "🎈": "balloon party",
      "🎉": "party celebration confetti",
      "📧": "email mail",
      "📦": "package box delivery",
      "📅": "calendar date",
      "📎": "paperclip attachment",
      "📝": "memo note write",
      "✏️": "pencil write",
      "🔍": "search magnify",
      "🔒": "lock secure security",
      "🔓": "unlock security",
      "🔑": "key password",

      "✅": "check correct yes done success",
      "❌": "cross wrong no cancel",
      "⚠️": "warning caution alert",
      "❗": "exclamation important",
      "❓": "question help",
      "💯": "hundred perfect score",
      "♻️": "recycle",
      "➡️": "right arrow next",
      "⬅️": "left arrow back",
      "⬆️": "up arrow",
      "⬇️": "down arrow",

      // "🇮🇳": "india indian flag",
      // "🇺🇸": "usa united states america flag",
      // "🇬🇧": "uk united kingdom britain flag",
      // "🇨🇦": "canada flag",
      // "🇦🇺": "australia flag",
      // "🇯🇵": "japan flag",
      // "🇰🇷": "south korea korean flag",
      // "🇨🇳": "china chinese flag",
      // "🇸🇬": "singapore flag",
      // "🇦🇪": "uae united arab emirates dubai flag",
      // "🇫🇷": "france french flag",
      // "🇩🇪": "germany german flag",
      // "🇮🇹": "italy italian flag",
      // "🇪🇸": "spain spanish flag",
      // "🇧🇷": "brazil flag",
      // "🇳🇵": "nepal flag",
    };

    /* =====================================================
   RECENTLY USED EMOJIS
===================================================== */

    const CHAT_RECENT_EMOJIS_KEY = "servicecall_chat_recent_emojis";

    function getRecentChatEmojis() {
      try {
        const stored = localStorage.getItem(CHAT_RECENT_EMOJIS_KEY);

        if (!stored) {
          return [];
        }

        const parsed = JSON.parse(stored);

        if (!Array.isArray(parsed)) {
          return [];
        }

        return parsed.slice(0, 24);
      } catch (error) {
        return [];
      }
    }

    function saveRecentChatEmoji(emoji) {
      const recent = getRecentChatEmojis();

      /*
       * Remove it first so the newest use
       * always moves to the front.
       */
      const updated = [emoji, ...recent.filter((item) => item !== emoji)].slice(
        0,
        24,
      );

      try {
        localStorage.setItem(CHAT_RECENT_EMOJIS_KEY, JSON.stringify(updated));
      } catch (error) {
        // Do not break Chat if storage fails.
      }
    }

    /* =====================================================
   PICKER SHELL
===================================================== */

    picker.innerHTML = `
  <div style="
    padding:14px 14px 10px;
    border-bottom:1px solid #edf1f0;
    background:#ffffff;
  ">

    <div style="
      display:flex;
      align-items:center;
      justify-content:space-between;
      margin-bottom:11px;
    ">
      <div style="
        font-size:14px;
        font-weight:750;
        color:#29463f;
      ">
        Emojis
      </div>

      <button
        id="chatEmojiPickerClose"
        type="button"
        title="Close"
        style="
          width:28px;
          height:28px;
          border:none;
          border-radius:8px;
          background:transparent;
          color:#71827d;
          font-size:19px;
          cursor:pointer;
        "
      >
        ×
      </button>
    </div>

    <div style="
      position:relative;
    ">
      <span style="
        position:absolute;
        left:11px;
        top:50%;
        transform:translateY(-50%);
        pointer-events:none;
        font-size:13px;
      ">
        🔍
      </span>

      <input
        id="chatEmojiSearch"
        type="text"
        autocomplete="off"
        placeholder="Search emojis..."
        style="
          width:100%;
          height:36px;
          padding:0 11px 0 34px;
          border:1px solid #d6e0dd;
          border-radius:9px;
          outline:none;
          background:#f9fbfa;
          color:#29463f;
          font-size:12px;
        "
      >
    </div>

    <div
      id="chatEmojiCategories"
      style="
        display:flex;
        align-items:center;
        justify-content:space-between;
        gap:3px;
        margin-top:10px;
      "
    ></div>

  </div>

  <div
    id="chatEmojiContent"
    style="
      flex:1;
      min-height:0;
      overflow-y:auto;
      padding:10px 10px 14px;
      scroll-behavior:smooth;
    "
  ></div>
`;

    /* =====================================================
   ELEMENT REFERENCES
===================================================== */

    const searchInput = picker.querySelector("#chatEmojiSearch");

    const categoriesElement = picker.querySelector("#chatEmojiCategories");

    const contentElement = picker.querySelector("#chatEmojiContent");

    const closeButton = picker.querySelector("#chatEmojiPickerClose");

    /* =====================================================
   INSERT EMOJI AT CURRENT CURSOR
===================================================== */

    function insertEmojiIntoComposer(emoji) {
      if (!chatMessageInput || chatMessageInput.disabled) {
        return;
      }

      const value = String(chatMessageInput.value || "");

      const start =
        typeof chatMessageInput.selectionStart === "number"
          ? chatMessageInput.selectionStart
          : value.length;

      const end =
        typeof chatMessageInput.selectionEnd === "number"
          ? chatMessageInput.selectionEnd
          : start;

      chatMessageInput.value = value.slice(0, start) + emoji + value.slice(end);

      const newCursorPosition = start + emoji.length;

      chatMessageInput.focus();

      chatMessageInput.setSelectionRange(newCursorPosition, newCursorPosition);

      /*
       * Reuse your existing composer input
       * handling for resize + Send/Save state.
       */
      chatMessageInput.dispatchEvent(
        new Event("input", {
          bubbles: true,
        }),
      );
    }

    /* =====================================================
   RENDER EMOJIS
===================================================== */

    function renderEmojiSections(searchText = "") {
      const normalizedSearch = String(searchText || "")
        .trim()
        .toLowerCase();

      contentElement.innerHTML = "";

      let visibleEmojiCount = 0;

      /*
       * Build the list that will be rendered.
       * Recently Used appears first when
       * we have at least one saved emoji.
       */
      const recentEmojis = getRecentChatEmojis();

      const categoriesToRender = [
        ...(recentEmojis.length
          ? [
              {
                id: "recent",
                icon: "🕘",
                title: "Recently Used",
                emojis: recentEmojis,
              },
            ]
          : []),

        ...emojiCategories,
      ];

      categoriesToRender.forEach((category) => {
        /*
         * Search currently matches category names.
         *
         * Searching "food", "heart", "flag",
         * "animal", etc. therefore works.
         */
        let filteredEmojis = category.emojis;

        if (normalizedSearch) {
          filteredEmojis = category.emojis.filter((emoji) => {
            const keywords = String(
              emojiSearchKeywords[emoji] || "",
            ).toLowerCase();

            return (
              category.title.toLowerCase().includes(normalizedSearch) ||
              keywords.includes(normalizedSearch)
            );
          });
        }

        if (filteredEmojis.length === 0) {
          return;
        }

        visibleEmojiCount += filteredEmojis.length;

        const section = document.createElement("div");

        section.id = `chatEmojiSection-${category.id}`;

        section.style.cssText = `
        margin-bottom:14px;
        scroll-margin-top:8px;
      `;

        const title = document.createElement("div");

        title.textContent = category.title;

        title.style.cssText = `
        position:sticky;
        top:-10px;
        z-index:2;

        padding:
          7px 3px 6px;

        margin-bottom:4px;

        background:
          rgba(255,255,255,0.96);

        color:#60736e;

        font-size:11px;
        font-weight:700;
      `;

        const grid = document.createElement("div");

        grid.style.cssText = `
        display:grid;
        grid-template-columns:
          repeat(8, 1fr);
        gap:3px;
      `;

        filteredEmojis.forEach((emoji) => {
          const emojiButton = document.createElement("button");

          emojiButton.type = "button";

          emojiButton.textContent = emoji;

          emojiButton.title = emoji;

          emojiButton.style.cssText = `
            width:38px;
            height:38px;

            display:flex;
            align-items:center;
            justify-content:center;

            padding:0;

            border:none;
            border-radius:9px;

            background:transparent;

            font-size:22px;
            line-height:1;

            cursor:pointer;

            transition:
              background 0.12s ease,
              transform 0.12s ease;
          `;

          emojiButton.addEventListener("mouseenter", () => {
            emojiButton.style.background = "#edf4f2";

            emojiButton.style.transform = "scale(1.12)";
          });

          emojiButton.addEventListener("mouseleave", () => {
            emojiButton.style.background = "transparent";

            emojiButton.style.transform = "scale(1)";
          });

          emojiButton.addEventListener("click", (event) => {
            event.stopPropagation();

            /*
             * Insert into the message composer.
             */
            insertEmojiIntoComposer(emoji);

            /*
             * Remember it for Recently Used.
             */
            saveRecentChatEmoji(emoji);

            /*
             * Refresh the picker immediately so
             * Recently Used updates live.
             *
             * Do not refresh while searching,
             * otherwise the current search results
             * would be disturbed.
             */
            if (!String(searchInput.value || "").trim()) {
              renderEmojiSections("");
            }
          });

          document.addEventListener("keydown", (event) => {
            if (event.key !== "Escape") {
              return;
            }

            const picker = document.getElementById("chatComposerEmojiPicker");

            if (!picker) {
              return;
            }

            picker.remove();

            if (chatMessageInput && !chatMessageInput.disabled) {
              chatMessageInput.focus();
            }
          });
          grid.appendChild(emojiButton);
        });

        section.appendChild(title);

        section.appendChild(grid);

        contentElement.appendChild(section);
      });

      if (visibleEmojiCount === 0) {
        const empty = document.createElement("div");

        empty.textContent = "No emojis found";

        empty.style.cssText = `
      padding:45px 20px;
      text-align:center;
      color:#82918d;
      font-size:12px;
    `;

        contentElement.appendChild(empty);
      }
    }

    /* =====================================================
   CATEGORY BUTTONS
===================================================== */

    const recentForNavigation = getRecentChatEmojis();

    const navigationCategories = [
      ...(recentForNavigation.length
        ? [
            {
              id: "recent",
              icon: "🕘",
              title: "Recently Used",
            },
          ]
        : []),

      ...emojiCategories,
    ];

    navigationCategories.forEach((category) => {
      const categoryButton = document.createElement("button");

      categoryButton.type = "button";

      categoryButton.textContent = category.icon;

      categoryButton.title = category.title;

      categoryButton.style.cssText = `
      width:31px;
      height:31px;

      display:flex;
      align-items:center;
      justify-content:center;

      padding:0;

      border:none;
      border-radius:8px;

      background:transparent;

      font-size:17px;
      cursor:pointer;
    `;

      categoryButton.addEventListener("mouseenter", () => {
        categoryButton.style.background = "#edf4f2";
      });

      categoryButton.addEventListener("mouseleave", () => {
        categoryButton.style.background = "transparent";
      });

      categoryButton.addEventListener("click", (event) => {
        event.stopPropagation();

        /*
         * Clear search so all sections exist.
         */
        searchInput.value = "";

        renderEmojiSections();

        const targetSection = contentElement.querySelector(
          `#chatEmojiSection-${category.id}`,
        );

        if (targetSection) {
          targetSection.scrollIntoView({
            behavior: "smooth",
            block: "start",
          });
        }
      });

      categoriesElement.appendChild(categoryButton);
    });

    /* =====================================================
   SEARCH
===================================================== */

    searchInput.addEventListener("input", () => {
      renderEmojiSections(searchInput.value);
    });

    /* =====================================================
   CLOSE
===================================================== */

    closeButton.addEventListener("click", (event) => {
      event.stopPropagation();

      picker.remove();

      if (chatMessageInput && !chatMessageInput.disabled) {
        chatMessageInput.focus();
      }
    });

    /* =====================================================
   FIRST RENDER
===================================================== */

    renderEmojiSections();

    document.body.appendChild(picker);

    /*
     * Position picker above the 😊 button.
     */
    const buttonRect = chatEmojiButton.getBoundingClientRect();

    const pickerWidth = 380;
    const pickerHeight = 420;
    const gap = 10;

    let left = buttonRect.left;

    let top = buttonRect.top - pickerHeight - gap;

    /*
     * Keep it inside the application window.
     */
    if (left + pickerWidth > window.innerWidth - 12) {
      left = window.innerWidth - pickerWidth - 12;
    }

    if (left < 12) {
      left = 12;
    }

    if (top < 12) {
      top = buttonRect.bottom + gap;
    }

    picker.style.left = `${left}px`;

    picker.style.top = `${top}px`;

    /*
     * Prevent clicks inside the picker from
     * reaching the document click handler.
     */
    picker.addEventListener("click", (event) => {
      event.stopPropagation();
    });
  }

  if (chatEmojiButton) {
    chatEmojiButton.addEventListener("click", (event) => {
      event.stopPropagation();

      toggleChatEmojiPicker();
    });
  }

  document.addEventListener("click", (event) => {
    const picker = document.getElementById("chatComposerEmojiPicker");

    if (
      picker &&
      !picker.contains(event.target) &&
      event.target !== chatEmojiButton
    ) {
      picker.remove();
    }
  });

  const chatMembershipMessage = document.getElementById(
    "chatMembershipMessage",
  );

  const chatSendButton = document.getElementById("chatSendButton");
  let activeChatConversation = null;
  let lastChatMessageSysId = "";

  let lastChatEditCheckpoint = "";
  /*
   * Older-message pagination state.
   *
   * These belong only to the currently
   * open conversation.
   */
  let oldestChatMessageSysId = "";

  let chatMessageHistoryHasMore = false;

  let chatMessageHistoryLoading = false;
  const chatMessageCache = new Map();
  /*
   * =========================================
   * NEW MESSAGE SCROLL STATE
   * =========================================
   */

  let chatNewMessageCount = 0;

  let chatUserWasNearBottom = true;

  function clearChatMessageSelection() {
    chatMessageSelectionMode = "";

    selectedChatMessages.clear();

    if (!chatMessages) {
      return;
    }

    chatMessages
      .querySelectorAll(".chat-message-row[data-message-selected='true']")
      .forEach((row) => {
        row.dataset.messageSelected = "false";

        row.style.background = "";
        row.style.borderRadius = "";
        row.style.boxShadow = "";
        row.style.paddingTop = "";
        row.style.paddingBottom = "";
      });

    const selectionBar = document.getElementById("chatMessageSelectionBar");

    if (selectionBar) {
      selectionBar.remove();
    }
  }

  function selectChatMessage(messageRow, message) {
    if (!messageRow || !message || !message.sys_id) {
      return;
    }

    const messageSysId = String(message.sys_id).trim();

    if (!messageSysId) {
      return;
    }

    /*
     * System messages and globally deleted
     * tombstones cannot be selected.
     */
    if (
      String(message.type || "").toLowerCase() === "system" ||
      message.deleted === true
    ) {
      return;
    }

    selectedChatMessages.set(messageSysId, {
      sys_id: messageSysId,

      is_mine: message.is_mine === true,

      type: String(message.type || "text"),

      text: String(message.text || ""),
    });

    messageRow.dataset.messageSelected = "true";

    /*
     * Temporary functional selection styling.
     * Final styling/animation comes during
     * the final UI phase.
     */
    messageRow.style.background = "rgba(79, 143, 125, 0.10)";

    messageRow.style.borderRadius = "10px";

    messageRow.style.boxShadow = message.is_mine
      ? "inset 3px 0 0 rgba(79, 143, 125, 0.65)"
      : "inset -3px 0 0 rgba(79, 143, 125, 0.65)";

    messageRow.style.paddingTop = "10px";
    messageRow.style.paddingBottom = "10px";

    updateChatMessageSelectionUI();
  }

  function openChatForwardModal() {
    /*
     * Never allow two Forward modals.
     */
    const existingModal = document.getElementById("chatForwardModal");

    if (existingModal) {
      existingModal.remove();
    }

    /*
     * Only:
     *
     * - Direct conversations
     * - Groups where I am still an active member
     *
     * Historical groups are intentionally excluded.
     */
    const destinations = loadedChatConversations.filter((conversation) => {
      if (!conversation || !conversation.sys_id) {
        return false;
      }

      const type = String(conversation.type || "")
        .trim()
        .toLowerCase();

      if (type === "direct") {
        return true;
      }

      if (type === "group" && conversation.membership_active === true) {
        return true;
      }

      return false;
    });

    /*
     * MULTIPLE destinations are supported.
     *
     * conversation sys_id -> conversation
     */
    const selectedDestinations = new Map();

    /* =========================================
     OVERLAY
  ========================================= */

    const overlay = document.createElement("div");

    overlay.id = "chatForwardModal";

    overlay.style.cssText = `
    position:fixed;
    inset:0;

    z-index:5000;

    display:flex;
    align-items:center;
    justify-content:center;

    padding:24px;
    box-sizing:border-box;

    background:rgba(17, 29, 25, 0.38);
  `;

    /* =========================================
     MODAL
  ========================================= */

    const modal = document.createElement("div");

    modal.style.cssText = `
    width:min(460px, 92vw);
    max-height:72vh;

    display:flex;
    flex-direction:column;

    overflow:hidden;

    background:#ffffff;

    border:
      1px solid #dfe6e4;

    border-radius:16px;

    box-shadow:
      0 20px 60px
      rgba(0,0,0,0.22);
  `;

    /* =========================================
     HEADER
  ========================================= */

    const header = document.createElement("div");

    header.style.cssText = `
    display:flex;
    align-items:center;
    gap:12px;

    padding:16px 18px;

    border-bottom:
      1px solid #edf1ef;
  `;

    const title = document.createElement("div");

    title.textContent = "Forward message";

    title.style.cssText = `
    flex:1;

    font-size:16px;
    font-weight:700;

    color:#263632;
  `;

    const closeButton = document.createElement("button");

    closeButton.type = "button";
    closeButton.textContent = "✕";
    closeButton.title = "Close";

    closeButton.style.cssText = `
    width:34px;
    height:34px;

    border:none;
    border-radius:50%;

    background:transparent;

    color:#52615d;

    font-size:17px;

    cursor:pointer;
  `;

    header.appendChild(title);
    header.appendChild(closeButton);

    modal.appendChild(header);

    /* =========================================
     SEARCH
  ========================================= */

    const searchWrapper = document.createElement("div");

    searchWrapper.style.cssText = `
    padding:12px 14px;
  `;

    const searchInput = document.createElement("input");

    searchInput.type = "text";

    searchInput.placeholder = "Search chats or groups...";

    searchInput.style.cssText = `
    width:100%;

    padding:10px 12px;

    box-sizing:border-box;

    border:
      1px solid #d7e0dd;

    border-radius:10px;

    outline:none;

    font-size:13px;

    color:#263632;
    background:#ffffff;
  `;

    searchWrapper.appendChild(searchInput);

    modal.appendChild(searchWrapper);

    /* =========================================
     DESTINATION LIST
  ========================================= */

    const destinationList = document.createElement("div");

    destinationList.style.cssText = `
    flex:1;

    min-height:180px;
    max-height:330px;

    overflow-y:auto;

    border-top:
      1px solid #f1f3f2;

    border-bottom:
      1px solid #edf1ef;
  `;

    modal.appendChild(destinationList);

    /* =========================================
     FOOTER
  ========================================= */

    const footer = document.createElement("div");

    footer.style.cssText = `
    display:flex;
    align-items:center;
    gap:10px;

    padding:13px 16px;
  `;

    const selectedText = document.createElement("div");

    selectedText.style.cssText = `
    flex:1;

    font-size:12px;
    font-weight:600;

    color:#65736f;
  `;

    const cancelButton = document.createElement("button");

    cancelButton.type = "button";
    cancelButton.textContent = "Cancel";

    cancelButton.style.cssText = `
    padding:9px 15px;

    border:
      1px solid #d6dfdc;

    border-radius:9px;

    background:#ffffff;

    color:#344641;

    font-size:13px;
    font-weight:600;

    cursor:pointer;
  `;

    const forwardButton = document.createElement("button");

    forwardButton.type = "button";
    forwardButton.textContent = "Forward";

    forwardButton.style.cssText = `
    padding:9px 17px;

    border:none;
    border-radius:9px;

    background:#32c7a0;

    color:#ffffff;

    font-size:13px;
    font-weight:700;

    cursor:pointer;
  `;

    footer.appendChild(selectedText);
    footer.appendChild(cancelButton);
    footer.appendChild(forwardButton);

    modal.appendChild(footer);

    overlay.appendChild(modal);

    document.body.appendChild(overlay);

    /* =========================================
     UPDATE FOOTER
  ========================================= */

    function updateForwardDestinationState() {
      const count = selectedDestinations.size;

      selectedText.textContent =
        count === 0
          ? "No chats selected"
          : count === 1
            ? "1 chat selected"
            : `${count} chats selected`;

      forwardButton.disabled = count === 0;

      forwardButton.style.opacity = count === 0 ? "0.45" : "1";

      forwardButton.style.cursor = count === 0 ? "default" : "pointer";
    }

    /* =========================================
     RENDER DESTINATIONS
  ========================================= */

    function renderDestinations(filterText = "") {
      destinationList.innerHTML = "";

      const normalizedFilter = String(filterText || "")
        .trim()
        .toLowerCase();

      const filteredDestinations = destinations.filter((conversation) => {
        const name = String(
          conversation.display_name || conversation.title || "",
        ).toLowerCase();

        return !normalizedFilter || name.includes(normalizedFilter);
      });

      if (filteredDestinations.length === 0) {
        const empty = document.createElement("div");

        empty.textContent = normalizedFilter
          ? "No matching chats."
          : "No chats available.";

        empty.style.cssText = `
        padding:28px 16px;

        text-align:center;

        font-size:12px;
        color:#71827d;
      `;

        destinationList.appendChild(empty);

        return;
      }

      filteredDestinations.forEach((conversation) => {
        const conversationSysId = String(conversation.sys_id);

        const type = String(conversation.type || "").toLowerCase();

        const displayName = String(
          conversation.display_name ||
            conversation.title ||
            (type === "group" ? "Group" : "Unknown User"),
        );

        const row = document.createElement("button");

        row.type = "button";

        row.style.cssText = `
          width:100%;

          display:flex;
          align-items:center;
          gap:12px;

          padding:11px 15px;

          box-sizing:border-box;

          border:none;
          border-bottom:
            1px solid #f0f3f2;

          background:#ffffff;

          text-align:left;

          cursor:pointer;
        `;

        /* -------------------------
           SELECTION CIRCLE
        ------------------------- */

        const selector = document.createElement("div");

        selector.style.cssText = `
          width:20px;
          height:20px;
          min-width:20px;

          display:flex;
          align-items:center;
          justify-content:center;

          border:
            1.5px solid #aab8b4;

          border-radius:50%;

          font-size:12px;
          font-weight:800;
        `;

        /* -------------------------
           AVATAR
        ------------------------- */

        const avatar = document.createElement("div");

        avatar.style.cssText = `
          width:38px;
          height:38px;
          min-width:38px;

          display:flex;
          align-items:center;
          justify-content:center;

          border-radius:50%;

          background:#dff3ec;
          color:#17634f;

          font-size:12px;
          font-weight:800;
        `;

        if (type === "group") {
          avatar.textContent = "G";
        } else {
          const parts = displayName.trim().split(/\s+/).filter(Boolean);

          avatar.textContent =
            parts.length >= 2
              ? (parts[0][0] + parts[parts.length - 1][0]).toUpperCase()
              : displayName.charAt(0).toUpperCase() || "?";
        }

        /* -------------------------
           INFORMATION
        ------------------------- */

        const information = document.createElement("div");

        information.style.cssText = `
          flex:1;
          min-width:0;
        `;

        const name = document.createElement("div");

        name.textContent = displayName;

        name.style.cssText = `
          font-size:13px;
          font-weight:650;

          color:#263632;

          white-space:nowrap;
          overflow:hidden;
          text-overflow:ellipsis;
        `;

        const subtype = document.createElement("div");

        subtype.textContent = type === "group" ? "Group" : "Direct chat";

        subtype.style.cssText = `
          margin-top:2px;

          font-size:11px;
          color:#7a8783;
        `;

        information.appendChild(name);
        information.appendChild(subtype);

        row.appendChild(selector);
        row.appendChild(avatar);
        row.appendChild(information);

        function refreshRowSelection() {
          const selected = selectedDestinations.has(conversationSysId);

          selector.textContent = selected ? "✓" : "";

          selector.style.background = selected ? "#32c7a0" : "#ffffff";

          selector.style.borderColor = selected ? "#32c7a0" : "#aab8b4";

          selector.style.color = selected ? "#ffffff" : "transparent";

          row.style.background = selected ? "#eef9f5" : "#ffffff";
        }

        row.addEventListener("click", (event) => {
          event.preventDefault();
          event.stopPropagation();

          if (selectedDestinations.has(conversationSysId)) {
            selectedDestinations.delete(conversationSysId);
          } else {
            selectedDestinations.set(conversationSysId, conversation);
          }

          refreshRowSelection();

          updateForwardDestinationState();
        });

        refreshRowSelection();

        destinationList.appendChild(row);
      });
    }

    /* =========================================
     SEARCH EXISTING DESTINATIONS
  ========================================= */

    searchInput.addEventListener("input", () => {
      renderDestinations(searchInput.value);
    });

    /* =========================================
     CLOSE
  ========================================= */

    function closeForwardModal() {
      overlay.remove();
    }

    closeButton.addEventListener("click", closeForwardModal);

    cancelButton.addEventListener("click", closeForwardModal);

    overlay.addEventListener("click", (event) => {
      if (event.target === overlay) {
        closeForwardModal();
      }
    });

    /* =========================================
     FORWARD BUTTON

     Backend comes NEXT.
  ========================================= */

    forwardButton.addEventListener("click", async (event) => {
      event.preventDefault();
      event.stopPropagation();

      if (selectedDestinations.size === 0) {
        return;
      }

      if (selectedChatMessages.size === 0) {
        return;
      }

      /*
       * Prevent duplicate forwarding if the
       * user clicks Forward repeatedly.
       */
      if (forwardButton.disabled) {
        return;
      }

      /*
       * Extract only the source message sys_ids.
       */
      const messageSysIds = Array.from(selectedChatMessages.values())
        .map((message) =>
          String(
            message && (message.sys_id || message.message_sys_id || ""),
          ).trim(),
        )
        .filter(Boolean);

      /*
       * Extract destination conversation sys_ids.
       */
      const destinationConversationIds = Array.from(selectedDestinations.keys())
        .map((value) => String(value || "").trim())
        .filter(Boolean);

      if (messageSysIds.length === 0) {
        console.error("No valid messages selected for forwarding.");

        return;
      }

      if (destinationConversationIds.length === 0) {
        console.error("No valid destinations selected for forwarding.");

        return;
      }

      /*
       * Lock the action immediately.
       */
      forwardButton.disabled = true;
      forwardButton.textContent = "Forwarding...";
      forwardButton.style.opacity = "0.65";
      forwardButton.style.cursor = "default";

      cancelButton.disabled = true;
      closeButton.disabled = true;
      searchInput.disabled = true;

      try {
        const result = await window.serviceCall.forwardMessages(
          messageSysIds,
          destinationConversationIds,
        );

        if (!result || result.success !== true) {
          throw new Error(
            result && result.message
              ? result.message
              : "Unable to forward the selected messages.",
          );
        }

        console.log("Messages forwarded successfully:", result);

        /*
         * Forward succeeded.
         *
         * Remove the modal first.
         */
        closeForwardModal();

        /*
         * Exit shared message-selection mode.
         */
        clearChatMessageSelection();

        /*
         * Immediately refresh the rendered
         * selection state.
         *
         * This hides the selection circles on the
         * currently-open conversation without
         * requiring the user to change chats.
         */
        updateChatMessageSelectionUI();

        /*
         * Silently refresh the sidebar so all
         * destination previews/order can update
         * without showing the normal loading state.
         */
        try {
          await syncChatConversationList();
        } catch (refreshError) {
          console.error(
            "Unable to silently refresh conversations after forwarding:",
            refreshError,
          );
        }
      } catch (error) {
        console.error("Unable to forward messages:", error);

        /*
         * Keep modal + selections intact so the
         * user can retry.
         */
        forwardButton.disabled = false;
        forwardButton.textContent = "Forward";
        forwardButton.style.opacity = "1";
        forwardButton.style.cursor = "pointer";

        cancelButton.disabled = false;
        closeButton.disabled = false;
        searchInput.disabled = false;
      }
    });

    renderDestinations();

    updateForwardDestinationState();

    setTimeout(() => {
      searchInput.focus();
    }, 50);
  }

  function updateChatMessageSelectionUI() {
    const selectedCount = selectedChatMessages.size;

    /*
     * Update every rendered message's
     * selection indicator.
     */
    if (chatMessages) {
      chatMessages.querySelectorAll(".chat-message-row").forEach((row) => {
        const messageSysId = String(row.dataset.messageSysId || "").trim();

        const indicator = row.querySelector(
          ".chat-message-selection-indicator",
        );

        if (!indicator) {
          return;
        }

        /*
         * Deleted-for-everyone tombstones are
         * never selectable.
         *
         * Their selection indicator must stay
         * completely hidden even while selection
         * mode is active.
         */
        if (row.dataset.messageDeleted === "true") {
          indicator.style.display = "none";
          indicator.style.opacity = "0";
          indicator.style.pointerEvents = "none";

          row.style.cursor = "";

          return;
        }

        indicator.style.display = "";

        row.style.cursor =
          chatMessageSelectionMode && messageSysId ? "pointer" : "";

        const isSelected = selectedChatMessages.has(messageSysId);

        indicator.textContent = isSelected ? "✓" : "";

        indicator.style.background = isSelected ? "#32c7a0" : "#ffffff";

        indicator.style.borderColor = isSelected ? "#32c7a0" : "#aab8b4";

        indicator.style.color = isSelected ? "#ffffff" : "transparent";

        indicator.style.opacity = chatMessageSelectionMode ? "1" : "0";

        indicator.style.pointerEvents = chatMessageSelectionMode
          ? "auto"
          : "none";
      });
    }

    /*
     * Selection bar.
     */
    const existingBar = document.getElementById("chatMessageSelectionBar");

    if (!chatMessageSelectionMode || selectedCount === 0) {
      if (existingBar) {
        existingBar.remove();
      }

      return;
    }

    let selectionBar = existingBar;

    if (!selectionBar) {
      selectionBar = document.createElement("div");

      selectionBar.id = "chatMessageSelectionBar";

      selectionBar.style.cssText = `
      position:absolute;
      left:0;
      right:0;
      bottom:0;
      z-index:1500;

      min-height:54px;
      padding:8px 16px;

      display:flex;
      align-items:center;
      gap:12px;

      box-sizing:border-box;

      background:#ffffff;

      border-top:
        1px solid #dfe6e4;

      box-shadow:
        0 -6px 20px
        rgba(0,0,0,0.08);
    `;

      /*
       * CLOSE / CANCEL SELECTION
       */
      const closeButton = document.createElement("button");

      closeButton.type = "button";
      closeButton.textContent = "✕";
      closeButton.title = "Cancel selection";

      closeButton.style.cssText = `
      width:36px;
      height:36px;
      min-width:36px;

      border:none;
      border-radius:50%;

      background:transparent;
      color:#45534f;

      font-size:18px;

      cursor:pointer;
    `;

      closeButton.addEventListener("click", () => {
        clearChatMessageSelection();

        updateChatMessageSelectionUI();
      });

      /*
       * SELECTED COUNT
       */
      const countText = document.createElement("div");

      countText.className = "chat-message-selection-count";

      countText.style.cssText = `
      flex:1;

      font-size:13px;
      font-weight:600;

      color:#263632;
    `;

      /*
       * ACTION BUTTON
       *
       * Delete mode -> trash
       * Forward mode -> arrow
       */
      const actionButton = document.createElement("button");

      actionButton.type = "button";

      actionButton.className = "chat-message-selection-action";

      actionButton.style.cssText = `
      width:38px;
      height:38px;
      min-width:38px;

      border:none;
      border-radius:50%;

      background:transparent;
      color:#45534f;

      font-size:20px;

      cursor:pointer;
    `;

      /*
       * SELECTION ACTION
       *
       * Delete mode:
       *   - All selected messages are mine:
       *       Delete for everyone
       *       Delete for me
       *       Cancel
       *
       *   - At least one selected message
       *     belongs to somebody else:
       *       Delete for me
       *       Cancel
       *
       * Forward mode will be wired later.
       */
      actionButton.addEventListener("click", (event) => {
        event.preventDefault();
        event.stopPropagation();

        if (selectedChatMessages.size === 0) {
          return;
        }

        /*
         * Forward will use this same
         * action button later.
         */
        if (chatMessageSelectionMode === "forward") {
          openChatForwardModal();
          return;
        }

        /*
         * Historical group conversations
         * remain read-only.
         */
        if (isActiveChatReadOnly()) {
          clearChatMessageSelection();

          updateChatMessageSelectionUI();

          showChatMembershipError();

          return;
        }

        /*
         * Close any previous Delete
         * choice popup.
         */
        document
          .querySelectorAll(".chat-selection-delete-menu")
          .forEach((existingMenu) => {
            existingMenu.remove();
          });

        const selectedMessages = Array.from(selectedChatMessages.values());

        /*
         * Delete for everyone is available
         * ONLY when every selected message
         * belongs to the current user.
         */
        const allMessagesAreMine =
          selectedMessages.length > 0 &&
          selectedMessages.every(
            (selectedMessage) => selectedMessage.is_mine === true,
          );

        /* =========================================
       DELETE CHOICE MENU
    ========================================= */

        const deleteMenu = document.createElement("div");

        deleteMenu.className = "chat-selection-delete-menu";

        deleteMenu.style.cssText = `
      position:absolute;
      right:12px;
      bottom:58px;

      z-index:1600;

      min-width:190px;

      padding:6px;

      display:flex;
      flex-direction:column;
      gap:2px;

      box-sizing:border-box;

      border:
        1px solid #dfe6e4;

      border-radius:12px;

      background:#ffffff;

      box-shadow:
        0 10px 30px
        rgba(0,0,0,0.16);
    `;

        /*
         * Small helper so all menu
         * buttons behave consistently.
         */
        function createDeleteChoiceButton(label, destructive = false) {
          const button = document.createElement("button");

          button.type = "button";

          button.textContent = label;

          button.style.cssText = `
        width:100%;

        border:none;
        border-radius:8px;

        background:transparent;

        padding:9px 11px;

        text-align:left;

        font-size:12px;

        color:${destructive ? "#b42318" : "#60706c"};

        cursor:pointer;
      `;

          button.addEventListener("mouseenter", () => {
            button.style.background = destructive ? "#fff1f0" : "#eef4f2";
          });

          button.addEventListener("mouseleave", () => {
            button.style.background = "transparent";
          });

          return button;
        }

        /* =========================================
       DELETE FOR EVERYONE
    ========================================= */

        if (allMessagesAreMine) {
          const deleteEveryoneButton = createDeleteChoiceButton(
            "Delete for everyone",
            true,
          );

          deleteEveryoneButton.addEventListener(
            "click",
            async (deleteEvent) => {
              deleteEvent.preventDefault();
              deleteEvent.stopPropagation();

              /*
               * Backend execution comes
               * in the NEXT step.
               */
              const selectedMessageSysIds = selectedMessages
                .map((selectedMessage) =>
                  String(selectedMessage.sys_id || "").trim(),
                )
                .filter(Boolean);

              if (selectedMessageSysIds.length === 0) {
                return;
              }

              deleteEveryoneButton.disabled = true;
              deleteEveryoneButton.textContent = "Deleting...";

              try {
                const conversationSysId = String(
                  activeChatConversation?.sys_id || "",
                ).trim();

                const result = await window.serviceCall.deleteMessages(
                  conversationSysId,
                  selectedMessageSysIds,
                  "everyone",
                );

                if (!result || result.success !== true) {
                  console.warn("Multi Delete for everyone failed:", result);

                  deleteEveryoneButton.disabled = false;

                  deleteEveryoneButton.textContent = "Delete for everyone";

                  return;
                }

                /*
                 * Convert every affected rendered
                 * message into a deleted tombstone.
                 */
                selectedMessageSysIds.forEach((deletedMessageSysId) => {
                  const row = Array.from(
                    chatMessages.querySelectorAll(".chat-message-row"),
                  ).find(
                    (candidateRow) =>
                      String(candidateRow.dataset.messageSysId || "") ===
                      deletedMessageSysId,
                  );

                  if (!row) {
                    loadChatConversations();
                    return;
                  }

                  const selectedMessage =
                    selectedChatMessages.get(deletedMessageSysId);

                  if (selectedMessage) {
                    /*
                     * IMPORTANT:
                     * Keep the tombstone in the EXACT SAME
                     * position as the original message.
                     *
                     * appendChatMessage() normally adds to the
                     * bottom, so remember the next row first.
                     */
                    const nextRow = row.nextSibling;

                    row.remove();

                    appendChatMessage({
                      ...selectedMessage,
                      sys_id: deletedMessageSysId,
                      text: "",
                      deleted: true,
                      reactions: [],
                      my_reaction: "",
                    });

                    /*
                     * appendChatMessage() created the tombstone
                     * at the bottom. Find it and move it back
                     * into the original position.
                     */
                    const tombstoneRow = Array.from(
                      chatMessages.querySelectorAll(".chat-message-row"),
                    ).find(
                      (candidateRow) =>
                        String(candidateRow.dataset.messageSysId || "") ===
                        deletedMessageSysId,
                    );

                    if (tombstoneRow) {
                      /*
                       * Explicitly mark it as deleted so
                       * selection UI never shows a checkbox.
                       */
                      tombstoneRow.dataset.messageDeleted = "true";

                      if (nextRow && nextRow.parentElement === chatMessages) {
                        chatMessages.insertBefore(tombstoneRow, nextRow);
                      } else {
                        chatMessages.appendChild(tombstoneRow);
                      }

                      const deletedIndicator = tombstoneRow.querySelector(
                        ".chat-message-selection-indicator",
                      );

                      if (deletedIndicator) {
                        deletedIndicator.style.display = "none";

                        deletedIndicator.style.opacity = "0";

                        deletedIndicator.style.pointerEvents = "none";
                      }

                      tombstoneRow.style.cursor = "";
                    }
                  }
                });

                /*
                 * Cached history must not resurrect
                 * the old versions.
                 */
                if (activeChatConversation && activeChatConversation.sys_id) {
                  chatMessageCache.delete(
                    String(activeChatConversation.sys_id),
                  );
                }

                deleteMenu.remove();

                clearChatMessageSelection();

                updateChatMessageSelectionUI();

                /*
                 * Refresh sidebar so its preview
                 * immediately reflects the deletion.
                 */
                try {
                  await syncChatConversationList();
                } catch (refreshError) {
                  console.error(
                    "Unable to silently refresh conversations after multi-delete:",
                    refreshError,
                  );
                }
              } catch (error) {
                console.error(
                  "Unable to delete selected messages for everyone:",
                  error,
                );

                deleteEveryoneButton.disabled = false;

                deleteEveryoneButton.textContent = "Delete for everyone";
              }
            },
          );

          deleteMenu.appendChild(deleteEveryoneButton);
        }

        /* =========================================
       DELETE FOR ME
    ========================================= */

        const deleteForMeButton = createDeleteChoiceButton(
          "Delete for me",
          true,
        );

        deleteForMeButton.addEventListener("click", async (deleteEvent) => {
          deleteEvent.preventDefault();
          deleteEvent.stopPropagation();

          const selectedMessageSysIds = selectedMessages
            .map((selectedMessage) =>
              String(selectedMessage.sys_id || "").trim(),
            )
            .filter(Boolean);

          if (selectedMessageSysIds.length === 0) {
            return;
          }

          const conversationSysId = String(
            activeChatConversation?.sys_id || "",
          ).trim();

          if (!conversationSysId) {
            return;
          }

          deleteForMeButton.disabled = true;
          deleteForMeButton.textContent = "Deleting...";

          try {
            /*
             * Use the same existing backend used by
             * single-message Delete for me.
             *
             * The backend already accepts an array
             * of message sys_ids.
             */
            const result = await window.serviceCall.deleteMessages(
              conversationSysId,
              selectedMessageSysIds,
              "me",
            );

            if (!result || result.success !== true) {
              console.warn("Multi Delete for me failed:", result);

              deleteForMeButton.disabled = false;
              deleteForMeButton.textContent = "Delete for me";

              return;
            }

            /*
             * Remove ONLY the selected messages
             * from this user's rendered chat.
             *
             * Unlike Delete for everyone,
             * there is NO tombstone.
             */
            selectedMessageSysIds.forEach((deletedMessageSysId) => {
              const row = Array.from(
                chatMessages.querySelectorAll(".chat-message-row"),
              ).find(
                (candidateRow) =>
                  String(candidateRow.dataset.messageSysId || "") ===
                  deletedMessageSysId,
              );

              if (row) {
                row.remove();
              }
            });

            /*
             * Remove cached history so reopening
             * the conversation cannot restore
             * messages hidden for this user.
             */
            chatMessageCache.delete(conversationSysId);

            /*
             * Close popup and exit selection mode.
             */
            deleteMenu.remove();

            clearChatMessageSelection();

            updateChatMessageSelectionUI();

            /*
             * Refresh the sidebar because its
             * user-specific preview may have changed.
             */
            try {
              await syncChatConversationList();
            } catch (refreshError) {
              console.error(
                "Unable to silently refresh conversations after multi Delete for me:",
                refreshError,
              );
            }
          } catch (error) {
            console.error("Unable to delete selected messages for me:", error);

            deleteForMeButton.disabled = false;
            deleteForMeButton.textContent = "Delete for me";
          }
        });

        deleteMenu.appendChild(deleteForMeButton);

        /* =========================================
       CANCEL
    ========================================= */

        const cancelButton = createDeleteChoiceButton("Cancel", false);

        cancelButton.addEventListener("click", (cancelEvent) => {
          cancelEvent.preventDefault();
          cancelEvent.stopPropagation();

          deleteMenu.remove();
        });

        deleteMenu.appendChild(cancelButton);

        /*
         * Attach popup to our existing
         * selection bar.
         */
        selectionBar.appendChild(deleteMenu);

        /*
         * Click anywhere outside the popup
         * to close ONLY the popup.
         *
         * Message selection itself remains.
         */
        setTimeout(() => {
          function closeDeleteMenu(outsideEvent) {
            if (
              deleteMenu.contains(outsideEvent.target) ||
              actionButton.contains(outsideEvent.target)
            ) {
              return;
            }

            deleteMenu.remove();

            document.removeEventListener("click", closeDeleteMenu, true);
          }

          document.addEventListener("click", closeDeleteMenu, true);
        }, 0);
      });

      selectionBar.appendChild(closeButton);

      selectionBar.appendChild(countText);

      selectionBar.appendChild(actionButton);

      /*
       * Put the bar over the bottom of
       * the chat conversation area.
       */
      const conversationPanel = chatMessages.parentElement;

      if (conversationPanel) {
        const panelPosition =
          window.getComputedStyle(conversationPanel).position;

        if (!panelPosition || panelPosition === "static") {
          conversationPanel.style.position = "relative";
        }

        conversationPanel.appendChild(selectionBar);
      }
    }

    const countText = selectionBar.querySelector(
      ".chat-message-selection-count",
    );

    if (countText) {
      countText.textContent =
        selectedCount === 1 ? "1 selected" : `${selectedCount} selected`;
    }

    const actionButton = selectionBar.querySelector(
      ".chat-message-selection-action",
    );

    if (actionButton) {
      if (chatMessageSelectionMode === "delete") {
        actionButton.textContent = "🗑";
        actionButton.title = "Delete selected messages";
      } else {
        actionButton.textContent = "➜";
        actionButton.title = "Forward selected messages";
      }
    }
  }

  function toggleChatMessageSelection(messageRow, message) {
    if (!messageRow || !message || !message.sys_id) {
      return;
    }

    const messageSysId = String(message.sys_id).trim();

    if (!messageSysId) {
      return;
    }

    /*
     * System messages and deleted tombstones
     * cannot be selected.
     */
    if (
      String(message.type || "").toLowerCase() === "system" ||
      message.deleted === true
    ) {
      return;
    }

    /*
     * Already selected -> deselect it.
     */
    if (selectedChatMessages.has(messageSysId)) {
      selectedChatMessages.delete(messageSysId);

      messageRow.dataset.messageSelected = "false";

      messageRow.style.background = "";
      messageRow.style.borderRadius = "";
      messageRow.style.boxShadow = "";
      messageRow.style.paddingTop = "";
      messageRow.style.paddingBottom = "";

      /*
       * If nothing remains selected,
       * leave selection mode completely.
       */
      if (selectedChatMessages.size === 0) {
        chatMessageSelectionMode = "";
      }

      updateChatMessageSelectionUI();

      return;
    }

    /*
     * Not selected yet -> select it.
     */
    selectChatMessage(messageRow, message);
  }

  /* =====================================================
   CHAT MESSAGE MULTI-SELECTION

   Shared by:
   - Delete
   - Forward
===================================================== */

  let chatMessageSelectionMode = "";

  const selectedChatMessages = new Map();

  /*
   * =========================================
   * CHAT SCROLL POSITION
   * =========================================
   */

  /*
   * =========================================
   * NEW MESSAGE INDICATOR
   * =========================================
   */

  function updateChatNewMessagesButton() {
    if (!chatNewMessagesButton) {
      return;
    }

    if (chatNewMessageCount <= 0) {
      chatNewMessagesButton.style.display = "none";

      chatNewMessagesButton.textContent = "";

      return;
    }

    const label = chatNewMessageCount === 1 ? "new message" : "new messages";

    chatNewMessagesButton.textContent = `↓ ${chatNewMessageCount} ${label}`;

    chatNewMessagesButton.style.display = "block";
  }

  function isChatNearBottom() {
    if (!chatMessages) {
      return true;
    }

    const distanceFromBottom =
      chatMessages.scrollHeight -
      chatMessages.scrollTop -
      chatMessages.clientHeight;

    /*
     * Treat the user as being at the bottom
     * when they are within 80px of it.
     *
     * This avoids requiring pixel-perfect
     * scroll positioning.
     */
    return distanceFromBottom <= 80;
  }
  let lastChatReactionCheckpoint = "";

  /* -------------------------------------------------
   CHAT ELEMENTS
------------------------------------------------- */

  const chatPeopleSearchInput = document.getElementById(
    "chatPeopleSearchInput",
  );

  const chatPeopleSearchResults = document.getElementById(
    "chatPeopleSearchResults",
  );

  const chatConversationList = document.getElementById("chatConversationList");

  const chatNewMessagesButton = document.getElementById(
    "chatNewMessagesButton",
  );

  /*
   * =========================================
   * NEW MESSAGE BUTTON CLICK
   * =========================================
   */

  if (chatNewMessagesButton) {
    chatNewMessagesButton.addEventListener("click", async () => {
      if (
        !chatMessages ||
        !activeChatConversation ||
        !activeChatConversation.sys_id
      ) {
        return;
      }

      const conversationSysId = String(activeChatConversation.sys_id).trim();

      /*
       * Move to the newest messages.
       */
      chatMessages.scrollTo({
        top: chatMessages.scrollHeight,
        behavior: "smooth",
      });

      /*
       * New messages are now intentionally
       * being viewed by the user.
       */
      chatUserWasNearBottom = true;

      chatNewMessageCount = 0;

      updateChatNewMessagesButton();

      try {
        /*
         * Persist the read state in
         * ServiceNow.
         */
        const readResult =
          await window.serviceCall.markConversationRead(conversationSysId);

        /*
         * The user may have switched chats
         * while the request was running.
         */
        if (
          !activeChatConversation ||
          String(activeChatConversation.sys_id || "") !== conversationSysId
        ) {
          return;
        }

        if (!readResult || readResult.success !== true) {
          console.warn("Unable to mark conversation read:", readResult);

          return;
        }

        activeChatConversation.unread_count = 0;

        if (readResult.last_read_at) {
          activeChatConversation.last_read_at = String(readResult.last_read_at);
        }

        /*
         * Refresh sidebar immediately so its
         * unread state disappears too.
         */
        await syncChatConversationList();
      } catch (error) {
        console.error(
          "Unable to mark conversation read from new-message button:",
          error,
        );
      }
    });
  }

  const chatEmptyState = document.getElementById("chatEmptyState");

  const chatConversationPanel = document.getElementById(
    "chatConversationPanel",
  );

  const chatUserAvatar = document.getElementById("chatUserAvatar");

  const chatUserName = document.getElementById("chatUserName");

  const chatUserPresenceDot = document.getElementById("chatUserPresenceDot");

  const chatUserPresenceText = document.getElementById("chatUserPresenceText");

  const chatCallButton = document.getElementById("chatCallButton");

  const chatCalendarButton = document.getElementById("chatCalendarButton");

  const chatGroupDetailsButton = document.getElementById(
    "chatGroupDetailsButton",
  );

  const chatGroupDetailsModal = document.getElementById(
    "chatGroupDetailsModal",
  );

  const chatGroupDetailsBackdrop = document.getElementById(
    "chatGroupDetailsBackdrop",
  );

  const chatGroupDetailsCloseButton = document.getElementById(
    "chatGroupDetailsCloseButton",
  );

  const chatGroupDetailsName = document.getElementById("chatGroupDetailsName");

  const chatGroupDetailsMembers = document.getElementById(
    "chatGroupDetailsMembers",
  );

  /* -------------------------------------------------
   GROUP DETAILS - ADD PEOPLE
------------------------------------------------- */

  const chatGroupAddPeopleButton = document.getElementById(
    "chatGroupAddPeopleButton",
  );

  const chatGroupAddPeoplePanel = document.getElementById(
    "chatGroupAddPeoplePanel",
  );

  const chatGroupAddPeopleSearch = document.getElementById(
    "chatGroupAddPeopleSearch",
  );

  const chatGroupAddPeopleResults = document.getElementById(
    "chatGroupAddPeopleResults",
  );

  const chatGroupAddPeopleMessage = document.getElementById(
    "chatGroupAddPeopleMessage",
  );

  const chatGroupAddPeopleCancelButton = document.getElementById(
    "chatGroupAddPeopleCancelButton",
  );

  const chatGroupAddPeopleSaveButton = document.getElementById(
    "chatGroupAddPeopleSaveButton",
  );

  let chatGroupAddPeopleSearchTimer = null;

  const chatGroupAddPeopleSelectedUsers = new Map();

  let currentChatGroupDetails = null;

  let chatPeopleSearchTimer = null;

  let activeChatUser = null;

  /* =====================================================
   TOP BAR PRESENCE
===================================================== */

  const presenceButton = document.getElementById("presenceButton");

  const presenceMenu = document.getElementById("presenceMenu");

  const presenceText = document.getElementById("presenceText");

  const presenceDot = document.getElementById("presenceDot");

  const presenceOptions = document.querySelectorAll(".presence-option");

  const oofReasonPanel = document.getElementById("oofReasonPanel");

  const oofReasonInput = document.getElementById("oofReasonInput");

  const saveOofButton = document.getElementById("saveOofButton");

  const cancelOofButton = document.getElementById("cancelOofButton");

  let peopleSearchTimer = null;

  const connectionPill = document.getElementById("connectionPill");

  const currentPageTitle = document.getElementById("currentPageTitle");

  const meetingsContainer = document.getElementById("meetingsContainer");

  const scheduleMeetingButton = document.getElementById(
    "scheduleMeetingButton",
  );

  const meetingSearchInput = document.getElementById("meetingSearchInput");

  const meetingSearchClear = document.getElementById("meetingSearchClear");

  const meetingPagination = document.getElementById("meetingPagination");

  const meetingStatusFilter = document.getElementById("meetingStatusFilter");

  let loadedChatConversations = [];

  /* -------------------------------------------------
   NOTIFICATION ELEMENTS
------------------------------------------------- */

  const notificationsContainer = document.getElementById(
    "notificationsContainer",
  );

  const notificationPagination = document.getElementById(
    "notificationPagination",
  );

  const notificationUnreadBadge = document.getElementById(
    "notificationUnreadBadge",
  );

  const notificationUnreadFilterCount = document.getElementById(
    "notificationUnreadFilterCount",
  );

  const refreshNotificationsButton = document.getElementById(
    "refreshNotificationsButton",
  );

  const markAllNotificationsReadButton = document.getElementById(
    "markAllNotificationsReadButton",
  );

  const notificationFilterButtons = document.querySelectorAll(
    ".notification-filter",
  );

  const notificationsNavButton = document.querySelector(
    '[data-view="notificationsView"]',
  );

  const notificationSearchInput = document.getElementById(
    "notificationSearchInput",
  );

  const notificationSearchClear = document.getElementById(
    "notificationSearchClear",
  );

  /* -------------------------------------------------
   NOTIFICATION DETAIL ELEMENTS
------------------------------------------------- */

  const notificationListPanel = document.getElementById(
    "notificationListPanel",
  );

  const notificationDetailPanel = document.getElementById(
    "notificationDetailPanel",
  );

  const notificationDetailBackButton = document.getElementById(
    "notificationDetailBackButton",
  );

  const notificationDetailIcon = document.getElementById(
    "notificationDetailIcon",
  );

  const notificationDetailType = document.getElementById(
    "notificationDetailType",
  );

  const notificationDetailTitle = document.getElementById(
    "notificationDetailTitle",
  );

  const notificationDetailTime = document.getElementById(
    "notificationDetailTime",
  );

  const notificationDetailMessage = document.getElementById(
    "notificationDetailMessage",
  );

  const notificationDetailActions = document.getElementById(
    "notificationDetailActions",
  );

  let currentNotificationPage = 1;

  let currentNotificationFilter = "all";

  let notificationAutoRefreshTimer = null;

  let currentNotificationSearch = "";

  let notificationSearchTimer = null;

  let knownNotificationIds = new Set();

  let chatMessageSyncTimer = null;

  let chatMessageSyncRunning = false;

  /*
   * Notifications currently loaded from ServiceNow.
   *
   * Filters can use this immediately without
   * making another API request.
   */
  let cachedNotifications = [];

  let cachedNotificationResult = null;

  let notificationsInitialized = false;

  let notificationSearchVersion = 0;

  let currentMeetingPage = 1;

  let currentMeetingSearch = "";

  let currentMeetingStatus = "";

  let meetingSearchTimer = null;

  let schedulePeopleSearchTimer = null;

  let selectedMeetingPeople = [];

  let currentMeetingTimezone = "";

  let meetingsAutoRefreshTimer = null;

  let currentMeetingDetails = "";

  /*
   * Meeting form mode:
   *
   * create = scheduling a new meeting
   * edit   = modifying an existing meeting
   */
  let meetingFormMode = "create";

  /*
   * Stores the meeting currently being edited.
   */
  let editingMeetingSysId = "";

  const meetingDetailsModal = document.getElementById("meetingDetailsModal");

  const meetingDetailsCloseButton = document.getElementById(
    "meetingDetailsCloseButton",
  );

  const meetingDetailsFooterCloseButton = document.getElementById(
    "meetingDetailsFooterCloseButton",
  );

  const meetingDetailsActionButton = document.getElementById(
    "meetingDetailsActionButton",
  );

  const meetingDetailsNumber = document.getElementById("meetingDetailsNumber");

  const meetingDetailsHeading = document.getElementById(
    "meetingDetailsHeading",
  );

  const meetingDetailsStatus = document.getElementById("meetingDetailsStatus");

  const meetingDetailsDescription = document.getElementById(
    "meetingDetailsDescription",
  );

  const meetingDetailsOrganizer = document.getElementById(
    "meetingDetailsOrganizer",
  );

  const meetingDetailsStart = document.getElementById("meetingDetailsStart");

  const meetingDetailsEnd = document.getElementById("meetingDetailsEnd");

  const meetingDetailsStartedBy = document.getElementById(
    "meetingDetailsStartedBy",
  );

  const meetingDetailsStartedAt = document.getElementById(
    "meetingDetailsStartedAt",
  );

  const meetingDetailsEndedAt = document.getElementById(
    "meetingDetailsEndedAt",
  );

  const meetingDetailsStartedByField = document.getElementById(
    "meetingDetailsStartedByField",
  );

  const meetingDetailsStartedAtField = document.getElementById(
    "meetingDetailsStartedAtField",
  );

  const meetingDetailsEndedAtField = document.getElementById(
    "meetingDetailsEndedAtField",
  );

  const meetingDetailsParticipantCount = document.getElementById(
    "meetingDetailsParticipantCount",
  );

  const meetingDetailsParticipants = document.getElementById(
    "meetingDetailsParticipants",
  );

  const scheduleMeetingModal = document.getElementById("scheduleMeetingModal");

  const scheduleMeetingCloseButton = document.getElementById(
    "scheduleMeetingCloseButton",
  );

  const scheduleMeetingCancelButton = document.getElementById(
    "scheduleMeetingCancelButton",
  );

  const scheduleMeetingTitle = document.getElementById("scheduleMeetingTitle");

  const scheduleMeetingDescription = document.getElementById(
    "scheduleMeetingDescription",
  );

  const scheduleMeetingStart = document.getElementById("scheduleMeetingStart");

  const scheduleMeetingEnd = document.getElementById("scheduleMeetingEnd");

  const scheduleMeetingTimezone = document.getElementById(
    "scheduleMeetingTimezone",
  );

  const scheduleMeetingHeading = document.getElementById(
    "scheduleMeetingHeading",
  );

  const scheduleMeetingSubtitle = document.getElementById(
    "scheduleMeetingSubtitle",
  );

  const scheduleMeetingPeopleSearch = document.getElementById(
    "scheduleMeetingPeopleSearch",
  );

  const scheduleMeetingPeopleResults = document.getElementById(
    "scheduleMeetingPeopleResults",
  );

  const resetPresenceButton = document.getElementById("resetPresenceButton");

  const scheduleMeetingSelectedPeople = document.getElementById(
    "scheduleMeetingSelectedPeople",
  );

  const scheduleMeetingMessage = document.getElementById(
    "scheduleMeetingMessage",
  );

  const scheduleMeetingSubmitButton = document.getElementById(
    "scheduleMeetingSubmitButton",
  );

  if (scheduleMeetingCloseButton) {
    scheduleMeetingCloseButton.addEventListener(
      "click",
      closeScheduleMeetingModal,
    );
  }

  if (scheduleMeetingCancelButton) {
    scheduleMeetingCancelButton.addEventListener(
      "click",
      closeScheduleMeetingModal,
    );
  }

  if (scheduleMeetingModal) {
    scheduleMeetingModal.addEventListener("click", (event) => {
      if (event.target.hasAttribute("data-schedule-meeting-close")) {
        closeScheduleMeetingModal();
      }
    });
  }

  /* -------------------------------------------------
           CONNECTION STATUS
        ------------------------------------------------- */

  function setConnectionDisplay(connected, text) {
    if (!connectionPill) {
      return;
    }

    connectionPill.textContent = text;

    connectionPill.classList.toggle("connected", connected);
  }

  /* -------------------------------------------------
           LOAD SAVED INSTANCE
        ------------------------------------------------- */

  try {
    const savedInstance = await window.serviceCall.getInstance();

    if (input && savedInstance.instanceUrl) {
      input.value = savedInstance.instanceUrl;
    }
  } catch (error) {
    console.error("Unable to load saved ServiceNow instance:", error);
  }

  /* -------------------------------------------------
           CHECK CONNECTION
        ------------------------------------------------- */

  try {
    const connectionStatus = await window.serviceCall.getConnectionStatus();

    if (connectionStatus.connected) {
      loginButton.disabled = true;

      loginButton.textContent = "Connected to ServiceNow";

      setConnectionDisplay(true, "Connected");

      await loadMyPresence();

      if (message) {
        message.textContent = connectionStatus.message || "";
      }
    } else {
      loginButton.disabled = false;

      loginButton.textContent = "Sign in to ServiceNow";

      setConnectionDisplay(false, "Not connected");
    }
  } catch (error) {
    console.error("Unable to check ServiceCall connection:", error);

    setConnectionDisplay(false, "Connection unavailable");
  }

  /* -------------------------------------------------
           SAVE INSTANCE
        ------------------------------------------------- */

  if (form) {
    form.addEventListener(
      "submit",

      async (event) => {
        event.preventDefault();

        const instanceUrl = input.value.trim();

        message.textContent = "Saving ServiceNow instance...";

        try {
          const result = await window.serviceCall.saveInstance(instanceUrl);

          message.textContent = result.message || "";
        } catch (error) {
          console.error("Save instance failed:", error);

          message.textContent = "Unable to save the ServiceNow instance.";
        }
      },
    );
  }

  /* -------------------------------------------------
           LOGIN
        ------------------------------------------------- */

  if (loginButton) {
    loginButton.addEventListener(
      "click",

      async () => {
        message.textContent = "Opening ServiceNow sign-in...";

        try {
          const result = await window.serviceCall.startLogin();

          message.textContent = result.message || "";
        } catch (error) {
          console.error("ServiceNow login failed:", error);

          message.textContent = "Unable to start ServiceNow sign-in.";
        }
      },
    );
  }

  /* -------------------------------------------------
           AUTH STATUS EVENTS
        ------------------------------------------------- */

  window.serviceCall.onAuthStatus(async (data) => {
    if (message) {
      message.textContent = data.message || "";
    }

    if (data.status === "connected") {
      loginButton.disabled = true;

      loginButton.textContent = "Connected to ServiceNow";

      setConnectionDisplay(true, "Connected");
      await loadMyPresence();
    } else if (
      data.status === "warning" ||
      data.status === "error" ||
      data.status === "authentication_required"
    ) {
      loginButton.disabled = false;

      loginButton.textContent = "Sign in to ServiceNow";

      setConnectionDisplay(false, "Connection required");
    }
  });

  /* -------------------------------------------------
           OPEN ACTIVE CALL
        ------------------------------------------------- */

  if (openActiveCallButton) {
    openActiveCallButton.addEventListener(
      "click",

      async () => {
        try {
          const result = await window.serviceCall.openActiveCall();

          if (!result.active_call && message) {
            message.textContent =
              result.message || "No active call is available.";
          }
        } catch (error) {
          console.error("Open active call failed:", error);

          if (message) {
            message.textContent = "Unable to open the active call.";
          }
        }
      },
    );
  }

  /* -------------------------------------------------
           SIDEBAR NAVIGATION
        ------------------------------------------------- */

  const navigationButtons = document.querySelectorAll(".nav-button[data-view]");

  const views = document.querySelectorAll(".view");

  navigationButtons.forEach((button) => {
    button.addEventListener(
      "click",

      async () => {
        const targetView = button.dataset.view;

        /*
         * Hide all pages.
         */
        views.forEach((view) => {
          view.classList.remove("active");
        });

        /*
         * Remove active state
         * from navigation.
         */
        navigationButtons.forEach((navButton) => {
          navButton.classList.remove("active");
        });

        /*
         * Show selected page.
         */
        const selectedView = document.getElementById(targetView);

        if (selectedView) {
          selectedView.classList.add("active");
        }

        button.classList.add("active");

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
        let pageName = "";

        const explicitPageNames = {
          homeView: "Home",

          peopleView: "People",

          meetingsView: "Meetings",

          notificationsView: "Notifications",

          chatView: "Chat",

          historyView: "History",

          recordingsView: "Recordings",

          settingsView: "Settings",
        };

        pageName = explicitPageNames[targetView] || button.textContent.trim();

        if (currentPageTitle) {
          currentPageTitle.textContent = pageName;
        }

        if (targetView === "meetingsView") {
          await loadMeetings();

          startMeetingsAutoRefresh();
        } else {
          stopMeetingsAutoRefresh();
        }

        /*
         * Load real Chat conversations
         * whenever Chat is opened.
         */
        if (targetView === "chatView") {
          await loadChatConversations();
        }

        /*
         * Notifications are different from Meetings.
         *
         * Their background monitor runs globally,
         * but opening the Notifications page performs
         * a normal visible refresh.
         */
        if (targetView === "notificationsView") {
          currentNotificationPage = 1;

          await loadNotifications(false);
        }
      },
    );
  });

  /* -------------------------------------------------
           MEETING HELPERS
        ------------------------------------------------- */

  function escapeHtml(value) {
    return String(value || "")
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;")
      .replace(/"/g, "&quot;")
      .replace(/'/g, "&#039;");
  }

  function formatMeetingDate(value) {
    if (!value) {
      return "";
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

  function getStatusClass(state) {
    const normalized = String(state || "")
      .toLowerCase()
      .trim();

    if (normalized === "in progress") {
      return "in-progress";
    }

    if (normalized === "scheduled") {
      return "scheduled";
    }

    return "";
  }

  function formatMeetingDetailsValue(value) {
    const text = String(value || "").trim();

    return text || "—";
  }

  function formatMeetingDetailsStatus(value) {
    const text = String(value || "")
      .trim()
      .toLowerCase();

    if (!text) {
      return "—";
    }

    return text
      .split(" ")
      .map((word) => (word ? word.charAt(0).toUpperCase() + word.slice(1) : ""))
      .join(" ");
  }

  function closeMeetingDetailsModal() {
    if (!meetingDetailsModal) {
      return;
    }

    meetingDetailsModal.classList.remove("open");

    meetingDetailsModal.setAttribute("aria-hidden", "true");
  }

  function getDateTimeLocalValueInTimezone(date, timeZone) {
    if (!date || !timeZone) {
      return "";
    }

    const parts = new Intl.DateTimeFormat("en-CA", {
      timeZone: timeZone,
      year: "numeric",
      month: "2-digit",
      day: "2-digit",
      hour: "2-digit",
      minute: "2-digit",
      hourCycle: "h23",
    }).formatToParts(date);

    const values = {};

    parts.forEach((part) => {
      if (part.type !== "literal") {
        values[part.type] = part.value;
      }
    });

    return (
      values.year +
      "-" +
      values.month +
      "-" +
      values.day +
      "T" +
      values.hour +
      ":" +
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
    meetingFormMode = "create";

    editingMeetingSysId = "";

    if (scheduleMeetingHeading) {
      scheduleMeetingHeading.textContent = "Schedule Meeting";
    }

    if (scheduleMeetingSubtitle) {
      scheduleMeetingSubtitle.textContent = "Create a new ServiceCall meeting.";
    }

    if (scheduleMeetingSubmitButton) {
      scheduleMeetingSubmitButton.textContent = "Schedule Meeting";
    }

    if (scheduleMeetingTimezone) {
      scheduleMeetingTimezone.textContent =
        currentMeetingTimezone || "Loading...";
    }

    /*
     * Start every new scheduling attempt
     * with a clean form.
     */

    if (scheduleMeetingTitle) {
      scheduleMeetingTitle.value = "";
    }

    if (scheduleMeetingDescription) {
      scheduleMeetingDescription.value = "";
    }

    /*
     * Default meeting times are based on
     * the authenticated ServiceNow user's
     * timezone, NOT the laptop timezone.
     */
    if (currentMeetingTimezone && scheduleMeetingStart && scheduleMeetingEnd) {
      const now = new Date();

      /*
       * Default start = 5 minutes from now.
       */
      const defaultStart = new Date(now.getTime() + 5 * 60 * 1000);

      /*
       * Default end = 35 minutes from now,
       * giving a 30-minute meeting.
       */
      const defaultEnd = new Date(now.getTime() + 35 * 60 * 1000);

      scheduleMeetingStart.value = getDateTimeLocalValueInTimezone(
        defaultStart,
        currentMeetingTimezone,
      );

      scheduleMeetingEnd.value = getDateTimeLocalValueInTimezone(
        defaultEnd,
        currentMeetingTimezone,
      );
    } else {
      if (scheduleMeetingStart) {
        scheduleMeetingStart.value = "";
      }

      if (scheduleMeetingEnd) {
        scheduleMeetingEnd.value = "";
      }
    }

    selectedMeetingPeople = [];

    if (scheduleMeetingPeopleSearch) {
      scheduleMeetingPeopleSearch.value = "";
    }

    if (scheduleMeetingPeopleResults) {
      scheduleMeetingPeopleResults.innerHTML = "";
      scheduleMeetingPeopleResults.style.display = "none";
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
      scheduleMeetingMessage.textContent = "";
    }

    scheduleMeetingModal.classList.add("open");

    scheduleMeetingModal.setAttribute("aria-hidden", "false");

    /*
     * Put the cursor directly in Title.
     */

    setTimeout(() => {
      if (scheduleMeetingTitle) {
        scheduleMeetingTitle.focus();
      }
    }, 0);
  }

  function closeScheduleMeetingModal() {
    if (!scheduleMeetingModal) {
      return;
    }

    scheduleMeetingModal.classList.remove("open");

    scheduleMeetingModal.setAttribute("aria-hidden", "true");

    if (scheduleMeetingPeopleResults) {
      scheduleMeetingPeopleResults.style.display = "none";
    }
  }

  function meetingDisplayValueToDateTimeLocal(value) {
    const text = String(value || "").trim();

    if (!text) {
      return "";
    }

    return text.replace(" ", "T").substring(0, 16);
  }

  async function openEditMeetingModal(meetingSysId) {
    if (!meetingSysId || !scheduleMeetingModal) {
      return;
    }

    /*
     * EDIT mode.
     */
    meetingFormMode = "edit";

    editingMeetingSysId = meetingSysId;

    /*
     * Change the existing modal UI.
     */
    if (scheduleMeetingHeading) {
      scheduleMeetingHeading.textContent = "Edit Meeting";
    }

    if (scheduleMeetingSubtitle) {
      scheduleMeetingSubtitle.textContent = "Update this ServiceCall meeting.";
    }

    if (scheduleMeetingSubmitButton) {
      scheduleMeetingSubmitButton.textContent = "Loading...";

      scheduleMeetingSubmitButton.disabled = true;
    }

    if (scheduleMeetingMessage) {
      scheduleMeetingMessage.textContent = "";
    }

    /*
     * Clear old participant search results.
     */
    if (scheduleMeetingPeopleSearch) {
      scheduleMeetingPeopleSearch.value = "";
    }

    if (scheduleMeetingPeopleResults) {
      scheduleMeetingPeopleResults.innerHTML = "";

      scheduleMeetingPeopleResults.style.display = "none";
    }

    /*
     * Open modal immediately while details load.
     */
    scheduleMeetingModal.classList.add("open");

    scheduleMeetingModal.setAttribute("aria-hidden", "false");

    try {
      const result = await window.serviceCall.getMeetingDetails(meetingSysId);

      if (!result || result.success !== true) {
        throw new Error(
          result && result.message ? result.message : "Unable to load meeting.",
        );
      }

      console.log("Editing meeting:", result);

      /*
       * Title + Description
       */
      if (scheduleMeetingTitle) {
        scheduleMeetingTitle.value = result.title || "";
      }

      if (scheduleMeetingDescription) {
        scheduleMeetingDescription.value = result.description || "";
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
        scheduleMeetingStart.value = meetingDisplayValueToDateTimeLocal(
          result.scheduled_start,
        );
      }

      if (scheduleMeetingEnd) {
        scheduleMeetingEnd.value = meetingDisplayValueToDateTimeLocal(
          result.scheduled_end,
        );
      }

      /*
       * Display authenticated user's
       * ServiceNow timezone.
       */
      if (scheduleMeetingTimezone) {
        scheduleMeetingTimezone.textContent =
          currentMeetingTimezone || "Unavailable";
      }

      /*
       * Populate existing attendees.
       *
       * Do not add the organizer as a
       * selectable attendee.
       */
      selectedMeetingPeople = Array.isArray(result.participants)
        ? result.participants
            .filter((participant) => participant.role !== "organizer")
            .map((participant) => ({
              sys_id: participant.user_sys_id,

              name: participant.user_name,
            }))
        : [];

      /*
       * Re-render selected participant chips.
       */
      renderSelectedMeetingPeople();

      if (scheduleMeetingSubmitButton) {
        scheduleMeetingSubmitButton.disabled = false;

        scheduleMeetingSubmitButton.textContent = "Save Changes";
      }

      if (scheduleMeetingTitle) {
        scheduleMeetingTitle.focus();
      }
    } catch (error) {
      console.error("Unable to open Edit Meeting:", error);

      if (scheduleMeetingMessage) {
        scheduleMeetingMessage.textContent =
          error.message || "Unable to load meeting.";
      }

      if (scheduleMeetingSubmitButton) {
        scheduleMeetingSubmitButton.disabled = true;

        scheduleMeetingSubmitButton.textContent = "Save Changes";
      }
    }
  }

  function openMeetingDetailsModal(details) {
    if (!meetingDetailsModal) {
      return;
    }

    /*
     * Remember the meeting currently
     * displayed in the Details modal.
     */
    currentMeetingDetails = details;

    console.log("ServiceCall Meeting Details:", details);

    /*
     * Reset the contextual action button.
     *
     * We will determine its exact action
     * from the meeting state/permissions next.
     */
    if (meetingDetailsActionButton) {
      meetingDetailsActionButton.style.display = "none";

      meetingDetailsActionButton.disabled = false;

      meetingDetailsActionButton.textContent = "Join Meeting";
    }

    /* -------------------------
   PRIMARY MEETING ACTION
------------------------- */

    if (meetingDetailsActionButton) {
      if (details.can_start === true) {
        meetingDetailsActionButton.textContent = "Start Meeting";

        meetingDetailsActionButton.style.display = "";
      } else if (details.can_join === true) {
        meetingDetailsActionButton.textContent = "Join Meeting";

        meetingDetailsActionButton.style.display = "";
      }
    }

    /* -------------------------
       BASIC INFORMATION
    ------------------------- */

    meetingDetailsNumber.textContent = formatMeetingDetailsValue(
      details.meeting_number,
    );

    meetingDetailsHeading.textContent = formatMeetingDetailsValue(
      details.title,
    );

    meetingDetailsStatus.textContent = formatMeetingDetailsStatus(
      details.state,
    );

    meetingDetailsDescription.textContent =
      String(details.description || "").trim() || "No description.";

    meetingDetailsOrganizer.textContent = formatMeetingDetailsValue(
      details.organizer_name,
    );

    meetingDetailsStart.textContent = formatMeetingDetailsValue(
      details.scheduled_start,
    );

    meetingDetailsEnd.textContent = formatMeetingDetailsValue(
      details.scheduled_end,
    );

    /* -------------------------
       STARTED INFORMATION
    ------------------------- */

    const hasStartedBy = Boolean(String(details.started_by_name || "").trim());

    const hasStartedAt = Boolean(String(details.started_at || "").trim());

    const hasEndedAt = Boolean(String(details.ended_at || "").trim());

    meetingDetailsStartedByField.style.display = hasStartedBy ? "" : "none";

    meetingDetailsStartedAtField.style.display = hasStartedAt ? "" : "none";

    meetingDetailsEndedAtField.style.display = hasEndedAt ? "" : "none";

    meetingDetailsStartedBy.textContent = formatMeetingDetailsValue(
      details.started_by_name,
    );

    meetingDetailsStartedAt.textContent = formatMeetingDetailsValue(
      details.started_at,
    );

    meetingDetailsEndedAt.textContent = formatMeetingDetailsValue(
      details.ended_at,
    );

    /* -------------------------
       PARTICIPANTS
    ------------------------- */

    const participants = Array.isArray(details.participants)
      ? details.participants
      : [];

    meetingDetailsParticipantCount.textContent = String(participants.length);

    meetingDetailsParticipants.innerHTML = "";

    if (participants.length === 0) {
      const empty = document.createElement("div");

      empty.className = "meeting-details-empty";

      empty.textContent = "No participants.";

      meetingDetailsParticipants.appendChild(empty);
    } else {
      participants.forEach((participant) => {
        const row = document.createElement("div");

        row.className = "meeting-details-participant";

        /* -----------------
                   PERSON
                ----------------- */

        const main = document.createElement("div");

        main.className = "meeting-details-participant-main";

        const name = document.createElement("div");

        name.className = "meeting-details-participant-name";

        name.textContent = formatMeetingDetailsValue(participant.user_name);

        const role = document.createElement("div");

        role.className = "meeting-details-participant-role";

        role.textContent = formatMeetingDetailsStatus(participant.role);

        main.appendChild(name);

        main.appendChild(role);

        /* -----------------
                   STATUSES
                ----------------- */

        const statuses = document.createElement("div");

        statuses.className = "meeting-details-participant-statuses";

        if (participant.invitation_status) {
          const invitationBadge = document.createElement("span");

          invitationBadge.className = "meeting-details-badge";

          invitationBadge.textContent = formatMeetingDetailsStatus(
            participant.invitation_status,
          );

          statuses.appendChild(invitationBadge);
        }

        if (participant.join_status) {
          const joinBadge = document.createElement("span");

          joinBadge.className = "meeting-details-badge";

          joinBadge.textContent = formatMeetingDetailsStatus(
            participant.join_status,
          );

          statuses.appendChild(joinBadge);
        }

        row.appendChild(main);

        row.appendChild(statuses);

        meetingDetailsParticipants.appendChild(row);
      });
    }

    /* -------------------------
       OPEN
    ------------------------- */

    meetingDetailsModal.classList.add("open");

    meetingDetailsModal.setAttribute("aria-hidden", "false");
  }

  /* =========================================
   MEETING DETAILS PRIMARY ACTION
========================================= */

  if (meetingDetailsActionButton) {
    meetingDetailsActionButton.addEventListener("click", async () => {
      if (!currentMeetingDetails || !currentMeetingDetails.meeting_sys_id) {
        return;
      }

      const meetingSysId = currentMeetingDetails.meeting_sys_id;

      meetingDetailsActionButton.disabled = true;

      try {
        /* -------------------------
                   START MEETING
                ------------------------- */

        if (currentMeetingDetails.can_start === true) {
          meetingDetailsActionButton.textContent = "Starting...";

          const result = await window.serviceCall.startMeeting(meetingSysId);

          if (!result || result.success !== true) {
            throw new Error(
              result && result.message
                ? result.message
                : "Unable to start meeting.",
            );
          }

          playMeetingJoinStartSound();

          /*
           * main.js already opens the
           * connected meeting call window.
           */

          meetingDetailsModal.classList.remove("open");

          meetingDetailsModal.setAttribute("aria-hidden", "true");

          currentMeetingDetails = null;

          await loadMeetings(true);

          return;
        }

        /* -------------------------
                   JOIN MEETING
                ------------------------- */

        if (currentMeetingDetails.can_join === true) {
          meetingDetailsActionButton.textContent = "Joining...";

          const result = await window.serviceCall.joinMeeting(meetingSysId);

          if (!result || result.success !== true) {
            throw new Error(
              result && result.message
                ? result.message
                : "Unable to join meeting.",
            );
          }

          playMeetingJoinStartSound();

          meetingDetailsModal.classList.remove("open");

          meetingDetailsModal.setAttribute("aria-hidden", "true");

          currentMeetingDetails = null;

          await loadMeetings(true);

          return;
        }
      } catch (error) {
        console.error("Meeting details action failed:", error);

        /*
         * Re-fetch the meeting because its
         * state may have changed on the server.
         */
        try {
          const refreshed =
            await window.serviceCall.getMeetingDetails(meetingSysId);

          if (refreshed && refreshed.success === true) {
            openMeetingDetailsModal(refreshed);

            return;
          }
        } catch (refreshError) {
          console.error("Unable to refresh meeting details:", refreshError);
        }

        meetingDetailsActionButton.textContent = "Try Again";
      } finally {
        meetingDetailsActionButton.disabled = false;
      }
    });
  }

  /* =================================================
   COPY MEETING LINK
================================================= */

  async function copyMeetingLink(meetingSysId) {
    if (!meetingSysId) {
      return false;
    }

    const meetingLink =
      "servicecall://meeting/" + encodeURIComponent(meetingSysId);

    try {
      await navigator.clipboard.writeText(meetingLink);

      console.log("Meeting link copied:", meetingLink);

      return true;
    } catch (error) {
      console.error("Unable to copy meeting link:", error);

      return false;
    }
  }

  /* =================================================
   OPEN MEETING FROM DEEP LINK
================================================= */

  async function openMeetingFromDeepLink(meetingSysId) {
    const cleanMeetingSysId = String(meetingSysId || "").trim();

    /*
     * ServiceNow sys_id must be
     * exactly 32 hexadecimal characters.
     */
    if (!/^[0-9a-f]{32}$/i.test(cleanMeetingSysId)) {
      console.error("Invalid ServiceCall meeting link.");

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
        await window.serviceCall.getMeetingDetails(cleanMeetingSysId);

      if (!result || result.success !== true) {
        throw new Error(
          result && result.message
            ? result.message
            : "Unable to retrieve meeting details.",
        );
      }

      /*
       * Reuse the exact same Details UI
       * used by the View Details button.
       */
      openMeetingDetailsModal(result);
    } catch (error) {
      console.error("Unable to open meeting link:", error);

      if (message) {
        message.textContent = error.message || "Unable to open this meeting.";
      }
    }
  }
  /* -------------------------------------------------
           RENDER MEETING
        ------------------------------------------------- */

  function createMeetingCard(meeting) {
    const card = document.createElement("div");

    card.className = "meeting-card";

    const statusClass = getStatusClass(meeting.state);

    const organizer = meeting.organizer_name || "Unknown organizer";

    card.innerHTML = `

                <div class="meeting-card-left">

                    <div class="meeting-title">
                        ${escapeHtml(meeting.title || "Untitled meeting")}
                    </div>

                    <div class="meeting-meta">

                        ${escapeHtml(meeting.meeting_number || "")}

                        <br>

                        ${escapeHtml(
                          formatMeetingDate(meeting.scheduled_start),
                        )}

                        ${
                          meeting.scheduled_end
                            ? " – " +
                              escapeHtml(
                                formatMeetingDate(meeting.scheduled_end),
                              )
                            : ""
                        }

                        <br>

                        Organizer:
                        ${escapeHtml(organizer)}

                    </div>

                </div>


                <div class="meeting-actions">

                    <span
                        class="
                            meeting-status
                            ${statusClass}
                        "
                    >
                        ${escapeHtml(meeting.state || "Unknown")}
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

    const actions = card.querySelector(".meeting-actions");

    /* =========================================
   VIEW MEETING DETAILS
========================================= */

    const detailsButton = document.createElement("button");

    detailsButton.type = "button";

    detailsButton.className = "secondary-button";

    detailsButton.textContent = "View Details";

    detailsButton.addEventListener(
      "click",

      async () => {
        detailsButton.disabled = true;

        detailsButton.textContent = "Loading...";

        try {
          const result = await window.serviceCall.getMeetingDetails(
            meeting.meeting_sys_id,
          );

          if (!result || result.success !== true) {
            throw new Error(
              result && result.message
                ? result.message
                : "Unable to retrieve meeting details.",
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
          console.error("Meeting details failed:", error);

          if (message) {
            message.textContent =
              error.message || "Unable to retrieve meeting details.";
          }
        } finally {
          detailsButton.disabled = false;

          detailsButton.textContent = "View Details";
        }
      },
    );

    actions.appendChild(detailsButton);

    /* -----------------------------------------
   COPY MEETING LINK
----------------------------------------- */

    const copyLinkButton = document.createElement("button");

    copyLinkButton.className = "secondary-button";

    copyLinkButton.textContent = "Copy Link";

    copyLinkButton.addEventListener("click", async () => {
      const copied = await copyMeetingLink(meeting.meeting_sys_id);

      if (!copied) {
        copyLinkButton.textContent = "Copy failed";

        setTimeout(() => {
          copyLinkButton.textContent = "Copy Link";
        }, 1500);

        return;
      }

      copyLinkButton.textContent = "Copied!";

      setTimeout(() => {
        copyLinkButton.textContent = "Copy Link";
      }, 1500);
    });

    actions.appendChild(copyLinkButton);

    /* =========================================
   EDIT MEETING
========================================= */

    if (meeting.can_edit === true) {
      const button = document.createElement("button");

      button.className = "secondary-button";

      button.textContent = "Edit";

      button.addEventListener(
        "click",

        async () => {
          await openEditMeetingModal(meeting.meeting_sys_id);
        },
      );

      actions.appendChild(button);
    }

    if (meeting.can_start === true) {
      const button = document.createElement("button");

      button.className = "primary-button";

      button.textContent = "Start";

      button.addEventListener(
        "click",

        async () => {
          /*
           * Prevent double-clicking Start.
           */
          button.disabled = true;

          button.textContent = "Starting...";

          try {
            const result = await window.serviceCall.startMeeting(
              meeting.meeting_sys_id,
            );

            if (!result || result.success !== true) {
              throw new Error(
                result && result.message
                  ? result.message
                  : "Unable to start meeting.",
              );
            }

            playMeetingJoinStartSound();

            console.log("Meeting started:", result);

            /*
             * Refresh Meetings so the card
             * changes from Scheduled to
             * In Progress.
             */
            await loadMeetings(true);
          } catch (error) {
            console.error("Start meeting failed:", error);

            button.disabled = false;

            button.textContent = "Start";

            if (message) {
              message.textContent = error.message || "Unable to start meeting.";
            }
          }
        },
      );

      actions.appendChild(button);
    }

    if (meeting.can_join === true) {
      const button = document.createElement("button");

      button.className = "primary-button";

      button.textContent = "Join";

      button.addEventListener(
        "click",

        async () => {
          button.disabled = true;

          button.textContent = "Joining...";

          try {
            const result = await window.serviceCall.joinMeeting(
              meeting.meeting_sys_id,
            );

            if (!result || result.success !== true) {
              throw new Error(
                result && result.message
                  ? result.message
                  : "Unable to join meeting.",
              );
            }

            playMeetingJoinStartSound();

            console.log("Meeting joined:", result);

            /*
             * Refresh the card because
             * can_join / can_leave may
             * now have changed.
             */
            await loadMeetings(true);
          } catch (error) {
            console.error("Join meeting failed:", error);

            button.disabled = false;

            button.textContent = "Join";

            if (message) {
              message.textContent = error.message || "Unable to join meeting.";
            }
          }
        },
      );

      actions.appendChild(button);
    }

    if (meeting.can_leave === true) {
      const button = document.createElement("button");

      button.className = "secondary-button";

      button.textContent = "Leave";

      button.addEventListener(
        "click",

        async () => {
          button.disabled = true;

          button.textContent = "Leaving...";

          try {
            const result = await window.serviceCall.leaveMeeting(
              meeting.meeting_sys_id,
            );

            if (!result || result.success !== true) {
              throw new Error(
                result && result.message
                  ? result.message
                  : "Unable to leave meeting.",
              );
            }

            console.log("Meeting left:", result);

            playMeetingLeaveEndSound();

            await loadMeetings(true);
          } catch (error) {
            console.error("Leave meeting failed:", error);

            button.disabled = false;

            button.textContent = "Leave";

            if (message) {
              message.textContent = error.message || "Unable to leave meeting.";
            }
          }
        },
      );

      actions.appendChild(button);
    }

    if (meeting.can_end === true) {
      const button = document.createElement("button");

      button.className = "danger-button";

      button.textContent = "End";

      button.addEventListener(
        "click",

        async () => {
          /*
           * Prevent accidental double-clicks.
           */
          button.disabled = true;

          button.textContent = "Ending...";

          try {
            const result = await window.serviceCall.endMeeting(
              meeting.meeting_sys_id,
            );

            if (!result || result.success !== true) {
              throw new Error(
                result && result.message
                  ? result.message
                  : "Unable to end meeting.",
              );
            }

            console.log("Meeting ended:", result);

            playMeetingLeaveEndSound();

            /*
             * Refresh meeting cards.
             * The ended meeting should now
             * display as Ended and its live
             * action buttons disappear.
             */
            await loadMeetings(true);
          } catch (error) {
            console.error("End meeting failed:", error);

            button.disabled = false;

            button.textContent = "End";

            if (message) {
              message.textContent = error.message || "Unable to end meeting.";
            }
          }
        },
      );

      actions.appendChild(button);
    }

    if (meeting.can_cancel === true) {
      const button = document.createElement("button");

      button.className = "danger-button";

      button.textContent = "Cancel";

      button.addEventListener(
        "click",

        async () => {
          button.disabled = true;

          button.textContent = "Cancelling...";

          try {
            const result = await window.serviceCall.cancelMeeting(
              meeting.meeting_sys_id,
            );

            if (!result || result.success !== true) {
              throw new Error(
                result && result.message
                  ? result.message
                  : "Unable to cancel meeting.",
              );
            }

            console.log("Meeting cancelled:", result);

            /*
             * Refresh meeting cards.
             *
             * State should now be Cancelled
             * and Start / Cancel disappear.
             */
            await loadMeetings(true);
          } catch (error) {
            console.error("Cancel meeting failed:", error);

            button.disabled = false;

            button.textContent = "Cancel";

            if (message) {
              message.textContent =
                error.message || "Unable to cancel meeting.";
            }
          }
        },
      );

      actions.appendChild(button);
    }

    return card;
  }

  if (meetingDetailsCloseButton) {
    meetingDetailsCloseButton.addEventListener(
      "click",
      closeMeetingDetailsModal,
    );
  }

  if (meetingDetailsFooterCloseButton) {
    meetingDetailsFooterCloseButton.addEventListener(
      "click",
      closeMeetingDetailsModal,
    );
  }

  if (meetingDetailsModal) {
    meetingDetailsModal.addEventListener("click", (event) => {
      if (event.target.hasAttribute("data-meeting-details-close")) {
        closeMeetingDetailsModal();
      }
    });
  }

  document.addEventListener("keydown", (event) => {
    if (event.key !== "Escape") {
      return;
    }

    if (
      scheduleMeetingModal &&
      scheduleMeetingModal.classList.contains("open")
    ) {
      closeScheduleMeetingModal();

      return;
    }

    if (meetingDetailsModal && meetingDetailsModal.classList.contains("open")) {
      closeMeetingDetailsModal();
    }
  });
  /* -------------------------------------------------
   RENDER MEETING PAGINATION
------------------------------------------------- */

  function renderMeetingPagination(result) {
    if (!meetingPagination) {
      return;
    }

    meetingPagination.innerHTML = "";

    const totalPages = parseInt(result.total_pages, 10) || 0;

    const currentPage = parseInt(result.current_page, 10) || 1;

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

    const previousButton = document.createElement("button");

    previousButton.type = "button";

    previousButton.className = "meeting-page-button";

    previousButton.textContent = "‹ Previous";

    previousButton.disabled = currentPage <= 1;

    previousButton.addEventListener("click", async () => {
      if (currentPage <= 1) {
        return;
      }

      currentMeetingPage = currentPage - 1;

      await loadMeetings();
    });

    meetingPagination.appendChild(previousButton);

    /* -----------------------------
   PAGE NUMBERS
----------------------------- */

    function addPageButton(pageNumber) {
      const pageButton = document.createElement("button");

      pageButton.type = "button";

      pageButton.className = "meeting-page-button";

      pageButton.textContent = String(pageNumber);

      if (pageNumber === currentPage) {
        pageButton.classList.add("active");

        pageButton.disabled = true;
      }

      pageButton.addEventListener("click", async () => {
        if (pageNumber === currentPage) {
          return;
        }

        currentMeetingPage = pageNumber;

        await loadMeetings();
      });

      meetingPagination.appendChild(pageButton);
    }

    function addEllipsis() {
      const ellipsis = document.createElement("span");

      ellipsis.className = "meeting-page-info";

      ellipsis.textContent = "…";

      meetingPagination.appendChild(ellipsis);
    }

    /*
     * Small number of pages:
     *
     * 1 2 3 4 5
     */
    if (totalPages <= 5) {
      for (let pageNumber = 1; pageNumber <= totalPages; pageNumber++) {
        addPageButton(pageNumber);
      }
    } else if (currentPage <= 3) {
      /*
       * Near the beginning:
       *
       * 1 2 3 … 15
       */
      addPageButton(1);
      addPageButton(2);
      addPageButton(3);

      addEllipsis();

      addPageButton(totalPages);
    } else if (currentPage >= totalPages - 2) {
      /*
       * Near the end:
       *
       * 1 … 13 14 15
       */
      addPageButton(1);

      addEllipsis();

      addPageButton(totalPages - 2);

      addPageButton(totalPages - 1);

      addPageButton(totalPages);
    } else {
      /*
       * Somewhere in the middle:
       *
       * 1 … 7 8 9 … 15
       */
      addPageButton(1);

      addEllipsis();

      addPageButton(currentPage - 1);

      addPageButton(currentPage);

      addPageButton(currentPage + 1);

      addEllipsis();

      addPageButton(totalPages);
    }

    /* -----------------------------
       NEXT
    ----------------------------- */

    const nextButton = document.createElement("button");

    nextButton.type = "button";

    nextButton.className = "meeting-page-button";

    nextButton.textContent = "Next ›";

    nextButton.disabled = currentPage >= totalPages;

    nextButton.addEventListener("click", async () => {
      if (currentPage >= totalPages) {
        return;
      }

      currentMeetingPage = currentPage + 1;

      await loadMeetings();
    });

    meetingPagination.appendChild(nextButton);

    /* -----------------------------
       PAGE INFORMATION
    ----------------------------- */

    const pageInfo = document.createElement("span");

    pageInfo.className = "meeting-page-info";

    pageInfo.textContent = "Page " + currentPage + " of " + totalPages;

    meetingPagination.appendChild(pageInfo);
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

    meetingsAutoRefreshTimer = setInterval(async () => {
      /*
       * Refresh only when the
       * Meetings page is actually open.
       */
      const meetingsView = document.getElementById("meetingsView");

      if (!meetingsView || !meetingsView.classList.contains("active")) {
        return;
      }

      /*
       * Don't disturb the user while
       * the Schedule Meeting modal
       * is open.
       */
      if (
        scheduleMeetingModal &&
        scheduleMeetingModal.classList.contains("open")
      ) {
        return;
      }

      try {
        console.log("Auto-refreshing meetings...");

        await loadMeetings(true);
      } catch (error) {
        console.error("Meeting auto-refresh failed:", error);
      }
    }, 15000);
  }

  function stopMeetingsAutoRefresh() {
    if (!meetingsAutoRefreshTimer) {
      return;
    }

    clearInterval(meetingsAutoRefreshTimer);

    meetingsAutoRefreshTimer = null;
  }

  /* -------------------------------------------------
           LOAD MEETINGS
        ------------------------------------------------- */

  async function loadMeetings(silent = false) {
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
        meetingPagination.innerHTML = "";
      }

      meetingsContainer.innerHTML = `
            <div class="loading">
                Loading your meetings...
            </div>
        `;
    }

    try {
      const result = await window.serviceCall.getMyMeetings(
        currentMeetingPage,
        currentMeetingSearch,
        currentMeetingStatus,
      );

      if (!result || result.success !== true) {
        throw new Error(
          result && result.message
            ? result.message
            : "Unable to retrieve meetings.",
        );
      }

      /*
       * Save the authenticated user's
       * ServiceNow timezone.
       */
      currentMeetingTimezone = String(result.user_timezone || "").trim();

      console.log("ServiceNow user timezone:", currentMeetingTimezone);

      if (scheduleMeetingTimezone) {
        scheduleMeetingTimezone.textContent =
          currentMeetingTimezone || "Unavailable";
      }

      const meetings = Array.isArray(result.meetings) ? result.meetings : [];

      currentMeetingPage = parseInt(result.current_page, 10) || 1;

      renderMeetingPagination(result);

      meetingsContainer.innerHTML = "";

      if (meetings.length === 0) {
        /*
         * Search and/or status filter is active.
         */
        if (currentMeetingSearch || currentMeetingStatus) {
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

      meetings.forEach((meeting) => {
        meetingsContainer.appendChild(createMeetingCard(meeting));
      });
    } catch (error) {
      console.error("Unable to load meetings:", error);

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
    meetingSearchInput.addEventListener("input", () => {
      const searchValue = meetingSearchInput.value.trim();

      /*
       * Show the clear button whenever
       * something has been entered.
       */
      if (meetingSearchClear) {
        meetingSearchClear.style.display = searchValue ? "flex" : "none";
      }

      /*
       * Cancel the previous pending search.
       *
       * This prevents an API request for
       * every individual keystroke.
       */
      if (meetingSearchTimer) {
        clearTimeout(meetingSearchTimer);
      }

      meetingSearchTimer = setTimeout(async () => {
        currentMeetingSearch = searchValue;

        /*
         * Every new search begins
         * from page 1.
         */
        currentMeetingPage = 1;

        await loadMeetings();
      }, 300);
    });
  }

  if (meetingSearchClear) {
    meetingSearchClear.addEventListener("click", async () => {
      if (meetingSearchTimer) {
        clearTimeout(meetingSearchTimer);

        meetingSearchTimer = null;
      }

      if (meetingSearchInput) {
        meetingSearchInput.value = "";

        meetingSearchInput.focus();
      }

      meetingSearchClear.style.display = "none";

      currentMeetingSearch = "";

      currentMeetingPage = 1;

      await loadMeetings();
    });
  }

  /* -------------------------------------------------
   MEETING STATUS FILTER
------------------------------------------------- */

  if (meetingStatusFilter) {
    meetingStatusFilter.addEventListener("change", async () => {
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
      currentMeetingStatus = String(meetingStatusFilter.value || "")
        .toLowerCase()
        .trim();

      /*
       * A new filter always begins
       * from page 1.
       */
      currentMeetingPage = 1;

      await loadMeetings();
    });
  }

  /* =======================================================
   MEETING CHANGE REFRESH
======================================================= */

  if (
    window.serviceCall &&
    typeof window.serviceCall.onMeetingChanged === "function"
  ) {
    window.serviceCall.onMeetingChanged(async (data) => {
      console.log("ServiceCall meeting changed:", data);

      await loadMeetings(true);
    });
  }

  /* -------------------------------------------------
           SCHEDULE MEETING
        ------------------------------------------------- */

  if (scheduleMeetingButton) {
    scheduleMeetingButton.addEventListener("click", () => {
      openScheduleMeetingModal();
    });
  }

  /* =================================================
   OPEN MEETING FROM DESKTOP NOTIFICATION
================================================= */

  if (
    window.serviceCall &&
    typeof window.serviceCall.onNotificationMeetingOpen === "function"
  ) {
    window.serviceCall.onNotificationMeetingOpen(async (data) => {
      try {
        const meetingSysId = String(
          data && data.meetingSysId ? data.meetingSysId : "",
        ).trim();

        console.log("ServiceCall meeting notification received:", data);

        /*
         * Validate the meeting sys_id
         * before doing anything.
         */
        if (!/^[0-9a-f]{32}$/i.test(meetingSysId)) {
          console.error("Invalid meeting sys_id from notification.");

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
        const meetingsNavigationButton = document.querySelector(
          '.nav-button[data-view="meetingsView"]',
        );

        if (meetingsNavigationButton) {
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
        await openMeetingFromDeepLink(meetingSysId);
      } catch (error) {
        console.error("Unable to open meeting from notification:", error);
      }
    });
  }
  /* =================================================
   SERVICECALL DEEP LINK
================================================= */

  if (window.serviceCall && window.serviceCall.onDeepLink) {
    window.serviceCall.onDeepLink(async (data) => {
      try {
        const deepLink = String(data && data.url ? data.url : "").trim();

        if (!deepLink) {
          return;
        }

        console.log("ServiceCall deep link received in renderer:", deepLink);

        /*
         * Expected formats:
         *
         * servicecall://meeting/<meeting_sys_id>
         * servicecall:///meeting/<meeting_sys_id>
         */
        const meetingLinkMatch = String(deepLink || "")
          .trim()
          .match(/^servicecall:\/\/\/?meeting\/([0-9a-f]{32})$/i);

        if (!meetingLinkMatch) {
          console.error("Unsupported ServiceCall deep link:", deepLink);

          return;
        }

        const meetingSysId = meetingLinkMatch[1];

        console.log("ServiceCall meeting sys_id from link:", meetingSysId);

        if (!/^[0-9a-f]{32}$/i.test(meetingSysId)) {
          console.error("Invalid meeting sys_id in ServiceCall link.");

          return;
        }

        /*
         * Open the meeting using our
         * existing secure details flow.
         */
        await openMeetingFromDeepLink(meetingSysId);
      } catch (error) {
        console.error("Unable to process ServiceCall meeting link:", error);
      }
    });
  }

  /*
   * Tell the main process that the renderer
   * has installed its deep-link listener.
   */
  if (window.serviceCall && window.serviceCall.rendererReady) {
    window.serviceCall.rendererReady();
  }

  function renderSelectedMeetingPeople() {
    if (!scheduleMeetingSelectedPeople) {
      return;
    }

    /*
     * Clear the current display.
     */
    scheduleMeetingSelectedPeople.innerHTML = "";

    /*
     * No selected people.
     */
    if (selectedMeetingPeople.length === 0) {
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
    selectedMeetingPeople.forEach((person) => {
      const selectedPerson = document.createElement("div");

      selectedPerson.className = "schedule-meeting-selected-person";

      const name = document.createElement("span");

      name.textContent = person.name || "Unknown User";

      const removeButton = document.createElement("button");

      removeButton.type = "button";

      removeButton.textContent = "×";

      if (isUploading) {
        removeButton.disabled = true;
        removeButton.title = "Upload in progress";
      }

      removeButton.title = "Remove";

      removeButton.addEventListener("click", () => {
        /*
         * Remove the person from
         * our selected array.
         */
        selectedMeetingPeople = selectedMeetingPeople.filter(
          (selected) => selected.sys_id !== person.sys_id,
        );

        /*
         * Re-render everything.
         */
        renderSelectedMeetingPeople();
      });

      selectedPerson.appendChild(name);

      selectedPerson.appendChild(removeButton);

      scheduleMeetingSelectedPeople.appendChild(selectedPerson);
    });
  }

  /* -------------------------------------------------
   SCHEDULE MEETING - PEOPLE SEARCH
------------------------------------------------- */

  if (scheduleMeetingPeopleSearch) {
    scheduleMeetingPeopleSearch.addEventListener("input", () => {
      const searchText = scheduleMeetingPeopleSearch.value.trim();

      if (schedulePeopleSearchTimer) {
        clearTimeout(schedulePeopleSearchTimer);
      }

      /*
       * Our existing /users API requires
       * at least 2 characters.
       */
      if (searchText.length < 2) {
        scheduleMeetingPeopleResults.innerHTML = "";

        scheduleMeetingPeopleResults.style.display = "none";

        return;
      }

      schedulePeopleSearchTimer = setTimeout(async () => {
        try {
          const result = await window.serviceCall.searchUsers(searchText);

          console.log("Schedule meeting user search:", result);

          if (!result || result.success !== true) {
            return;
          }

          const users = Array.isArray(result.users)
            ? result.users.filter(
                (user) =>
                  !selectedMeetingPeople.some(
                    (person) => person.sys_id === user.sys_id,
                  ),
              )
            : [];

          scheduleMeetingPeopleResults.innerHTML = "";

          users.forEach((user) => {
            const item = document.createElement("div");

            item.className = "schedule-meeting-person-result";

            item.textContent = user.name || "Unknown User";

            item.addEventListener("click", () => {
              /*
               * Don't add the same person twice.
               */
              const alreadySelected = selectedMeetingPeople.some(
                (person) => person.sys_id === user.sys_id,
              );

              if (!alreadySelected) {
                selectedMeetingPeople.push({
                  sys_id: user.sys_id,
                  name: user.name || "Unknown User",
                });
              }

              renderSelectedMeetingPeople();
              /*
               * Clear search and hide results.
               */
              scheduleMeetingPeopleSearch.value = "";

              scheduleMeetingPeopleResults.innerHTML = "";

              scheduleMeetingPeopleResults.style.display = "none";
            });

            scheduleMeetingPeopleResults.appendChild(item);
          });

          scheduleMeetingPeopleResults.style.display =
            users.length > 0 ? "block" : "none";
        } catch (error) {
          console.error("Schedule meeting user search failed:", error);
        }
      }, 300);
    });
  }

  /* -------------------------------------------------
   SCHEDULE MEETING - VALIDATION
------------------------------------------------- */

  if (scheduleMeetingSubmitButton) {
    scheduleMeetingSubmitButton.addEventListener("click", async () => {
      const title = scheduleMeetingTitle.value.trim();

      const description = scheduleMeetingDescription.value.trim();

      const start = scheduleMeetingStart.value;

      const end = scheduleMeetingEnd.value;

      /*
       * Clear previous message.
       */
      scheduleMeetingMessage.textContent = "";

      if (!title) {
        scheduleMeetingMessage.textContent = "Please enter a meeting title.";

        scheduleMeetingTitle.focus();

        return;
      }

      if (!start) {
        scheduleMeetingMessage.textContent =
          "Please select a start date and time.";

        scheduleMeetingStart.focus();

        return;
      }

      if (!end) {
        scheduleMeetingMessage.textContent =
          "Please select an end date and time.";

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
          "End time must be after the start time.";

        scheduleMeetingEnd.focus();

        return;
      }

      if (!currentMeetingTimezone) {
        scheduleMeetingMessage.textContent =
          "Unable to determine your ServiceNow time zone. Please refresh Meetings and try again.";

        return;
      }

      if (selectedMeetingPeople.length === 0) {
        scheduleMeetingMessage.textContent =
          "Please select at least one person.";

        scheduleMeetingPeopleSearch.focus();

        return;
      }

      /*
       * Build participant sys_id array.
       */
      const participants = selectedMeetingPeople.map((person) => person.sys_id);

      /*
       * Build meeting payload.
       */
      const meetingData = {
        title: title,

        description: description,

        scheduled_start: start,

        scheduled_end: end,

        timezone: currentMeetingTimezone,

        participants: participants,
      };

      console.log(
        meetingFormMode === "edit"
          ? "Updating ServiceCall meeting:"
          : "Creating ServiceCall meeting:",
        meetingData,
      );

      /*
       * Prevent duplicate clicks while
       * the meeting is being created.
       */
      scheduleMeetingSubmitButton.disabled = true;

      scheduleMeetingMessage.textContent = "Scheduling meeting...";

      try {
        console.log("Meeting timezone test:", {
          start: start,
          end: end,
          timezone: currentMeetingTimezone,
          browserTimezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
        });

        let result;

        /*
         * CREATE MODE
         */
        if (meetingFormMode === "create") {
          result = await window.serviceCall.createMeeting(meetingData);
        } else if (meetingFormMode === "edit") {
          /*
           * EDIT MODE
           */
          if (!editingMeetingSysId) {
            throw new Error("Meeting sys_id is missing.");
          }

          result = await window.serviceCall.updateMeeting(
            editingMeetingSysId,
            meetingData,
          );
        }

        console.log("Create meeting result:", result);

        if (!result || result.success !== true) {
          scheduleMeetingMessage.textContent =
            result && result.message
              ? result.message
              : "Unable to schedule meeting.";

          return;
        }

        /*
         * Meeting created successfully.
         */
        scheduleMeetingMessage.textContent =
          meetingFormMode === "edit"
            ? "Meeting updated successfully."
            : "Meeting scheduled successfully.";

        /*
         * Refresh Meetings list.
         */
        await loadMeetings();

        /*
         * Close modal shortly after
         * successful creation.
         */
        setTimeout(() => {
          closeScheduleMeetingModal();
        }, 700);
      } catch (error) {
        console.error("Schedule meeting failed:", error);

        scheduleMeetingMessage.textContent =
          error && error.message
            ? error.message
            : "Unable to schedule meeting.";
      } finally {
        scheduleMeetingSubmitButton.disabled = false;
      }
    });
  }

  /* =======================================================
   SERVICECALL CHAT - GROUP DETAILS
======================================================= */

  async function openChatGroupDetails() {
    /* =========================================
       VALIDATE ACTIVE GROUP
    ========================================= */

    if (
      !activeChatConversation ||
      activeChatConversation.type !== "group" ||
      !activeChatConversation.sys_id
    ) {
      return;
    }

    const conversationSysId = String(activeChatConversation.sys_id);

    const groupName =
      activeChatConversation.display_name ||
      activeChatConversation.title ||
      "Group";

    /* =========================================
     RESET LEAVE GROUP UI
  ========================================= */

    if (chatGroupLeaveButton) {
      chatGroupLeaveButton.disabled = false;
      chatGroupLeaveButton.textContent = "Leave Group";
    }

    if (chatGroupLeaveMessage) {
      chatGroupLeaveMessage.textContent = "";
      chatGroupLeaveMessage.style.display = "none";
    }

    /* =========================================
       OPEN MODAL IMMEDIATELY
    ========================================= */

    if (chatGroupDetailsName) {
      chatGroupDetailsName.textContent = groupName;
    }

    if (chatGroupDetailsMembers) {
      chatGroupDetailsMembers.innerHTML = `
            <div style="
                color:#71827d;
                font-size:13px;
                padding:8px 0;
            ">
                Loading members...
            </div>
        `;
    }

    if (chatGroupDetailsModal) {
      chatGroupDetailsModal.style.display = "flex";

      chatGroupDetailsModal.setAttribute("aria-hidden", "false");
    }

    /* =========================================
       LOAD REAL GROUP DETAILS
    ========================================= */

    try {
      const result =
        await window.serviceCall.getGroupDetails(conversationSysId);

      if (!result || !result.success) {
        throw new Error(
          result && result.message
            ? result.message
            : "Unable to load group details.",
        );
      }

      currentChatGroupDetails = result;

      /*
       * User may have switched conversations
       * while the request was running.
       */

      if (
        !activeChatConversation ||
        String(activeChatConversation.sys_id) !== conversationSysId
      ) {
        return;
      }

      /* =========================================
   ADD PEOPLE PERMISSION
========================================= */

      if (chatGroupAddPeopleButton) {
        const currentMember = result.current_member || {};

        const currentRole = String(currentMember.role || "")
          .trim()
          .toLowerCase();

        /*
         * UI visibility is convenience only.
         *
         * ServiceNow remains the real authority.
         */
        const canAddPeople = currentRole === "owner";

        chatGroupAddPeopleButton.style.display = canAddPeople ? "" : "none";
      }

      /* =========================================
     GROUP NAME
========================================= */

      if (chatGroupDetailsName && result.group) {
        chatGroupDetailsName.textContent = result.group.title || groupName;
      }
      /* =========================================
           MEMBERS
        ========================================= */

      const members = Array.isArray(result.members) ? result.members : [];

      if (!chatGroupDetailsMembers) {
        return;
      }

      /*
       * The whole members section will now
       * live inside this existing container.
       */

      chatGroupDetailsMembers.innerHTML = "";

      /* =========================================
           MEMBERS HEADER
        ========================================= */

      const membersHeader = document.createElement("div");

      membersHeader.style.display = "flex";

      membersHeader.style.alignItems = "center";

      membersHeader.style.justifyContent = "space-between";

      membersHeader.style.gap = "10px";

      membersHeader.style.marginBottom = "10px";

      const membersCount = document.createElement("div");

      membersCount.style.fontSize = "13px";

      membersCount.style.fontWeight = "700";

      membersCount.textContent = "Members (" + members.length + ")";

      membersHeader.appendChild(membersCount);

      chatGroupDetailsMembers.appendChild(membersHeader);

      /* =========================================
           MEMBER SEARCH
        ========================================= */

      const searchInput = document.createElement("input");

      searchInput.type = "text";

      searchInput.placeholder = "Search members...";

      searchInput.autocomplete = "off";

      searchInput.style.width = "100%";

      searchInput.style.boxSizing = "border-box";

      searchInput.style.padding = "9px 11px";

      searchInput.style.border = "1px solid rgba(0, 0, 0, 0.12)";

      searchInput.style.borderRadius = "9px";

      searchInput.style.outline = "none";

      searchInput.style.fontSize = "13px";

      searchInput.style.marginBottom = "10px";

      chatGroupDetailsMembers.appendChild(searchInput);

      /* =========================================
           SCROLLABLE MEMBER LIST
        ========================================= */

      const memberList = document.createElement("div");

      memberList.style.maxHeight = "260px";

      memberList.style.overflowY = "auto";

      memberList.style.overflowX = "hidden";

      memberList.style.paddingRight = "4px";

      chatGroupDetailsMembers.appendChild(memberList);

      /* =========================================
           EMPTY GROUP
        ========================================= */

      if (members.length === 0) {
        const emptyState = document.createElement("div");

        emptyState.style.color = "#71827d";

        emptyState.style.fontSize = "13px";

        emptyState.style.padding = "8px 0";

        emptyState.textContent = "No members found.";

        memberList.appendChild(emptyState);

        searchInput.disabled = true;

        return;
      }

      /* =========================================
           RENDER MEMBERS
        ========================================= */

      function renderMembers(searchValue) {
        const query = String(searchValue || "")
          .trim()
          .toLowerCase();

        memberList.innerHTML = "";

        /* =========================================
     FILTER MEMBERS
  ========================================= */

        const filteredMembers = members.filter((member) => {
          if (!query) {
            return true;
          }

          const memberName = String(member.name || "").toLowerCase();

          const memberUsername = String(member.user_name || "").toLowerCase();

          const memberEmail = String(member.email || "").toLowerCase();

          return (
            memberName.includes(query) ||
            memberUsername.includes(query) ||
            memberEmail.includes(query)
          );
        });

        /* =========================================
     NO SEARCH RESULTS
  ========================================= */

        if (filteredMembers.length === 0) {
          const noResults = document.createElement("div");

          noResults.style.color = "#71827d";

          noResults.style.fontSize = "13px";

          noResults.style.padding = "12px 2px";

          noResults.textContent = "No matching members.";

          memberList.appendChild(noResults);

          return;
        }

        /* =========================================
     CURRENT USER ROLE
  ========================================= */

        const currentMemberRole = String(
          result.current_member && result.current_member.role
            ? result.current_member.role
            : "",
        )
          .trim()
          .toLowerCase();

        const currentUserIsOwner = currentMemberRole === "owner";

        /* =========================================
     MEMBER ROWS
  ========================================= */

        filteredMembers.forEach((member) => {
          const row = document.createElement("div");

          row.style.display = "flex";

          row.style.alignItems = "center";

          row.style.justifyContent = "space-between";

          row.style.gap = "12px";

          row.style.padding = "10px 2px";

          row.style.borderBottom = "1px solid rgba(0, 0, 0, 0.06)";

          /* =====================================
         LEFT SIDE
      ===================================== */

          const person = document.createElement("div");

          person.style.minWidth = "0";

          person.style.flex = "1";

          /* -------------------------
         NAME
      ------------------------- */

          const name = document.createElement("div");

          name.style.fontSize = "14px";

          name.style.fontWeight = "600";

          name.style.whiteSpace = "nowrap";

          name.style.overflow = "hidden";

          name.style.textOverflow = "ellipsis";

          name.textContent = member.name || member.user_name || "Unknown User";

          if (member.is_me) {
            name.textContent += " (You)";
          }

          person.appendChild(name);

          /* -------------------------
         USERNAME
      ------------------------- */

          if (member.user_name) {
            const username = document.createElement("div");

            username.style.fontSize = "12px";

            username.style.color = "#71827d";

            username.style.marginTop = "2px";

            username.style.whiteSpace = "nowrap";

            username.style.overflow = "hidden";

            username.style.textOverflow = "ellipsis";

            username.textContent = "@" + member.user_name;

            person.appendChild(username);
          }

          /* -------------------------
         EMAIL
      ------------------------- */

          if (member.email) {
            const email = document.createElement("div");

            email.style.fontSize = "11px";

            email.style.color = "#8a9995";

            email.style.marginTop = "2px";

            email.style.whiteSpace = "nowrap";

            email.style.overflow = "hidden";

            email.style.textOverflow = "ellipsis";

            email.textContent = member.email;

            person.appendChild(email);
          }

          /* =====================================
         RIGHT SIDE
      ===================================== */

          const rightSide = document.createElement("div");

          rightSide.style.display = "flex";

          rightSide.style.alignItems = "center";

          rightSide.style.gap = "8px";

          rightSide.style.flexShrink = "0";

          /* =====================================
         MEMBER ROLE
      ===================================== */

          const rawMemberRole = String(member.role || "member")
            .trim()
            .toLowerCase();

          /*
           * New group model:
           *
           * Owner
           * Member
           *
           * Legacy "admin" records are
           * temporarily displayed as Member.
           */

          const memberRole = rawMemberRole === "owner" ? "owner" : "member";

          /* =====================================
         ROLE LABEL
      ===================================== */

          const role = document.createElement("div");

          role.style.fontSize = "12px";

          role.style.fontWeight = "600";

          role.style.whiteSpace = "nowrap";

          role.textContent = memberRole === "owner" ? "Owner" : "Member";

          rightSide.appendChild(role);

          /* =====================================
         OWNER MANAGEMENT
      ===================================== */

          /*
           * Only Owners receive management
           * controls.
           *
           * Never show management controls
           * against yourself.
           */

          if (currentUserIsOwner && !member.is_me) {
            const targetUserSysId = String(member.user_sys_id || "").trim();

            /* =================================
           ROLE BUTTON
        ================================= */

            const newRole = memberRole === "owner" ? "member" : "owner";

            const roleButton = document.createElement("button");

            roleButton.type = "button";

            roleButton.style.fontSize = "11px";

            roleButton.style.padding = "5px 8px";

            roleButton.style.borderRadius = "7px";

            roleButton.style.cursor = "pointer";

            roleButton.textContent =
              memberRole === "owner" ? "Make Member" : "Make Owner";

            roleButton.addEventListener("click", async () => {
              if (roleButton.disabled) {
                return;
              }

              if (!targetUserSysId) {
                console.error("Missing member user sys_id.", member);

                return;
              }

              /*
               * Lock BOTH actions while
               * this member is being changed.
               */

              roleButton.disabled = true;

              removeButton.disabled = true;

              roleButton.textContent =
                newRole === "owner" ? "Making Owner..." : "Making Member...";

              try {
                const changeResult =
                  await window.serviceCall.setGroupMemberRole(
                    conversationSysId,
                    targetUserSysId,
                    newRole,
                  );

                if (!changeResult || changeResult.success !== true) {
                  throw new Error(
                    changeResult && changeResult.message
                      ? changeResult.message
                      : "Unable to change member role.",
                  );
                }

                /* -------------------------
                 STALE GROUP GUARD
              ------------------------- */

                if (
                  !activeChatConversation ||
                  String(activeChatConversation.sys_id) !== conversationSysId
                ) {
                  return;
                }

                /* -------------------------
                 REFRESH DETAILS
              ------------------------- */

                await openChatGroupDetails();
              } catch (error) {
                console.error("Unable to change group member role:", error);

                roleButton.disabled = false;

                removeButton.disabled = false;

                roleButton.textContent =
                  memberRole === "owner" ? "Make Member" : "Make Owner";
              }
            });

            /* =================================
           REMOVE BUTTON
        ================================= */

            const removeButton = document.createElement("button");

            removeButton.type = "button";

            removeButton.style.fontSize = "11px";

            removeButton.style.padding = "5px 8px";

            removeButton.style.borderRadius = "7px";

            removeButton.style.cursor = "pointer";

            removeButton.textContent = "Remove";

            removeButton.addEventListener("click", async () => {
              if (removeButton.disabled) {
                return;
              }

              if (!targetUserSysId) {
                console.error("Missing member user sys_id.", member);

                return;
              }

              /*
               * We deliberately do not use
               * window.confirm() here.
               *
               * Electron confirmation UX can
               * be added later with our own
               * modal during UI polish.
               */

              /* -------------------------
               LOCK ACTIONS
            ------------------------- */

              removeButton.disabled = true;

              roleButton.disabled = true;

              removeButton.textContent = "Removing...";

              try {
                const removeResult = await window.serviceCall.removeGroupMember(
                  conversationSysId,
                  targetUserSysId,
                );

                if (!removeResult || removeResult.success !== true) {
                  throw new Error(
                    removeResult && removeResult.message
                      ? removeResult.message
                      : "Unable to remove group member.",
                  );
                }

                /* -------------------------
                 STALE GROUP GUARD
              ------------------------- */

                if (
                  !activeChatConversation ||
                  String(activeChatConversation.sys_id) !== conversationSysId
                ) {
                  return;
                }

                /* -------------------------
                 REFRESH GROUP DETAILS
              ------------------------- */

                await openChatGroupDetails();

                /*
                 * The existing chat sync will
                 * retrieve the system message:
                 *
                 * "X removed Y from the group."
                 */
              } catch (error) {
                console.error("Unable to remove group member:", error);

                removeButton.disabled = false;

                roleButton.disabled = false;

                removeButton.textContent = "Remove";
              }
            });

            /* =================================
           ADD OWNER CONTROLS
        ================================= */

            rightSide.appendChild(roleButton);

            rightSide.appendChild(removeButton);
          }

          /* =====================================
         ADD ROW
      ===================================== */

          row.appendChild(person);

          row.appendChild(rightSide);

          memberList.appendChild(row);
        });
      }

      /* =========================================
           INITIAL MEMBER RENDER
        ========================================= */

      renderMembers("");

      /* =========================================
           LIVE SEARCH
        ========================================= */

      searchInput.addEventListener("input", () => {
        renderMembers(searchInput.value);
      });
    } catch (error) {
      console.error("Unable to load group details:", error);

      if (chatGroupDetailsMembers) {
        chatGroupDetailsMembers.innerHTML = `
                <div style="
                    color:#a33;
                    font-size:13px;
                    padding:8px 0;
                ">
                    Unable to load group members.
                </div>
            `;
      }
    }
  }

  function closeChatGroupDetails() {
    if (!chatGroupDetailsModal) {
      return;
    }

    chatGroupDetailsModal.style.display = "none";

    chatGroupDetailsModal.setAttribute("aria-hidden", "true");
  }

  /* =======================================================
   GROUP DETAILS - ADD PEOPLE PANEL
======================================================= */

  function resetChatGroupAddPeople() {
    chatGroupAddPeopleSelectedUsers.clear();

    if (chatGroupAddPeopleSearch) {
      chatGroupAddPeopleSearch.value = "";
    }

    if (chatGroupAddPeopleResults) {
      chatGroupAddPeopleResults.innerHTML = "";
    }

    if (chatGroupAddPeopleMessage) {
      chatGroupAddPeopleMessage.textContent = "";
    }

    if (chatGroupAddPeopleSaveButton) {
      chatGroupAddPeopleSaveButton.disabled = true;

      chatGroupAddPeopleSaveButton.textContent = "Add Selected";
    }
  }

  function openChatGroupAddPeople() {
    if (
      !activeChatConversation ||
      activeChatConversation.type !== "group" ||
      !activeChatConversation.sys_id ||
      !currentChatGroupDetails
    ) {
      return;
    }

    resetChatGroupAddPeople();

    if (chatGroupAddPeoplePanel) {
      chatGroupAddPeoplePanel.style.display = "block";
    }

    if (chatGroupAddPeopleSearch) {
      setTimeout(() => {
        chatGroupAddPeopleSearch.focus();
      }, 0);
    }
  }

  function closeChatGroupAddPeople() {
    if (chatGroupAddPeoplePanel) {
      chatGroupAddPeoplePanel.style.display = "none";
    }

    resetChatGroupAddPeople();
  }

  /* =========================================
   ADD PEOPLE - OPEN
========================================= */

  if (chatGroupAddPeopleButton) {
    chatGroupAddPeopleButton.addEventListener("click", () => {
      openChatGroupAddPeople();
    });
  }

  /* =========================================
   ADD PEOPLE - CANCEL
========================================= */

  if (chatGroupAddPeopleCancelButton) {
    chatGroupAddPeopleCancelButton.addEventListener("click", () => {
      closeChatGroupAddPeople();
    });
  }

  function renderChatGroupAddPeopleResults(users) {
    if (!chatGroupAddPeopleResults) {
      return;
    }

    chatGroupAddPeopleResults.innerHTML = "";

    const safeUsers = Array.isArray(users) ? users : [];

    safeUsers.forEach((user) => {
      const userSysId = String(user.sys_id || "").trim();

      if (!userSysId) {
        return;
      }

      const row = document.createElement("div");

      row.style.display = "flex";

      row.style.alignItems = "center";

      row.style.gap = "10px";

      row.style.padding = "9px 2px";

      row.style.cursor = "pointer";

      row.style.borderBottom = "1px solid rgba(0,0,0,0.06)";

      /* -------------------------
       CHECKBOX
    ------------------------- */

      const checkbox = document.createElement("input");

      checkbox.type = "checkbox";

      checkbox.checked = chatGroupAddPeopleSelectedUsers.has(userSysId);

      /* -------------------------
       PERSON
    ------------------------- */

      const person = document.createElement("div");

      person.style.flex = "1";

      person.style.minWidth = "0";

      const name = document.createElement("div");

      name.style.fontSize = "13px";

      name.style.fontWeight = "600";

      name.textContent = user.name || user.user_name || "Unknown User";

      person.appendChild(name);

      if (user.user_name) {
        const username = document.createElement("div");

        username.style.fontSize = "11px";

        username.style.color = "#71827d";

        username.style.marginTop = "2px";

        username.textContent = "@" + user.user_name;

        person.appendChild(username);
      }

      /* -------------------------
       SELECTION
    ------------------------- */

      function updateSelection(selected) {
        checkbox.checked = selected;

        if (selected) {
          chatGroupAddPeopleSelectedUsers.set(userSysId, user);
        } else {
          chatGroupAddPeopleSelectedUsers.delete(userSysId);
        }

        if (chatGroupAddPeopleSaveButton) {
          const count = chatGroupAddPeopleSelectedUsers.size;

          chatGroupAddPeopleSaveButton.disabled = count === 0;

          chatGroupAddPeopleSaveButton.textContent =
            count > 0 ? "Add Selected (" + count + ")" : "Add Selected";
        }
      }

      /*
       * Row click.
       */
      row.addEventListener("click", (event) => {
        /*
         * Checkbox has its own change event.
         * Don't toggle twice.
         */
        if (event.target === checkbox) {
          return;
        }

        updateSelection(!checkbox.checked);
      });

      /*
       * Checkbox click/change.
       *
       * This explicitly fixes the checkbox
       * problem we saw in Create Group where
       * row clicks worked but the checkbox
       * itself could behave inconsistently.
       */
      checkbox.addEventListener("change", () => {
        updateSelection(checkbox.checked);
      });

      row.appendChild(checkbox);

      row.appendChild(person);

      chatGroupAddPeopleResults.appendChild(row);
    });
  }

  /* =======================================================
   GROUP DETAILS - ADD SELECTED PEOPLE
======================================================= */

  if (chatGroupAddPeopleSaveButton) {
    chatGroupAddPeopleSaveButton.addEventListener("click", async () => {
      /* -----------------------------------------
         VALIDATE ACTIVE GROUP
      ----------------------------------------- */

      if (
        !activeChatConversation ||
        activeChatConversation.type !== "group" ||
        !activeChatConversation.sys_id
      ) {
        return;
      }

      /* -----------------------------------------
         SELECTED USERS
      ----------------------------------------- */

      const selectedUserSysIds = Array.from(
        chatGroupAddPeopleSelectedUsers.keys(),
      );

      if (selectedUserSysIds.length === 0) {
        return;
      }

      const conversationSysId = String(activeChatConversation.sys_id).trim();

      /* -----------------------------------------
         LOCK BUTTON
      ----------------------------------------- */

      chatGroupAddPeopleSaveButton.disabled = true;

      chatGroupAddPeopleSaveButton.textContent = "Adding...";

      if (chatGroupAddPeopleMessage) {
        chatGroupAddPeopleMessage.textContent = "";
      }

      try {
        /* -----------------------------------------
           ADD MEMBERS
        ----------------------------------------- */

        const result = await window.serviceCall.addGroupMembers(
          conversationSysId,
          selectedUserSysIds,
        );

        if (!result || result.success !== true) {
          throw new Error(
            result && result.message ? result.message : "Unable to add people.",
          );
        }

        console.log("Group members added:", result);

        /* -----------------------------------------
           CLOSE ADD PEOPLE PANEL
        ----------------------------------------- */

        closeChatGroupAddPeople();

        /*
         * Make sure we're still looking at
         * the same group before refreshing.
         */
        if (
          !activeChatConversation ||
          String(activeChatConversation.sys_id) !== conversationSysId
        ) {
          return;
        }

        /* -----------------------------------------
           REFRESH GROUP DETAILS
        ----------------------------------------- */

        await openChatGroupDetails();

        /*
         * Existing chat synchronization will
         * retrieve the system message created
         * by ServiceNow:
         *
         * "<User> added X to the group."
         */
      } catch (error) {
        console.error("Unable to add group members:", error);

        if (chatGroupAddPeopleMessage) {
          chatGroupAddPeopleMessage.textContent =
            error && error.message ? error.message : "Unable to add people.";
        }

        /*
         * Restore button because the operation
         * failed.
         */

        const count = chatGroupAddPeopleSelectedUsers.size;

        chatGroupAddPeopleSaveButton.disabled = count === 0;

        chatGroupAddPeopleSaveButton.textContent =
          count > 0 ? "Add Selected (" + count + ")" : "Add Selected";
      }
    });
  }

  /* =======================================================
   GROUP DETAILS - SEARCH PEOPLE TO ADD
======================================================= */

  if (chatGroupAddPeopleSearch) {
    chatGroupAddPeopleSearch.addEventListener("input", () => {
      clearTimeout(chatGroupAddPeopleSearchTimer);

      const searchText = String(chatGroupAddPeopleSearch.value || "").trim();

      /*
       * Same minimum search length used by
       * the existing ServiceCall user search.
       */
      if (searchText.length < 2) {
        if (chatGroupAddPeopleResults) {
          chatGroupAddPeopleResults.innerHTML = "";
        }

        if (chatGroupAddPeopleMessage) {
          chatGroupAddPeopleMessage.textContent = searchText
            ? "Enter at least 2 characters."
            : "";
        }

        return;
      }

      chatGroupAddPeopleSearchTimer = setTimeout(async () => {
        if (chatGroupAddPeopleMessage) {
          chatGroupAddPeopleMessage.textContent = "Searching...";
        }

        try {
          const result = await window.serviceCall.searchUsers(searchText);

          /*
           * Ignore an old response if the user
           * has already typed something else.
           */
          if (
            !chatGroupAddPeopleSearch ||
            chatGroupAddPeopleSearch.value.trim() !== searchText
          ) {
            return;
          }

          if (!result || result.success !== true) {
            throw new Error(
              result && result.message
                ? result.message
                : "Unable to search users.",
            );
          }

          const users = Array.isArray(result.users) ? result.users : [];

          /*
           * Current active group members must
           * not appear in Add People results.
           */
          const existingMemberIds = new Set(
            (Array.isArray(
              currentChatGroupDetails && currentChatGroupDetails.members,
            )
              ? currentChatGroupDetails.members
              : []
            )
              .map((member) =>
                String(member.user_sys_id || member.sys_id || "").trim(),
              )
              .filter(Boolean),
          );

          const availableUsers = users.filter((user) => {
            const userSysId = String(user.sys_id || "").trim();

            if (!userSysId) {
              return false;
            }

            return !existingMemberIds.has(userSysId);
          });

          renderChatGroupAddPeopleResults(availableUsers);

          if (chatGroupAddPeopleMessage) {
            chatGroupAddPeopleMessage.textContent = availableUsers.length
              ? ""
              : "No users available to add.";
          }
        } catch (error) {
          console.error("Unable to search group users:", error);

          if (chatGroupAddPeopleResults) {
            chatGroupAddPeopleResults.innerHTML = "";
          }

          if (chatGroupAddPeopleMessage) {
            chatGroupAddPeopleMessage.textContent =
              error.message || "Unable to search users.";
          }
        }
      }, 300);
    });
  }

  if (chatGroupDetailsButton) {
    chatGroupDetailsButton.addEventListener("click", () => {
      openChatGroupDetails();
    });
  }

  if (chatGroupDetailsCloseButton) {
    chatGroupDetailsCloseButton.addEventListener("click", () => {
      closeChatGroupDetails();
    });
  }

  if (chatGroupDetailsBackdrop) {
    chatGroupDetailsBackdrop.addEventListener("click", () => {
      closeChatGroupDetails();
    });
  }

  /* =========================================
   LEAVE GROUP
========================================= */

  if (chatGroupLeaveButton) {
    let leaveGroupRunning = false;

    chatGroupLeaveButton.addEventListener("click", async () => {
      /* -----------------------------------------
         PREVENT DUPLICATE EXECUTION
      ----------------------------------------- */

      if (leaveGroupRunning) {
        return;
      }

      /* -----------------------------------------
         VALIDATE ACTIVE GROUP
      ----------------------------------------- */

      if (
        !activeChatConversation ||
        activeChatConversation.type !== "group" ||
        !activeChatConversation.sys_id
      ) {
        return;
      }

      const conversationSysId = String(activeChatConversation.sys_id).trim();

      if (!conversationSysId) {
        return;
      }

      /* -----------------------------------------
         LOCK ACTION
      ----------------------------------------- */

      leaveGroupRunning = true;

      chatGroupLeaveButton.disabled = true;

      chatGroupLeaveButton.textContent = "Leaving...";

      if (chatGroupLeaveMessage) {
        chatGroupLeaveMessage.style.display = "none";

        chatGroupLeaveMessage.textContent = "";
      }

      try {
        /* -----------------------------------------
           SERVICECALL BACKEND
        ----------------------------------------- */

        const result = await window.serviceCall.leaveGroup(conversationSysId);

        console.log("ServiceCall Leave Group:", result);

        if (!result || result.success !== true) {
          throw new Error(
            result && result.message
              ? result.message
              : "Unable to leave group.",
          );
        }

        /* -----------------------------------------
           STALE CONVERSATION GUARD

           The user could theoretically switch
           chats while ServiceNow is responding.
        ----------------------------------------- */

        if (
          activeChatConversation &&
          String(activeChatConversation.sys_id || "") !== conversationSysId
        ) {
          return;
        }

        /* -----------------------------------------
           CLOSE GROUP DETAILS
        ----------------------------------------- */

        closeChatGroupDetails();

        closeChatGroupAddPeople();

        currentChatGroupDetails = null;

        /* -----------------------------------------
   CONVERT CURRENT GROUP TO
   HISTORICAL / READ-ONLY STATE
----------------------------------------- */

        if (
          activeChatConversation &&
          String(activeChatConversation.sys_id || "") === conversationSysId
        ) {
          activeChatConversation.membership_active = false;

          /*
           * Preserve the leave boundary locally when
           * the backend returns it.
           */
          if (result.membership_left_at) {
            activeChatConversation.membership_left_at =
              result.membership_left_at;
          }
        }

        activeChatUser = null;

        /* -----------------------------------------
   REFRESH CURRENT CHAT AS HISTORICAL
----------------------------------------- */

        if (
          activeChatConversation &&
          String(activeChatConversation.sys_id || "") === conversationSysId
        ) {
          await openChatConversation(activeChatConversation);
        }

        /* -----------------------------------------
   KEEP HISTORICAL CHAT OPEN

   The leave request succeeded, so this
   conversation immediately becomes a
   local read-only historical conversation.

   Do NOT clear messages.
   Do NOT hide the conversation panel.
   Do NOT reload the whole sidebar.
----------------------------------------- */

        if (chatEmptyState) {
          chatEmptyState.style.display = "none";
        }

        if (chatConversationPanel) {
          chatConversationPanel.style.display = "flex";
        }

        /* -----------------------------------------
   RESET COMPOSER CONTENT
----------------------------------------- */

        if (chatMessageInput) {
          chatMessageInput.value = "";

          /*
           * Keep the input available so our existing
           * read-only safeguard can explain why the
           * user can no longer send if they type.
           */
          chatMessageInput.disabled = false;

          chatMessageInput.placeholder = "Message group...";

          resizeChatMessageInput();
        }

        if (chatSendButton) {
          chatSendButton.disabled = true;
        }

        /* -----------------------------------------
   CLEAR MEMBERSHIP WARNING

   Leaving itself should not immediately show
   an error. The warning appears only if the
   former member attempts to type/send/react.
----------------------------------------- */

        if (chatMembershipMessage) {
          chatMembershipMessage.textContent = "";
          chatMembershipMessage.style.display = "none";
        }

        /* -----------------------------------------
   REMOVE ACTIVE-ONLY GROUP ACTIONS
----------------------------------------- */

        if (chatGroupDetailsButton) {
          chatGroupDetailsButton.style.display = "none";
        }

        /* -----------------------------------------
   UPDATE ONLY THIS SIDEBAR ROW LOCALLY

   Background conversation sync can reconcile
   the server state later without a visible
   full-list reload.
----------------------------------------- */

        const leftConversationRow = chatConversationList
          ? chatConversationList.querySelector(
              `button[data-conversation-sys-id="${conversationSysId}"]`,
            )
          : null;

        if (leftConversationRow) {
          leftConversationRow.dataset.membershipActive = "false";
        }

        await syncChatConversationList();

        /* -----------------------------------------
           DIAGNOSTIC

           Useful for our ownership-transfer test.
        ----------------------------------------- */

        if (result.ownership_transferred === true) {
          console.log(
            "Group ownership automatically transferred:",
            result.new_owner,
          );
        }

        if (result.group_closed === true) {
          console.log("Last member left. Group is now inactive.");
        }
      } catch (error) {
        console.error("Unable to leave ServiceCall group:", error);

        /* -----------------------------------------
           SHOW ERROR IN MODAL
        ----------------------------------------- */

        if (chatGroupLeaveMessage) {
          chatGroupLeaveMessage.textContent =
            error && error.message ? error.message : "Unable to leave group.";

          chatGroupLeaveMessage.style.display = "block";
        }

        /* -----------------------------------------
           RESTORE ACTION
        ----------------------------------------- */

        chatGroupLeaveButton.disabled = false;

        chatGroupLeaveButton.textContent = "Leave Group";
      } finally {
        leaveGroupRunning = false;
      }
    });
  }

  /* =========================================
   RENAME GROUP
========================================= */

  if (
    chatGroupRenameButton &&
    chatGroupRenameEditor &&
    chatGroupRenameInput &&
    chatGroupRenameSaveButton &&
    chatGroupRenameCancelButton
  ) {
    function closeGroupRenameEditor() {
      chatGroupRenameEditor.style.display = "none";

      chatGroupRenameButton.style.display = "";

      chatGroupRenameInput.value = "";
    }

    chatGroupRenameButton.addEventListener("click", () => {
      if (
        !activeChatConversation ||
        activeChatConversation.type !== "group" ||
        !activeChatConversation.sys_id
      ) {
        return;
      }

      const currentTitle =
        activeChatConversation.title ||
        activeChatConversation.display_name ||
        "";

      chatGroupRenameInput.value = currentTitle;

      chatGroupRenameButton.style.display = "none";

      chatGroupRenameEditor.style.display = "flex";

      chatGroupRenameInput.focus();

      chatGroupRenameInput.select();
    });

    chatGroupRenameCancelButton.addEventListener("click", () => {
      closeGroupRenameEditor();
    });

    async function saveGroupRename() {
      if (
        !activeChatConversation ||
        activeChatConversation.type !== "group" ||
        !activeChatConversation.sys_id
      ) {
        return;
      }

      const conversationSysId = String(activeChatConversation.sys_id).trim();

      const oldTitle = String(
        activeChatConversation.title ||
          activeChatConversation.display_name ||
          "",
      ).trim();

      const newTitle = String(chatGroupRenameInput.value || "").trim();

      if (!newTitle) {
        chatGroupRenameInput.focus();

        return;
      }

      if (newTitle === oldTitle) {
        closeGroupRenameEditor();

        return;
      }

      /* =========================================
       LOCK RENAME ACTION IMMEDIATELY
    ========================================= */

      chatGroupRenameSaveButton.disabled = true;

      chatGroupRenameCancelButton.disabled = true;

      chatGroupRenameInput.disabled = true;

      chatGroupRenameSaveButton.textContent = "Saving...";

      try {
        const result = await window.serviceCall.renameGroup(
          conversationSysId,
          newTitle,
        );

        console.log(
          "🔥 RENAME FIRST RESPONSE:",
          JSON.stringify(result, null, 2),
        );

        if (!result || !result.success) {
          throw new Error(
            result && result.message
              ? result.message
              : "Unable to rename group.",
          );
        }

        /*
         * User may have switched conversations
         * while ServiceNow was responding.
         */
        if (
          !activeChatConversation ||
          String(activeChatConversation.sys_id) !== conversationSysId
        ) {
          return;
        }

        const finalTitle = String(
          result.group && result.group.title ? result.group.title : newTitle,
        ).trim();

        /* =========================================
           1. UPDATE ACTIVE CONVERSATION OBJECT
        ========================================= */

        activeChatConversation.title = finalTitle;

        activeChatConversation.display_name = finalTitle;

        /* =========================================
           2. UPDATE OPEN CHAT HEADER IMMEDIATELY
        ========================================= */

        if (chatUserName) {
          chatUserName.textContent = finalTitle;
        }

        /* =========================================
           3. UPDATE GROUP DETAILS IMMEDIATELY
        ========================================= */

        if (chatGroupDetailsName) {
          chatGroupDetailsName.textContent = finalTitle;
        }

        /* =========================================
           4. UPDATE SIDEBAR IMMEDIATELY
        ========================================= */

        if (chatConversationList) {
          const conversationRows = chatConversationList.querySelectorAll(
            "button[data-conversation-sys-id]",
          );

          conversationRows.forEach((row) => {
            if (
              String(row.dataset.conversationSysId || "") !== conversationSysId
            ) {
              return;
            }

            /*
             * Current sidebar structure:
             *
             * row
             *   avatar
             *   information
             *      name
             *      preview
             */

            const information = row.children[1];

            if (information) {
              const nameElement = information.children[0];

              if (nameElement) {
                nameElement.textContent = finalTitle;
              }
            }
          });
        }

        /* =========================================
           5. CLOSE RENAME EDITOR
        ========================================= */

        closeGroupRenameEditor();

        /* =========================================
           6. BACKGROUND AUTHORITATIVE SIDEBAR SYNC
        ========================================= */

        if (typeof syncChatConversationList === "function") {
          await syncChatConversationList();
        }

        /* =========================================
           7. FETCH SYSTEM MESSAGE
        ========================================= */

        if (typeof checkForNewChatMessages === "function") {
          await checkForNewChatMessages();
        }
      } catch (error) {
        console.error("Unable to rename group:", error);

        /*
         * Keep editor open and preserve
         * the typed title on failure.
         */
        chatGroupRenameInput.focus();
      } finally {
        chatGroupRenameSaveButton.disabled = false;

        chatGroupRenameCancelButton.disabled = false;

        chatGroupRenameInput.disabled = false;

        chatGroupRenameSaveButton.textContent = "Save";
      }
    }

    chatGroupRenameSaveButton.addEventListener("click", async () => {
      await saveGroupRename();
    });

    chatGroupRenameInput.addEventListener("keydown", async (event) => {
      if (event.key === "Enter") {
        event.preventDefault();

        await saveGroupRename();

        return;
      }

      if (event.key === "Escape") {
        event.preventDefault();

        closeGroupRenameEditor();
      }
    });
  }

  async function prewarmChatMessageCache(conversations = []) {
    if (!Array.isArray(conversations)) {
      return;
    }

    /*
     * Start small.
     *
     * Pre-warm only the first 5 conversations
     * currently returned in sidebar order.
     *
     * This avoids hammering ServiceNow if the
     * user has many conversations.
     */
    const conversationsToWarm = conversations.slice(0, 5);

    for (const conversation of conversationsToWarm) {
      const conversationSysId = String(
        conversation && conversation.sys_id ? conversation.sys_id : "",
      ).trim();

      if (!conversationSysId) {
        continue;
      }

      /*
       * Already cached from an earlier open
       * or pre-warm.
       */
      if (chatMessageCache.has(conversationSysId)) {
        continue;
      }

      try {
        const result = await window.serviceCall.getMessages(conversationSysId);

        if (!result || result.success !== true) {
          continue;
        }

        const messages = Array.isArray(result.messages) ? result.messages : [];

        chatMessageCache.set(conversationSysId, {
          messages: messages.slice(),

          pagination: {
            has_more: !!(
              result.pagination && result.pagination.has_more === true
            ),

            before:
              result.pagination && result.pagination.before
                ? String(result.pagination.before).trim()
                : messages.length > 0
                  ? String(messages[0].sys_id || "").trim()
                  : "",
          },
        });
      } catch (error) {
        /*
         * Pre-warming is only an optimization.
         *
         * It must never interfere with normal
         * Chat operation.
         */
        console.warn("Unable to pre-warm chat:", conversationSysId, error);
      }
    }
  }

  /* =============================================
   SERVICECALL CHAT - OPEN CONVERSATION
============================================= */

  async function openChatConversation(conversation) {
    if (chatMembershipMessage) {
      chatMembershipMessage.textContent = "";
      chatMembershipMessage.style.display = "none";
    }

    if (!conversation || !conversation.sys_id) {
      return;
    }

    /*
     * Remove any unsent temporary direct chat
     * before opening a persisted conversation.
     *
     * A temporary row exists only locally and
     * has no ServiceNow conversation sys_id.
     */
    if (chatConversationList) {
      chatConversationList
        .querySelectorAll('button[data-temporary-chat="true"]')
        .forEach((row) => {
          row.remove();
        });
    }

    activeChatConversation = conversation;

    /*
     * IMPORTANT:
     *
     * A message checkpoint belongs to one
     * specific conversation.
     *
     * Clear the previous conversation's
     * checkpoint immediately when switching
     * conversations.
     *
     * The fresh checkpoint will be established
     * after this conversation's messages load.
     */
    lastChatMessageSysId = "";

    lastChatEditCheckpoint = "";
    /*
     * Reset older-history pagination whenever
     * a different conversation is opened.
     */
    oldestChatMessageSysId = "";

    chatMessageHistoryHasMore = false;

    chatMessageHistoryLoading = false;

    /*
     * Reaction checkpoints are also scoped
     * to a conversation.
     *
     * Do not allow the previous conversation's
     * checkpoint to be used while this one loads.
     */
    lastChatReactionCheckpoint = "";

    const openingConversationSysId = String(conversation.sys_id);

    /* =========================================
   ACTIVE DIRECT-CHAT USER
========================================= */

    /*
     * activeChatUser represents ONE person.
     *
     * Therefore it must exist only for a
     * direct conversation.
     *
     * A group conversation has members,
     * not one "active user".
     */
    if (conversation.type === "direct" && conversation.other_user_sys_id) {
      activeChatUser = {
        sys_id: conversation.other_user_sys_id,

        name:
          conversation.other_user_name ||
          conversation.display_name ||
          "Unknown User",

        user_name: conversation.other_user_user_name || "",

        display_status: "Offline",
      };
    } else {
      activeChatUser = null;
    }

    const displayName =
      conversation.display_name || conversation.title || "Conversation";

    /* =========================================
       HEADER
    ========================================= */

    if (chatUserName) {
      chatUserName.textContent = displayName;
    }

    /* =========================================
   GROUP DETAILS BUTTON
========================================= */

    if (chatGroupDetailsButton) {
      const canOpenGroupDetails =
        conversation.type === "group" &&
        conversation.membership_active !== false;

      chatGroupDetailsButton.style.display = canOpenGroupDetails ? "" : "none";
    }

    /* =========================================
       AVATAR
    ========================================= */

    if (chatUserAvatar) {
      const nameParts = displayName.trim().split(/\s+/).filter(Boolean);

      let initials = "?";

      if (nameParts.length >= 2) {
        initials = (
          nameParts[0][0] + nameParts[nameParts.length - 1][0]
        ).toUpperCase();
      } else if (nameParts.length === 1) {
        initials = nameParts[0][0].toUpperCase();
      }

      chatUserAvatar.textContent = initials;
    }

    /* =========================================
       PRESENCE
    ========================================= */

    if (chatUserPresenceText) {
      chatUserPresenceText.textContent =
        conversation.type === "group" ? "" : "Offline";
    }

    if (chatUserPresenceDot) {
      if (conversation.type === "group") {
        /*
         * Groups do not have a single
         * presence state.
         *
         * Individual member presence will
         * be shown later in Group Details.
         */
        chatUserPresenceDot.style.display = "none";
      } else {
        /*
         * Direct conversation:
         * show normal user presence.
         */
        chatUserPresenceDot.style.display = "";

        chatUserPresenceDot.className = "chat-user-presence-dot offline";
      }
    }

    /* =========================================
       SHOW CONVERSATION
    ========================================= */

    if (chatEmptyState) {
      chatEmptyState.style.display = "none";
    }

    if (chatConversationPanel) {
      chatConversationPanel.style.display = "flex";
    }

    /*
     * =========================================
     * INSTANT CACHED MESSAGE RENDER
     * =========================================
     */

    const cachedConversation = chatMessageCache.get(openingConversationSysId);

    const openedFromCache = !!(
      cachedConversation && Array.isArray(cachedConversation.messages)
    );

    if (cachedConversation && Array.isArray(cachedConversation.messages)) {
      const cachedMessages = cachedConversation.messages;

      /*
       * Restore pagination state belonging
       * to this conversation.
       */
      oldestChatMessageSysId = String(
        cachedConversation.pagination && cachedConversation.pagination.before
          ? cachedConversation.pagination.before
          : cachedMessages.length > 0
            ? cachedMessages[0].sys_id || ""
            : "",
      ).trim();

      chatMessageHistoryHasMore = !!(
        cachedConversation.pagination &&
        cachedConversation.pagination.has_more === true
      );

      /*
       * Establish the live-sync checkpoint
       * immediately from the cached newest
       * message.
       */
      if (cachedMessages.length > 0) {
        const cachedNewestMessage = cachedMessages[cachedMessages.length - 1];

        lastChatMessageSysId = String(cachedNewestMessage.sys_id || "").trim();
      } else {
        lastChatMessageSysId = "";
      }

      /*
       * Render immediately.
       *
       * No ServiceNow wait.
       */
      if (chatMessages) {
        chatMessages.innerHTML = "";

        if (cachedMessages.length === 0) {
          chatMessages.innerHTML = `
        <div class="chat-message-placeholder">
          <div>
            This is the beginning of your conversation.
          </div>
        </div>
      `;
        } else {
          cachedMessages.forEach((message) => {
            appendChatMessage(message);
          });

          chatMessages.scrollTop = chatMessages.scrollHeight;
        }
      }
    } else {
      /*
       * First-ever open during this renderer
       * session has no cache yet.
       *
       * Clear the previous conversation while
       * ServiceNow retrieves the initial page.
       */
      if (chatMessages) {
        chatMessages.innerHTML = "";
      }
    }

    /*
     * Disable composer while the authoritative
     * request is running.
     */
    if (chatMessageInput) {
      chatMessageInput.disabled = true;
    }

    if (chatSendButton) {
      chatSendButton.disabled = true;
    }

    try {
      /* =========================================
           ONLY BLOCKING REQUEST:
           LOAD MESSAGES
        ========================================= */

      const result = await window.serviceCall.getMessages(conversation.sys_id);

      /*
       * User switched conversations while
       * ServiceNow was responding.
       *
       * Ignore this old response.
       */
      if (
        !activeChatConversation ||
        String(activeChatConversation.sys_id) !== openingConversationSysId
      ) {
        return;
      }

      console.log("ServiceCall conversation messages:", result);

      if (!result || result.success !== true) {
        throw new Error(
          result && result.message
            ? result.message
            : "Unable to load messages.",
        );
      }

      const messages = Array.isArray(result.messages) ? result.messages : [];

      /* =========================================
   HISTORY PAGINATION CHECKPOINT
========================================= */

      oldestChatMessageSysId =
        messages.length > 0 ? String(messages[0].sys_id || "").trim() : "";

      chatMessageHistoryHasMore = !!(
        result.pagination && result.pagination.has_more === true
      );

      chatMessageHistoryLoading = false;

      /* =========================================
   CACHE RECENT MESSAGE PAGE
========================================= */

      chatMessageCache.set(
        String(conversation.sys_id || "").trim(),

        {
          /*
           * Keep our own array so later changes to
           * the current response array do not
           * accidentally change the cache.
           */
          messages: messages.slice(),

          pagination: {
            has_more: chatMessageHistoryHasMore,

            before: oldestChatMessageSysId,
          },
        },
      );

      /* =========================================
     MESSAGE SYNC CHECKPOINT
  ========================================= */

      if (messages.length > 0) {
        const newestMessage = messages[messages.length - 1];

        lastChatMessageSysId = String(newestMessage.sys_id || "").trim();
      } else {
        lastChatMessageSysId = "";
      }

      /*
       * Establish the edit-sync checkpoint
       * from the newest edit currently loaded.
       *
       * This prevents old historical edits from
       * being treated as fresh edits when the
       * silent sync starts.
       */
      lastChatEditCheckpoint = "";

      for (const loadedMessage of messages) {
        const editedAt = String(loadedMessage.edited_at || "").trim();

        if (
          editedAt &&
          (!lastChatEditCheckpoint || editedAt > lastChatEditCheckpoint)
        ) {
          lastChatEditCheckpoint = editedAt;
        }
      }

      /*
       * Prefer the ServiceNow server checkpoint.
       *
       * This also initializes edit sync when this
       * conversation has never had an edit before.
       */
      const serverEditCheckpoint = String(result.edit_checkpoint || "").trim();

      if (serverEditCheckpoint) {
        lastChatEditCheckpoint = serverEditCheckpoint;
      }

      /* =========================================
     AUTHORITATIVE RENDER
  ========================================= */

      if (!chatMessages) {
        return;
      }

      /*
       * If cached messages were already visible,
       * the user may have started reading/scrolled
       * upward while the authoritative ServiceNow
       * request was still running.
       *
       * Their manual movement must win.
       */
      const preserveUserScroll = openedFromCache && !isChatNearBottom();

      const preservedScrollTop = preserveUserScroll
        ? chatMessages.scrollTop
        : 0;

      chatMessages.innerHTML = "";
      if (messages.length === 0) {
        chatMessages.innerHTML = `
                <div class="chat-message-placeholder">
                    <div>
                        This is the beginning of your conversation.
                    </div>
                </div>
            `;
      } else {
        messages.forEach((message) => {
          appendChatMessage(message);
        });

        /*
         * appendChatMessage() currently scrolls to
         * the bottom while rendering.
         *
         * If the user had already moved upward during
         * the cache → authoritative refresh, restore
         * their position after the complete page has
         * finished rendering.
         */
        if (preserveUserScroll) {
          chatMessages.scrollTop = preservedScrollTop;
        } else {
          chatMessages.scrollTop = chatMessages.scrollHeight;
        }
      }

      /* =========================================
   ENABLE COMPOSER
========================================= */

      const canSendToConversation =
        (conversation.type === "direct" && conversation.other_user_sys_id) ||
        (conversation.type === "group" && conversation.sys_id);

      if (canSendToConversation) {
        if (chatMessageInput) {
          chatMessageInput.disabled = false;

          chatMessageInput.placeholder =
            conversation.type === "group"
              ? "Message group..."
              : "Type a message...";
        }

        if (chatSendButton) {
          chatSendButton.disabled = !String(
            chatMessageInput ? chatMessageInput.value : "",
          ).trim();
        }
      }

      /*
       * IMPORTANT:
       *
       * Everything the user needs to SEE
       * has now been rendered.
       *
       * The operations below must not delay
       * conversation rendering.
       */

      /* =========================================
           BACKGROUND:
           MARK CONVERSATION READ
        ========================================= */

      window.serviceCall
        .markConversationRead(conversation.sys_id)
        .then((readResult) => {
          /*
           * Don't modify the currently
           * displayed conversation if
           * the user has already moved.
           */
          if (
            !activeChatConversation ||
            String(activeChatConversation.sys_id) !== openingConversationSysId
          ) {
            return;
          }

          if (readResult && readResult.success) {
            conversation.unread_count = 0;

            if (readResult.last_read_at) {
              conversation.last_read_at = String(readResult.last_read_at);
            }

            console.log("Conversation marked as read:", readResult);
          } else {
            console.warn("Unable to mark conversation as read:", readResult);
          }
        })
        .catch((readError) => {
          console.error("Unable to mark conversation as read:", readError);
        });

      /* =========================================
           BACKGROUND:
           REACTION CHECKPOINT
        ========================================= */

      lastChatReactionCheckpoint = "";

      window.serviceCall
        .getReactionUpdates(conversation.sys_id)
        .then((reactionSyncResult) => {
          if (
            !activeChatConversation ||
            String(activeChatConversation.sys_id) !== openingConversationSysId
          ) {
            return;
          }

          if (
            reactionSyncResult &&
            reactionSyncResult.success &&
            reactionSyncResult.checkpoint
          ) {
            lastChatReactionCheckpoint = String(
              reactionSyncResult.checkpoint,
            ).trim();
          }
        })
        .catch((error) => {
          console.error("Unable to establish chat reaction checkpoint:", error);
        });
    } catch (error) {
      /*
       * Don't show an error from an old
       * conversation after the user has
       * already switched elsewhere.
       */
      if (
        !activeChatConversation ||
        String(activeChatConversation.sys_id) !== openingConversationSysId
      ) {
        return;
      }

      console.error("Unable to open ServiceCall conversation:", error);

      if (chatMessages) {
        chatMessages.innerHTML = `
                <div class="chat-message-placeholder">
                    <div>
                        Unable to load messages.
                    </div>
                </div>
            `;
      }
    }
  }

  /* =======================================================
   SERVICECALL CHAT - SEND MESSAGE
======================================================= */

  let chatMessageSending = false;

  let chatReplyTarget = null;

  let chatEditTarget = null;

  /* =======================================================
   SERVICECALL CHAT - APPEND MESSAGE
======================================================= */

  function appendChatMessage(message) {
    if (!chatMessages || !message) {
      return;
    }

    /*
     * Remove beginning/loading placeholder
     * if one is currently visible.
     */
    const placeholder = chatMessages.querySelector(".chat-message-placeholder");

    if (placeholder) {
      placeholder.remove();
    }

    const messageSysId = String(message.sys_id || "").trim();

    /* =====================================================
     SYSTEM MESSAGE
  ===================================================== */

    const messageType = String(message.type || "text")
      .trim()
      .toLowerCase();

    if (messageType === "system") {
      /*
       * Duplicate protection for system messages.
       */
      if (messageSysId) {
        const existingSystemMessage = Array.from(
          chatMessages.querySelectorAll(".chat-message-row"),
        ).find(
          (existingRow) =>
            String(existingRow.dataset.messageSysId || "") === messageSysId,
        );

        if (existingSystemMessage) {
          console.log("Duplicate system message ignored:", messageSysId);

          return;
        }
      }

      /* =========================================
       SYSTEM MESSAGE ROW
    ========================================= */

      const systemRow = document.createElement("div");

      systemRow.className = "chat-message-row chat-system-message-row";

      if (messageSysId) {
        systemRow.dataset.messageSysId = messageSysId;
      }

      if (message.deleted === true) {
        messageRow.dataset.messageDeleted = "true";
      }

      systemRow.style.cssText = `
      width:100%;
      display:flex;
      flex-direction:column;
      align-items:center;
      justify-content:center;
      box-sizing:border-box;
      margin:16px 0;
      padding:0 24px;
      position:relative;
    `;

      /* =========================================
       EVENT LINE
    ========================================= */

      const systemLine = document.createElement("div");

      systemLine.style.cssText = `
      width:100%;
      display:flex;
      align-items:center;
      justify-content:center;
      gap:12px;
    `;

      /* -------------------------
       LEFT LINE
    ------------------------- */

      const leftLine = document.createElement("div");

      leftLine.style.cssText = `
      width:42px;
      height:1px;
      flex-shrink:0;
      background:linear-gradient(
        to right,
        transparent,
        rgba(96, 112, 108, 0.28)
      );
    `;

      /* -------------------------
       SYSTEM TEXT
    ------------------------- */

      const systemText = document.createElement("div");

      systemText.textContent = String(message.text || "");

      systemText.style.cssText = `
      max-width:70%;
      text-align:center;
      color:#7b8884;
      font-size:11.5px;
      font-weight:500;
      font-style:italic;
      line-height:1.45;
      letter-spacing:0.25px;
      white-space:normal;
      overflow-wrap:anywhere;
      opacity:0.88;
      text-shadow:
        0 1px 0 rgba(255,255,255,0.7);
    `;

      /* -------------------------
       RIGHT LINE
    ------------------------- */

      const rightLine = document.createElement("div");

      rightLine.style.cssText = `
      width:42px;
      height:1px;
      flex-shrink:0;
      background:linear-gradient(
        to left,
        transparent,
        rgba(96, 112, 108, 0.28)
      );
    `;

      systemLine.appendChild(leftLine);
      systemLine.appendChild(systemText);
      systemLine.appendChild(rightLine);

      /* =========================================
       TIME
    ========================================= */

      const systemTime = document.createElement("div");

      systemTime.textContent = String(message.sent_at || "");

      systemTime.style.cssText = `
      margin-top:4px;
      color:#a0aaa7;
      font-size:9px;
      font-weight:400;
      letter-spacing:0.2px;
      text-align:center;
      opacity:0.78;
    `;

      /* =========================================
       BUILD
    ========================================= */

      systemRow.appendChild(systemLine);

      if (message.sent_at) {
        systemRow.appendChild(systemTime);
      }

      chatMessages.appendChild(systemRow);

      chatMessages.scrollTop = chatMessages.scrollHeight;

      return;
    }

    /* =====================================================
     DUPLICATE MESSAGE PROTECTION
  ===================================================== */

    if (messageSysId) {
      const existingMessageRow = Array.from(
        chatMessages.querySelectorAll(".chat-message-row"),
      ).find(
        (existingRow) =>
          String(existingRow.dataset.messageSysId || "") === messageSysId,
      );

      if (existingMessageRow) {
        console.log("Duplicate chat message ignored:", messageSysId);

        return;
      }
    }

    /* =====================================================
     MESSAGE ROW
  ===================================================== */

    const messageRow = document.createElement("div");

    messageRow.className = "chat-message-row";

    messageRow._serviceCallMessage = message;

    if (messageSysId) {
      messageRow.dataset.messageSysId = messageSysId;
    }

    messageRow.style.cssText = `
    display:flex;
    flex-direction:column;
    align-items:${message.is_mine ? "flex-end" : "flex-start"};
    margin:8px 14px;
    position:relative;
  `;

    /*
     * Bubble + action buttons wrapper.
     */

    /* =====================================================
   MESSAGE SELECTION INDICATOR
===================================================== */

    const selectionIndicator = document.createElement("button");

    selectionIndicator.type = "button";

    selectionIndicator.className = "chat-message-selection-indicator";

    selectionIndicator.title = "Select message";

    selectionIndicator.style.cssText = `
  position:absolute;
  top:50%;

  width:20px;
  height:20px;

  padding:0;

  display:flex;
  align-items:center;
  justify-content:center;

  transform:translateY(-50%);

  border:1.5px solid #aab8b4;
  border-radius:50%;

  background:#ffffff;
  color:transparent;

  font-size:12px;
  font-weight:700;

  cursor:pointer;

  opacity:0;
  pointer-events:none;

  transition:
    opacity 0.15s ease,
    background 0.15s ease,
    border-color 0.15s ease,
    transform 0.15s ease;
`;

    /*
     * Keep the selection control on the
     * opposite side of the message bubble.
     *
     * My message       -> checkbox on left
     * Received message -> checkbox on right
     */
    if (message.is_mine) {
      selectionIndicator.style.left = "8px";
      selectionIndicator.style.right = "auto";
    } else {
      selectionIndicator.style.right = "8px";
      selectionIndicator.style.left = "auto";
    }

    selectionIndicator.addEventListener("click", (event) => {
      event.preventDefault();
      event.stopPropagation();

      if (!chatMessageSelectionMode) {
        return;
      }

      toggleChatMessageSelection(messageRow, message);
    });

    /*
     * Deleted-for-everyone tombstones are
     * not selectable.
     *
     * Do not add a selection control to the
     * DOM at all for deleted messages.
     */
    if (message.deleted !== true) {
      messageRow.appendChild(selectionIndicator);
    }

    const bubbleWrapper = document.createElement("div");

    bubbleWrapper.style.cssText = `
    display:flex;
    align-items:center;
    gap:6px;
    max-width:78%;
    position:relative;
  `;

    /*
     * Incoming:
     * [actions] [message]
     *
     * Outgoing:
     * [message] [actions]
     */
    bubbleWrapper.style.flexDirection = message.is_mine ? "row-reverse" : "row";

    const bubble = document.createElement("div");

    bubble.classList.add("chat-message-bubble");

    /* =====================================================
 FORWARDED MESSAGE LABEL
===================================================== */

    if (message.forwarded === true && message.deleted !== true) {
      const forwardedLabel = document.createElement("div");

      forwardedLabel.className = "chat-message-forwarded-label";

      forwardedLabel.textContent = "↪ Forwarded";

      forwardedLabel.style.cssText = `
    margin-bottom:4px;
    color:#66736f;
    font-size:10.5px;
    font-weight:600;
    line-height:1.2;
    white-space:nowrap;
    opacity:0.9;
  `;

      bubble.appendChild(forwardedLabel);
    }

    /* =====================================================
     REPLIED MESSAGE PREVIEW
  ===================================================== */

    const replyTo = message.reply_to || null;

    if (replyTo) {
      const replyPreview = document.createElement("div");

      replyPreview.className = "chat-message-reply-preview";

      replyPreview.title = "Go to original message";

      replyPreview.style.cssText = `
      margin-bottom:6px;
      padding:6px 8px;

      border-left:3px solid
        ${message.is_mine ? "#4f8f7d" : "#7b8d88"};

      border-radius:6px;

      background:
        ${
          message.is_mine ? "rgba(255,255,255,0.48)" : "rgba(255,255,255,0.72)"
        };

      overflow:hidden;
      cursor:pointer;
    `;

      const replySender = document.createElement("div");

      replySender.textContent = String(replyTo.sender_name || "Unknown User");

      replySender.style.cssText = `
      margin-bottom:2px;
      color:#397261;
      font-size:10.5px;
      font-weight:700;
      white-space:nowrap;
      overflow:hidden;
      text-overflow:ellipsis;
    `;

      const replyText = document.createElement("div");

      if (replyTo.deleted === true) {
        replyText.textContent = "Message deleted";

        replyText.style.fontStyle = "italic";
      } else {
        const originalText = String(replyTo.text || "");

        replyText.textContent =
          originalText.length > 140
            ? originalText.substring(0, 137) + "..."
            : originalText;
      }

      replyText.style.cssText += `
      color:#66736f;
      font-size:10.5px;
      line-height:1.35;
      white-space:nowrap;
      overflow:hidden;
      text-overflow:ellipsis;
    `;

      replyPreview.appendChild(replySender);

      replyPreview.appendChild(replyText);

      replyPreview.addEventListener("click", async (event) => {
        event.stopPropagation();

        const originalMessageSysId = String(replyTo.sys_id || "").trim();

        if (!originalMessageSysId) {
          return;
        }

        await jumpToChatMessage(originalMessageSysId);
      });

      bubble.appendChild(replyPreview);
    }

    const messageAttachments = Array.isArray(message.attachments)
      ? message.attachments.filter(
          (attachment) => attachment && attachment.sys_id,
        )
      : [];

    /*
     * Temporary backward compatibility:
     *
     * Older/cached responses may still contain
     * the previous singular "attachment" property.
     */
    if (
      messageAttachments.length === 0 &&
      message.attachment &&
      message.attachment.sys_id
    ) {
      messageAttachments.push(message.attachment);
    }

    const isAttachmentMessage =
      messageType === "attachment" &&
      message.deleted !== true &&
      messageAttachments.length > 0;

    /* -----------------------------------------------------
   DELETED MESSAGE
----------------------------------------------------- */

    if (message.deleted === true) {
      const messageText = document.createElement("div");

      messageText.textContent = "Message deleted";

      messageText.style.cssText = `
    white-space:pre-wrap;
    overflow-wrap:anywhere;
    font-style:italic;
    color:#7f8986;
  `;

      bubble.appendChild(messageText);
    } else if (isAttachmentMessage) {
      /* -----------------------------------------------------
     ATTACHMENT MESSAGE

     One message may contain:
     - optional text/caption
     - one or more attachments
  ----------------------------------------------------- */

      const attachmentMessageText = String(message.text || "").trim();

      /* =========================================
     OPTIONAL TEXT / CAPTION
  ========================================= */

      if (attachmentMessageText) {
        const messageText = document.createElement("div");

        messageText.className = "chat-attachment-caption";

        messageText.textContent = attachmentMessageText;

        messageText.style.cssText = `
    margin-bottom:8px;
    white-space:pre-wrap;
    overflow-wrap:anywhere;
  `;

        bubble.appendChild(messageText);
      }

      /* =========================================
     ATTACHMENTS CONTAINER
  ========================================= */

      const attachmentsContainer = document.createElement("div");

      attachmentsContainer.className = "chat-message-attachments";

      attachmentsContainer.style.cssText = `
    display:flex;
    flex-direction:column;
    gap:6px;
    width:100%;
  `;

      /* =========================================
     RENDER EVERY ATTACHMENT
  ========================================= */

      messageAttachments.forEach((attachment) => {
        const attachmentCard = document.createElement("div");

        attachmentCard.className = "chat-message-attachment-card";

        attachmentCard.style.cssText = `
        min-width:220px;
        max-width:320px;

        display:flex;
        align-items:center;
        gap:10px;

        padding:8px 10px;

        border:
          1px solid rgba(
            88,
            112,
            105,
            0.16
          );

        border-radius:10px;

        background:
          rgba(
            255,
            255,
            255,
            0.58
          );

        box-sizing:border-box;
      `;

        /* -------------------------
         FILE ICON
      ------------------------- */

        const attachmentIcon = document.createElement("div");

        attachmentIcon.textContent = "📄";

        attachmentIcon.style.cssText = `
        width:36px;
        height:36px;

        flex-shrink:0;

        display:flex;
        align-items:center;
        justify-content:center;

        border-radius:9px;

        background:
          rgba(
            79,
            143,
            125,
            0.12
          );

        font-size:20px;
      `;

        /* -------------------------
         FILE INFORMATION
      ------------------------- */

        const attachmentInfo = document.createElement("div");

        attachmentInfo.style.cssText = `
        min-width:0;
        flex:1;
      `;

        const attachmentName = document.createElement("div");

        attachmentName.textContent = String(
          attachment.file_name || "Attachment",
        );

        attachmentName.title = attachmentName.textContent;

        attachmentName.style.cssText = `
        overflow:hidden;
        text-overflow:ellipsis;
        white-space:nowrap;

        color:#24332f;

        font-size:12.5px;
        font-weight:600;
        line-height:1.35;
      `;

        const attachmentDetails = document.createElement("div");

        const mimeType = String(attachment.mime_type || "");

        let fileTypeLabel = "FILE";

        if (mimeType === "application/pdf") {
          fileTypeLabel = "PDF";
        } else if (mimeType.startsWith("image/")) {
          fileTypeLabel = "IMAGE";
        } else if (mimeType.startsWith("video/")) {
          fileTypeLabel = "VIDEO";
        } else if (mimeType.startsWith("audio/")) {
          fileTypeLabel = "AUDIO";
        }

        const attachmentSize = formatChatFileSize(
          Number(attachment.file_size || 0),
        );

        attachmentDetails.textContent =
          fileTypeLabel + (attachmentSize ? " · " + attachmentSize : "");

        attachmentDetails.style.cssText = `
        margin-top:2px;

        color:#71807c;

        font-size:10.5px;
        line-height:1.3;
      `;

        attachmentInfo.appendChild(attachmentName);

        attachmentInfo.appendChild(attachmentDetails);

        attachmentCard.appendChild(attachmentIcon);

        attachmentCard.appendChild(attachmentInfo);

        /* =========================================
         SECURE OPEN
      ========================================= */

        attachmentCard.style.cursor = "pointer";

        attachmentCard.title = "Open attachment";

        attachmentCard.addEventListener("click", async () => {
          console.log("ATTACHMENT CARD CLICKED", attachment);

          const attachmentSysId = String(attachment.sys_id || "").trim();

          const fileName = String(attachment.file_name || "attachment").trim();

          if (!attachmentSysId) {
            console.error("Attachment has no valid sys_id.");

            return;
          }

          /*
           * Prevent repeated open requests
           * while this particular file is
           * already being processed.
           */
          if (attachmentCard.dataset.opening === "true") {
            return;
          }

          attachmentCard.dataset.opening = "true";

          try {
            const result = await window.serviceCall.openChatAttachment(
              attachmentSysId,
              fileName,
            );

            console.log("Open attachment result:", result);

            if (!result || result.success !== true) {
              throw new Error(result?.message || "Unable to open attachment.");
            }
          } catch (error) {
            console.error("Unable to open attachment:", error);
          } finally {
            delete attachmentCard.dataset.opening;
          }
        });

        attachmentsContainer.appendChild(attachmentCard);
      });

      bubble.appendChild(attachmentsContainer);
    } else {
      /* -----------------------------------------------------
     NORMAL TEXT MESSAGE
  ----------------------------------------------------- */

      const messageText = document.createElement("div");

      messageText.textContent = message.text || "";

      messageText.style.cssText = `
    white-space:pre-wrap;
    overflow-wrap:anywhere;
  `;

      bubble.appendChild(messageText);
    }

    bubble.style.cssText = `
    max-width:100%;
    padding:9px 12px;
    border-radius:12px;
    font-size:13px;
    line-height:1.4;
    white-space:pre-wrap;
    overflow-wrap:anywhere;
    background:${message.is_mine ? "#dff3ec" : "#f1f3f2"};
    color:#1f2927;
  `;

    /* =====================================================
     REPLY BUTTON
  ===================================================== */

    const replyButton = document.createElement("button");

    replyButton.type = "button";
    replyButton.textContent = "↩";
    replyButton.title = "Reply";

    replyButton.style.cssText = `
    width:24px;
    height:24px;
    min-width:24px;
    border:none;
    border-radius:50%;
    background:#eef2f1;
    color:#60706c;
    font-size:14px;
    line-height:24px;
    padding:0;
    cursor:pointer;
    opacity:0;
    pointer-events:none;
    transition:
      opacity 0.15s ease,
      background 0.15s ease,
      transform 0.15s ease;
  `;

    /*
     * A persisted, non-deleted message is
     * required before it can become a
     * reply target.
     */
    if (!messageSysId || message.deleted === true) {
      replyButton.style.display = "none";
    }

    replyButton.addEventListener("mouseenter", () => {
      replyButton.style.background = "#dfe8e5";

      replyButton.style.transform = "scale(1.08)";
    });

    replyButton.addEventListener("mouseleave", () => {
      replyButton.style.background = "#eef2f1";

      replyButton.style.transform = "scale(1)";
    });

    replyButton.addEventListener("click", (event) => {
      event.stopPropagation();

      if (isActiveChatReadOnly()) {
        showChatMembershipError();
        return;
      }

      if (!messageSysId || message.deleted === true) {
        return;
      }

      chatReplyTarget = {
        sys_id: messageSysId,

        sender_name: String(
          message.sender_name ||
            message.sender_user_name ||
            (message.is_mine ? "You" : "Unknown User"),
        ),

        text: String(message.text || ""),
      };

      renderChatReplyPreview();

      if (chatMessageInput) {
        chatMessageInput.focus();
      }
    });

    /* =====================================================
     MORE MESSAGE ACTIONS
     Forward / Delete
  ===================================================== */

    const moreButton = document.createElement("button");

    moreButton.type = "button";
    moreButton.textContent = "⋯";
    moreButton.title = "More";

    moreButton.className = "chat-message-more-button";

    moreButton.style.cssText = `
    width:24px;
    height:24px;
    min-width:24px;
    border:none;
    border-radius:50%;
    background:#eef2f1;
    color:#60706c;
    font-size:16px;
    font-weight:600;
    line-height:20px;
    padding:0;
    cursor:pointer;
    opacity:0;
    transition:
      opacity 0.15s ease,
      background 0.15s ease,
      transform 0.15s ease;
  `;

    /*
     * System messages never reach here.
     *
     * Deleted-for-everyone tombstones
     * also do not need Forward/Delete.
     */
    if (!messageSysId || message.deleted === true) {
      moreButton.style.display = "none";
    }

    moreButton.addEventListener("mouseenter", () => {
      moreButton.style.background = "#dfe8e5";

      moreButton.style.transform = "scale(1.08)";
    });

    moreButton.addEventListener("mouseleave", () => {
      moreButton.style.background = "#eef2f1";

      moreButton.style.transform = "scale(1)";
    });

    /* =====================================================
     REACTION BUTTON
  ===================================================== */

    const reactionButton = document.createElement("button");

    reactionButton.type = "button";
    reactionButton.textContent = "+";
    reactionButton.title = "Add reaction";

    reactionButton.style.cssText = `
    width:24px;
    height:24px;
    min-width:24px;
    border:none;
    border-radius:50%;
    background:#eef2f1;
    color:#60706c;
    font-size:16px;
    line-height:24px;
    padding:0;
    cursor:pointer;
    opacity:0;
    transition:
      opacity 0.15s ease,
      background 0.15s ease,
      transform 0.15s ease;
  `;

    /*
     * Reactions are allowed only on
     * another user's non-deleted message.
     */
    if (!messageSysId || message.is_mine || message.deleted === true) {
      reactionButton.style.display = "none";
    }

    /* =====================================================
   MESSAGE ACTION HOVER

   IMPORTANT:
   Controls appear ONLY when the pointer
   is directly over the message bubble.
===================================================== */

    /* =====================================================
   MESSAGE ACTION HOVER

   Actions become interactive ONLY while
   the actual message bubble is hovered.
===================================================== */

    function showMessageActions() {
      if (!messageSysId || message.deleted === true) {
        return;
      }

      replyButton.style.opacity = "1";
      replyButton.style.pointerEvents = "auto";

      moreButton.style.opacity = "1";
      moreButton.style.pointerEvents = "auto";

      if (!message.is_mine) {
        reactionButton.style.opacity = "1";
        reactionButton.style.pointerEvents = "auto";
      }
    }

    function hideMessageActions() {
      replyButton.style.opacity = "0";
      replyButton.style.pointerEvents = "none";

      reactionButton.style.opacity = "0";
      reactionButton.style.pointerEvents = "none";

      moreButton.style.opacity = "0";
      moreButton.style.pointerEvents = "none";
    }

    /*
     * While multi-selection mode is active,
     * clicking a message bubble selects or
     * deselects that message.
     */
    /*
     * SELECTION MODE
     *
     * Once Delete / Forward selection mode
     * is active, the entire message row is
     * selectable — not only the bubble or
     * checkbox.
     */

    /*
     * SELECTION MODE HOVER
     *
     * While Delete / Forward selection mode
     * is active, the whole row behaves like
     * a selectable surface.
     */
    messageRow.addEventListener("mouseenter", () => {
      if (!chatMessageSelectionMode) {
        return;
      }

      messageRow.style.cursor = "pointer";

      /*
       * Only apply the light hover when
       * this message is NOT already selected.
       */
      if (!selectedChatMessages.has(messageSysId)) {
        messageRow.style.background = "rgba(79, 143, 125, 0.045)";

        messageRow.style.borderRadius = "10px";
      }
    });

    messageRow.addEventListener("mouseleave", () => {
      if (!chatMessageSelectionMode) {
        messageRow.style.cursor = "";
        return;
      }

      /*
       * Selected messages keep their
       * stronger selection background.
       */
      if (selectedChatMessages.has(messageSysId)) {
        return;
      }

      messageRow.style.background = "";
      messageRow.style.borderRadius = "";
    });

    messageRow.addEventListener("click", (event) => {
      if (!chatMessageSelectionMode) {
        return;
      }

      /*
       * Ignore clicks on controls.
       *
       * The checkbox has its own selection
       * handler.
       */
      if (event.target.closest(".chat-message-selection-indicator")) {
        return;
      }

      event.preventDefault();
      event.stopPropagation();

      toggleChatMessageSelection(messageRow, message);
    });

    bubble.addEventListener("mouseenter", () => {
      showMessageActions();
    });

    bubble.addEventListener("mouseleave", () => {
      setTimeout(() => {
        const pointerStillInActions = bubbleWrapper.matches(":hover");

        const menuOpen = !!bubbleWrapper.querySelector(
          ".chat-message-more-menu",
        );

        if (!pointerStillInActions && !menuOpen) {
          hideMessageActions();
        }
      }, 80);
    });

    bubbleWrapper.addEventListener("mouseleave", () => {
      if (bubbleWrapper.querySelector(".chat-message-more-menu")) {
        return;
      }

      hideMessageActions();
    });

    reactionButton.addEventListener("mouseenter", () => {
      reactionButton.style.background = "#dfe8e5";

      reactionButton.style.transform = "scale(1.08)";
    });

    reactionButton.addEventListener("mouseleave", () => {
      reactionButton.style.background = "#eef2f1";

      reactionButton.style.transform = "scale(1)";
    });

    /* =====================================================
     REACTION SUMMARY
  ===================================================== */

    const reactionSummary = document.createElement("div");

    reactionSummary.className = "chat-message-reactions";

    reactionSummary.style.cssText = `
    display:flex;
    flex-wrap:wrap;
    gap:4px;
    margin-top:4px;
    min-height:0;
  `;

    function renderReactions(reactions, myReaction) {
      reactionSummary.innerHTML = "";

      const reactionList = Array.isArray(reactions) ? reactions : [];

      reactionList.forEach((reactionItem) => {
        if (!reactionItem || !reactionItem.reaction) {
          return;
        }

        const chip = document.createElement("button");

        chip.type = "button";

        const reactionEmoji = String(reactionItem.reaction);

        const reactionCount = Number(reactionItem.count || 0);

        chip.textContent =
          reactionEmoji + (reactionCount > 0 ? " " + reactionCount : "");

        const isMine = reactionEmoji === myReaction;

        chip.style.cssText = `
          border:1px solid ${isMine ? "#67a995" : "#d9e0de"};
          background:${isMine ? "#e1f3ed" : "#f7f9f8"};
          border-radius:12px;
          padding:2px 7px;
          font-size:12px;
          line-height:18px;
          cursor:pointer;
          color:#31433e;
        `;

        chip.addEventListener("click", async () => {
          await applyReaction(reactionEmoji);
        });

        reactionSummary.appendChild(chip);
      });

      reactionSummary.dataset.ready = "true";
    }

    /*
     * Allow silent reaction synchronization
     * to update this exact message later.
     */
    if (messageSysId) {
      messageRow._serviceCallRenderReactions = (reactions, myReaction) => {
        renderReactions(reactions, myReaction);
      };
    }

    /* =====================================================
     APPLY REACTION
  ===================================================== */

    let reactionRequestRunning = false;

    async function applyReaction(reaction) {
      if (isActiveChatReadOnly()) {
        showChatMembershipError();
        return;
      }

      if (reactionRequestRunning || !messageSysId || message.deleted === true) {
        return;
      }

      reactionRequestRunning = true;

      try {
        const result = await window.serviceCall.setMessageReaction(
          messageSysId,
          reaction,
        );

        if (!result || result.success !== true) {
          console.warn("Unable to update reaction:", result);

          return;
        }

        renderReactions(result.reactions, result.my_reaction || "");
      } catch (error) {
        console.error("Unable to update message reaction:", error);
      } finally {
        reactionRequestRunning = false;
      }
    }

    /* =====================================================
     EMOJI PICKER
  ===================================================== */

    reactionButton.addEventListener("click", (event) => {
      event.stopPropagation();

      document.querySelectorAll(".chat-reaction-picker").forEach((picker) => {
        picker.remove();
      });

      /*
       * Close More menus as well.
       */
      document.querySelectorAll(".chat-message-more-menu").forEach((menu) => {
        menu.remove();
      });

      const picker = document.createElement("div");

      picker.className = "chat-reaction-picker";

      picker.style.cssText = `
        position:absolute;
        z-index:1000;
        width:300px;
        max-height:300px;
        overflow-y:auto;
        padding:10px;
        border:1px solid #dfe6e4;
        border-radius:14px;
        background:#ffffff;
        box-shadow:
          0 10px 30px
          rgba(0,0,0,0.14);
        display:flex;
        flex-direction:column;
        gap:10px;
      `;

      if (message.is_mine) {
        picker.style.right = "30px";
      } else {
        picker.style.left = "30px";
      }

      picker.style.bottom = "30px";

      const emojiSections = [
        {
          title: "Smileys",

          emojis: [
            "😀",
            "😃",
            "😄",
            "😁",
            "😆",
            "😅",
            "😂",
            "🤣",
            "😊",
            "🙂",
            "🙃",
            "😉",
            "😍",
            "🥰",
            "😘",
            "😎",
            "🤩",
            "🥳",
            "😋",
            "😜",
            "🤪",
            "🤗",
            "🤭",
            "🫢",
            "🤔",
            "🫡",
            "😐",
            "😑",
            "🙄",
            "😏",
            "😒",
            "😔",
            "😢",
            "😭",
            "😤",
            "😡",
            "🤬",
            "😱",
            "😨",
            "😴",
            "🤯",
            "🥶",
          ],
        },

        {
          title: "Gestures",

          emojis: [
            "👍",
            "👎",
            "👌",
            "🤌",
            "✌️",
            "🤞",
            "🤟",
            "🤘",
            "🤙",
            "👏",
            "🙌",
            "🫶",
            "🤝",
            "🙏",
            "💪",
            "👊",
            "✊",
            "🤜",
            "🤛",
            "👀",
          ],
        },

        {
          title: "Hearts",

          emojis: [
            "❤️",
            "🩷",
            "🧡",
            "💛",
            "💚",
            "💙",
            "🩵",
            "💜",
            "🤎",
            "🖤",
            "🤍",
            "💔",
            "❤️‍🔥",
            "❤️‍🩹",
            "💕",
            "💖",
            "💗",
            "💓",
            "💞",
            "💘",
            "💝",
          ],
        },

        {
          title: "More",

          emojis: [
            "💯",
            "🔥",
            "✨",
            "⭐",
            "🌟",
            "💫",
            "⚡",
            "🎉",
            "🎊",
            "🎈",
            "🎁",
            "🏆",
            "🥇",
            "🚀",
            "✅",
            "❌",
            "⚠️",
            "💡",
            "📌",
            "☕",
          ],
        },
      ];

      emojiSections.forEach((section) => {
        const sectionElement = document.createElement("div");

        const title = document.createElement("div");

        title.textContent = section.title;

        title.style.cssText = `
            margin-bottom:5px;
            font-size:11px;
            font-weight:600;
            color:#788681;
          `;

        const emojiGrid = document.createElement("div");

        emojiGrid.style.cssText = `
            display:grid;
            grid-template-columns:
              repeat(8, 1fr);
            gap:3px;
          `;

        section.emojis.forEach((emoji) => {
          const emojiButton = document.createElement("button");

          emojiButton.type = "button";

          emojiButton.textContent = emoji;

          emojiButton.style.cssText = `
                width:30px;
                height:30px;
                border:none;
                border-radius:7px;
                background:transparent;
                font-size:19px;
                cursor:pointer;
                padding:0;
              `;

          emojiButton.addEventListener("mouseenter", () => {
            emojiButton.style.background = "#eef4f2";
          });

          emojiButton.addEventListener("mouseleave", () => {
            emojiButton.style.background = "transparent";
          });

          emojiButton.addEventListener("click", async (pickerEvent) => {
            pickerEvent.stopPropagation();

            picker.remove();

            await applyReaction(emoji);
          });

          emojiGrid.appendChild(emojiButton);
        });

        sectionElement.appendChild(title);

        sectionElement.appendChild(emojiGrid);

        picker.appendChild(sectionElement);
      });

      bubbleWrapper.appendChild(picker);
    });

    /* =====================================================
     MORE ACTIONS MENU
  ===================================================== */

    moreButton.addEventListener("click", (event) => {
      event.stopPropagation();

      if (!messageSysId || message.deleted === true) {
        return;
      }

      /*
       * If this exact menu is already
       * open, clicking More again closes it.
       */
      const existingOwnMenu = bubbleWrapper.querySelector(
        ".chat-message-more-menu",
      );

      if (existingOwnMenu) {
        existingOwnMenu.remove();

        moreButton.style.opacity = "0";

        return;
      }

      /*
       * Close menus belonging to other
       * messages.
       */
      document.querySelectorAll(".chat-message-more-menu").forEach((menu) => {
        menu.remove();
      });

      /*
       * Close reaction picker.
       */
      document.querySelectorAll(".chat-reaction-picker").forEach((picker) => {
        picker.remove();
      });

      const menu = document.createElement("div");

      menu.className = "chat-message-more-menu";

      menu.style.cssText = `
        position:absolute;
        z-index:1100;
        min-width:140px;
        padding:5px;
        border:1px solid #dfe6e4;
        border-radius:10px;
        background:#ffffff;
        box-shadow:
          0 8px 24px
          rgba(0,0,0,0.14);
        display:flex;
        flex-direction:column;
        gap:2px;
        bottom:30px;
      `;

      if (message.is_mine) {
        menu.style.right = "30px";
      } else {
        menu.style.left = "30px";
      }

      /* -------------------------
   EDIT MESSAGE

   Available only for:
   - my own message
   - non-deleted message
   - normal text message
   - writable conversation
------------------------- */

      if (
        message.is_mine === true &&
        message.deleted !== true &&
        (String(message.type || "text") === "text" ||
          String(message.type || "") === "attachment") &&
        !isActiveChatReadOnly()
      ) {
        const editButton = document.createElement("button");

        editButton.type = "button";

        editButton.textContent = "Edit message";

        editButton.style.cssText = `
    border:none;
    border-radius:7px;
    background:transparent;
    padding:8px 10px;
    text-align:left;
    font-size:12px;
    color:#31433e;
    cursor:pointer;
  `;

        editButton.addEventListener("mouseenter", () => {
          editButton.style.background = "#eef4f2";
        });

        editButton.addEventListener("mouseleave", () => {
          editButton.style.background = "transparent";
        });

        editButton.addEventListener("click", (editEvent) => {
          editEvent.stopPropagation();

          menu.remove();

          if (
            !messageSysId ||
            message.deleted === true ||
            message.is_mine !== true ||
            isActiveChatReadOnly()
          ) {
            return;
          }

          /*
           * Editing and replying are mutually
           * exclusive composer modes.
           */
          chatReplyTarget = null;

          renderChatReplyPreview();

          /*
           * Remember exactly which persisted
           * message is being edited.
           */
          chatEditTarget = {
            sys_id: messageSysId,

            original_text: String(message.text || ""),

            type: String(message.type || "text").toLowerCase(),
          };
          /*
           * Put the existing message text
           * into the normal composer.
           */
          if (chatMessageInput) {
            chatMessageInput.value = chatEditTarget.original_text;

            chatMessageInput.disabled = false;

            /*
             * Reuse normal composer resizing
             * and Send-button state handling.
             */
            chatMessageInput.dispatchEvent(
              new Event("input", {
                bubbles: true,
              }),
            );

            chatMessageInput.focus();

            /*
             * Put cursor at the end.
             */
            const endPosition = chatMessageInput.value.length;

            chatMessageInput.setSelectionRange(endPosition, endPosition);
          }

          /*
           * For now the existing Send button
           * becomes the Save button.
           *
           * We are NOT sending/updating yet.
           */
          if (chatSendButton) {
            chatSendButton.textContent = "Save";

            const editingMessageType = String(
              chatEditTarget.type || "text",
            ).toLowerCase();

            const hasEditText = !!String(
              chatMessageInput ? chatMessageInput.value : "",
            ).trim();

            /*
             * Text message:
             *   Save requires text.
             *
             * Attachment message:
             *   Empty caption is valid.
             */
            chatSendButton.disabled =
              editingMessageType === "text" && !hasEditText;
          }
        });
        menu.appendChild(editButton);
      }

      /* -------------------------
         FORWARD
      ------------------------- */

      const forwardButton = document.createElement("button");

      forwardButton.type = "button";

      forwardButton.textContent = "Forward";

      forwardButton.style.cssText = `
        border:none;
        border-radius:7px;
        background:transparent;
        padding:8px 10px;
        text-align:left;
        font-size:12px;
        color:#31433e;
        cursor:pointer;
      `;

      forwardButton.addEventListener("mouseenter", () => {
        forwardButton.style.background = "#eef4f2";
      });

      forwardButton.addEventListener("mouseleave", () => {
        forwardButton.style.background = "transparent";
      });

      forwardButton.addEventListener("click", (forwardEvent) => {
        forwardEvent.preventDefault();
        forwardEvent.stopPropagation();

        /*
         * Do not allow system messages,
         * deleted tombstones, or messages
         * without a valid sys_id.
         */
        if (
          !messageSysId ||
          message.deleted === true ||
          String(message.type || "").toLowerCase() === "system"
        ) {
          menu.remove();
          hideMessageActions();
          return;
        }

        /*
         * Start Forward multi-selection mode.
         */
        chatMessageSelectionMode = "forward";

        /*
         * Start with a clean selection.
         */
        selectedChatMessages.clear();

        /*
         * Automatically select the message
         * where Forward was clicked.
         */
        selectChatMessage(messageRow, message);

        /*
         * Close the normal More menu.
         */
        menu.remove();

        hideMessageActions();
      });

      /* -------------------------
   DELETE
------------------------- */

      const deleteButton = document.createElement("button");

      deleteButton.type = "button";

      deleteButton.textContent = "Delete";

      deleteButton.style.cssText = `
  border:none;
  border-radius:7px;
  background:transparent;
  padding:8px 10px;
  text-align:left;
  font-size:12px;
  color:#b42318;
  cursor:pointer;
`;

      deleteButton.addEventListener("mouseenter", () => {
        deleteButton.style.background = "#fff1f0";
      });

      deleteButton.addEventListener("mouseleave", () => {
        deleteButton.style.background = "transparent";
      });

      deleteButton.addEventListener("click", (deleteEvent) => {
        deleteEvent.stopPropagation();

        /*
         * MULTI-SELECT DELETE MODE
         *
         * First Delete click enters selection mode
         * and selects this message.
         */
        if (!chatMessageSelectionMode) {
          chatMessageSelectionMode = "delete";

          selectedChatMessages.clear();

          selectChatMessage(messageRow, message);

          menu.remove();

          hideMessageActions();

          return;
        }

        /*
         * Historical group conversations
         * are read-only.
         */
        if (isActiveChatReadOnly()) {
          menu.remove();

          hideMessageActions();

          showChatMembershipError();

          return;
        }

        if (!messageSysId || message.deleted === true) {
          return;
        }

        /*
         * Replace the normal More menu
         * with the Delete choices.
         */
        menu.innerHTML = "";

        menu.style.minWidth = "175px";

        /* =========================================
       DELETE FOR EVERYONE
       Only shown for my own message.
    ========================================= */

        if (message.is_mine) {
          const deleteForEveryoneButton = document.createElement("button");

          deleteForEveryoneButton.type = "button";

          deleteForEveryoneButton.textContent = "Delete for everyone";

          deleteForEveryoneButton.style.cssText = `
        border:none;
        border-radius:7px;
        background:transparent;
        padding:8px 10px;
        text-align:left;
        font-size:12px;
        color:#b42318;
        cursor:pointer;
      `;

          deleteForEveryoneButton.addEventListener("mouseenter", () => {
            deleteForEveryoneButton.style.background = "#fff1f0";
          });

          deleteForEveryoneButton.addEventListener("mouseleave", () => {
            deleteForEveryoneButton.style.background = "transparent";
          });

          deleteForEveryoneButton.addEventListener(
            "click",
            async (optionEvent) => {
              optionEvent.stopPropagation();

              if (deleteForEveryoneButton.disabled) {
                return;
              }

              deleteForEveryoneButton.disabled = true;

              deleteForEveryoneButton.textContent = "Deleting...";

              try {
                const result = await window.serviceCall.deleteMessages(
                  String(activeChatConversation?.sys_id || "").trim(),
                  [messageSysId],
                  "everyone",
                );

                if (!result || result.success !== true) {
                  console.warn("Delete for everyone failed:", result);

                  deleteForEveryoneButton.disabled = false;

                  deleteForEveryoneButton.textContent = "Delete for everyone";

                  return;
                }

                /*
                 * Remove the old row.
                 *
                 * Then render the deleted
                 * tombstone locally.
                 */
                messageRow.remove();

                appendChatMessage({
                  ...message,
                  text: "",
                  deleted: true,
                  reactions: [],
                  my_reaction: "",
                });

                /*
                 * Prevent an old cached copy
                 * from being trusted after
                 * this mutation.
                 */
                if (activeChatConversation && activeChatConversation.sys_id) {
                  chatMessageCache.delete(
                    String(activeChatConversation.sys_id),
                  );
                }

                menu.remove();

                hideMessageActions();

                /*
                 * Refresh sidebar immediately after
                 * Delete for everyone.
                 */
                /*
                 * Silently refresh sidebar after
                 * Delete for everyone.
                 */
                try {
                  await syncChatConversationList();
                } catch (refreshError) {
                  console.error(
                    "Unable to silently refresh conversations after delete:",
                    refreshError,
                  );
                }
              } catch (error) {
                console.error("Unable to delete message for everyone:", error);

                deleteForEveryoneButton.disabled = false;

                deleteForEveryoneButton.textContent = "Delete for everyone";
              }
            },
          );

          menu.appendChild(deleteForEveryoneButton);
        }

        /* =========================================
       DELETE FOR ME
       Available for both sent and received
       messages.
    ========================================= */

        const deleteForMeButton = document.createElement("button");

        deleteForMeButton.type = "button";

        deleteForMeButton.textContent = "Delete for me";

        deleteForMeButton.style.cssText = `
      border:none;
      border-radius:7px;
      background:transparent;
      padding:8px 10px;
      text-align:left;
      font-size:12px;
      color:#b42318;
      cursor:pointer;
    `;

        deleteForMeButton.addEventListener("mouseenter", () => {
          deleteForMeButton.style.background = "#fff1f0";
        });

        deleteForMeButton.addEventListener("mouseleave", () => {
          deleteForMeButton.style.background = "transparent";
        });

        deleteForMeButton.addEventListener("click", async (optionEvent) => {
          optionEvent.stopPropagation();

          if (deleteForMeButton.disabled) {
            return;
          }

          deleteForMeButton.disabled = true;

          deleteForMeButton.textContent = "Deleting...";

          try {
            const result = await window.serviceCall.deleteMessages(
              String(activeChatConversation?.sys_id || "").trim(),
              [messageSysId],
              "me",
            );

            if (!result || result.success !== true) {
              console.warn("Delete for me failed:", result);

              deleteForMeButton.disabled = false;

              deleteForMeButton.textContent = "Delete for me";

              return;
            }

            /*
             * Delete-for-me means this
             * message disappears completely
             * from this user's view.
             */
            messageRow.remove();

            /*
             * Invalidate this conversation's
             * cached messages so reopening it
             * cannot resurrect the hidden
             * message.
             */
            if (activeChatConversation && activeChatConversation.sys_id) {
              chatMessageCache.delete(String(activeChatConversation.sys_id));
            }

            menu.remove();

            hideMessageActions();

            /*
             * Refresh sidebar immediately after
             * Delete for me.
             */
            /*
             * Silently refresh sidebar after
             * Delete for me.
             */
            try {
              await syncChatConversationList();
            } catch (refreshError) {
              console.error(
                "Unable to silently refresh conversations after delete for me:",
                refreshError,
              );
            }
          } catch (error) {
            console.error("Unable to delete message for me:", error);

            deleteForMeButton.disabled = false;

            deleteForMeButton.textContent = "Delete for me";
          }
        });

        menu.appendChild(deleteForMeButton);

        /* =========================================
       CANCEL
    ========================================= */

        const cancelDeleteButton = document.createElement("button");

        cancelDeleteButton.type = "button";

        cancelDeleteButton.textContent = "Cancel";

        cancelDeleteButton.style.cssText = `
      border:none;
      border-radius:7px;
      background:transparent;
      padding:8px 10px;
      text-align:left;
      font-size:12px;
      color:#60706c;
      cursor:pointer;
    `;

        cancelDeleteButton.addEventListener("mouseenter", () => {
          cancelDeleteButton.style.background = "#eef4f2";
        });

        cancelDeleteButton.addEventListener("mouseleave", () => {
          cancelDeleteButton.style.background = "transparent";
        });

        cancelDeleteButton.addEventListener("click", (optionEvent) => {
          optionEvent.stopPropagation();

          menu.remove();

          hideMessageActions();
        });

        menu.appendChild(cancelDeleteButton);
      });

      menu.appendChild(forwardButton);

      menu.appendChild(deleteButton);

      bubbleWrapper.appendChild(menu);

      moreButton.style.opacity = "1";
      moreButton.style.pointerEvents = "auto";

      /*
       * Close this menu when the user
       * clicks anywhere outside it.
       *
       * setTimeout prevents the same click
       * that opened the menu from immediately
       * closing it again.
       */
      setTimeout(() => {
        function closeMoreMenuOnOutsideClick(outsideEvent) {
          if (
            menu.contains(outsideEvent.target) ||
            moreButton.contains(outsideEvent.target)
          ) {
            return;
          }

          menu.remove();

          hideMessageActions();

          document.removeEventListener(
            "click",
            closeMoreMenuOnOutsideClick,
            true,
          );
        }

        document.addEventListener("click", closeMoreMenuOnOutsideClick, true);
      }, 0);
    });

    /* =====================================================
     BUILD MESSAGE ACTION AREA
  ===================================================== */

    bubbleWrapper.appendChild(bubble);

    bubbleWrapper.appendChild(replyButton);

    bubbleWrapper.appendChild(reactionButton);

    bubbleWrapper.appendChild(moreButton);

    /* =====================================================
     METADATA
  ===================================================== */

    const metadata = document.createElement("div");

    metadata.className = "chat-message-metadata";

    const sentAt = String(message.sent_at || "").trim();

    const editedAt = String(message.edited_at || "").trim();

    metadata.textContent = sentAt + (editedAt ? " · Edited" : "");

    metadata.style.cssText = `
  margin-top:3px;
  font-size:10px;
  color:#89918f;
`;

    messageRow.appendChild(bubbleWrapper);

    messageRow.appendChild(reactionSummary);

    messageRow.appendChild(metadata);

    chatMessages.appendChild(messageRow);

    /*
     * If a future /messages response
     * already contains reactions,
     * render them immediately.
     */
    renderReactions(message.reactions || [], message.my_reaction || "");

    /*
     * Keep newest message visible.
     */
    chatMessages.scrollTop = chatMessages.scrollHeight;
  }

  function updateChatMessageReactions(messageSysId, reactions, myReaction) {
    const targetSysId = String(messageSysId || "").trim();

    if (!targetSysId) {
      return;
    }

    const rows = document.querySelectorAll(".chat-message-row");

    let targetRow = null;

    rows.forEach((row) => {
      if (String(row.dataset.messageSysId || "") === targetSysId) {
        targetRow = row;
      }
    });

    if (
      !targetRow ||
      typeof targetRow._serviceCallRenderReactions !== "function"
    ) {
      return;
    }

    targetRow._serviceCallRenderReactions(
      Array.isArray(reactions) ? reactions : [],
      String(myReaction || ""),
    );
  }

  async function loadOlderChatMessages() {
    /*
     * Nothing to load unless a persisted
     * conversation is currently open.
     */
    if (
      !activeChatConversation ||
      !activeChatConversation.sys_id ||
      !chatMessages
    ) {
      return;
    }

    /*
     * ServiceNow already told us that
     * there is no older history.
     */
    if (!chatMessageHistoryHasMore) {
      return;
    }

    /*
     * Prevent multiple requests when several
     * scroll events fire near the top.
     */
    if (chatMessageHistoryLoading) {
      return;
    }

    if (!oldestChatMessageSysId) {
      return;
    }

    const conversationSysId = String(activeChatConversation.sys_id).trim();

    const beforeMessageSysId = String(oldestChatMessageSysId).trim();

    chatMessageHistoryLoading = true;

    /*
     * Remember the current document height.
     *
     * After older messages are inserted above,
     * we'll compensate for the added height so
     * the user's viewport stays on the same
     * message.
     */
    const previousScrollHeight = chatMessages.scrollHeight;

    const previousScrollTop = chatMessages.scrollTop;

    try {
      const result = await window.serviceCall.getMessages(
        conversationSysId,
        "",
        beforeMessageSysId,
        50,
      );

      /*
       * User may have switched conversations
       * while ServiceNow was responding.
       */
      if (
        !activeChatConversation ||
        String(activeChatConversation.sys_id) !== conversationSysId
      ) {
        return;
      }

      if (!result || result.success !== true) {
        console.warn("Unable to load older chat messages:", result);

        return;
      }

      const olderMessages = Array.isArray(result.messages)
        ? result.messages
        : [];

      /*
       * No older records returned.
       */
      if (olderMessages.length === 0) {
        chatMessageHistoryHasMore = false;

        return;
      }

      /*
       * appendChatMessage() currently appends
       * messages to the bottom and also scrolls
       * downward.
       *
       * Therefore we render the older page into
       * a temporary container first, then move
       * those generated rows above the existing
       * conversation.
       */
      const existingFirstChild = chatMessages.firstChild;

      olderMessages.forEach((message) => {
        appendChatMessage(message);
      });

      /*
       * appendChatMessage() placed the newly
       * generated rows at the bottom.
       *
       * Collect those rows by their authoritative
       * message sys_ids.
       */
      const olderRows = [];

      olderMessages.forEach((message) => {
        const messageSysId = String(message.sys_id || "").trim();

        if (!messageSysId) {
          return;
        }

        const row = Array.from(
          chatMessages.querySelectorAll(".chat-message-row"),
        ).find(
          (candidateRow) =>
            String(candidateRow.dataset.messageSysId || "") === messageSysId,
        );

        if (row) {
          olderRows.push(row);
        }
      });

      /*
       * Move the page above the messages that
       * were already visible.
       *
       * olderMessages already arrives from the
       * backend in oldest -> newest order.
       */
      olderRows.forEach((row) => {
        chatMessages.insertBefore(row, existingFirstChild);
      });

      /*
       * The first message returned is now the
       * oldest loaded message and therefore
       * becomes our next "before" checkpoint.
       */
      oldestChatMessageSysId = String(olderMessages[0].sys_id || "").trim();

      chatMessageHistoryHasMore = !!(
        result.pagination && result.pagination.has_more === true
      );

      /*
       * Restore the exact visual position.
       *
       * The content above the user became taller,
       * so move scrollTop by exactly that increase.
       */
      const newScrollHeight = chatMessages.scrollHeight;

      const addedHeight = newScrollHeight - previousScrollHeight;

      chatMessages.scrollTop = previousScrollTop + addedHeight;

      console.log(
        "Loaded older chat messages:",
        olderMessages.length,
        "hasMore:",
        chatMessageHistoryHasMore,
      );
    } catch (error) {
      console.error("Unable to load older chat history:", error);
    } finally {
      /*
       * Only unlock if we're still looking
       * at the same conversation.
       */
      if (
        activeChatConversation &&
        String(activeChatConversation.sys_id) === conversationSysId
      ) {
        chatMessageHistoryLoading = false;
      }
    }
  }

  async function jumpToChatMessage(messageSysId) {
    const targetMessageSysId = String(messageSysId || "").trim();

    if (
      !targetMessageSysId ||
      !chatMessages ||
      !activeChatConversation ||
      !activeChatConversation.sys_id
    ) {
      return;
    }

    /*
     * Remember which conversation started
     * this operation.
     *
     * If the user changes conversations while
     * older history is loading, stop immediately.
     */
    const conversationSysId = String(activeChatConversation.sys_id).trim();

    function findTargetRow() {
      return Array.from(
        chatMessages.querySelectorAll(".chat-message-row"),
      ).find(
        (row) => String(row.dataset.messageSysId || "") === targetMessageSysId,
      );
    }

    function highlightTargetRow(row) {
      if (!row) {
        return;
      }

      row.scrollIntoView({
        behavior: "smooth",
        block: "center",
      });

      const previousTransition = row.style.transition;

      const previousBackground = row.style.background;

      row.style.transition = "background 0.25s ease";

      row.style.background = "rgba(79, 143, 125, 0.18)";

      window.setTimeout(() => {
        row.style.background = previousBackground;

        window.setTimeout(() => {
          row.style.transition = previousTransition;
        }, 300);
      }, 900);
    }

    /*
     * The original message may already be
     * among the currently rendered messages.
     */
    let targetRow = findTargetRow();

    if (targetRow) {
      highlightTargetRow(targetRow);
      return;
    }

    /*
     * Otherwise progressively request older
     * pages using the pagination logic that
     * already works for normal Chat history.
     */
    while (chatMessageHistoryHasMore) {
      /*
       * Stop if the user switched conversations.
       */
      if (
        !activeChatConversation ||
        String(activeChatConversation.sys_id || "").trim() !== conversationSysId
      ) {
        return;
      }

      /*
       * loadOlderChatMessages() has its own
       * request lock.
       *
       * If another history request is currently
       * running, wait briefly for it to finish
       * instead of starting a competing request.
       */
      if (chatMessageHistoryLoading) {
        await new Promise((resolve) => {
          window.setTimeout(resolve, 100);
        });

        continue;
      }

      const previousOldestMessageSysId = String(
        oldestChatMessageSysId || "",
      ).trim();

      await loadOlderChatMessages();

      /*
       * The requested older page may now contain
       * our original reply target.
       */
      targetRow = findTargetRow();

      if (targetRow) {
        highlightTargetRow(targetRow);
        return;
      }

      /*
       * Safety guard:
       *
       * If pagination made no progress, do not
       * accidentally loop forever.
       */
      const currentOldestMessageSysId = String(
        oldestChatMessageSysId || "",
      ).trim();

      if (
        chatMessageHistoryHasMore &&
        previousOldestMessageSysId === currentOldestMessageSysId
      ) {
        console.warn(
          "Chat history pagination made no progress while locating reply target:",
          targetMessageSysId,
        );

        return;
      }
    }

    console.log(
      "Original reply message was not found in available history:",
      targetMessageSysId,
    );
  }

  /*
   * Load older history when the user
   * reaches the top of the conversation.
   */
  if (chatMessages) {
    chatMessages.addEventListener(
      "scroll",

      async () => {
        /*
         * =========================================
         * REACHED NEWEST MESSAGES
         * =========================================
         *
         * If new messages arrived while the user
         * was reading older content, manually
         * reaching the bottom means those messages
         * have now been viewed.
         */
        if (
          chatNewMessageCount > 0 &&
          isChatNearBottom() &&
          activeChatConversation &&
          activeChatConversation.sys_id
        ) {
          const conversationSysId = String(
            activeChatConversation.sys_id,
          ).trim();

          /*
           * Clear local indicator immediately.
           */
          chatUserWasNearBottom = true;

          chatNewMessageCount = 0;

          updateChatNewMessagesButton();

          try {
            /*
             * Persist the read state in ServiceNow.
             */
            const readResult =
              await window.serviceCall.markConversationRead(conversationSysId);

            /*
             * The user may have switched chats
             * while ServiceNow was responding.
             */
            if (
              !activeChatConversation ||
              String(activeChatConversation.sys_id || "") !== conversationSysId
            ) {
              return;
            }

            if (readResult && readResult.success === true) {
              activeChatConversation.unread_count = 0;

              if (readResult.last_read_at) {
                activeChatConversation.last_read_at = String(
                  readResult.last_read_at,
                );
              }

              /*
               * Reconcile the sidebar immediately.
               */
              await syncChatConversationList();
            } else {
              console.warn(
                "Unable to mark manually viewed chat as read:",
                readResult,
              );
            }
          } catch (error) {
            console.error(
              "Unable to mark manually viewed chat as read:",
              error,
            );
          }

          return;
        }

        /*
         * =========================================
         * LOAD OLDER HISTORY
         * =========================================
         */

        if (chatMessages.scrollTop > 40) {
          return;
        }

        if (!chatMessageHistoryHasMore) {
          return;
        }

        if (chatMessageHistoryLoading) {
          return;
        }

        if (!oldestChatMessageSysId) {
          return;
        }

        await loadOlderChatMessages();
      },
    );
  }

  async function checkForNewChatMessages() {
    /*
     * Nothing to synchronize unless
     * a conversation is currently open.
     */
    if (!activeChatConversation || !activeChatConversation.sys_id) {
      return;
    }

    /*
     * We need an existing checkpoint.
     *
     * Initial conversation loading is
     * responsible for establishing it.
     */
    if (!lastChatMessageSysId) {
      return;
    }

    /*
     * Remember which conversation this
     * request belongs to.
     *
     * The user may switch conversations
     * while the request is running.
     */
    const conversationSysId = activeChatConversation.sys_id;

    const checkpointSysId = lastChatMessageSysId;

    try {
      const result = await window.serviceCall.getMessages(
        conversationSysId,
        checkpointSysId,
        "",
        50,
        lastChatEditCheckpoint,
      );

      if (!result || result.success !== true) {
        console.warn("Silent chat synchronization failed:", result);

        return;
      }

      /*
       * Conversation changed while
       * ServiceNow was responding.
       *
       * Ignore this old response.
       */
      if (
        !activeChatConversation ||
        String(activeChatConversation.sys_id) !== String(conversationSysId) ||
        isActiveChatReadOnly()
      ) {
        return;
      }

      const newMessages = Array.isArray(result.messages) ? result.messages : [];

      const editedMessages = Array.isArray(result.edited_messages)
        ? result.edited_messages
        : [];

      /*
       * Apply edits to existing messages without
       * rebuilding or reloading the conversation.
       */
      editedMessages.forEach((editedMessage) => {
        if (!editedMessage || !editedMessage.sys_id) {
          return;
        }

        updateEditedChatMessageInView(
          editedMessage.sys_id,
          editedMessage.text || "",
          editedMessage.edited_at || "",
        );
      });

      /*
       * Always advance the server edit checkpoint,
       * even when this poll contained no edits.
       */
      const serverEditCheckpoint = String(result.edit_checkpoint || "").trim();

      if (serverEditCheckpoint) {
        lastChatEditCheckpoint = serverEditCheckpoint;
      }

      /*
       * No NEW messages.
       *
       * Edited messages above have already been
       * synchronized, so there is nothing else
       * to do in the new-message path.
       */
      if (newMessages.length === 0) {
        return;
      }

      /*
       * Append ONLY messages returned
       * after our checkpoint.
       */
      /*
       * =========================================
       * LIVE MESSAGE SCROLL BEHAVIOR
       * =========================================
       *
       * Capture the user's position BEFORE
       * adding the new messages.
       */
      const wasNearBottom = isChatNearBottom();

      const previousScrollTop = chatMessages ? chatMessages.scrollTop : 0;

      /*
       * Append the newly received messages.
       *
       * appendChatMessage() currently scrolls
       * to the bottom internally, so we'll
       * correct that immediately below when
       * the user was reading older content.
       */
      newMessages.forEach((newMessage) => {
        appendChatMessage(newMessage);
      });

      /*
       * User was already reading the newest
       * part of the conversation.
       *
       * Keep them at the bottom naturally.
       */
      if (wasNearBottom) {
        chatUserWasNearBottom = true;

        chatNewMessageCount = 0;

        updateChatNewMessagesButton();

        if (chatMessages) {
          chatMessages.scrollTop = chatMessages.scrollHeight;
        }
      } else {
        chatUserWasNearBottom = false;

        chatNewMessageCount += newMessages.length;

        if (chatMessages) {
          chatMessages.scrollTop = previousScrollTop;
        }

        updateChatNewMessagesButton();
      }

      /*
       * =========================================
       * MARK LIVE MESSAGES READ
       * =========================================
       *
       * An open conversation is no longer enough
       * to consider incoming messages "seen".
       *
       * If the user was already near the bottom
       * before these messages arrived, they are
       * effectively viewing the newest content.
       *
       * If the user was scrolled upward, leave
       * the messages unread in ServiceNow.
       */
      if (wasNearBottom) {
        try {
          const readResult =
            await window.serviceCall.markConversationRead(conversationSysId);

          /*
           * The user may have switched chats while
           * ServiceNow was processing the request.
           */
          if (
            readResult &&
            readResult.success &&
            activeChatConversation &&
            String(activeChatConversation.sys_id || "") ===
              String(conversationSysId)
          ) {
            activeChatConversation.unread_count = 0;

            if (readResult.last_read_at) {
              activeChatConversation.last_read_at = String(
                readResult.last_read_at,
              );
            }
          }
        } catch (readError) {
          console.error(
            "Unable to mark visible live messages as read:",
            readError,
          );
        }
      }

      /*
       * Move checkpoint to the newest
       * message we just received.
       */
      const newestMessage = newMessages[newMessages.length - 1];

      lastChatMessageSysId = String(
        newestMessage.sys_id || lastChatMessageSysId,
      ).trim();

      console.log(
        "Silent Chat sync:",
        newMessages.length,
        "new message(s). New checkpoint:",
        lastChatMessageSysId,
      );
    } catch (error) {
      /*
       * Silent synchronization should
       * never destroy/open/reload Chat.
       */
      console.error("Silent Chat synchronization error:", error);
    }
  }

  async function checkForChatReactionUpdates() {
    if (
      !activeChatConversation ||
      !activeChatConversation.sys_id ||
      !lastChatReactionCheckpoint
    ) {
      return;
    }

    const conversationSysId = String(activeChatConversation.sys_id);

    const checkpoint = String(lastChatReactionCheckpoint);

    try {
      const result = await window.serviceCall.getReactionUpdates(
        conversationSysId,
        checkpoint,
      );

      /*
       * Conversation may have changed
       * while the request was running.
       */
      if (
        !activeChatConversation ||
        String(activeChatConversation.sys_id) !== conversationSysId
      ) {
        return;
      }

      if (!result || !result.success) {
        return;
      }

      const updates = Array.isArray(result.updates) ? result.updates : [];

      updates.forEach((update) => {
        updateChatMessageReactions(
          update.message_sys_id,
          update.reactions || [],
          update.my_reaction || "",
        );
      });

      if (result.checkpoint) {
        lastChatReactionCheckpoint = String(result.checkpoint).trim();
      }
    } catch (error) {
      console.error("Unable to silently sync chat reactions:", error);
    }
  }

  function stopChatMessageSync() {
    if (chatMessageSyncTimer) {
      clearInterval(chatMessageSyncTimer);

      chatMessageSyncTimer = null;
    }
  }

  function startChatMessageSync() {
    /*
     * Never allow multiple polling
     * timers to run together.
     */
    stopChatMessageSync();

    chatMessageSyncTimer = setInterval(
      async () => {
        /*
         * Prevent overlapping requests
         * if ServiceNow responds slowly.
         */
        if (chatMessageSyncRunning) {
          return;
        }

        chatMessageSyncRunning = true;

        try {
          /*
           * =================================================
           * SIDEBAR CONVERSATION SYNC
           * =================================================
           *
           * This must run even when no conversation
           * is currently open.
           *
           * It allows:
           *
           * - unread counts
           * - latest previews
           * - conversation ordering
           *
           * to update automatically.
           */
          await syncChatConversationList();

          /*
           * =================================================
           * ACTIVE CONVERSATION SYNC
           * =================================================
           */

          if (
            activeChatConversation &&
            activeChatConversation.sys_id &&
            !isActiveChatReadOnly()
          ) {
            /*
             * New messages
             */
            await checkForNewChatMessages();

            /*
             * Reaction changes
             */
            await checkForChatReactionUpdates();
          }
        } catch (error) {
          console.error("Chat sync failed:", error);
        } finally {
          chatMessageSyncRunning = false;
        }
      },

      2000,
    );
  }

  async function syncChatConversationList() {
    try {
      const result = await window.serviceCall.getConversations();

      if (!result || result.success !== true) {
        console.warn("Silent conversation list sync failed:", result);

        return;
      }

      const conversations = Array.isArray(result.conversations)
        ? result.conversations
        : [];

      conversations.forEach((conversation) => {
        const conversationSysId = String(conversation.sys_id || "").trim();

        if (!conversationSysId) {
          return;
        }

        /*
         * Find the existing sidebar row.
         */
        const row = Array.from(
          chatConversationList.querySelectorAll(
            "button[data-conversation-sys-id]",
          ),
        ).find(
          (existingRow) =>
            String(existingRow.dataset.conversationSysId || "") ===
            conversationSysId,
        );

        /*
         * Conversation does not exist in
         * the current sidebar yet.
         *
         * For now reload the list so a
         * newly-created conversation can
         * appear.
         */
        if (!row) {
          return;
        }

        /* =========================================
                   UPDATE PREVIEW
                ========================================= */

        const information = row.children[1];

        if (information) {
          const preview = information.children[1];

          if (preview) {
            preview.textContent =
              conversation.last_message_preview || "No messages yet.";
          }
        }

        /* =========================================
                   UNREAD COUNT
                ========================================= */

        let unreadCount = Math.max(
          0,
          parseInt(conversation.unread_count, 10) || 0,
        );

        /*
         * If this exact conversation is
         * currently open, we do NOT want
         * to show an unread badge for it.
         *
         * The user is actively looking at
         * this conversation.
         */
        const isActiveConversation =
          activeChatConversation &&
          String(activeChatConversation.sys_id) === conversationSysId;

        /*
         * =========================================
         * ACTIVE CHAT READ STATE
         * =========================================
         *
         * An open conversation is NOT automatically
         * considered read anymore.
         *
         * If the user is reading older messages and
         * new messages have arrived below them,
         * preserve the authoritative unread count so
         * the sidebar can highlight the conversation.
         *
         * Only suppress the unread badge when the
         * active conversation is actually at the
         * newest messages.
         */
        if (
          isActiveConversation &&
          chatNewMessageCount <= 0 &&
          isChatNearBottom()
        ) {
          unreadCount = 0;
        }

        let badge = row.querySelector(".chat-unread-badge");

        /*
         * Visually distinguish conversations
         * containing unread messages.
         */
        if (unreadCount > 0) {
          row.classList.add("chat-conversation-unread");

          /*
           * Unread visual state.
           */
          row.style.background = "#eef8f4";

          const unreadInformation = row.children[1];

          if (unreadInformation) {
            const unreadName = unreadInformation.children[0];
            const unreadPreview = unreadInformation.children[1];

            if (unreadName) {
              unreadName.style.fontWeight = "700";
            }

            if (unreadPreview) {
              unreadPreview.style.fontWeight = "600";
              unreadPreview.style.color = "#46524f";
            }
          }
        } else {
          row.classList.remove("chat-conversation-unread");

          /*
           * Restore normal visual state.
           */
          row.style.background = "white";

          const normalInformation = row.children[1];

          if (normalInformation) {
            const normalName = normalInformation.children[0];
            const normalPreview = normalInformation.children[1];

            if (normalName) {
              normalName.style.fontWeight = "600";
            }

            if (normalPreview) {
              normalPreview.style.fontWeight = "400";
              normalPreview.style.color = "#78827f";
            }
          }
        }

        if (unreadCount > 0) {
          if (!badge) {
            badge = document.createElement("div");

            badge.className = "chat-unread-badge";

            badge.dataset.conversationSysId = conversationSysId;

            badge.style.cssText = `
                            min-width:20px;
                            height:20px;
                            padding:0 6px;
                            border-radius:10px;
                            display:flex;
                            align-items:center;
                            justify-content:center;
                            flex-shrink:0;
                            background:#17634f;
                            color:white;
                            font-size:10px;
                            font-weight:700;
                            line-height:1;
                        `;

            row.appendChild(badge);
          }

          badge.textContent = unreadCount > 99 ? "99+" : String(unreadCount);
        } else if (badge) {
          /*
           * REMOVE BADGE
           */
          badge.remove();
        }

        /*
         * Keep our existing conversation
         * object synchronized when this
         * is the active conversation.
         */
        if (isActiveConversation) {
          activeChatConversation.unread_count = 0;
        }
      });

      /*
       * Keep sidebar order synchronized with
       * ServiceNow's authoritative conversation order.
       *
       * The /conversations response is ordered by
       * latest activity, so newer conversations
       * naturally rise to the top.
       */
      if (chatConversationList) {
        conversations.forEach((conversation) => {
          const conversationSysId = String(conversation.sys_id || "").trim();

          if (!conversationSysId) {
            return;
          }

          const row = chatConversationList.querySelector(
            `button[data-conversation-sys-id="${conversationSysId}"]`,
          );

          if (row) {
            chatConversationList.appendChild(row);
          }
        });
      }
    } catch (error) {
      console.error("Silent conversation list synchronization error:", error);
    }
  }

  window.testChatSync = checkForNewChatMessages;

  startChatMessageSync();

  function isActiveChatReadOnly() {
    return !!(
      activeChatConversation &&
      activeChatConversation.type === "group" &&
      activeChatConversation.membership_active === false
    );
  }

  function showChatMembershipError() {
    if (!chatMembershipMessage) return;

    chatMembershipMessage.textContent =
      "You are no longer a member of this group and can't send messages.";

    chatMembershipMessage.style.display = "block";
  }

  function renderChatReplyPreview() {
    if (!chatMessageInput) {
      return;
    }

    /*
     * Remove an existing preview first.
     */
    const existingPreview = document.getElementById("chatReplyPreview");

    if (existingPreview) {
      existingPreview.remove();
    }

    if (!chatReplyTarget) {
      return;
    }

    const preview = document.createElement("div");

    preview.id = "chatReplyPreview";

    preview.style.cssText = `
    width:100%;
    box-sizing:border-box;
    margin:0 0 6px 0;
    padding:8px 10px;

    display:flex;
    align-items:center;
    justify-content:space-between;
    gap:10px;

    border-left:3px solid #4f8f7d;
    border-radius:7px;

    background:#f0f6f4;
  `;

    const content = document.createElement("div");

    content.style.cssText = `
    min-width:0;
    flex:1;
  `;

    const sender = document.createElement("div");

    sender.textContent = chatReplyTarget.sender_name || "Message";

    sender.style.cssText = `
    margin-bottom:2px;
    color:#397261;
    font-size:11px;
    font-weight:700;
  `;

    const text = document.createElement("div");

    const originalText = String(chatReplyTarget.text || "");

    text.textContent =
      originalText.length > 120
        ? originalText.substring(0, 117) + "..."
        : originalText;

    text.style.cssText = `
    color:#66736f;
    font-size:11px;

    white-space:nowrap;
    overflow:hidden;
    text-overflow:ellipsis;
  `;

    const cancelButton = document.createElement("button");

    cancelButton.type = "button";

    cancelButton.textContent = "×";

    cancelButton.title = "Cancel reply";

    cancelButton.style.cssText = `
    width:24px;
    height:24px;
    min-width:24px;

    border:none;
    border-radius:50%;

    background:transparent;
    color:#65736f;

    font-size:18px;
    line-height:22px;

    cursor:pointer;
  `;

    cancelButton.addEventListener("click", (event) => {
      event.preventDefault();
      event.stopPropagation();

      chatReplyTarget = null;

      renderChatReplyPreview();

      if (chatMessageInput) {
        chatMessageInput.focus();
      }
    });

    content.appendChild(sender);
    content.appendChild(text);

    preview.appendChild(content);
    preview.appendChild(cancelButton);

    /*
     * Your textarea is already inside its own
     * flex-column wrapper with the membership
     * warning above it.
     *
     * Therefore insert the reply preview directly
     * before the textarea.
     */
    chatMessageInput.parentElement.insertBefore(preview, chatMessageInput);
  }

  async function sendActiveChatMessage() {
    if (chatMessageSending) {
      return;
    }

    /*
     * Never send while one or more
     * attachments are still uploading.
     *
     * This protects Enter/key events too,
     * not just the disabled Send button.
     */
    if (chatAttachmentUploadsInProgress > 0) {
      console.warn("Waiting for attachments to finish uploading.");

      return;
    }

    if (isActiveChatReadOnly()) {
      showChatMembershipError();
      return;
    }

    /* =========================================
       VALIDATE ACTIVE CONVERSATION
    ========================================= */

    if (!activeChatConversation) {
      console.warn("No active conversation.");

      return;
    }

    const conversationType = String(activeChatConversation.type || "").trim();

    const isDirectConversation = conversationType === "direct";

    const isGroupConversation = conversationType === "group";

    if (!isDirectConversation && !isGroupConversation) {
      console.warn("Unsupported conversation type:", conversationType);

      return;
    }

    /*
     * Direct conversations require the
     * other ServiceCall user's sys_id.
     */
    if (isDirectConversation && !activeChatConversation.other_user_sys_id) {
      console.warn("Direct conversation has no recipient.");

      return;
    }

    /*
     * Group conversations already exist.
     *
     * Their conversation sys_id becomes
     * the message target.
     */
    if (isGroupConversation && !activeChatConversation.sys_id) {
      console.warn("Group conversation has no sys_id.");

      return;
    }

    if (!chatMessageInput) {
      return;
    }

    /* =========================================
       MESSAGE
    ========================================= */

    const message = String(chatMessageInput.value || "").trim();

    /*
     * A send is valid when we have:
     *
     * 1. text
     * OR
     * 2. a prepared attachment
     */
    const hasText = !!message;

    /*
     * Capture ALL ready attachments before
     * asynchronous sending begins.
     *
     * This snapshot is important because the
     * live composer state may change later.
     */
    const sendingAttachments = Array.isArray(pendingChatAttachments)
      ? pendingChatAttachments
          .filter((attachment) => attachment && attachment.sysId)
          .map((attachment) => ({
            sysId: String(attachment.sysId || "").trim(),

            fileName: String(attachment.fileName || ""),

            mimeType: String(attachment.mimeType || ""),

            fileSize: Number(attachment.fileSize || 0),
          }))
      : [];

    const hasAttachments = pendingChatAttachments.some(
      (attachment) =>
        attachment &&
        attachment.status !== "failed" &&
        !!String(attachment.sysId || "").trim(),
    );

    if (!hasText && !hasAttachments) {
      return;
    }

    if (hasText && message.length > 10000) {
      console.error("Message exceeds 10000 characters.");

      return;
    }

    /*
     * Capture the conversation being sent
     * to BEFORE the asynchronous request.
     *
     * If the user changes conversations
     * while ServiceNow is processing the
     * message, we must not accidentally
     * modify the new conversation UI.
     */
    const sendingConversationSysId = String(
      activeChatConversation.sys_id || "",
    ).trim();

    const recipientSysId = isDirectConversation
      ? String(activeChatConversation.other_user_sys_id || "").trim()
      : "";

    const conversationSysId = isGroupConversation
      ? sendingConversationSysId
      : "";

    /*
     * Capture the identity of a temporary direct
     * chat before the asynchronous send begins.
     *
     * A temporary chat has no conversation sys_id
     * yet, so its recipient identifies it.
     */
    const sendingTemporaryRecipientSysId =
      isDirectConversation && activeChatConversation.temporary === true
        ? recipientSysId
        : "";

    /* =========================================
       LOCK SEND
    ========================================= */

    chatMessageSending = true;

    chatMessageInput.disabled = true;

    if (chatSendButton) {
      chatSendButton.disabled = true;

      chatSendButton.textContent = "Sending...";
    }

    try {
      /* =========================================
           SEND

           DIRECT:
           recipientSysId + empty conversation

           GROUP:
           empty recipient + conversationSysId
        ========================================= */

      /*
       * Capture the reply target for this send.
       *
       * Empty string = ordinary message.
       */
      const replyToMessageSysId =
        chatReplyTarget && chatReplyTarget.sys_id
          ? String(chatReplyTarget.sys_id).trim()
          : "";

      /*
       * =========================================
       * SEND TEXT / ATTACHMENT
       * =========================================
       */

      let result = null;

      /*
       * =========================================
       * SEND CONTENT
       * =========================================
       *
       * ATTACHMENTS:
       * text + all files are sent through ONE
       * server operation and become ONE message.
       *
       * TEXT ONLY:
       * preserve our existing proven sendMessage()
       * flow, including temporary direct-chat
       * creation/promotion.
       */
      if (hasAttachments) {
        /*
         * Attachments currently require an
         * existing real conversation.
         *
         * Temporary direct chats are already
         * blocked during attachment selection.
         */
        const realConversationSysId = String(
          activeChatConversation?.sys_id || "",
        ).trim();

        if (
          !realConversationSysId ||
          !/^[0-9a-f]{32}$/i.test(realConversationSysId)
        ) {
          throw new Error("Attachments require an existing conversation.");
        }

        const attachmentSysIds = sendingAttachments.map(
          (attachment) => attachment.sysId,
        );

        result = await window.serviceCall.sendChatContent(
          realConversationSysId,
          message,
          attachmentSysIds,
          replyToMessageSysId,
        );

        console.log("ServiceCall chat content sent:", result);

        if (!result || result.success !== true) {
          throw new Error(result?.message || "Unable to send chat content.");
        }
      } else {
        /*
         * Existing text-only flow remains
         * unchanged.
         */
        result = await window.serviceCall.sendMessage(
          recipientSysId,
          conversationSysId,
          message,
          replyToMessageSysId,
        );

        console.log("ServiceCall message sent:", result);

        if (!result || result.success !== true) {
          throw new Error(result?.message || "Unable to send message.");
        }
      }
      /*
       * =========================================
       * PROMOTE TEMPORARY DIRECT CHAT
       * =========================================
       *
       * Opening a searched user creates only a
       * local temporary conversation.
       *
       * After the first successful message,
       * ServiceNow has now created/reused the
       * real direct conversation.
       */
      if (
        isDirectConversation &&
        activeChatConversation &&
        activeChatConversation.temporary === true &&
        result &&
        result.conversation &&
        result.conversation.sys_id
      ) {
        const realConversationSysId = String(result.conversation.sys_id).trim();

        /*
         * Only promote the currently-open temporary
         * chat if it is still the same recipient
         * that this message was sent to.
         */
        const isSameTemporaryRecipient =
          String(activeChatConversation.other_user_sys_id || "").trim() ===
          recipientSysId;

        if (realConversationSysId && isSameTemporaryRecipient) {
          activeChatConversation.sys_id = realConversationSysId;

          activeChatConversation.temporary = false;

          /*
           * Promote the existing temporary sidebar
           * row instead of adding a duplicate row.
           */
          const temporaryRow = chatConversationList
            ? Array.from(
                chatConversationList.querySelectorAll(
                  'button[data-temporary-chat="true"]',
                ),
              ).find(
                (row) => String(row.dataset.userSysId || "") === recipientSysId,
              )
            : null;

          if (temporaryRow) {
            /*
             * Before promoting the temporary row,
             * remove any OTHER sidebar row that
             * already represents the same real
             * conversation.
             *
             * This protects against the authoritative
             * sidebar sync racing with temporary-chat
             * promotion.
             */
            if (chatConversationList) {
              Array.from(
                chatConversationList.querySelectorAll(
                  "button[data-conversation-sys-id]",
                ),
              ).forEach((row) => {
                if (row === temporaryRow) {
                  return;
                }

                const rowConversationSysId = String(
                  row.dataset.conversationSysId || "",
                ).trim();

                if (rowConversationSysId === realConversationSysId) {
                  row.remove();
                }
              });
            }

            /*
             * Promote this existing temporary row
             * into the authoritative conversation row.
             */
            temporaryRow.dataset.temporaryChat = "false";

            temporaryRow.dataset.conversationSysId = realConversationSysId;

            delete temporaryRow.dataset.userSysId;
          }
        }
      }

      /*
       * The user may have changed to another
       * conversation while the message was
       * being stored.
       *
       * The message was successfully sent,
       * but we must NOT append it into the
       * wrong conversation.
       */
      const stillViewingSentConversation =
        !!activeChatConversation &&
        /*
         * Existing direct/group conversation:
         * identify it using conversation sys_id.
         */
        ((sendingConversationSysId &&
          String(activeChatConversation.sys_id || "").trim() ===
            sendingConversationSysId) ||
          /*
           * Temporary direct conversation:
           * there was no conversation sys_id when
           * sending started, so identify it using
           * the recipient's sys_id.
           */
          (!sendingConversationSysId &&
            sendingTemporaryRecipientSysId &&
            activeChatConversation.type === "direct" &&
            String(activeChatConversation.other_user_sys_id || "").trim() ===
              sendingTemporaryRecipientSysId));
      /*
       * Only clear the composer when the user
       * is still viewing the conversation from
       * which this message was sent.
       */
      if (stillViewingSentConversation) {
        /*
         * Clear text only when text was sent.
         */
        if (hasText) {
          chatMessageInput.value = "";

          resizeChatMessageInput();

          /*
           * The reply was successfully stored.
           * Leave reply mode only after server success.
           */
          chatReplyTarget = null;

          renderChatReplyPreview();
        }

        /*
         * Attachment was accepted by ServiceNow.
         *
         * Only now remove it from the composer.
         */
        if (hasAttachments && result && result.success === true) {
          /*
           * Remove only the attachments that were
           * part of THIS successful send.
           *
           * This is safer than blindly clearing
           * future composer state.
           */
          const sentAttachmentIds = new Set(
            sendingAttachments.map((attachment) => attachment.sysId),
          );

          pendingChatAttachments = pendingChatAttachments.filter(
            (attachment) =>
              !sentAttachmentIds.has(String(attachment.sysId || "").trim()),
          );

          renderPendingChatAttachment();

          if (chatAttachmentInput && pendingChatAttachments.length === 0) {
            chatAttachmentInput.value = "";
          }
        }
      }

      /* =========================================
   AUTHORITATIVE SAVED TEXT MESSAGE
========================================= */

      /*
       * Only locally render a text message when
       * a text message was actually sent.
       *
       * Attachment rendering will come through
       * our attachment-aware message flow later.
       */
      if (
        hasText &&
        !hasAttachments &&
        result &&
        result.message &&
        stillViewingSentConversation
      ) {
        const savedMessage = result.message || {};

        const savedReplyTo = savedMessage.reply_to || null;

        appendChatMessage({
          sys_id: String(savedMessage.sys_id || ""),

          sender_sys_id: String(savedMessage.sender_sys_id || ""),

          sender_name: String(savedMessage.sender_name || ""),

          type: String(savedMessage.type || "text"),

          text: String(savedMessage.text || message),

          sent_at: String(savedMessage.sent_at || ""),

          is_mine: true,

          reply_to: savedReplyTo,
        });

        /*
         * Prevent polling from fetching the
         * locally-rendered TEXT message again.
         */
        if (savedMessage.sys_id) {
          lastChatMessageSysId = String(savedMessage.sys_id).trim();
        }

        /* =====================================
               UPDATE ACTIVE CONVERSATION MEMORY
            ===================================== */

        activeChatConversation.last_message_at =
          result && result.conversation && result.conversation.last_message_at
            ? String(result.conversation.last_message_at)
            : result && result.message && result.message.sent_at
              ? String(result.message.sent_at)
              : activeChatConversation.last_message_at || "";

        /* =====================================
               UPDATE SIDEBAR PREVIEW
            ===================================== */

        const conversationRows = chatConversationList
          ? chatConversationList.querySelectorAll("button")
          : [];

        conversationRows.forEach((row) => {
          const rowName = row.querySelector("div > div:first-child");

          if (
            !rowName ||
            rowName.textContent !==
              (activeChatConversation.display_name ||
                activeChatConversation.title ||
                "Conversation")
          ) {
            return;
          }

          const information = row.children[1];

          if (!information) {
            return;
          }

          const preview = information.children[1];

          if (preview) {
            preview.textContent = sentPreviewText;
          }

          /*
           * This conversation now has the newest activity.
           * Move its existing sidebar row to the top.
           */
          if (
            chatConversationList &&
            row.parentElement === chatConversationList &&
            chatConversationList.firstElementChild !== row
          ) {
            /*
             * Move the active conversation
             * to the top of the sidebar.
             */
            chatConversationList.prepend(row);

            /*
             * Bring the refreshed top of the
             * conversation list into view.
             */
            chatConversationList.scrollTo({
              top: 0,
              behavior: "smooth",
            });
          }
        });
      }
    } catch (error) {
      console.error("Unable to send ServiceCall message:", error);

      /*
       * Keep the text so the user can retry.
       */
      if (chatMessageInput) {
        chatMessageInput.disabled = false;
      }

      if (chatSendButton) {
        chatSendButton.disabled = false;
      }
    } finally {
      chatMessageSending = false;

      if (chatSendButton) {
        chatSendButton.textContent = "Send";
      }
      /*
       * Only restore/focus the composer when
       * the user is still in the conversation
       * where this send began.
       *
       * Existing conversation:
       * compare conversation sys_id.
       *
       * Temporary direct conversation:
       * the first successful send promotes it
       * from an empty sys_id to the real sys_id,
       * so identify it using the recipient.
       */
      const stillInSendingConversation =
        !!activeChatConversation &&
        ((sendingConversationSysId &&
          String(activeChatConversation.sys_id || "").trim() ===
            sendingConversationSysId) ||
          (!sendingConversationSysId &&
            sendingTemporaryRecipientSysId &&
            activeChatConversation.type === "direct" &&
            String(activeChatConversation.other_user_sys_id || "").trim() ===
              sendingTemporaryRecipientSysId));

      if (chatMessageInput && stillInSendingConversation) {
        chatMessageInput.disabled = false;

        if (chatSendButton) {
          const hasCurrentText = !!String(chatMessageInput.value || "").trim();

          const hasCurrentAttachments =
            Array.isArray(pendingChatAttachments) &&
            pendingChatAttachments.some(
              (attachment) =>
                attachment &&
                attachment.status !== "failed" &&
                !!String(attachment.sysId || "").trim(),
            );
          const uploadsStillRunning = chatAttachmentUploadsInProgress > 0;

          chatSendButton.disabled =
            uploadsStillRunning || (!hasCurrentText && !hasCurrentAttachments);
        }

        chatMessageInput.focus();
      }
    }
  }

  function updateEditedChatMessageInView(messageSysId, newText, editedAt) {
    const targetSysId = String(messageSysId || "").trim();

    if (!targetSysId || !chatMessages) {
      return;
    }

    const row = Array.from(
      chatMessages.querySelectorAll(".chat-message-row"),
    ).find(
      (candidateRow) =>
        String(candidateRow.dataset.messageSysId || "") === targetSysId,
    );

    if (!row) {
      return;
    }

    /*
     * Find this message in the current cache first.
     * This also gives us the authoritative message
     * object used by the renderer.
     */
    const conversationSysId = String(
      activeChatConversation ? activeChatConversation.sys_id || "" : "",
    ).trim();

    const cachedConversation = conversationSysId
      ? chatMessageCache.get(conversationSysId)
      : null;

    let cachedMessage = null;

    if (cachedConversation && Array.isArray(cachedConversation.messages)) {
      cachedMessage =
        cachedConversation.messages.find(
          (message) => String(message.sys_id || "").trim() === targetSysId,
        ) || null;
    }

    /*
     * Update cache.
     */
    if (cachedMessage) {
      cachedMessage.text = String(newText || "");

      cachedMessage.edited_at = String(editedAt || "");
    }

    /*
     * Keep the exact message object used by
     * this rendered row synchronized.
     *
     * The Edit button closes over this object,
     * so future edits must see the latest text.
     */
    if (row._serviceCallMessage) {
      row._serviceCallMessage.text = String(newText || "");

      row._serviceCallMessage.edited_at = String(editedAt || "");
    }

    /*
     * Find the actual bubble.
     *
     * The bubble is inside the first wrapper
     * in the message row.
     */
    const bubble = row.querySelector(".chat-message-bubble");

    console.log("EDIT ATTACHMENT DOM DEBUG:", {
      messageSysId: targetSysId,
      messageType: cachedMessage?.type,
      bubbleFound: !!bubble,
      captionFound: !!bubble?.querySelector(".chat-attachment-caption"),
      attachmentsFound: !!bubble?.querySelector(".chat-message-attachments"),
      bubbleChildren: bubble
        ? Array.from(bubble.children).map(
            (child) => child.className || child.tagName,
          )
        : [],
    });

    if (bubble) {
      const hasAttachmentContainer = !!bubble.querySelector(
        ".chat-message-attachments",
      );

      const messageType = String(
        cachedMessage?.type ||
          (hasAttachmentContainer ? "attachment" : "") ||
          (chatEditTarget &&
          String(chatEditTarget.sys_id || "").trim() === targetSysId
            ? chatEditTarget.type
            : "") ||
          "text",
      ).toLowerCase();

      /*
       * Normal text message:
       * replacing the bubble text is safe.
       */
      if (messageType === "text") {
        bubble.textContent = String(newText || "");
      } else if (messageType === "attachment") {
        const editedCaption = String(newText || "").trim();

        let caption = bubble.querySelector(".chat-attachment-caption");

        const attachmentsContainer = bubble.querySelector(
          ".chat-message-attachments",
        );

        /*
         * Existing caption:
         * update it.
         */
        if (caption && editedCaption) {
          caption.textContent = editedCaption;
        } else if (caption && !editedCaption) {
          /*
           * Caption was completely removed:
           * remove only the caption element.
           *
           * Attachment cards remain untouched.
           */
          caption.remove();
        } else if (!caption && editedCaption) {
          /*
           * Files-only message now received a caption:
           * create the caption and insert it directly
           * before the existing attachment cards.
           */
          caption = document.createElement("div");

          caption.className = "chat-attachment-caption";

          caption.textContent = editedCaption;

          caption.style.cssText = `
      margin-bottom:8px;
      white-space:pre-wrap;
      overflow-wrap:anywhere;
    `;

          if (attachmentsContainer) {
            bubble.insertBefore(caption, attachmentsContainer);
          } else {
            bubble.appendChild(caption);
          }
        }
      }
    }

    /*
     * Metadata is the final element of
     * the message row in the current renderer.
     */
    const metadata = row.querySelector(".chat-message-metadata");
    if (metadata) {
      const sentAt = cachedMessage
        ? String(cachedMessage.sent_at || "").trim()
        : "";

      const editTime = String(editedAt || "").trim();

      metadata.textContent =
        (editTime || sentAt) + (editTime ? " · Edited" : "");
    }
  }

  /* =======================================================
   SERVICECALL CHAT - SAVE EDITED MESSAGE
======================================================= */

  async function saveEditedChatMessage() {
    if (!chatEditTarget || !chatMessageInput || !activeChatConversation) {
      return;
    }

    if (chatMessageSending) {
      return;
    }

    const conversationSysId = String(
      activeChatConversation.sys_id || "",
    ).trim();

    const messageSysId = String(chatEditTarget.sys_id || "").trim();

    const message = String(chatMessageInput.value || "").trim();

    const editingMessageType = String(
      chatEditTarget.type || "text",
    ).toLowerCase();

    if (!conversationSysId || !messageSysId) {
      return;
    }

    /*
     * Normal text messages cannot be saved empty.
     *
     * Attachment messages may have an empty caption
     * because the attached files remain in the message.
     */
    if (editingMessageType === "text" && !message) {
      return;
    }

    /*
     * Nothing changed.
     */
    if (message === String(chatEditTarget.original_text || "").trim()) {
      chatEditTarget = null;

      chatMessageInput.value = "";

      resizeChatMessageInput();

      if (chatSendButton) {
        chatSendButton.textContent = "Send";
        chatSendButton.disabled = true;
      }

      chatMessageInput.focus();

      return;
    }

    chatMessageSending = true;

    if (chatMessageInput) {
      chatMessageInput.disabled = true;
    }

    if (chatSendButton) {
      chatSendButton.disabled = true;
      chatSendButton.textContent = "Saving...";
    }

    try {
      const result = await window.serviceCall.editMessage(
        conversationSysId,
        messageSysId,
        message,
      );

      console.log("ServiceCall edit message result:", result);

      if (!result || result.success !== true) {
        throw new Error(
          result && result.message ? result.message : "Unable to edit message.",
        );
      }

      /*
       * Edit succeeded.
       *
       * Leave edit mode first.
       */

      updateEditedChatMessageInView(
        messageSysId,
        message,
        result.edited_at || "",
      );

      chatEditTarget = null;

      /*
       * Clear composer.
       */

      chatMessageInput.value = "";

      resizeChatMessageInput();

      /*
       * Reload the active conversation from
       * ServiceNow so the existing message
       * bubble receives authoritative data.
       *
       * We are intentionally NOT creating
       * another message bubble here.
       */
      /*
       * Refresh sidebar preview as well.
       */
    } catch (error) {
      console.error("Unable to edit ServiceCall message:", error);

      /*
       * IMPORTANT:
       *
       * Keep edit mode + typed text intact
       * so the user can retry.
       */
    } finally {
      chatMessageSending = false;

      if (chatMessageInput) {
        chatMessageInput.disabled = false;
      }

      if (chatSendButton) {
        if (chatEditTarget) {
          chatSendButton.textContent = "Save";

          const editingMessageType = String(
            chatEditTarget.type || "text",
          ).toLowerCase();

          const hasEditText = !!String(
            chatMessageInput ? chatMessageInput.value : "",
          ).trim();

          /*
           * Text message:
           *   empty text cannot be saved.
           *
           * Attachment message:
           *   empty caption is valid because
           *   the files remain attached.
           */
          chatSendButton.disabled =
            editingMessageType === "text" && !hasEditText;
        } else {
          chatSendButton.textContent = "Send";

          chatSendButton.disabled = !String(
            chatMessageInput ? chatMessageInput.value : "",
          ).trim();
        }
      }

      if (chatMessageInput) {
        chatMessageInput.focus();
      }
    }
  }

  /* =======================================================
   SERVICECALL CHAT - SEND BUTTON
======================================================= */

  if (chatSendButton) {
    chatSendButton.addEventListener(
      "click",

      async () => {
        /*
         * EDIT MODE
         *
         * Do NOT allow the normal send flow
         * to create a new message while an
         * existing message is being edited.
         *
         * Actual save-to-ServiceNow comes next.
         */
        if (chatEditTarget) {
          await saveEditedChatMessage();

          return;
        }

        await sendActiveChatMessage();
      },
    );
  }

  /* =======================================================
   SERVICECALL CHAT - MESSAGE INPUT
======================================================= */

  if (chatMessageInput) {
    chatMessageInput.addEventListener(
      "input",

      () => {
        if (isActiveChatReadOnly()) {
          if (String(chatMessageInput.value || "").trim()) {
            showChatMembershipError();
          } else if (chatMembershipMessage) {
            chatMembershipMessage.style.display = "none";
          }
        }
        /*
         * Grow / shrink composer
         * according to message content.
         */
        resizeChatMessageInput();

        if (!chatSendButton || chatMessageSending) {
          return;
        }

        const hasText = !!String(chatMessageInput.value || "").trim();

        /*
         * EDIT MODE
         *
         * Text messages require text.
         * Attachment messages may have an empty caption.
         */
        if (chatEditTarget) {
          const editingMessageType = String(
            chatEditTarget.type || "text",
          ).toLowerCase();

          chatSendButton.disabled = editingMessageType === "text" && !hasText;

          return;
        }

        const hasAttachments = pendingChatAttachments.some(
          (attachment) =>
            attachment &&
            attachment.status !== "failed" &&
            !!String(attachment.sysId || "").trim(),
        );

        const uploadsStillRunning = chatAttachmentUploadsInProgress > 0;

        chatSendButton.disabled =
          uploadsStillRunning || (!hasText && !hasAttachments);
      },
    );
  }

  /* =======================================================
   SERVICECALL CHAT - KEYBOARD SEND
======================================================= */

  if (chatMessageInput) {
    chatMessageInput.addEventListener(
      "keydown",

      async (event) => {
        if (event.key !== "Enter") {
          return;
        }

        /*
         * CTRL + ENTER
         * Explicitly insert a newline.
         */
        if (event.ctrlKey) {
          event.preventDefault();

          const start = chatMessageInput.selectionStart;

          const end = chatMessageInput.selectionEnd;

          const currentValue = chatMessageInput.value;

          chatMessageInput.value =
            currentValue.substring(0, start) +
            "\n" +
            currentValue.substring(end);

          const newPosition = start + 1;

          chatMessageInput.setSelectionRange(newPosition, newPosition);

          /*
           * Trigger normal input behavior
           * after changing value manually.
           */
          chatMessageInput.dispatchEvent(
            new Event("input", {
              bubbles: true,
            }),
          );

          return;
        }

        /*
         * ENTER
         * Send.
         *
         * Shift + Enter is also allowed
         * as a normal newline.
         */
        if (event.shiftKey || event.altKey || event.metaKey) {
          return;
        }

        event.preventDefault();

        if (chatMessageSending) {
          return;
        }

        const message = String(chatMessageInput.value || "").trim();

        /*
         * EDIT MODE
         *
         * Handle this BEFORE normal new-message validation.
         *
         * saveEditedChatMessage() already decides whether
         * an empty value is valid:
         *
         * - text message       -> empty NOT allowed
         * - attachment message -> empty caption allowed
         */
        if (chatEditTarget) {
          await saveEditedChatMessage();

          return;
        }

        const hasAttachments = pendingChatAttachments.some(
          (attachment) =>
            attachment &&
            attachment.status !== "failed" &&
            !!String(attachment.sysId || "").trim(),
        );

        if (
          chatAttachmentUploadsInProgress > 0 ||
          (!message && !hasAttachments)
        ) {
          return;
        }

        await sendActiveChatMessage();
      },
    );
  }

  /* =======================================================
   SERVICECALL CHAT - CONVERSATIONS
======================================================= */

  async function loadChatConversations() {
    if (!chatConversationList) {
      return;
    }

    chatConversationList.innerHTML = `
        <div style="
            padding:14px;
            font-size:12px;
            color:#6b7774;
        ">
            Loading conversations...
        </div>
    `;

    try {
      const result = await window.serviceCall.getConversations();

      console.log("ServiceCall conversations:", result);

      if (!result || result.success !== true) {
        throw new Error(
          result && result.message
            ? result.message
            : "Unable to load conversations.",
        );
      }

      const conversations = Array.isArray(result.conversations)
        ? result.conversations
        : [];

      /*
       * Keep the latest authoritative conversation
       * list in renderer memory.
       *
       * Search-result clicks can use this to detect
       * whether a real direct conversation already
       * exists for that user.
       */
      loadedChatConversations = conversations;

      /*
       * =========================================
       * PRE-WARM RECENT CHAT MESSAGE CACHE
       * =========================================
       *
       * Do NOT await this.
       *
       * The conversation sidebar should continue
       * rendering immediately while recent chats
       * are warmed silently in the background.
       */
      prewarmChatMessageCache(conversations).catch((error) => {
        console.warn("Chat cache pre-warm failed:", error);
      });

      chatConversationList.innerHTML = "";

      if (conversations.length === 0) {
        chatConversationList.innerHTML = `
                <div style="
                    padding:14px;
                    font-size:12px;
                    color:#6b7774;
                ">
                    No conversations yet.
                </div>
            `;

        return;
      }

      conversations.forEach((conversation) => {
        const row = document.createElement("button");

        row.type = "button";

        /*
         * Keep the conversation sys_id
         * on the row.
         *
         * This will also help us later
         * with silent sidebar updates.
         */
        row.dataset.conversationSysId = String(conversation.sys_id || "");

        row.dataset.conversationSysId = String(conversation.sys_id || "");

        /* =====================================================
   DIRECT CHAT - RIGHT CLICK MENU
===================================================== */

        row.addEventListener("contextmenu", (event) => {
          return;
          /*
           * DELETE CHAT AVAILABILITY
           *
           * Direct chat:
           *   Delete Chat is allowed.
           *
           * Active group:
           *   Delete Chat is NOT allowed.
           *   The user must Leave Group first.
           *
           * Historical group:
           *   After leaving / being removed,
           *   Delete Chat is allowed.
           */

          const conversationType = String(conversation.type || "")
            .trim()
            .toLowerCase();

          const membershipActive = conversation.membership_active === true;

          /*
           * Only direct and group conversations
           * support Delete Chat.
           */
          if (conversationType !== "direct" && conversationType !== "group") {
            return;
          }

          /*
           * An active group cannot be deleted.
           *
           * The user must Leave Group first.
           */
          if (conversationType === "group" && membershipActive) {
            return;
          }

          event.preventDefault();
          event.stopPropagation();

          /*
           * Only one conversation context menu
           * may exist at a time.
           */
          document
            .querySelectorAll(".chat-conversation-context-menu")
            .forEach((existingMenu) => {
              existingMenu.remove();
            });

          const menu = document.createElement("div");

          menu.className = "chat-conversation-context-menu";

          menu.style.cssText = `
    position:fixed;
    left:${event.clientX}px;
    top:${event.clientY}px;
 
    min-width:150px;
 
    padding:5px;
 
    border:1px solid #dfe7e4;
    border-radius:8px;
 
    background:#ffffff;
 
    box-shadow:
      0 8px 24px rgba(0, 0, 0, 0.12);
 
    z-index:10000;
  `;

          const deleteChatButton = document.createElement("button");

          deleteChatButton.type = "button";

          deleteChatButton.textContent = "Delete chat";

          deleteChatButton.style.cssText = `
    width:100%;
 
    border:none;
    border-radius:6px;
 
    padding:8px 10px;
 
    background:transparent;
 
    color:#b42318;
 
    font-size:12px;
    text-align:left;
 
    cursor:pointer;
  `;

          deleteChatButton.addEventListener("mouseenter", () => {
            deleteChatButton.style.background = "#fff1f0";
          });

          deleteChatButton.addEventListener("mouseleave", () => {
            deleteChatButton.style.background = "transparent";
          });

          /*
           * Execution comes in the next step.
           *
           * For now we only verify that the
           * right-click interaction works.
           */
          deleteChatButton.addEventListener("click", async (deleteEvent) => {
            deleteEvent.preventDefault();
            deleteEvent.stopPropagation();

            const conversationSysId = String(conversation.sys_id || "").trim();

            const conversationName = String(
              conversation.display_name || conversation.title || "this chat",
            ).trim();

            /*
             * Close the context menu first.
             */
            menu.remove();

            if (!conversationSysId) {
              return;
            }

            /* =====================================================
     CONFIRM DELETE CHAT
  ===================================================== */

            const confirmed = window.confirm(
              `Delete chat with ${conversationName}?\n\n` +
                `This chat will be removed only from your chat list.`,
            );

            /*
             * Cancel / X:
             * absolutely nothing changes.
             */
            if (!confirmed) {
              return;
            }

            /* =====================================================
     DELETE CHAT
  ===================================================== */

            deleteChatButton.disabled = true;

            try {
              const result =
                await window.serviceCall.deleteChat(conversationSysId);

              console.log("ServiceCall delete chat:", result);

              if (!result || result.success !== true) {
                throw new Error(
                  result && result.message
                    ? result.message
                    : "Unable to delete chat.",
                );
              }

              /*
               * Remove cached messages for this
               * conversation.
               */
              chatMessageCache.delete(conversationSysId);

              /*
               * If the deleted conversation is currently
               * open, clear the active conversation.
               */
              if (
                activeChatConversation &&
                String(activeChatConversation.sys_id || "") ===
                  conversationSysId
              ) {
                /*
                 * The currently-open conversation has
                 * just been deleted for this user.
                 *
                 * Clear ALL renderer state belonging
                 * to that conversation immediately.
                 */
                activeChatConversation = null;
                activeChatUser = null;

                lastChatMessageSysId = "";
                lastChatReactionCheckpoint = "";

                oldestChatMessageSysId = "";
                chatMessageHistoryHasMore = false;
                chatMessageHistoryLoading = false;

                chatNewMessageCount = 0;
                chatUserWasNearBottom = true;

                clearChatMessageSelection();
                updateChatNewMessagesButton();

                /*
                 * Remove the deleted conversation from
                 * the visible conversation area immediately.
                 */
                if (chatMessages) {
                  chatMessages.innerHTML = "";
                }

                /*
                 * No conversation is currently active,
                 * therefore the composer must not remain
                 * attached to the deleted conversation.
                 */
                if (chatMessageInput) {
                  chatMessageInput.value = "";
                  chatMessageInput.disabled = true;
                  chatMessageInput.placeholder =
                    "Select a conversation to start messaging.";
                }

                if (chatSendButton) {
                  chatSendButton.disabled = true;
                }

                /*
                 * Return the right side to the normal
                 * no-conversation-selected state.
                 */
                if (chatConversationPanel) {
                  chatConversationPanel.style.display = "none";
                }

                if (chatEmptyState) {
                  chatEmptyState.style.display = "";
                }
              }

              /*
               * Reload from ServiceNow.
               *
               * Because /conversations now excludes
               * u_hidden=true direct memberships,
               * this chat should disappear.
               */
              await loadChatConversations();
            } catch (error) {
              console.error("Unable to delete ServiceCall chat:", error);

              window.alert(
                error && error.message
                  ? error.message
                  : "Unable to delete chat.",
              );
            } finally {
              deleteChatButton.disabled = false;
            }
          });

          menu.appendChild(deleteChatButton);

          document.body.appendChild(menu);

          /*
           * Prevent menu from overflowing beyond
           * the visible Electron window.
           */
          const menuRect = menu.getBoundingClientRect();

          if (menuRect.right > window.innerWidth) {
            menu.style.left =
              Math.max(8, window.innerWidth - menuRect.width - 8) + "px";
          }

          if (menuRect.bottom > window.innerHeight) {
            menu.style.top =
              Math.max(8, window.innerHeight - menuRect.height - 8) + "px";
          }

          /*
           * Clicking anywhere outside closes it.
           */
          setTimeout(() => {
            function closeConversationMenu(outsideEvent) {
              if (menu.contains(outsideEvent.target)) {
                return;
              }

              menu.remove();

              document.removeEventListener(
                "mousedown",
                closeConversationMenu,
                true,
              );
            }

            document.addEventListener("mousedown", closeConversationMenu, true);
          }, 0);
        });

        row.style.cssText = `
                    width:100%;
                    display:flex;
                    align-items:center;
                    gap:12px;
                    padding:12px 14px;
                    border:0;
                    border-bottom:1px solid #edf1ef;
                    background:white;
                    text-align:left;
                    cursor:pointer;
                `;

        /* -------------------------
                   DISPLAY NAME
                ------------------------- */

        const displayName =
          conversation.display_name || conversation.title || "Conversation";

        /* -------------------------
                   INITIALS
                ------------------------- */

        const nameParts = displayName.trim().split(/\s+/).filter(Boolean);

        let initials = "?";

        if (nameParts.length >= 2) {
          initials = (
            nameParts[0][0] + nameParts[nameParts.length - 1][0]
          ).toUpperCase();
        } else if (nameParts.length === 1) {
          initials = nameParts[0][0].toUpperCase();
        }

        const avatar = document.createElement("div");

        avatar.textContent = initials;

        avatar.style.cssText = `
                    width:38px;
                    height:38px;
                    min-width:38px;
                    border-radius:50%;
                    display:flex;
                    align-items:center;
                    justify-content:center;
                    background:#dff3ec;
                    color:#17634f;
                    font-size:12px;
                    font-weight:700;
                `;

        /* -------------------------
                   TEXT
                ------------------------- */

        const information = document.createElement("div");

        information.style.cssText = `
                    min-width:0;
                    flex:1;
                `;

        const name = document.createElement("div");

        name.textContent = displayName;

        name.style.cssText = `
                    font-size:13px;
                    font-weight:600;
                    color:#1f2927;
                    white-space:nowrap;
                    overflow:hidden;
                    text-overflow:ellipsis;
                `;

        const preview = document.createElement("div");

        preview.textContent =
          conversation.last_message_preview || "No messages yet.";

        preview.style.cssText = `
                    margin-top:3px;
                    font-size:11px;
                    color:#78827f;
                    white-space:nowrap;
                    overflow:hidden;
                    text-overflow:ellipsis;
                `;

        information.appendChild(name);

        information.appendChild(preview);

        /* -------------------------
                   UNREAD COUNT
                ------------------------- */

        const unreadCount = Math.max(
          0,
          parseInt(conversation.unread_count, 10) || 0,
        );

        let unreadBadge = null;

        if (unreadCount > 0) {
          unreadBadge = document.createElement("div");

          unreadBadge.className = "chat-unread-badge";

          /*
           * Keep the count sensible
           * if a conversation has a
           * very large unread total.
           */
          unreadBadge.textContent =
            unreadCount > 99 ? "99+" : String(unreadCount);

          unreadBadge.style.cssText = `
                        min-width:20px;
                        height:20px;
                        padding:0 6px;
                        border-radius:10px;
                        display:flex;
                        align-items:center;
                        justify-content:center;
                        flex-shrink:0;
                        background:#17634f;
                        color:white;
                        font-size:10px;
                        font-weight:700;
                        line-height:1;
                    `;

          /*
           * Useful later for silent
           * sidebar synchronization.
           */
          unreadBadge.dataset.conversationSysId = String(
            conversation.sys_id || "",
          );
        }

        /* -------------------------
                   BUILD ROW
                ------------------------- */

        row.appendChild(avatar);

        row.appendChild(information);

        if (unreadBadge) {
          row.appendChild(unreadBadge);
        }

        /* -------------------------
                   OPEN CONVERSATION
                ------------------------- */

        row.addEventListener(
          "click",

          async () => {
            console.log("Chat conversation selected:", conversation);

            const selectedConversationSysId = String(
              conversation.sys_id || "",
            ).trim();

            const activeConversationSysId = activeChatConversation
              ? String(activeChatConversation.sys_id || "").trim()
              : "";

            /*
             * =========================================
             * ALREADY-OPEN CONVERSATION
             * =========================================
             *
             * Do NOT reopen it.
             *
             * Reopening would rerender the message list
             * and move the user to the newest message,
             * destroying their current reading position.
             */
            if (
              selectedConversationSysId &&
              activeConversationSysId === selectedConversationSysId
            ) {
              return;
            }

            /*
             * Different conversation:
             * open normally.
             */
            await openChatConversation(conversation);

            /*
             * Update local badge after the newly
             * selected conversation has been opened.
             */
            if (conversation.unread_count === 0) {
              const currentBadge = row.querySelector(".chat-unread-badge");

              if (currentBadge) {
                currentBadge.remove();
              }
            }
          },
        );

        /* -------------------------
                   HOVER
                ------------------------- */

        row.addEventListener(
          "mouseenter",

          () => {
            /*
             * Slightly stronger hover for unread chats.
             * Normal chats keep the existing hover.
             */
            row.style.background = row.classList.contains(
              "chat-conversation-unread",
            )
              ? "#e4f3ed"
              : "#f5faf8";
          },
        );

        row.addEventListener(
          "mouseleave",

          () => {
            /*
             * Preserve unread highlight after
             * the mouse leaves the row.
             */
            row.style.background = row.classList.contains(
              "chat-conversation-unread",
            )
              ? "#eef8f4"
              : "white";
          },
        );

        chatConversationList.appendChild(row);
      });
    } catch (error) {
      console.error("Unable to load ServiceCall conversations:", error);

      chatConversationList.innerHTML = `
            <div style="
                padding:14px;
                font-size:12px;
                color:#a33f3f;
            ">
                Unable to load conversations.
            </div>
        `;
    }
  }

  function resizeChatMessageInput() {
    if (!chatMessageInput) {
      return;
    }

    const MIN_HEIGHT = 44;
    const MAX_HEIGHT = 200;

    /*
     * Reset height first.
     * This is what allows the textarea
     * to SHRINK after deleting text.
     */
    chatMessageInput.style.height = "0px";

    const requiredHeight = Math.max(
      MIN_HEIGHT,
      Math.min(chatMessageInput.scrollHeight, MAX_HEIGHT),
    );

    chatMessageInput.style.height = requiredHeight + "px";

    /*
     * Scroll only after maximum
     * height has been reached.
     */
    chatMessageInput.style.overflowY =
      chatMessageInput.scrollHeight > MAX_HEIGHT ? "auto" : "hidden";
  }

  /* =======================================================
   SERVICECALL CHAT - PEOPLE SEARCH
======================================================= */

  /* -------------------------------------------------
   CHAT STATUS CLASS
------------------------------------------------- */

  function getChatPresenceClass(status) {
    const normalizedStatus = String(status || "")
      .toLowerCase()
      .trim();

    if (normalizedStatus === "available") {
      return "available";
    }

    if (normalizedStatus === "busy") {
      return "busy";
    }

    if (normalizedStatus === "away") {
      return "away";
    }

    if (normalizedStatus === "out of office") {
      return "out-of-office";
    }

    if (
      normalizedStatus === "in another call" ||
      normalizedStatus === "in call"
    ) {
      return "in-call";
    }

    return "offline";
  }

  /* -------------------------------------------------
   ADD TEMPORARY CHAT TO SIDEBAR
------------------------------------------------- */

  function addTemporaryChatToSidebar(user) {
    if (!chatConversationList || !user || !user.sys_id) {
      return;
    }

    const userSysId = String(user.sys_id).trim();

    if (!userSysId) {
      return;
    }

    /*
     * Only one temporary direct chat should exist.
     * Remove an older unsent temporary row first.
     */
    chatConversationList
      .querySelectorAll('button[data-temporary-chat="true"]')
      .forEach((row) => {
        if (String(row.dataset.userSysId || "") !== userSysId) {
          row.remove();
        }
      });

    /*
     * If this exact temporary user is already
     * in the sidebar, don't add them twice.
     */
    const existingTemporaryRow = Array.from(
      chatConversationList.querySelectorAll(
        'button[data-temporary-chat="true"]',
      ),
    ).find((row) => String(row.dataset.userSysId || "") === userSysId);

    if (existingTemporaryRow) {
      return;
    }

    const displayName = user.name || user.user_name || "Unknown User";

    const nameParts = displayName.trim().split(/\s+/).filter(Boolean);

    let initials = "?";

    if (nameParts.length >= 2) {
      initials = (
        nameParts[0][0] + nameParts[nameParts.length - 1][0]
      ).toUpperCase();
    } else if (nameParts.length === 1) {
      initials = nameParts[0][0].toUpperCase();
    }

    const row = document.createElement("button");

    row.type = "button";

    /*
     * This row does NOT have a ServiceNow
     * conversation sys_id yet.
     */
    row.dataset.temporaryChat = "true";
    row.dataset.userSysId = userSysId;

    row.style.cssText = `
    width:100%;
    display:flex;
    align-items:center;
    gap:12px;
    padding:12px 14px;
    border:0;
    border-bottom:1px solid #edf1ef;
    background:white;
    text-align:left;
    cursor:pointer;
  `;

    const avatar = document.createElement("div");

    avatar.textContent = initials;

    avatar.style.cssText = `
    width:38px;
    height:38px;
    min-width:38px;
    border-radius:50%;
    display:flex;
    align-items:center;
    justify-content:center;
    background:#dff3ec;
    color:#17634f;
    font-size:12px;
    font-weight:700;
  `;

    const information = document.createElement("div");

    information.style.cssText = `
    min-width:0;
    flex:1;
  `;

    const name = document.createElement("div");

    name.textContent = displayName;

    name.style.cssText = `
    font-size:13px;
    font-weight:600;
    color:#1f2927;
    white-space:nowrap;
    overflow:hidden;
    text-overflow:ellipsis;
  `;

    const preview = document.createElement("div");

    preview.textContent = "No messages yet.";

    preview.style.cssText = `
    margin-top:3px;
    font-size:11px;
    color:#78827f;
    white-space:nowrap;
    overflow:hidden;
    text-overflow:ellipsis;
  `;

    information.appendChild(name);
    information.appendChild(preview);

    row.appendChild(avatar);
    row.appendChild(information);

    /*
     * Clicking the temporary sidebar row
     * simply reopens the same local chat.
     */
    row.addEventListener(
      "click",

      async () => {
        const searchedUserSysId = String(user.sys_id || "").trim();

        /*
         * Close and reset the people-search dropdown
         * as soon as a user is selected.
         */
        if (chatPeopleSearchInput) {
          chatPeopleSearchInput.value = "";
        }

        if (chatPeopleSearchResults) {
          chatPeopleSearchResults.innerHTML = "";
          chatPeopleSearchResults.style.display = "none";
        }

        /*
         * Check whether we already have a real
         * direct conversation with this user.
         */
        const existingConversation = loadedChatConversations.find(
          (conversation) =>
            conversation.type === "direct" &&
            String(conversation.other_user_sys_id || "").trim() ===
              searchedUserSysId,
        );

        /*
         * Existing real conversation:
         * open it instead of creating a temporary row.
         */
        if (existingConversation) {
          await openChatConversation(existingConversation);

          /*
           * Close people-search dropdown
           * after the conversation opens.
           */
          if (chatPeopleSearchInput) {
            chatPeopleSearchInput.value = "";
            chatPeopleSearchInput.blur();
          }

          if (chatPeopleSearchResults) {
            chatPeopleSearchResults.innerHTML = "";
            chatPeopleSearchResults.style.display = "none";
          }

          return;
        }
        /*
         * No real conversation exists yet.
         * Open the local temporary chat.
         */
        openTemporaryChat(user);
      },
    );

    row.addEventListener("mouseenter", () => {
      row.style.background = "#f5faf8";
    });

    row.addEventListener("mouseleave", () => {
      row.style.background = "white";
    });

    /*
     * Temporary/new chat should appear at
     * the top of the conversation sidebar.
     */
    chatConversationList.prepend(row);
  }

  /* -------------------------------------------------
   OPEN TEMPORARY CHAT
------------------------------------------------- */

  function openTemporaryChat(user) {
    if (chatMembershipMessage) {
      chatMembershipMessage.textContent = "";
      chatMembershipMessage.style.display = "none";
    }

    if (!user || !user.sys_id) {
      return;
    }

    if (chatGroupDetailsButton) {
      chatGroupDetailsButton.style.display = "none";
    }

    /*
     * This is a NEW / TEMPORARY direct chat.
     *
     * It exists only in the renderer for now.
     * No ServiceNow conversation is created
     * until the first message is successfully sent.
     */
    activeChatConversation = {
      sys_id: "",
      type: "direct",

      temporary: true,

      other_user_sys_id: String(user.sys_id || "").trim(),

      display_name: user.name || user.user_name || "Unknown User",

      title: "",

      last_message_preview: "",
      last_message_at: "",

      unread_count: 0,

      membership_active: true,
    };

    activeChatUser = user;

    addTemporaryChatToSidebar(user);
    /*
     * Clear synchronization checkpoints
     * from the previously-open conversation.
     */
    lastChatMessageSysId = "";

    lastChatReactionCheckpoint = "";

    const displayName = user.name || user.user_name || "Unknown User";

    const displayStatus = user.display_status || "Offline";

    /* -------------------------
       NAME
    ------------------------- */

    if (chatUserName) {
      chatUserName.textContent = displayName;
    }

    /* -------------------------
       AVATAR INITIALS
    ------------------------- */

    if (chatUserAvatar) {
      const nameParts = displayName.trim().split(/\s+/).filter(Boolean);

      let initials = "?";

      if (nameParts.length >= 2) {
        initials = (
          nameParts[0][0] + nameParts[nameParts.length - 1][0]
        ).toUpperCase();
      } else if (nameParts.length === 1) {
        initials = nameParts[0][0].toUpperCase();
      }

      chatUserAvatar.textContent = initials;
    }

    /* -------------------------
       PRESENCE
    ------------------------- */

    if (chatUserPresenceText) {
      chatUserPresenceText.textContent = displayStatus;
    }

    if (chatUserPresenceDot) {
      chatUserPresenceDot.className =
        "chat-user-presence-dot " + getChatPresenceClass(displayStatus);
    }

    /* -------------------------
       SHOW CHAT
    ------------------------- */

    if (chatEmptyState) {
      chatEmptyState.style.display = "none";
    }

    if (chatConversationPanel) {
      chatConversationPanel.style.display = "flex";
    }

    /* =========================================
       IMPORTANT:
       REMOVE PREVIOUS USER'S MESSAGES
    ========================================= */

    if (chatMessages) {
      chatMessages.innerHTML = "";
    }

    /* -------------------------
       ENABLE COMPOSER
    ------------------------- */

    if (chatMessageInput) {
      chatMessageInput.disabled = false;

      chatMessageInput.placeholder = "Type a message...";
    }

    if (chatSendButton) {
      chatSendButton.disabled = !String(
        chatMessageInput ? chatMessageInput.value : "",
      ).trim();
    }

    /* -------------------------
       CLEAR SEARCH
    ------------------------- */

    if (chatPeopleSearchInput) {
      chatPeopleSearchInput.value = "";
    }

    if (chatPeopleSearchResults) {
      chatPeopleSearchResults.innerHTML = "";

      chatPeopleSearchResults.style.display = "none";
    }

    console.log("Temporary ServiceCall chat opened:", {
      sysId: user.sys_id,

      name: displayName,

      status: displayStatus,
    });
  }

  /* -------------------------------------------------
   CHAT PEOPLE SEARCH
------------------------------------------------- */

  if (chatPeopleSearchInput && chatPeopleSearchResults) {
    chatPeopleSearchInput.addEventListener(
      "input",

      () => {
        const searchText = chatPeopleSearchInput.value.trim();

        console.log("CHAT SEARCH INPUT FIRED:", searchText);

        /* -------------------------
               CANCEL PREVIOUS SEARCH
            ------------------------- */

        if (chatPeopleSearchTimer) {
          clearTimeout(chatPeopleSearchTimer);

          chatPeopleSearchTimer = null;
        }

        /* -------------------------
               EMPTY / TOO SHORT
            ------------------------- */

        if (searchText.length < 2) {
          chatPeopleSearchResults.innerHTML = "";

          chatPeopleSearchResults.style.display = "none";

          return;
        }

        /* -------------------------
               DEBOUNCE
            ------------------------- */

        chatPeopleSearchTimer = setTimeout(
          async () => {
            try {
              const result = await window.serviceCall.searchUsers(searchText);

              /*
               * Ignore an old result if
               * the user changed the search
               * while the request was running.
               */
              if (chatPeopleSearchInput.value.trim() !== searchText) {
                return;
              }

              if (!result || result.success !== true) {
                throw new Error(
                  result && result.message
                    ? result.message
                    : "Unable to search users.",
                );
              }

              const users = Array.isArray(result.users) ? result.users : [];

              chatPeopleSearchResults.innerHTML = "";

              /* -------------------------
                               NO USERS
                            ------------------------- */

              if (users.length === 0) {
                const emptyResult = document.createElement("div");

                emptyResult.textContent = "No users found.";

                emptyResult.style.padding = "14px";

                emptyResult.style.color = "#71827d";

                emptyResult.style.fontSize = "12px";

                chatPeopleSearchResults.appendChild(emptyResult);

                chatPeopleSearchResults.style.display = "block";

                return;
              }

              /* -------------------------
                               RESULTS
                            ------------------------- */

              users.forEach((user) => {
                const row = document.createElement("button");

                row.type = "button";

                row.style.width = "100%";

                row.style.display = "flex";

                row.style.alignItems = "center";

                row.style.gap = "11px";

                row.style.padding = "11px 12px";

                row.style.border = "0";

                row.style.borderBottom = "1px solid #edf1f0";

                row.style.background = "white";

                row.style.textAlign = "left";

                row.style.cursor = "pointer";

                /* -------------------------
                                       AVATAR
                                    ------------------------- */

                const avatar = document.createElement("div");

                avatar.style.width = "36px";

                avatar.style.height = "36px";

                avatar.style.flexShrink = "0";

                avatar.style.display = "flex";

                avatar.style.alignItems = "center";

                avatar.style.justifyContent = "center";

                avatar.style.borderRadius = "50%";

                avatar.style.background = "#dff3ec";

                avatar.style.color = "#17634f";

                avatar.style.fontSize = "12px";

                avatar.style.fontWeight = "800";

                const displayName =
                  user.name || user.user_name || "Unknown User";

                const nameParts = displayName
                  .trim()
                  .split(/\s+/)
                  .filter(Boolean);

                if (nameParts.length >= 2) {
                  avatar.textContent = (
                    nameParts[0][0] + nameParts[nameParts.length - 1][0]
                  ).toUpperCase();
                } else {
                  avatar.textContent =
                    displayName.charAt(0).toUpperCase() || "?";
                }

                /* -------------------------
                                       INFORMATION
                                    ------------------------- */

                const information = document.createElement("div");

                information.style.minWidth = "0";

                information.style.flex = "1";

                const name = document.createElement("div");

                name.textContent = displayName;

                name.style.color = "#29463f";

                name.style.fontSize = "13px";

                name.style.fontWeight = "700";

                name.style.whiteSpace = "nowrap";

                name.style.overflow = "hidden";

                name.style.textOverflow = "ellipsis";

                const status = document.createElement("div");

                status.textContent = user.display_status || "Offline";

                status.style.marginTop = "3px";

                status.style.color = "#71827d";

                status.style.fontSize = "11px";

                information.appendChild(name);

                information.appendChild(status);

                row.appendChild(avatar);

                row.appendChild(information);

                /* -------------------------
                                       OPEN USER
                                    ------------------------- */

                row.addEventListener(
                  "click",

                  async () => {
                    const searchedUserSysId = String(user.sys_id || "").trim();

                    /*
                     * Close the actual people-search dropdown
                     * immediately when a result is selected.
                     */
                    if (chatPeopleSearchTimer) {
                      clearTimeout(chatPeopleSearchTimer);
                      chatPeopleSearchTimer = null;
                    }

                    if (chatPeopleSearchInput) {
                      chatPeopleSearchInput.value = "";
                      chatPeopleSearchInput.blur();
                    }

                    if (chatPeopleSearchResults) {
                      chatPeopleSearchResults.innerHTML = "";
                      chatPeopleSearchResults.style.display = "none";
                    }

                    /*
                     * Check whether a real direct conversation
                     * already exists with this exact user.
                     */
                    const existingConversation = loadedChatConversations.find(
                      (conversation) =>
                        conversation.type === "direct" &&
                        String(conversation.other_user_sys_id || "").trim() ===
                          searchedUserSysId,
                    );

                    /*
                     * Existing conversation:
                     * open the real chat.
                     */
                    if (existingConversation) {
                      await openChatConversation(existingConversation);

                      return;
                    }

                    /*
                     * New person:
                     * open the local temporary chat.
                     */
                    openTemporaryChat(user);
                  },
                );

                /* -------------------------
                                       HOVER
                                    ------------------------- */

                row.addEventListener(
                  "mouseenter",

                  () => {
                    row.style.background = "#f1f7f4";
                  },
                );

                row.addEventListener(
                  "mouseleave",

                  () => {
                    row.style.background = "white";
                  },
                );

                chatPeopleSearchResults.appendChild(row);
              });

              chatPeopleSearchResults.style.display = "block";
            } catch (error) {
              console.error("Chat people search failed:", error);

              chatPeopleSearchResults.innerHTML = "";

              const errorResult = document.createElement("div");

              errorResult.textContent =
                error && error.message
                  ? error.message
                  : "Unable to search users.";

              errorResult.style.padding = "14px";

              errorResult.style.color = "#a33f3f";

              errorResult.style.fontSize = "12px";

              chatPeopleSearchResults.appendChild(errorResult);

              chatPeopleSearchResults.style.display = "block";
            }
          },

          300,
        );
      },
    );
  }

  /* -------------------------------------------------
   CLOSE SEARCH RESULTS WHEN CLICKING OUTSIDE
------------------------------------------------- */

  document.addEventListener(
    "click",

    (event) => {
      if (!chatPeopleSearchInput || !chatPeopleSearchResults) {
        return;
      }

      if (
        event.target === chatPeopleSearchInput ||
        chatPeopleSearchResults.contains(event.target)
      ) {
        return;
      }

      chatPeopleSearchResults.style.display = "none";
    },
  );

  /* =======================================================
   SERVICECALL CHAT - DIRECT CALL
======================================================= */

  if (chatCallButton) {
    chatCallButton.addEventListener(
      "click",

      async () => {
        /*
         * A user must currently be open
         * in Chat.
         */
        if (!activeChatUser || !activeChatUser.sys_id) {
          console.warn("No active Chat user selected.");

          return;
        }

        /*
         * Presence intentionally does NOT
         * block the call.
         *
         * Available, Away, Offline,
         * Out of Office, Busy or
         * In another call may all still
         * receive a call attempt.
         *
         * Server-side call rules remain
         * authoritative.
         */

        const targetUser = activeChatUser;

        const targetName = targetUser.name || targetUser.user_name || "user";

        /*
         * Prevent duplicate clicks while
         * the call request is starting.
         */
        chatCallButton.disabled = true;

        const originalContent = chatCallButton.innerHTML;

        chatCallButton.textContent = "...";

        try {
          console.log("Starting ServiceCall from Chat:", {
            targetUserSysId: targetUser.sys_id,

            targetUserName: targetName,

            displayStatus: targetUser.display_status || "",
          });

          const result = await window.serviceCall.startCall(targetUser.sys_id);

          console.log("Chat start call result:", result);

          if (!result || result.success !== true) {
            throw new Error(
              result && result.message
                ? result.message
                : "Unable to start call.",
            );
          }

          /*
           * IMPORTANT:
           *
           * Do NOT open another call
           * window here.
           *
           * The existing ServiceCall
           * outgoing-call architecture
           * detects the call and opens
           * the normal call window.
           */

          console.log(
            "ServiceCall started from Chat:",
            result.call_number || result.call_sys_id || targetName,
          );
        } catch (error) {
          console.error("Unable to start ServiceCall from Chat:", error);

          /*
           * Only restore the button on
           * failure.
           */
          chatCallButton.disabled = false;

          chatCallButton.innerHTML = originalContent;

          return;
        }

        /*
         * Restore the Chat button shortly
         * after the request succeeds.
         *
         * This lock is only preventing
         * duplicate Start Call requests.
         * The actual call lifecycle is
         * controlled by main.js.
         */
        setTimeout(
          () => {
            chatCallButton.disabled = false;

            chatCallButton.innerHTML = originalContent;
          },

          1200,
        );
      },
    );
  }

  /* =======================================================
   SERVICECALL PEOPLE
======================================================= */

  if (peopleSearchInput && peopleSearchResults) {
    peopleSearchInput.addEventListener("input", () => {
      const searchText = peopleSearchInput.value.trim();

      /*
       * Cancel previous pending search.
       */
      if (peopleSearchTimer) {
        clearTimeout(peopleSearchTimer);

        peopleSearchTimer = null;
      }

      /*
       * Existing /users API requires
       * at least two characters.
       */
      if (searchText.length < 2) {
        peopleSearchResults.innerHTML = "";

        if (peopleSearchMessage) {
          peopleSearchMessage.textContent =
            searchText.length === 1 ? "Type at least 2 characters." : "";
        }

        return;
      }

      if (peopleSearchMessage) {
        peopleSearchMessage.textContent = "Searching...";
      }

      peopleSearchTimer = setTimeout(async () => {
        try {
          const result = await window.serviceCall.searchUsers(searchText);

          /*
           * User may have typed something
           * different while this request
           * was running.
           */
          if (peopleSearchInput.value.trim() !== searchText) {
            return;
          }

          if (!result || result.success !== true) {
            throw new Error(
              result && result.message
                ? result.message
                : "Unable to search users.",
            );
          }

          const users = Array.isArray(result.users) ? result.users : [];

          peopleSearchResults.innerHTML = "";

          if (users.length === 0) {
            if (peopleSearchMessage) {
              peopleSearchMessage.textContent = "No users found.";
            }

            return;
          }

          if (peopleSearchMessage) {
            peopleSearchMessage.textContent = "";
          }

          users.forEach((user) => {
            const row = document.createElement("div");

            row.className = "people-result";

            /* -------------------------
                                       USER INFORMATION
                                    ------------------------- */

            const userInfo = document.createElement("div");

            const userName = document.createElement("div");

            userName.textContent = user.name || "Unknown User";

            userName.style.fontWeight = "600";

            const userDetails = document.createElement("div");

            userDetails.style.fontSize = "13px";

            userDetails.style.marginTop = "4px";

            const identityParts = [];

            if (user.user_name) {
              identityParts.push(user.user_name);
            }

            if (user.email) {
              identityParts.push(user.email);
            }

            userDetails.textContent = identityParts.join(" • ");

            /* -------------------------
                                       CURRENT STATUS
                                    ------------------------- */

            const userStatus = document.createElement("div");

            userStatus.style.fontSize = "13px";

            userStatus.style.marginTop = "5px";

            userStatus.textContent = user.display_status || "Offline";

            userInfo.appendChild(userName);

            if (identityParts.length > 0) {
              userInfo.appendChild(userDetails);
            }

            userInfo.appendChild(userStatus);

            /* -------------------------
                                       CALL BUTTON
                                    ------------------------- */

            const callButton = document.createElement("button");

            callButton.type = "button";

            callButton.className = "primary-button";

            callButton.textContent = "Call";

            callButton.addEventListener(
              "click",

              async () => {
                callButton.disabled = true;

                callButton.textContent = "Calling...";

                if (peopleSearchMessage) {
                  peopleSearchMessage.textContent =
                    "Calling " + (user.name || "user") + "...";
                }

                try {
                  const callResult = await window.serviceCall.startCall(
                    user.sys_id,
                  );

                  console.log("Start call result:", callResult);

                  if (!callResult || callResult.success !== true) {
                    throw new Error(
                      callResult && callResult.message
                        ? callResult.message
                        : "Unable to start call.",
                    );
                  }

                  if (peopleSearchMessage) {
                    peopleSearchMessage.textContent =
                      "Calling " +
                      (callResult.target_user_name || user.name || "user") +
                      "...";
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
                  console.error("Start ServiceCall failed:", error);

                  if (peopleSearchMessage) {
                    peopleSearchMessage.textContent =
                      error.message || "Unable to start call.";
                  }

                  callButton.disabled = false;

                  callButton.textContent = "Call";
                }
              },
            );

            /* -------------------------
                                       RESULT ROW
                                    ------------------------- */

            row.appendChild(userInfo);

            row.appendChild(callButton);

            /*
             * Temporary functional layout.
             * Final UI comes later.
             */
            row.style.display = "flex";

            row.style.alignItems = "center";

            row.style.justifyContent = "space-between";

            row.style.gap = "16px";

            row.style.padding = "12px 0";

            row.style.borderBottom = "1px solid #e5e5e5";

            peopleSearchResults.appendChild(row);
          });
        } catch (error) {
          console.error("People search failed:", error);

          peopleSearchResults.innerHTML = "";

          if (peopleSearchMessage) {
            peopleSearchMessage.textContent =
              error.message || "Unable to search users.";
          }
        }
      }, 300);
    });
  }

  /* =======================================================
   SERVICECALL NOTIFICATIONS
======================================================= */

  /* -------------------------------------------------
   NOTIFICATION TYPE HELPERS
------------------------------------------------- */

  function getNotificationCategory(notification) {
    const type = String(
      notification && notification.type ? notification.type : "",
    )
      .toLowerCase()
      .trim();

    if (type.includes("meeting")) {
      return "meeting";
    }

    if (type.includes("chat") || type.includes("message")) {
      return "chat";
    }

    if (type.includes("call") || type.includes("recording")) {
      return "call";
    }

    return "system";
  }

  /* -------------------------------------------------
   NOTIFICATION ICON
------------------------------------------------- */

  function getNotificationIcon(notification) {
    const category = getNotificationCategory(notification);

    switch (category) {
      case "meeting":
        return "📅";

      case "chat":
        return "💬";

      case "call":
        return "☎";

      default:
        return "🔔";
    }
  }

  /* -------------------------------------------------
   NOTIFICATION TIME
------------------------------------------------- */

  function formatNotificationTime(notification) {
    if (!notification) {
      return "";
    }

    return notification.created_at_display || notification.created_at || "";
  }

  /* -------------------------------------------------
   UPDATE UNREAD BADGE
------------------------------------------------- */

  function updateNotificationUnreadBadge(unreadCount) {
    const count = Math.max(0, parseInt(unreadCount, 10) || 0);

    /*
     * -----------------------------------------
     * SIDEBAR BADGE
     * -----------------------------------------
     */

    if (notificationUnreadBadge) {
      notificationUnreadBadge.textContent = count > 99 ? "99+" : String(count);

      notificationUnreadBadge.style.display = count > 0 ? "flex" : "none";
    }

    /*
     * -----------------------------------------
     * UNREAD FILTER COUNT
     * -----------------------------------------
     */

    if (notificationUnreadFilterCount) {
      notificationUnreadFilterCount.textContent =
        count > 99 ? "99+" : String(count);

      notificationUnreadFilterCount.style.display =
        count > 0 ? "inline-flex" : "none";
    }

    /*
     * -----------------------------------------
     * MARK ALL AS READ
     * -----------------------------------------
     *
     * Keep hidden until the Mark All backend
     * functionality is implemented.
     */

    if (markAllNotificationsReadButton) {
      markAllNotificationsReadButton.style.display = "none";
    }
  }

  /* -------------------------------------------------
   BRIEF NEW-NOTIFICATION PULSE
------------------------------------------------- */

  function pulseNotificationsNavigation() {
    if (!notificationsNavButton) {
      return;
    }

    notificationsNavButton.classList.remove("notification-arrived");

    /*
     * Force a reflow so the animation can
     * restart even if another notification
     * arrives shortly afterwards.
     */
    void notificationsNavButton.offsetWidth;

    notificationsNavButton.classList.add("notification-arrived");

    setTimeout(() => {
      notificationsNavButton.classList.remove("notification-arrived");
    }, 2200);
  }

  /* -------------------------------------------------
   HIGHLIGHT NOTIFICATION SEARCH MATCH
------------------------------------------------- */

  function highlightNotificationSearch(text, search) {
    const value = String(text || "");

    const searchValue = String(search || "").trim();

    /*
     * No active search.
     */
    if (!searchValue) {
      return escapeHtml(value);
    }

    /*
     * Escape the original text first
     * so notification content cannot
     * inject HTML.
     */
    const safeText = escapeHtml(value);

    /*
     * Escape special RegExp characters
     * entered by the user.
     */
    const safeSearch = searchValue.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");

    const expression = new RegExp("(" + safeSearch + ")", "gi");

    return safeText.replace(
      expression,
      '<mark class="notification-search-highlight">$1</mark>',
    );
  }

  /* -------------------------------------------------
   OPEN NOTIFICATION DETAIL
------------------------------------------------- */

  function openNotificationDetail(notification) {
    if (!notification || !notificationListPanel || !notificationDetailPanel) {
      return;
    }

    /*
     * Populate detail information.
     */
    if (notificationDetailIcon) {
      notificationDetailIcon.textContent = getNotificationIcon(notification);
    }

    if (notificationDetailType) {
      notificationDetailType.textContent =
        notification.type_display || notification.type || "Notification";
    }

    if (notificationDetailTitle) {
      /*
       * Detail view deliberately does not
       * use search highlighting.
       */
      notificationDetailTitle.textContent =
        notification.title ||
        notification.type_display ||
        "ServiceCall notification";
    }

    if (notificationDetailTime) {
      notificationDetailTime.textContent = formatNotificationTime(notification);
    }

    if (notificationDetailMessage) {
      notificationDetailMessage.textContent =
        notification.message || "No additional information.";
    }

    /*
     * Clear actions left by the previously
     * opened notification.
     */
    if (notificationDetailActions) {
      notificationDetailActions.innerHTML = "";

      /*
       * Meeting notification.
       */
      if (
        notification.action_type === "open_meeting" &&
        notification.meeting_sys_id
      ) {
        const openMeetingButton = document.createElement("button");

        openMeetingButton.type = "button";

        openMeetingButton.className = "primary-button";

        openMeetingButton.textContent = "Open Meeting";

        openMeetingButton.addEventListener(
          "click",

          async (event) => {
            event.stopPropagation();

            openMeetingButton.disabled = true;

            openMeetingButton.textContent = "Opening...";

            try {
              await openMeetingFromDeepLink(notification.meeting_sys_id);
            } catch (error) {
              console.error("Unable to open notification meeting:", error);
            } finally {
              openMeetingButton.disabled = false;

              openMeetingButton.textContent = "Open Meeting";
            }
          },
        );

        notificationDetailActions.appendChild(openMeetingButton);
      }
    }

    /*
     * Move from list → detail.
     */
    notificationListPanel.classList.add("detail-open");

    notificationDetailPanel.classList.add("active");

    notificationDetailPanel.setAttribute("aria-hidden", "false");

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
    if (notification.read !== true && notification.sys_id) {
      window.serviceCall
        .markNotificationRead(notification.sys_id)
        .then((result) => {
          console.log("Mark notification read result:", result);

          if (!result || result.success !== true) {
            console.error(
              "ServiceNow did not mark notification as read:",
              result,
            );

            return;
          }

          /*
           * -----------------------------------------
           * UPDATE THIS NOTIFICATION LOCALLY
           * -----------------------------------------
           */

          notification.read = true;

          notification.read_at = result.read_at || "";

          /*
           * cachedNotifications normally contains
           * this same object, but update by sys_id
           * as well so we are explicit.
           */
          const cachedNotification = cachedNotifications.find(
            (item) => String(item.sys_id || "") === String(notification.sys_id),
          );

          if (cachedNotification) {
            cachedNotification.read = true;

            cachedNotification.read_at = result.read_at || "";
          }

          /*
           * -----------------------------------------
           * UPDATE AUTHORITATIVE UNREAD COUNT
           * -----------------------------------------
           */

          const unreadCount = Number(result.unread_count || 0);

          if (cachedNotificationResult) {
            cachedNotificationResult.unread_count = unreadCount;
          }

          /*
           * Sidebar Notifications badge.
           */
          updateNotificationUnreadBadge(unreadCount);

          /*
           * Do NOT redraw the list while the
           * detail screen is open.
           *
           * Back will render the correct state.
           */
        })
        .catch((error) => {
          console.error("Unable to mark notification as read:", error);
        });
    }
  }

  /* -------------------------------------------------
   CREATE NOTIFICATION CARD
------------------------------------------------- */

  function createNotificationCard(notification, animateArrival = false) {
    const card = document.createElement("div");

    card.className = "notification-card";

    if (notification.read === true) {
      card.classList.add("read");
    } else {
      card.classList.add("unread");
    }

    if (animateArrival) {
      card.classList.add("notification-card-arriving");
    }

    /*
     * Unread indicator.
     */
    if (notification.read !== true) {
      const unreadDot = document.createElement("div");

      unreadDot.className = "notification-unread-dot";

      card.appendChild(unreadDot);
    }

    /*
     * Icon.
     */
    const icon = document.createElement("div");

    icon.className = "notification-icon";

    icon.textContent = getNotificationIcon(notification);

    card.appendChild(icon);

    /*
     * Main body.
     */
    const body = document.createElement("div");

    body.className = "notification-body";

    const titleRow = document.createElement("div");

    titleRow.className = "notification-title-row";

    const title = document.createElement("div");

    title.className = "notification-title";

    title.innerHTML = highlightNotificationSearch(
      notification.title ||
        notification.type_display ||
        "ServiceCall notification",

      currentNotificationSearch,
    );

    const time = document.createElement("div");

    time.className = "notification-time";

    time.textContent = formatNotificationTime(notification);

    titleRow.appendChild(title);

    titleRow.appendChild(time);

    body.appendChild(titleRow);

    /*
     * Message.
     */
    if (notification.message) {
      const notificationMessage = document.createElement("div");

      notificationMessage.className = "notification-message";

      notificationMessage.innerHTML = highlightNotificationSearch(
        notification.message,
        currentNotificationSearch,
      );

      body.appendChild(notificationMessage);
    }

    /*
     * Actions.
     */
    const actions = document.createElement("div");

    actions.className = "notification-actions";

    /*
     * OPEN MEETING
     *
     * Reuses the existing secure meeting
     * details flow.
     */
    if (
      notification.action_type === "open_meeting" &&
      notification.meeting_sys_id
    ) {
      const openMeetingButton = document.createElement("button");

      openMeetingButton.type = "button";

      openMeetingButton.className = "notification-action-button primary";

      openMeetingButton.textContent = "Open Meeting";

      openMeetingButton.addEventListener(
        "click",

        async () => {
          openMeetingButton.disabled = true;

          openMeetingButton.textContent = "Opening...";

          try {
            await openMeetingFromDeepLink(notification.meeting_sys_id);
          } catch (error) {
            console.error("Unable to open notification meeting:", error);
          } finally {
            openMeetingButton.disabled = false;

            openMeetingButton.textContent = "Open Meeting";
          }
        },
      );

      actions.appendChild(openMeetingButton);
    }

    /*
     * Only append the action row if
     * something was actually added.
     */
    if (actions.children.length > 0) {
      body.appendChild(actions);
    }

    card.appendChild(body);

    /*
     * Open the notification in the
     * same-page detail view.
     */
    card.addEventListener(
      "click",

      () => {
        openNotificationDetail(notification);
      },
    );

    card.setAttribute("role", "button");

    card.setAttribute("tabindex", "0");

    return card;
  }

  /* -------------------------------------------------
   CLOSE NOTIFICATION DETAIL
------------------------------------------------- */

  function closeNotificationDetail() {
    if (!notificationListPanel || !notificationDetailPanel) {
      return;
    }

    /*
     * Animate the detail screen out.
     */
    notificationDetailPanel.classList.remove("active");

    notificationDetailPanel.classList.add("closing");

    /*
     * Wait for the exit animation before
     * restoring the notification list.
     */
    setTimeout(() => {
      notificationDetailPanel.classList.remove("closing");

      notificationDetailPanel.setAttribute("aria-hidden", "true");

      /*
       * Restore list.
       */
      notificationListPanel.classList.remove("detail-open");

      notificationListPanel.classList.add("returning");

      /*
       * Clean temporary animation class.
       */
      setTimeout(() => {
        notificationListPanel.classList.remove("returning");

        /*
         * Re-render from our local cache.
         *
         * No ServiceNow request is required.
         */
        renderNotifications(cachedNotifications);

        if (cachedNotificationResult) {
          renderNotificationPagination(cachedNotificationResult);
        }
      }, 220);
    }, 180);
  }

  if (notificationDetailBackButton) {
    notificationDetailBackButton.addEventListener(
      "click",
      closeNotificationDetail,
    );
  }

  /* -------------------------------------------------
   RENDER NOTIFICATIONS
------------------------------------------------- */

  function renderNotifications(notifications, newlyArrivedIds = new Set()) {
    if (!notificationsContainer) {
      return;
    }

    notificationsContainer.innerHTML = "";

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

    const filteredNotifications = notifications.filter((notification) => {
      /*
       * ALL
       */
      if (currentNotificationFilter === "all") {
        return true;
      }

      /*
       * UNREAD
       */
      if (currentNotificationFilter === "unread") {
        return notification.read !== true;
      }

      /*
       * READ
       */
      if (currentNotificationFilter === "read") {
        return notification.read === true;
      }

      return true;
    });

    /*
     * -----------------------------------------
     * EMPTY STATE
     * -----------------------------------------
     */

    if (filteredNotifications.length === 0) {
      let emptyMessage = "You don't have any ServiceCall notifications yet.";

      if (currentNotificationFilter === "unread") {
        emptyMessage = "You have no unread notifications.";
      }

      if (currentNotificationFilter === "read") {
        emptyMessage = "You have no read notifications.";
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

    filteredNotifications.forEach((notification) => {
      notificationsContainer.appendChild(
        createNotificationCard(
          notification,

          newlyArrivedIds.has(notification.sys_id),
        ),
      );
    });
  }

  /* -------------------------------------------------
   NOTIFICATION PAGINATION
------------------------------------------------- */

  function renderNotificationPagination(result) {
    if (!notificationPagination) {
      return;
    }

    notificationPagination.innerHTML = "";

    const page = parseInt(result.page, 10) || 1;

    const hasMore = result.has_more === true;

    /*
     * With the current notification API,
     * we know the current page and whether
     * another page exists.
     *
     * Therefore use simple Previous/Next
     * pagination instead of pretending we
     * know a total page count.
     */
    if (page <= 1 && !hasMore) {
      return;
    }

    const previousButton = document.createElement("button");

    previousButton.type = "button";

    previousButton.className = "meeting-page-button";

    previousButton.textContent = "‹ Previous";

    previousButton.disabled = page <= 1;

    previousButton.addEventListener(
      "click",

      async () => {
        if (page <= 1) {
          return;
        }

        currentNotificationPage = page - 1;

        await loadNotifications(false);
      },
    );

    notificationPagination.appendChild(previousButton);

    const pageInfo = document.createElement("span");

    pageInfo.className = "meeting-page-info";

    pageInfo.textContent = "Page " + page;

    notificationPagination.appendChild(pageInfo);

    const nextButton = document.createElement("button");

    nextButton.type = "button";

    nextButton.className = "meeting-page-button";

    nextButton.textContent = "Next ›";

    nextButton.disabled = !hasMore;

    nextButton.addEventListener(
      "click",

      async () => {
        if (!hasMore) {
          return;
        }

        currentNotificationPage = page + 1;

        await loadNotifications(false);
      },
    );

    notificationPagination.appendChild(nextButton);
  }

  /* -------------------------------------------------
   LOAD NOTIFICATIONS
------------------------------------------------- */

  async function loadNotifications(silent = false) {
    const requestSearch = currentNotificationSearch;

    const requestSearchVersion = notificationSearchVersion;

    if (!notificationsContainer) {
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
    if (!silent) {
      notificationsContainer.innerHTML = `
            <div class="loading">
                Loading notifications...
            </div>
        `;

      if (notificationPagination) {
        notificationPagination.innerHTML = "";
      }
    }

    try {
      const result = await window.serviceCall.getNotifications(
        currentNotificationPage,
        20,
        requestSearch,
      );

      /*
       * The user typed something else while
       * this request was running.
       *
       * Ignore this old response completely.
       */
      if (
        requestSearchVersion !== notificationSearchVersion ||
        requestSearch !== currentNotificationSearch
      ) {
        return;
      }

      if (!result || result.success !== true) {
        throw new Error(
          result && result.message
            ? result.message
            : "Unable to retrieve notifications.",
        );
      }

      const notifications = Array.isArray(result.notifications)
        ? result.notifications
        : [];

      /*
       * Keep the latest ServiceNow result in memory.
       *
       * All / Unread / Read can now switch instantly.
       */
      cachedNotifications = notifications;

      cachedNotificationResult = result;

      /*
       * Update unread count globally,
       * regardless of which page the
       * user currently has open.
       */
      updateNotificationUnreadBadge(result.unread_count);

      const newlyArrivedIds = new Set();

      /*
       * IMPORTANT:
       *
       * The first successful load establishes
       * our baseline.
       *
       * Existing notifications must NOT all
       * pulse as though they just arrived.
       */
      if (notificationsInitialized) {
        notifications.forEach((notification) => {
          const sysId = String(notification.sys_id || "").trim();

          if (sysId && !knownNotificationIds.has(sysId)) {
            newlyArrivedIds.add(sysId);
          }
        });
      }

      /*
       * Remember everything returned by
       * this API response.
       */
      notifications.forEach((notification) => {
        const sysId = String(notification.sys_id || "").trim();

        if (sysId) {
          knownNotificationIds.add(sysId);
        }
      });

      notificationsInitialized = true;

      if (newlyArrivedIds.size > 0) {
        pulseNotificationsNavigation();

        /*
         * Show a desktop popup only for
         * genuinely new notifications.
         */
        notifications.forEach((notification) => {
          const sysId = String(notification.sys_id || "").trim();

          if (sysId && newlyArrivedIds.has(sysId)) {
            window.serviceCall
              .showNotificationPopup({
                notificationSysId: sysId,

                type:
                  notification.type_display ||
                  notification.type ||
                  "Notification",

                title: notification.title || "ServiceCall",

                message: notification.message || "",

                meetingSysId: notification.meeting_sys_id || "",
              })
              .catch((error) => {
                console.error(
                  "Unable to show ServiceCall notification popup:",
                  error,
                );
              });
          }
        });
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
      const notificationsView = document.getElementById("notificationsView");

      if (
        !silent ||
        (notificationsView && notificationsView.classList.contains("active"))
      ) {
        renderNotifications(notifications, newlyArrivedIds);

        renderNotificationPagination(result);
      }
    } catch (error) {
      console.error("Unable to load notifications:", error);

      /*
       * Never destroy existing cards because
       * a silent background refresh failed.
       */
      if (!silent) {
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

  if (refreshNotificationsButton) {
    refreshNotificationsButton.addEventListener(
      "click",

      async () => {
        refreshNotificationsButton.disabled = true;

        const originalText = refreshNotificationsButton.textContent;

        refreshNotificationsButton.textContent = "Refreshing...";

        try {
          await loadNotifications(true);
        } finally {
          refreshNotificationsButton.disabled = false;

          refreshNotificationsButton.textContent = originalText;
        }
      },
    );
  }

  /* -------------------------------------------------
   NOTIFICATION SEARCH
------------------------------------------------- */

  if (notificationSearchInput) {
    notificationSearchInput.addEventListener("input", () => {
      const searchValue = notificationSearchInput.value.trim();

      /*
       * Update the active search immediately.
       *
       * This lets the already-loaded cards respond
       * instantly while the full ServiceNow search
       * is waiting for the debounce timer.
       */
      currentNotificationSearch = searchValue;

      notificationSearchVersion++;

      /*
       * Show / hide clear button.
       */
      if (notificationSearchClear) {
        notificationSearchClear.style.display = searchValue ? "flex" : "none";
      }

      /*
       * Cancel previous pending search.
       */
      if (notificationSearchTimer) {
        clearTimeout(notificationSearchTimer);
      }

      /*
       * Wait briefly before searching
       * so we don't call ServiceNow on
       * every keystroke.
       */
      notificationSearchTimer = setTimeout(async () => {
        /*
         * New search always begins
         * from page 1.
         */
        currentNotificationPage = 1;

        await loadNotifications(true);
      }, 180);
    });
  }

  /* -------------------------------------------------
   CLEAR NOTIFICATION SEARCH
------------------------------------------------- */

  if (notificationSearchClear) {
    notificationSearchClear.addEventListener(
      "click",

      async () => {
        if (notificationSearchTimer) {
          clearTimeout(notificationSearchTimer);

          notificationSearchTimer = null;
        }

        if (notificationSearchInput) {
          notificationSearchInput.value = "";

          notificationSearchInput.focus();
        }

        notificationSearchClear.style.display = "none";

        currentNotificationSearch = "";

        notificationSearchVersion++;

        currentNotificationPage = 1;

        await loadNotifications(true);
      },
    );
  }

  /* -------------------------------------------------
   NOTIFICATION FILTERS
------------------------------------------------- */

  notificationFilterButtons.forEach((button) => {
    button.addEventListener(
      "click",

      () => {
        /*
         * Update selected filter visually.
         */
        notificationFilterButtons.forEach((filterButton) => {
          filterButton.classList.remove("active");
        });

        button.classList.add("active");

        currentNotificationFilter = String(
          button.dataset.notificationFilter || "all",
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
        renderNotifications(cachedNotifications);

        /*
         * Pagination still belongs to the
         * server result currently loaded.
         */
        if (cachedNotificationResult) {
          renderNotificationPagination(cachedNotificationResult);
        }
      },
    );
  });

  /* -------------------------------------------------
   GLOBAL NOTIFICATION AUTO REFRESH
------------------------------------------------- */

  function startNotificationsAutoRefresh() {
    if (notificationAutoRefreshTimer) {
      return;
    }

    /*
     * Establish baseline immediately.
     *
     * This is silent because ServiceCall may
     * currently be displaying Home/Meetings/etc.
     */
    loadNotifications(true);

    notificationAutoRefreshTimer = setInterval(async () => {
      try {
        /*
         * Background monitoring always
         * checks page 1 because that's
         * where newly created notifications
         * appear.
         *
         * Preserve the user's pagination.
         */
        const originalPage = currentNotificationPage;

        currentNotificationPage = 1;

        await loadNotifications(true);

        currentNotificationPage = originalPage;
      } catch (error) {
        console.error("Notification auto-refresh failed:", error);
      }
    }, 15000);
  }

  /*
   * Start notification monitoring for the
   * lifetime of the ServiceCall renderer.
   */
  startNotificationsAutoRefresh();

  const refreshRecordingsButton = document.getElementById(
    "refreshRecordingsButton",
  );

  const recordingsMessage = document.getElementById("recordingsMessage");

  const recordingsList = document.getElementById("recordingsList");

  window.addEventListener(
    "scroll",
    () => {
      if (scheduleMeetingPeopleResults) {
        scheduleMeetingPeopleResults.style.display = "none";
      }
    },
    true,
  );

  document.addEventListener("click", (event) => {
    if (!scheduleMeetingPeopleSearch || !scheduleMeetingPeopleResults) {
      return;
    }

    const clickedSearch = scheduleMeetingPeopleSearch.contains(event.target);

    const clickedResults = scheduleMeetingPeopleResults.contains(event.target);

    /*
     * Click anywhere outside the
     * People search/results → close dropdown.
     */
    if (!clickedSearch && !clickedResults) {
      scheduleMeetingPeopleResults.style.display = "none";
    }
  });

  async function loadRecordingHistory() {
    if (!recordingsMessage || !recordingsList) {
      return;
    }

    recordingsMessage.textContent = "Loading recordings...";

    recordingsList.innerHTML = "";

    try {
      const result = await window.serviceCall.getRecordingHistory();

      if (!result || result.success !== true) {
        recordingsMessage.textContent =
          result && result.message
            ? result.message
            : "Unable to load recordings.";

        return;
      }

      const recordings = Array.isArray(result.recordings)
        ? result.recordings
        : [];

      if (recordings.length === 0) {
        recordingsMessage.textContent = "No recordings found.";

        return;
      }

      recordingsMessage.textContent =
        recordings.length +
        (recordings.length === 1 ? " recording" : " recordings");

      recordings.forEach((recording) => {
        const card = document.createElement("div");

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

        const title = document.createElement("div");

        title.style.cssText = `
                    font-weight:600;
                    font-size:15px;
                    margin-bottom:8px;
                `;

        title.textContent = recording.number || "ServiceCall Recording";

        card.appendChild(title);

        /*
         * ---------------------------------
         * DETAILS
         * ---------------------------------
         */

        const details = document.createElement("div");

        details.style.cssText = `
                    font-size:13px;
                    line-height:1.7;
                    color:#52635f;
                `;

        const participants = Array.isArray(recording.participants)
          ? recording.participants.join(", ")
          : "";

        details.textContent =
          "Call: " +
          (recording.call_number || "-") +
          "\n" +
          "Participants: " +
          (participants || "-") +
          "\n" +
          "Status: " +
          (recording.status || "-") +
          "\n" +
          "Format: " +
          (recording.format || "-") +
          "\n" +
          "Started: " +
          (recording.started_at || "-");

        details.style.whiteSpace = "pre-line";

        card.appendChild(details);

        /*
         * ---------------------------------
         * AVAILABLE → DOWNLOAD
         * ---------------------------------
         */

        if (recording.status === "available") {
          const downloadButton = document.createElement("button");

          downloadButton.type = "button";

          downloadButton.className = "button";

          downloadButton.textContent = "Download";

          downloadButton.style.marginTop = "12px";

          downloadButton.addEventListener(
            "click",

            async () => {
              downloadButton.disabled = true;

              downloadButton.textContent = "Downloading...";

              try {
                const downloadResult =
                  await window.serviceCall.downloadRecording(
                    recording.recording_sys_id,
                  );

                if (downloadResult && downloadResult.success) {
                  recordingsMessage.textContent =
                    "Recording downloaded successfully.";
                } else if (
                  downloadResult &&
                  downloadResult.code === "DOWNLOAD_CANCELLED"
                ) {
                  recordingsMessage.textContent = "Download cancelled.";
                } else {
                  recordingsMessage.textContent =
                    downloadResult && downloadResult.message
                      ? downloadResult.message
                      : "Unable to download recording.";
                }
              } catch (error) {
                recordingsMessage.textContent =
                  error.message || "Unable to download recording.";
              } finally {
                downloadButton.disabled = false;

                downloadButton.textContent = "Download";
              }
            },
          );

          card.appendChild(downloadButton);
        }

        /*
         * ---------------------------------
         * PROCESSING
         * ---------------------------------
         */

        if (recording.status === "processing") {
          const state = document.createElement("div");

          state.style.cssText = `
                        margin-top:12px;
                        font-size:13px;
                        color:#667773;
                    `;

          state.textContent = "Recording is being processed...";

          card.appendChild(state);
        }

        /*
         * ---------------------------------
         * EXPIRED
         * ---------------------------------
         */

        if (recording.status === "expired") {
          const state = document.createElement("div");

          state.style.cssText = `
                        margin-top:12px;
                        font-size:13px;
                        color:#667773;
                    `;

          state.textContent = "Recording expired";

          card.appendChild(state);
        }

        recordingsList.appendChild(card);
      });
    } catch (error) {
      console.error("Unable to load recording history:", error);

      recordingsMessage.textContent =
        error.message || "Unable to load recordings.";
    }
  }

  if (refreshRecordingsButton) {
    refreshRecordingsButton.addEventListener("click", loadRecordingHistory);
  }

  function updatePresenceDisplay(status) {
    const normalized = String(status || "available")
      .trim()
      .toLowerCase();

    let label = "Available";

    let cssClass = "available";

    if (normalized === "busy") {
      label = "Busy";
      cssClass = "busy";
    } else if (normalized === "away") {
      label = "Away";
      cssClass = "away";
    } else if (normalized === "out of office") {
      label = "Out of Office";
      cssClass = "out-of-office";
    } else if (normalized === "in call") {
      label = "In a Call";
      cssClass = "in-call";
    } else if (normalized === "offline") {
      label = "Offline";
      cssClass = "offline";
    }

    if (presenceText) {
      presenceText.textContent = label;
    }

    if (presenceDot) {
      presenceDot.className = "presence-dot " + cssClass;
    }
  }

  async function loadMyPresence() {
    try {
      const result = await window.serviceCall.getMyPresence();

      if (!result || result.success !== true) {
        console.warn("Unable to load presence:", result);

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
        result.effective_status || result.presence_status || "available",
      );

      /*
       * Keep the saved OOF reason ready
       * for editing/reuse.
       */
      if (oofReasonInput) {
        oofReasonInput.value = result.oof_reason || "";
      }

      console.log("ServiceCall presence loaded:", result);
    } catch (error) {
      console.error("Unable to load ServiceCall presence:", error);
    }
  }

  /* =====================================================
   PRESENCE MENU
===================================================== */

  if (presenceButton && presenceMenu) {
    presenceButton.addEventListener("click", (event) => {
      event.stopPropagation();

      const isOpen = presenceMenu.style.display === "block";

      presenceMenu.style.display = isOpen ? "none" : "block";
    });
  }

  /*
   * Available / Busy / Away / OOF
   */

  presenceOptions.forEach((option) => {
    option.addEventListener(
      "click",

      async (event) => {
        event.stopPropagation();

        const status = String(option.dataset.presence || "")
          .trim()
          .toLowerCase();

        if (!status) {
          return;
        }

        /*
         * OOF needs a reason before
         * being saved.
         */

        if (status === "out of office") {
          if (oofReasonPanel) {
            oofReasonPanel.style.display = "block";
          }

          if (oofReasonInput) {
            oofReasonInput.focus();
          }

          return;
        }

        /*
         * Available / Busy / Away
         */

        try {
          const result = await window.serviceCall.updatePresence(status, "");

          if (!result || result.success !== true) {
            throw new Error(
              result && result.message
                ? result.message
                : "Unable to update presence.",
            );
          }

          updatePresenceDisplay(result.effective_status || status);

          if (oofReasonPanel) {
            oofReasonPanel.style.display = "none";
          }

          if (presenceMenu) {
            presenceMenu.style.display = "none";
          }
        } catch (error) {
          console.error("Unable to update presence:", error);
        }
      },
    );
  });

  /* =====================================================
   OUT OF OFFICE
===================================================== */

  if (saveOofButton) {
    saveOofButton.addEventListener(
      "click",

      async (event) => {
        event.stopPropagation();

        const reason = String(
          oofReasonInput ? oofReasonInput.value : "",
        ).trim();

        if (!reason) {
          if (oofReasonInput) {
            oofReasonInput.focus();
          }

          return;
        }

        try {
          const result = await window.serviceCall.updatePresence(
            "out of office",
            reason,
          );

          if (!result || result.success !== true) {
            throw new Error(
              result && result.message
                ? result.message
                : "Unable to update Out of Office.",
            );
          }

          updatePresenceDisplay(result.effective_status || "out of office");

          if (oofReasonPanel) {
            oofReasonPanel.style.display = "none";
          }

          if (presenceMenu) {
            presenceMenu.style.display = "none";
          }
        } catch (error) {
          console.error("Unable to update Out of Office:", error);
        }
      },
    );
  }

  if (cancelOofButton) {
    cancelOofButton.addEventListener("click", (event) => {
      event.stopPropagation();

      if (oofReasonPanel) {
        oofReasonPanel.style.display = "none";
      }
    });
  }

  document.addEventListener("click", (event) => {
    if (
      presenceMenu &&
      presenceButton &&
      !presenceMenu.contains(event.target) &&
      !presenceButton.contains(event.target)
    ) {
      presenceMenu.style.display = "none";

      if (oofReasonPanel) {
        oofReasonPanel.style.display = "none";
      }
    }
  });

  if (resetPresenceButton) {
    resetPresenceButton.addEventListener("click", async () => {
      try {
        resetPresenceButton.disabled = true;

        const result = await window.serviceCall.updatePresence("available", "");

        if (!result || result.success !== true) {
          console.error("Unable to reset presence:", result);

          return;
        }

        updatePresenceDisplay(
          result.effective_status || result.presence_status || "available",
        );

        if (oofReasonPanel) {
          oofReasonPanel.style.display = "none";
        }

        if (presenceMenu) {
          presenceMenu.style.display = "none";
        }

        console.log("ServiceCall presence reset:", result);
      } catch (error) {
        console.error("Unable to reset ServiceCall presence:", error);
      } finally {
        resetPresenceButton.disabled = false;
      }
    });
  }

  /* =====================================================
   MEETING ACTION SOUNDS
===================================================== */

  function playMeetingActionSound(soundFile) {
    try {
      if (!soundFile) {
        return;
      }

      const audio = new Audio(`assets/sounds/${soundFile}`);

      audio.volume = 0.7;

      audio.play().catch((error) => {
        console.error("Unable to play meeting action sound:", error);
      });
    } catch (error) {
      console.error("Meeting action sound failed:", error);
    }
  }

  function playMeetingJoinStartSound() {
    playMeetingActionSound("joinorstart.mp3");
  }

  function playMeetingLeaveEndSound() {
    playMeetingActionSound("end-meet.mp3");
  }

  async function loadCurrentAccount() {
    const nameElement = document.getElementById("currentAccountName");

    const usernameElement = document.getElementById("currentAccountUsername");

    const serviceCallIdElement = document.getElementById(
      "currentAccountServiceCallId",
    );

    const accountMessage = document.getElementById("accountMessage");

    try {
      const result = await window.serviceCall.getCurrentAccount();

      console.log("Current ServiceCall account:", result);

      if (!result?.success || !result?.user) {
        throw new Error("Current ServiceCall account could not be loaded.");
      }

      const user = result.user;

      const authorization = result.authorization || {};

      /*
       * NAME
       */

      if (nameElement) {
        nameElement.textContent = user.name || "ServiceCall User";
      }

      /*
       * USERNAME
       */

      if (usernameElement) {
        usernameElement.textContent = user.user_name
          ? `@${user.user_name}`
          : "";
      }

      /*
       * SERVICECALL ID
       */

      if (serviceCallIdElement) {
        serviceCallIdElement.textContent = user.servicecall_id
          ? `ServiceCall ID: ${user.servicecall_id}`
          : "";
      }

      /*
       * ACCESS LEVEL
       */

      if (accountMessage) {
        if (authorization.is_servicecall_admin === true) {
          accountMessage.textContent = "ServiceCall Administrator";
        } else {
          accountMessage.textContent = "ServiceCall User";
        }
      }
    } catch (error) {
      console.error("Failed to load current account:", error);

      if (accountMessage) {
        accountMessage.textContent = "Unable to load account information.";
      }
    }
  }

  /* =========================================
   CREATE GROUP MODAL
========================================= */

  function openChatCreateGroupModal() {
    if (!chatCreateGroupModal) {
      return;
    }

    chatCreateGroupModal.classList.add("open");

    chatCreateGroupModal.setAttribute("aria-hidden", "false");

    if (chatCreateGroupMessage) {
      chatCreateGroupMessage.textContent = "";
    }

    setTimeout(() => {
      if (chatCreateGroupNameInput) {
        chatCreateGroupNameInput.focus();
      }
    }, 50);
  }

  function closeChatCreateGroupModal() {
    if (!chatCreateGroupModal) {
      return;
    }

    chatCreateGroupModal.classList.remove("open");

    chatCreateGroupModal.setAttribute("aria-hidden", "true");
  }

  /* =========================================
   CREATE GROUP MODAL EVENTS
========================================= */

  if (chatNewGroupButton) {
    chatNewGroupButton.addEventListener("click", () => {
      openChatCreateGroupModal();
    });
  }

  if (chatCreateGroupCloseButton) {
    chatCreateGroupCloseButton.addEventListener("click", () => {
      closeChatCreateGroupModal();
    });
  }

  if (chatCreateGroupCancelButton) {
    chatCreateGroupCancelButton.addEventListener("click", () => {
      closeChatCreateGroupModal();
    });
  }

  if (chatCreateGroupBackdrop) {
    chatCreateGroupBackdrop.addEventListener("click", () => {
      closeChatCreateGroupModal();
    });
  }

  document.addEventListener("keydown", (event) => {
    if (
      event.key === "Escape" &&
      chatCreateGroupModal &&
      chatCreateGroupModal.classList.contains("open")
    ) {
      closeChatCreateGroupModal();
    }
  });

  if (chatCreateGroupSubmitButton) {
    chatCreateGroupSubmitButton.addEventListener(
      "click",

      async () => {
        const title = chatCreateGroupNameInput
          ? chatCreateGroupNameInput.value.trim()
          : "";

        const participantSysIds = Array.from(
          chatCreateGroupSelectedUsers.keys(),
        );

        /* -------------------------
               VALIDATION
            ------------------------- */

        if (!title) {
          if (chatCreateGroupMessage) {
            chatCreateGroupMessage.textContent = "Enter a group name.";
          }

          return;
        }

        if (participantSysIds.length === 0) {
          if (chatCreateGroupMessage) {
            chatCreateGroupMessage.textContent = "Select at least one person.";
          }

          return;
        }

        /*
         * Lock immediately.
         * Prevents duplicate groups from
         * repeated clicks.
         */
        chatCreateGroupSubmitButton.disabled = true;

        const originalText = chatCreateGroupSubmitButton.textContent;

        chatCreateGroupSubmitButton.textContent = "Creating...";

        if (chatCreateGroupMessage) {
          chatCreateGroupMessage.textContent = "";
        }

        try {
          const result = await window.serviceCall.createGroup(
            title,
            participantSysIds,
          );

          console.log("ServiceCall create group:", result);

          if (!result || result.success !== true) {
            throw new Error(
              result && result.message
                ? result.message
                : "Unable to create group.",
            );
          }

          /* -------------------------
                   SUCCESS
                ------------------------- */

          closeChatCreateGroupModal();

          /*
           * Clear local group state.
           */
          chatCreateGroupSelectedUsers.clear();

          if (chatCreateGroupNameInput) {
            chatCreateGroupNameInput.value = "";
          }

          if (chatCreateGroupPeopleSearch) {
            chatCreateGroupPeopleSearch.value = "";
          }

          if (chatCreateGroupPeopleResults) {
            chatCreateGroupPeopleResults.innerHTML = "";

            chatCreateGroupPeopleResults.style.display = "none";
          }

          renderChatCreateGroupSelectedPeople();

          /*
           * Reload sidebar.
           *
           * The newly-created group should
           * now come from /conversations.
           */
          await loadChatConversations();
        } catch (error) {
          console.error("Create ServiceCall group failed:", error);

          if (chatCreateGroupMessage) {
            chatCreateGroupMessage.textContent =
              error && error.message
                ? error.message
                : "Unable to create group.";
          }
        } finally {
          chatCreateGroupSubmitButton.textContent = originalText;

          /*
           * Re-evaluate instead of blindly
           * enabling the button.
           */
          updateChatCreateGroupSubmitState();
        }
      },
    );
  }

  /* =====================================================
   CREATE GROUP VALIDATION
===================================================== */

  function updateChatCreateGroupSubmitState() {
    if (!chatCreateGroupSubmitButton) {
      return;
    }

    const groupName = chatCreateGroupNameInput
      ? chatCreateGroupNameInput.value.trim()
      : "";

    const hasPeople = chatCreateGroupSelectedUsers.size > 0;

    chatCreateGroupSubmitButton.disabled = !groupName || !hasPeople;
  }

  /* =====================================================
   RENDER SELECTED GROUP USERS
===================================================== */

  function renderChatCreateGroupSelectedPeople() {
    if (!chatCreateGroupSelectedPeople) {
      return;
    }

    chatCreateGroupSelectedPeople.innerHTML = "";

    /* -------------------------
       EMPTY
    ------------------------- */

    if (chatCreateGroupSelectedUsers.size === 0) {
      const empty = document.createElement("div");

      empty.className = "chat-create-group-no-people";

      empty.textContent = "No people selected yet.";

      chatCreateGroupSelectedPeople.appendChild(empty);

      updateChatCreateGroupSubmitState();

      return;
    }

    /* -------------------------
       CHIPS
    ------------------------- */

    chatCreateGroupSelectedUsers.forEach((user) => {
      const chip = document.createElement("div");

      chip.className = "chat-create-group-chip";

      const name = document.createElement("span");

      name.textContent = user.name || user.user_name || "Unknown User";

      const removeButton = document.createElement("button");

      removeButton.type = "button";

      removeButton.className = "chat-create-group-chip-remove";

      removeButton.textContent = "×";

      removeButton.title = "Remove";

      removeButton.addEventListener(
        "click",

        () => {
          const sysId = String(user.sys_id || "");

          chatCreateGroupSelectedUsers.delete(sysId);

          renderChatCreateGroupSelectedPeople();

          /*
           * Refresh visible search
           * results so its checkbox
           * immediately becomes
           * unchecked.
           */
          searchChatCreateGroupPeople();
        },
      );

      chip.appendChild(name);

      chip.appendChild(removeButton);

      chatCreateGroupSelectedPeople.appendChild(chip);
    });

    updateChatCreateGroupSubmitState();
  }

  function renderPendingChatAttachment() {
    const preview = document.getElementById("chatAttachmentPreview");

    if (!preview) {
      return;
    }

    if (
      !Array.isArray(pendingChatAttachments) ||
      pendingChatAttachments.length === 0
    ) {
      preview.hidden = true;
      preview.replaceChildren();
      return;
    }

    preview.hidden = false;
    preview.replaceChildren();

    pendingChatAttachments.forEach((attachment) => {
      const card = document.createElement("div");

      card.className = "chat-attachment-preview-card";

      /* -------------------------
         ICON
      ------------------------- */

      const icon = document.createElement("span");

      icon.className = "chat-attachment-preview-icon";

      icon.textContent = "📄";

      /* -------------------------
         FILE INFORMATION
      ------------------------- */

      const info = document.createElement("div");

      info.className = "chat-attachment-preview-info";

      const name = document.createElement("div");

      name.className = "chat-attachment-preview-name";

      name.textContent = attachment.fileName;

      const size = document.createElement("div");

      size.className = "chat-attachment-preview-size";

      const isUploading = attachment.status === "uploading";

      const isFailed = attachment.status === "failed";

      if (isUploading) {
        size.textContent =
          formatChatFileSize(attachment.fileSize) + " · Uploading...";
      } else if (isFailed) {
        size.textContent =
          formatChatFileSize(attachment.fileSize) + " · Upload failed";
      } else {
        size.textContent = formatChatFileSize(attachment.fileSize);
      }

      info.appendChild(name);
      info.appendChild(size);

      /* -------------------------
         REMOVE BUTTON
      ------------------------- */

      const removeButton = document.createElement("button");

      removeButton.type = "button";

      removeButton.className = "chat-attachment-preview-remove";

      removeButton.title = "Remove attachment";

      removeButton.textContent = "×";

      /* -------------------------
         SECURE CANCEL
      ------------------------- */

      removeButton.addEventListener("click", async () => {
        const attachmentSysId = String(attachment.sysId || "").trim();

        /*
         * FAILED UPLOAD
         *
         * There is no usable server attachment left,
         * so remove only the local failed placeholder.
         */
        if (attachment.status === "failed") {
          pendingChatAttachments = pendingChatAttachments.filter(
            (item) => item !== attachment,
          );

          renderPendingChatAttachment();

          if (pendingChatAttachments.length === 0 && chatAttachmentInput) {
            chatAttachmentInput.value = "";
          }

          if (chatSendButton) {
            const hasCurrentText = !!String(
              chatMessageInput?.value || "",
            ).trim();

            const hasReadyAttachments = pendingChatAttachments.some(
              (item) =>
                item &&
                item.status !== "failed" &&
                !!String(item.sysId || "").trim(),
            );

            const uploadsStillRunning = chatAttachmentUploadsInProgress > 0;

            chatSendButton.disabled =
              uploadsStillRunning || (!hasCurrentText && !hasReadyAttachments);
          }

          return;
        }

        if (!attachmentSysId) {
          return;
        }

        /*
         * Prevent duplicate cancellation.
         */
        removeButton.disabled = true;

        try {
          const result =
            await window.serviceCall.cancelChatAttachment(attachmentSysId);

          console.log("Cancel attachment result:", result);

          if (!result || result.success !== true) {
            throw new Error(result?.message || "Unable to cancel attachment.");
          }

          /*
           * Remove ONLY this attachment
           * from the pending array.
           */
          pendingChatAttachments = pendingChatAttachments.filter(
            (item) => item.sysId !== attachmentSysId,
          );

          renderPendingChatAttachment();

          /*
           * Reset picker only when there
           * are no pending files.
           */
          if (pendingChatAttachments.length === 0 && chatAttachmentInput) {
            chatAttachmentInput.value = "";
          }

          /* -------------------------
               SEND BUTTON STATE
            ------------------------- */

          if (chatSendButton) {
            const hasCurrentText = !!String(
              chatMessageInput?.value || "",
            ).trim();

            const hasCurrentAttachments = pendingChatAttachments.some(
              (attachment) =>
                attachment &&
                attachment.status !== "failed" &&
                !!String(attachment.sysId || "").trim(),
            );

            chatSendButton.disabled = !hasCurrentText && !hasCurrentAttachments;
          }
        } catch (error) {
          console.error("Unable to cancel attachment:", error);

          /*
           * Server cancellation failed,
           * therefore keep the file.
           */
          removeButton.disabled = false;
        }
      });

      /* -------------------------
         BUILD CARD
      ------------------------- */

      card.appendChild(icon);
      card.appendChild(info);
      card.appendChild(removeButton);

      preview.appendChild(card);
    });
  }

  function formatChatFileSize(bytes) {
    const size = Number(bytes || 0);

    if (size < 1024) {
      return size + " B";
    }

    if (size < 1024 * 1024) {
      return (size / 1024).toFixed(1) + " KB";
    }

    return (size / (1024 * 1024)).toFixed(1) + " MB";
  }

  /* =====================================================
   SEARCH USERS FOR GROUP
===================================================== */

  async function searchChatCreateGroupPeople() {
    if (!chatCreateGroupPeopleSearch || !chatCreateGroupPeopleResults) {
      return;
    }

    const searchText = chatCreateGroupPeopleSearch.value.trim();

    /* -------------------------
       TOO SHORT
    ------------------------- */

    if (searchText.length < 2) {
      chatCreateGroupPeopleResults.innerHTML = "";

      chatCreateGroupPeopleResults.style.display = "none";

      return;
    }

    try {
      const result = await window.serviceCall.searchUsers(searchText);

      /*
       * Ignore stale search response.
       */
      if (chatCreateGroupPeopleSearch.value.trim() !== searchText) {
        return;
      }

      if (!result || result.success !== true) {
        throw new Error(
          result && result.message ? result.message : "Unable to search users.",
        );
      }

      const users = Array.isArray(result.users) ? result.users : [];

      chatCreateGroupPeopleResults.innerHTML = "";

      /* -------------------------
           NO RESULTS
        ------------------------- */

      if (users.length === 0) {
        const empty = document.createElement("div");

        empty.textContent = "No users found.";

        empty.style.padding = "14px";

        empty.style.color = "#71827d";

        empty.style.fontSize = "12px";

        chatCreateGroupPeopleResults.appendChild(empty);

        chatCreateGroupPeopleResults.style.display = "block";

        return;
      }

      /* -------------------------
           RESULTS
        ------------------------- */

      users.forEach((user) => {
        const sysId = String(user.sys_id || "").trim();

        if (!sysId) {
          return;
        }

        const selected = chatCreateGroupSelectedUsers.has(sysId);

        const row = document.createElement("button");

        row.type = "button";

        row.className = selected
          ? "chat-create-group-result selected"
          : "chat-create-group-result";

        /* -------------------------
                   CHECKBOX
                ------------------------- */

        const checkbox = document.createElement("input");

        checkbox.type = "checkbox";

        checkbox.className = "chat-create-group-result-checkbox";

        checkbox.checked = selected;

        checkbox.addEventListener("click", (event) => {
          /*
           * Do not let the click bubble to
           * the row, otherwise selection
           * would toggle twice.
           */
          event.stopPropagation();

          if (chatCreateGroupSelectedUsers.has(sysId)) {
            chatCreateGroupSelectedUsers.delete(sysId);
          } else {
            chatCreateGroupSelectedUsers.set(sysId, user);
          }

          const nowSelected = chatCreateGroupSelectedUsers.has(sysId);

          checkbox.checked = nowSelected;

          row.classList.toggle("selected", nowSelected);

          renderChatCreateGroupSelectedPeople();
        });

        /*
         * Row handles selection.
         * Prevent checkbox itself
         * from producing a second click.
         */
        checkbox.addEventListener(
          "click",

          (event) => {
            event.preventDefault();
          },
        );

        /* -------------------------
                   NAME
                ------------------------- */

        const name = document.createElement("div");

        name.className = "chat-create-group-result-name";

        name.textContent = user.name || user.user_name || "Unknown User";

        row.appendChild(checkbox);

        row.appendChild(name);

        /* -------------------------
                   MULTI-SELECT
                ------------------------- */

        row.addEventListener(
          "click",

          () => {
            if (chatCreateGroupSelectedUsers.has(sysId)) {
              chatCreateGroupSelectedUsers.delete(sysId);
            } else {
              chatCreateGroupSelectedUsers.set(sysId, user);
            }

            renderChatCreateGroupSelectedPeople();

            /*
             * Update this result immediately.
             */
            const nowSelected = chatCreateGroupSelectedUsers.has(sysId);

            checkbox.checked = nowSelected;

            row.classList.toggle("selected", nowSelected);
          },
        );

        chatCreateGroupPeopleResults.appendChild(row);
      });

      chatCreateGroupPeopleResults.style.display = "block";
    } catch (error) {
      console.error("Create group user search failed:", error);

      chatCreateGroupPeopleResults.innerHTML = "";

      const errorResult = document.createElement("div");

      errorResult.textContent =
        error && error.message ? error.message : "Unable to search users.";

      errorResult.style.padding = "14px";

      errorResult.style.color = "#a33f3f";

      errorResult.style.fontSize = "12px";

      chatCreateGroupPeopleResults.appendChild(errorResult);

      chatCreateGroupPeopleResults.style.display = "block";
    }
  }

  /* =====================================================
   CREATE GROUP SEARCH EVENTS
===================================================== */

  if (chatCreateGroupPeopleSearch) {
    chatCreateGroupPeopleSearch.addEventListener(
      "input",

      () => {
        if (chatCreateGroupSearchTimer) {
          clearTimeout(chatCreateGroupSearchTimer);
        }

        const searchText = chatCreateGroupPeopleSearch.value.trim();

        if (searchText.length < 2) {
          if (chatCreateGroupPeopleResults) {
            chatCreateGroupPeopleResults.innerHTML = "";

            chatCreateGroupPeopleResults.style.display = "none";
          }

          return;
        }

        chatCreateGroupSearchTimer = setTimeout(() => {
          searchChatCreateGroupPeople();
        }, 300);
      },
    );
  }

  if (chatCreateGroupNameInput) {
    chatCreateGroupNameInput.addEventListener(
      "input",

      () => {
        updateChatCreateGroupSubmitState();
      },
    );
  }

  /* =====================================================
   SIGN OUT
===================================================== */

  if (signOutButton) {
    signOutButton.addEventListener("click", async () => {
      /*
       * Prevent duplicate sign-out requests.
       */
      signOutButton.disabled = true;

      const originalText = signOutButton.textContent;

      signOutButton.textContent = "Signing out...";

      const signOutMessage = document.getElementById("accountMessage");

      try {
        const result = await window.serviceCall.signOut();

        console.log("ServiceCall sign out result:", result);

        if (!result || result.success !== true) {
          throw new Error(result?.message || "Unable to sign out.");
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
        console.error("ServiceCall sign out failed:", error);

        if (accountMessage) {
          accountMessage.textContent = error?.message || "Unable to sign out.";
        }

        signOutButton.disabled = false;

        signOutButton.textContent = originalText;
      }
    });
  }

  await loadCurrentAccount();
});


preload.js

const { contextBridge, ipcRenderer } = require("electron");

contextBridge.exposeInMainWorld("serviceCall", {
  /* -------------------------
           INSTANCE / CONNECTION
        ------------------------- */

  saveInstance: (instanceUrl) =>
    ipcRenderer.invoke("servicecall-save-instance", instanceUrl),

  getInstance: () => ipcRenderer.invoke("servicecall-get-instance"),

  getConnectionStatus: () =>
    ipcRenderer.invoke("servicecall-get-connection-status"),

  /* -------------------------
           AUTHENTICATION
        ------------------------- */

  startLogin: () => ipcRenderer.invoke("servicecall-start-login"),

  getGroupDetails: (conversationSysId) =>
    ipcRenderer.invoke("servicecall-get-group-details", conversationSysId),

  renameGroup: (conversationSysId, title) =>
    ipcRenderer.invoke("servicecall-rename-group", {
      conversationSysId,
      title,
    }),

  addGroupMembers: (conversationSysId, participantSysIds) =>
    ipcRenderer.invoke("servicecall-add-group-members", {
      conversationSysId,
      participantSysIds,
    }),

  setGroupMemberRole: (conversationSysId, memberUserSysId, role) =>
    ipcRenderer.invoke("servicecall-set-group-member-role", {
      conversationSysId,
      memberUserSysId,
      role,
    }),

  removeGroupMember: (conversationSysId, memberUserSysId) =>
    ipcRenderer.invoke("servicecall-remove-group-member", {
      conversationSysId,
      memberUserSysId,
    }),

  leaveGroup: (conversationSysId) =>
    ipcRenderer.invoke("servicecall-leave-group", {
      conversationSysId,
    }),

  onAuthStatus: (callback) => {
    ipcRenderer.on("servicecall-auth-status", (event, data) => {
      callback(data);
    });
  },

  /* -------------------------
           ACTIVE CALL
        ------------------------- */

  openActiveCall: () => ipcRenderer.invoke("servicecall-open-active-call"),

  /* -------------------------
           CALL ACTIONS
        ------------------------- */

  acceptCall: (callSysId) =>
    ipcRenderer.invoke("servicecall-accept-call", callSysId),

  declineCall: (callSysId) =>
    ipcRenderer.invoke("servicecall-decline-call", callSysId),

  cancelCall: (callSysId) =>
    ipcRenderer.invoke("servicecall-cancel-call", callSysId),

  endCall: (callSysId) => ipcRenderer.invoke("servicecall-end-call", callSysId),

  /* -------------------------
   CHAT
------------------------- */

  getConversations: () => ipcRenderer.invoke("servicecall-get-conversations"),

  getMessages: (
    conversationSysId,
    afterMessageSysId = "",
    beforeMessageSysId = "",
    limit = 50,
    editedAfter = "",
  ) =>
    ipcRenderer.invoke("servicecall-get-messages", {
      conversationSysId: conversationSysId,

      afterMessageSysId: afterMessageSysId,

      beforeMessageSysId: beforeMessageSysId,

      limit: limit,

      editedAfter: editedAfter,
    }),

  prepareChatAttachment: (payload) =>
    ipcRenderer.invoke("servicecall-prepare-chat-attachment", payload),

  uploadChatAttachmentBinary: (attachmentSysId, fileBytes) =>
    ipcRenderer.invoke("servicecall-upload-chat-attachment-binary", {
      attachmentSysId,
      fileBytes,
    }),

  sendAttachmentMessage: (attachmentSysId) =>
    ipcRenderer.invoke("servicecall-send-attachment-message", {
      attachmentSysId,
    }),

  downloadChatAttachment: (attachmentSysId) =>
    ipcRenderer.invoke("servicecall-download-chat-attachment", {
      attachmentSysId,
    }),

  getReactionUpdates: (conversationSysId, afterCheckpoint = "") =>
    ipcRenderer.invoke("servicecall-get-reaction-updates", {
      conversationSysId: conversationSysId,

      afterCheckpoint: afterCheckpoint,
    }),

  setMessageReaction: (messageSysId, reaction) =>
    ipcRenderer.invoke("servicecall-set-message-reaction", {
      messageSysId: messageSysId,

      reaction: reaction,
    }),

  openChatAttachment: (attachmentSysId, fileName) =>
    ipcRenderer.invoke("servicecall-open-chat-attachment", {
      attachmentSysId,
      fileName,
    }),

  cancelChatAttachment: (attachmentSysId) =>
    ipcRenderer.invoke("servicecall-cancel-chat-attachment", {
      attachmentSysId,
    }),

  sendChatContent: (
    conversationSysId,
    message,
    attachmentSysIds,
    replyToMessageSysId = "",
  ) =>
    ipcRenderer.invoke("servicecall-send-chat-content", {
      conversationSysId,
      message,
      attachmentSysIds,
      replyToMessageSysId,
    }),

  markConversationRead: (conversationSysId) =>
    ipcRenderer.invoke("servicecall-mark-conversation-read", conversationSysId),

  createGroup: (title, participantSysIds) =>
    ipcRenderer.invoke("servicecall-create-group", {
      title,
      participantSysIds,
    }),

  setMessageReaction: (messageSysId, reaction) =>
    ipcRenderer.invoke("servicecall-set-message-reaction", {
      messageSysId: messageSysId,

      reaction: reaction,
    }),

  sendMessage: (
    recipientSysId,
    conversationSysId,
    message,
    replyToMessageSysId = "",
  ) =>
    ipcRenderer.invoke("servicecall-send-message", {
      recipientSysId: recipientSysId || "",

      conversationSysId: conversationSysId || "",

      message: message,

      replyToMessageSysId: replyToMessageSysId || "",
    }),

  deleteMessages: (conversationSysId, messageSysIds, mode) =>
    ipcRenderer.invoke("servicecall-delete-messages", {
      conversationSysId,
      messageSysIds,
      mode,
    }),

  forwardMessages: (messageSysIds, destinationConversationIds) =>
    ipcRenderer.invoke("servicecall-forward-messages", {
      messageSysIds,
      destinationConversationIds,
    }),

  editMessage: (conversationSysId, messageSysId, message) =>
    ipcRenderer.invoke("servicecall-edit-message", {
      conversationSysId,
      messageSysId,
      message,
    }),

  leaveCall: (callSysId) =>
    ipcRenderer.invoke("servicecall-leave-call", callSysId),

  getCallStatus: (callSysId) =>
    ipcRenderer.invoke("servicecall-get-call-status", callSysId),

  checkAccess: () => ipcRenderer.invoke("servicecall-check-access"),

  /* -------------------------
           PARTICIPANTS
        ------------------------- */

  searchUsers: (searchText) =>
    ipcRenderer.invoke("servicecall-search-users", searchText),

  inviteParticipant: (callSysId, userSysId) =>
    ipcRenderer.invoke("servicecall-invite-participant", callSysId, userSysId),

  /* -------------------------
           DYNAMIC MEDIA CREDENTIALS
        ------------------------- */

  getMediaCredentials: (callSysId) =>
    ipcRenderer.invoke("servicecall-get-media-credentials", callSysId),

  /* -------------------------
   SCREEN SHARING
------------------------- */

  getScreenSources: () => ipcRenderer.invoke("servicecall-get-screen-sources"),

  setCallWindowLayout: (layout) =>
    ipcRenderer.invoke("servicecall-set-call-window-layout", layout),

  /* -------------------------
   RECORDING
------------------------- */

  startRecording: (callSysId) =>
    ipcRenderer.invoke("servicecall-start-recording", callSysId),

  startCall: (targetUserSysId) => {
    return ipcRenderer.invoke("servicecall-start-call", targetUserSysId);
  },

  updatePresence: (status, oofReason = "") => {
    return ipcRenderer.invoke("servicecall-update-presence", {
      status: status,
      oofReason: oofReason,
    });
  },

  checkAccess: () => ipcRenderer.invoke("servicecall-check-access"),

  onAuthorizationStatus: (callback) => {
    const handler = (event, data) => {
      callback(data);
    };

    ipcRenderer.on("servicecall-authorization-status", handler);

    return () => {
      ipcRenderer.removeListener("servicecall-authorization-status", handler);
    };
  },

  getMyPresence: () => {
    return ipcRenderer.invoke("servicecall-get-my-presence");
  },

  signOut: () => ipcRenderer.invoke("servicecall-sign-out"),

  getSavedAccounts: () => ipcRenderer.invoke("servicecall-get-saved-accounts"),

  getCurrentAccount: () =>
    ipcRenderer.invoke("servicecall-get-current-account"),

  activateSavedAccount: (accountKey) =>
    ipcRenderer.invoke("servicecall-activate-saved-account", accountKey),

  removeSavedAccount: (accountKey) =>
    ipcRenderer.invoke("servicecall-remove-saved-account", accountKey),

  finishRecording: (recordingSysId) =>
    ipcRenderer.invoke("servicecall-finish-recording", recordingSysId),

  uploadRecording: (recordingSysId, fileData, fileName, format) =>
    ipcRenderer.invoke(
      "servicecall-upload-recording",
      recordingSysId,
      fileData,
      fileName,
      format,
    ),

  finalizeVoiceRecording: (recordingSysId, webmData) =>
    ipcRenderer.invoke(
      "servicecall-finalize-voice-recording",
      recordingSysId,
      webmData,
    ),

  finalizeScreenRecording: (recordingSysId, webmData) =>
    ipcRenderer.invoke(
      "servicecall-finalize-screen-recording",
      recordingSysId,
      webmData,
    ),

  getRecordingHistory: () =>
    ipcRenderer.invoke("servicecall-get-recording-history"),

  downloadRecording: (recordingSysId) =>
    ipcRenderer.invoke("servicecall-download-recording", recordingSysId),

  completeRecording: (recordingSysId, attachmentSysId, format) =>
    ipcRenderer.invoke(
      "servicecall-complete-recording",
      recordingSysId,
      attachmentSysId,
      format,
    ),

  getMyMeetings: (page = 1, search = "", status = "") =>
    ipcRenderer.invoke("servicecall-get-my-meetings", {
      page: page,
      search: search,
      status: status,
    }),

  getMeetingDetails: (meetingSysId) =>
    ipcRenderer.invoke("servicecall-get-meeting-details", meetingSysId),

  createMeeting: (meetingData) =>
    ipcRenderer.invoke("servicecall-create-meeting", meetingData),

  onDeepLink: (callback) => {
    ipcRenderer.on("servicecall-deep-link", (event, data) => {
      callback(data);
    });
  },

  onDeepLink: (callback) => {
    const handler = (event, data) => {
      callback(data);
    };

    ipcRenderer.on("servicecall-deep-link", handler);

    return () => {
      ipcRenderer.removeListener("servicecall-deep-link", handler);
    };
  },

  rendererReady: () => {
    ipcRenderer.send("servicecall-renderer-ready");
  },

  getNotifications: (page = 1, pageSize = 20, search = "") => {
    return ipcRenderer.invoke("servicecall-get-notifications", {
      page: page,
      pageSize: pageSize,
      search: search,
    });
  },

  markNotificationRead: (notificationSysId) => {
    return ipcRenderer.invoke(
      "servicecall-mark-notification-read",
      notificationSysId,
    );
  },

  testNotificationPopup: () => {
    return ipcRenderer.invoke("servicecall-test-notification-popup");
  },

  dismissNotificationPopup: () => {
    ipcRenderer.send("servicecall-dismiss-notification-popup");
  },

  openNotification: (notificationData) => {
    ipcRenderer.send("servicecall-open-notification", notificationData);
  },

  onNotificationMeetingOpen: (callback) => {
    ipcRenderer.on("servicecall-open-notification-meeting", (event, data) => {
      callback(data);
    });
  },

  deleteChat: (conversationSysId) =>
    ipcRenderer.invoke("servicecall-delete-chat", conversationSysId),

  updateMeeting: (meetingSysId, meetingData) =>
    ipcRenderer.invoke("servicecall-update-meeting", meetingSysId, meetingData),

  startMeeting: (meetingSysId) =>
    ipcRenderer.invoke("servicecall-start-meeting", meetingSysId),

  joinMeeting: (meetingSysId) =>
    ipcRenderer.invoke("servicecall-join-meeting", meetingSysId),

  leaveMeeting: (meetingSysId) =>
    ipcRenderer.invoke("servicecall-leave-meeting", meetingSysId),

  endMeeting: (meetingSysId) =>
    ipcRenderer.invoke("servicecall-end-meeting", meetingSysId),

  cancelMeeting: (meetingSysId) =>
    ipcRenderer.invoke("servicecall-cancel-meeting", meetingSysId),

  notifyMeetingChanged: (meetingSysId) =>
    ipcRenderer.send("servicecall-meeting-changed", meetingSysId),

  onMeetingChanged: (callback) => {
    ipcRenderer.on("servicecall-meeting-changed", (event, data) => {
      callback(data);
    });
  },
});

  }
});
