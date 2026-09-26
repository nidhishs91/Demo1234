document.addEventListener("DOMContentLoaded", async () => {
  /* -------------------------------------------------
           ELEMENTS
        ------------------------------------------------- */

  const form = document.getElementById("instanceForm");

  const input = document.getElementById("instanceUrl");

  const message = document.getElementById("instanceMessage");

  const loginButton = document.getElementById("loginButton");

  const signOutButton = document.getElementById("signOutButton");

  const openActiveCallButton = document.getElementById("openActiveCallButton");

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

  const chatMessageInput = document.getElementById("chatMessageInput");

  const chatSendButton = document.getElementById("chatSendButton");
  let activeChatConversation = null;
  let lastChatMessageSysId = "";
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

  /* =============================================
   SERVICECALL CHAT - OPEN CONVERSATION
============================================= */

  async function openChatConversation(conversation) {
    if (!conversation || !conversation.sys_id) {
      return;
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
      chatGroupDetailsButton.style.display =
        conversation.type === "group" ? "" : "none";
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
     * Remove the previous conversation
     * immediately.
     *
     * No "Loading messages..." flash.
     */
    if (chatMessages) {
      chatMessages.innerHTML = "";
    }

    /*
     * Disable composer while the actual
     * conversation is being established.
     *
     * This prevents sending into the wrong
     * conversation during a very fast switch.
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
           MESSAGE SYNC CHECKPOINT
        ========================================= */

      if (messages.length > 0) {
        const newestMessage = messages[messages.length - 1];

        lastChatMessageSysId = String(newestMessage.sys_id || "").trim();
      } else {
        lastChatMessageSysId = "";
      }

      /* =========================================
           RENDER IMMEDIATELY
        ========================================= */

      if (!chatMessages) {
        return;
      }

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

        chatMessages.scrollTop = chatMessages.scrollHeight;
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

   System events are not normal chat messages.

   Examples:
   - Group created by System Administrator
   - System Administrator added Abel Tuter
   - System Administrator made Abel Tuter an owner
   - System Administrator changed Abel Tuter to a member
   - System Administrator removed Test User from the group

   Render them as a centered, subtle event imprint.
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

      /*
       * System messages deliberately have:
       *
       * - no chat bubble
       * - no sender alignment
       * - no reaction button
       * - no reaction summary
       * - no message background
       *
       * They visually behave like events
       * imprinted into the conversation.
       */

      chatMessages.scrollTop = chatMessages.scrollHeight;

      return;
    }
    /* =====================================================
   DUPLICATE MESSAGE PROTECTION

   The same ServiceNow message can reach the renderer
   through more than one path:

   1. /send-message response
   2. silent message synchronization

   A ServiceNow message sys_id must only ever have
   one visible message row.
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

    const messageRow = document.createElement("div");

    messageRow.className = "chat-message-row";

    /*
     * Store the REAL ServiceNow message
     * sys_id on the DOM element.
     *
     * Reactions, edits, deletion, etc.
     * can use this later.
     */
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
     * Bubble + reaction button wrapper.
     */
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
     *
     * [+] [message]
     *
     * Outgoing:
     *
     * [message] [+]
     */
    bubbleWrapper.style.flexDirection = message.is_mine ? "row-reverse" : "row";

    const bubble = document.createElement("div");

    bubble.textContent = message.text || "";

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
     * another user's message.
     *
     * Own messages can still contain
     * emojis as normal chat messages.
     */
    if (!messageSysId || message.is_mine) {
      reactionButton.style.display = "none";
    }

    bubbleWrapper.addEventListener("mouseenter", () => {
      if (messageSysId && !message.is_mine) {
        reactionButton.style.opacity = "1";
      }
    });

    bubbleWrapper.addEventListener("mouseleave", () => {
      reactionButton.style.opacity = "0";
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

    /*
     * Render reaction summary returned
     * by ServiceNow.
     */
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

        /*
         * Clicking an existing reaction
         * uses the same backend toggle.
         */
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
      if (reactionRequestRunning || !messageSysId) {
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

      /*
       * Close any picker already open
       * elsewhere in Chat.
       */
      document.querySelectorAll(".chat-reaction-picker").forEach((picker) => {
        picker.remove();
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

      /*
       * Keep the picker toward the
       * message side of the screen.
       */
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

    bubbleWrapper.appendChild(bubble);

    bubbleWrapper.appendChild(reactionButton);

    const metadata = document.createElement("div");

    metadata.textContent = message.sent_at || "";

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
     * this will render them immediately.
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
        activeChatConversation.sys_id !== conversationSysId
      ) {
        return;
      }

      const newMessages = Array.isArray(result.messages) ? result.messages : [];

      /*
       * Nothing new.
       *
       * Do absolutely nothing to the UI.
       */
      if (newMessages.length === 0) {
        return;
      }

      /*
       * Append ONLY messages returned
       * after our checkpoint.
       */
      newMessages.forEach((newMessage) => {
        appendChatMessage(newMessage);
      });

      /*
       * The user is actively viewing this
       * conversation.
       *
       * Any message that arrived through
       * silent synchronization has therefore
       * been seen and should immediately be
       * marked as read.
       */
      try {
        const readResult =
          await window.serviceCall.markConversationRead(conversationSysId);

        if (
          readResult &&
          readResult.success &&
          activeChatConversation &&
          String(activeChatConversation.sys_id) === String(conversationSysId)
        ) {
          activeChatConversation.unread_count = 0;
        }
      } catch (readError) {
        console.error(
          "Unable to mark silently received messages as read:",
          readError,
        );
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

          if (activeChatConversation && activeChatConversation.sys_id) {
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
          loadChatConversations();

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

        if (isActiveConversation) {
          unreadCount = 0;
        }

        let badge = row.querySelector(".chat-unread-badge");

        /*
         * SHOW / UPDATE BADGE
         */
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
    } catch (error) {
      console.error("Silent conversation list synchronization error:", error);
    }
  }

  window.testChatSync = checkForNewChatMessages;

  startChatMessageSync();

  async function sendActiveChatMessage() {
    if (chatMessageSending) {
      return;
    }

    /* =========================================
       VALIDATE ACTIVE CONVERSATION
    ========================================= */

    if (!activeChatConversation || !activeChatConversation.sys_id) {
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

    if (!message) {
      return;
    }

    if (message.length > 10000) {
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
    const sendingConversationSysId = String(activeChatConversation.sys_id);

    const recipientSysId = isDirectConversation
      ? String(activeChatConversation.other_user_sys_id || "").trim()
      : "";

    const conversationSysId = isGroupConversation
      ? sendingConversationSysId
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

      const result = await window.serviceCall.sendMessage(
        recipientSysId,
        conversationSysId,
        message,
      );

      console.log("ServiceCall message sent:", result);

      if (!result || result.success !== true) {
        throw new Error(
          result && result.message ? result.message : "Unable to send message.",
        );
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
        activeChatConversation &&
        String(activeChatConversation.sys_id) === sendingConversationSysId;

      /*
       * Only clear the composer when the user
       * is still viewing the conversation from
       * which this message was sent.
       */
      if (stillViewingSentConversation) {
        chatMessageInput.value = "";

        resizeChatMessageInput();
      }

      /* =========================================
           AUTHORITATIVE SAVED MESSAGE
        ========================================= */

      const savedMessage = result.message || {};

      /*
       * Only render locally when this is still
       * the conversation currently displayed.
       */
      if (stillViewingSentConversation) {
        appendChatMessage({
          sys_id: String(savedMessage.sys_id || ""),

          sender_sys_id: String(savedMessage.sender_sys_id || ""),

          sender_name: String(savedMessage.sender_name || ""),

          type: String(savedMessage.type || "text"),

          text: String(savedMessage.text || message),

          sent_at: String(savedMessage.sent_at || ""),

          is_mine: true,
        });

        /*
         * Prevent the silent polling loop
         * from fetching our locally-rendered
         * message again.
         */
        if (savedMessage.sys_id) {
          lastChatMessageSysId = String(savedMessage.sys_id).trim();
        }

        /* =====================================
               UPDATE ACTIVE CONVERSATION MEMORY
            ===================================== */

        activeChatConversation.last_message_preview = message;

        activeChatConversation.last_message_at =
          result.conversation && result.conversation.last_message_at
            ? String(result.conversation.last_message_at)
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
            preview.textContent = message;
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
       */
      if (
        chatMessageInput &&
        activeChatConversation &&
        String(activeChatConversation.sys_id) === sendingConversationSysId
      ) {
        chatMessageInput.disabled = false;

        if (chatSendButton) {
          chatSendButton.disabled = !String(
            chatMessageInput.value || "",
          ).trim();
        }

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
        /*
         * Grow / shrink composer
         * according to message content.
         */
        resizeChatMessageInput();

        if (!chatSendButton || chatMessageSending) {
          return;
        }

        chatSendButton.disabled = !String(chatMessageInput.value || "").trim();
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

        if (!message) {
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

            await openChatConversation(conversation);

            /*
             * openChatConversation()
             * marks this conversation
             * as read through ServiceNow.
             *
             * Update the local sidebar
             * immediately as well.
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
            row.style.background = "#f5faf8";
          },
        );

        row.addEventListener(
          "mouseleave",

          () => {
            row.style.background = "white";
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
    const MAX_HEIGHT = 140;

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
   OPEN TEMPORARY CHAT
------------------------------------------------- */

  function openTemporaryChat(user) {
    if (!user || !user.sys_id) {
      return;
    }

    if (chatGroupDetailsButton) {
      chatGroupDetailsButton.style.display = "none";
    }

    /*
     * This is a NEW / TEMPORARY chat.
     *
     * No ServiceNow conversation exists
     * merely because the user opened it.
     */
    activeChatConversation = null;

    activeChatUser = user;

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

                  () => {
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
