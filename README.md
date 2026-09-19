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

        const openActiveCallButton =
            document.getElementById(
                'openActiveCallButton'
            );

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


let currentMeetingPage = 1;

let currentMeetingSearch = '';

let currentMeetingStatus = '';

let meetingSearchTimer = null;

let schedulePeopleSearchTimer = null;

let selectedMeetingPeople = [];

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

const scheduleMeetingPeopleSearch =
    document.getElementById(
        'scheduleMeetingPeopleSearch'
    );

const scheduleMeetingPeopleResults =
    document.getElementById(
        'scheduleMeetingPeopleResults'
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
            (data) => {

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
                         * Update topbar title.
                         */
                        const pageName =
                            button.textContent.trim();


                        if (currentPageTitle) {

                            currentPageTitle.textContent =
                                pageName;
                        }


                        /*
                         * Meetings are loaded from
                         * ServiceNow whenever the user
                         * opens the Meetings page.
                         */
                        if (
                            targetView ===
                            'meetingsView'
                        ) {

                            await loadMeetings();
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

function openScheduleMeetingModal() {

    if (!scheduleMeetingModal) {
        return;
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

    if (scheduleMeetingStart) {
        scheduleMeetingStart.value = '';
    }

    if (scheduleMeetingEnd) {
        scheduleMeetingEnd.value = '';
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


function openMeetingDetailsModal(
    details
) {

    if (!meetingDetailsModal) {
        return;
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
                await loadMeetings();


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
                await loadMeetings();


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


                await loadMeetings();


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
                await loadMeetings();


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
                await loadMeetings();


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
           LOAD MEETINGS
        ------------------------------------------------- */

        async function loadMeetings() {

            if (!meetingsContainer) {
                return;
            }

            if (meetingPagination) {

    meetingPagination.innerHTML =
        '';
}


            meetingsContainer.innerHTML = `
                <div class="loading">
                    Loading your meetings...
                </div>
            `;


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


                meetingsContainer.innerHTML = `
                    <div class="empty-state">
                        Unable to load your meetings.
                    </div>
                `;
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

                await loadMeetings();
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


        /*
         * Show selected people.
         */
        scheduleMeetingSelectedPeople.innerHTML =
            '';


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
            person.name;


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

                selectedMeetingPeople =
                    selectedMeetingPeople.filter(
                        selected =>
                            selected.sys_id !==
                            person.sys_id
                    );


                selectedPerson.remove();


                /*
                 * If nobody remains selected,
                 * show our empty message again.
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
                }
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


            if (
                new Date(end) <=
                new Date(start)
            ) {

                scheduleMeetingMessage.textContent =
                    'End time must be after the start time.';

                scheduleMeetingEnd.focus();

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

                participants:
                    participants
            };


            console.log(
                'Creating ServiceCall meeting:',
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

                const result =
                    await window.serviceCall.createMeeting(
                        meetingData
                    );


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
                    'Meeting scheduled successfully.';


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
});
