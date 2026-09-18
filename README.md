const params =
    new URLSearchParams(
        window.location.search
    );
    
const mode =
    params.get('mode') ||
    'incoming';

const callSysId =
    params.get('callSysId') ||
    '';

const personName =
    params.get('name') ||
    'Unknown User';

const department =
    params.get('department') ||
    '';

const callNumber =
    params.get('callNumber') ||
    '';

const isConference =
    params.get('isConference') === 'true';

/* -------------------------
   MEETING CONTEXT
------------------------- */

const isMeeting =
    params.get('isMeeting') === 'true';

const meetingSysId =
    params.get('meetingSysId') ||
    '';

const meetingNumber =
    params.get('meetingNumber') ||
    '';

const meetingTitle =
    params.get('meetingTitle') ||
    '';

console.log(
    'SERVICECALL MEETING CONTEXT:',
    {
        isMeeting,
        meetingSysId,
        meetingNumber,
        meetingTitle
    }
);

let currentMode =
    mode;

let callStatusTimer =
    null;

let callStartedAt =
    null;

let durationTimer =
    null;

let closeTimer = null;

let ringtoneInterval = null;
let audioContext = null;

let currentUserIsOwner =
    false;

let participantRole =
    'participant';

/* -------------------------
   ELEMENTS
------------------------- */

const avatar =
    document.getElementById(
        'avatar'
    );

const personNameElement =
    document.getElementById(
        'personName'
    );

const departmentElement =
    document.getElementById(
        'department'
    );

const statusText =
    document.getElementById(
        'statusText'
    );

const callNumberElement =
    document.getElementById(
        'callNumber'
    );

const timerElement =
    document.getElementById(
        'timer'
    );

const incomingActions =
    document.getElementById(
        'incomingActions'
    );

const callingActions =
    document.getElementById(
        'callingActions'
    );

const connectedActions =
    document.getElementById(
        'connectedActions'
    );


/* -------------------------
   INITIAL DISPLAY
------------------------- */

personNameElement.textContent =
    personName;

departmentElement.textContent =
    department;

callNumberElement.textContent =
    callNumber;


const initials =
    personName
        .split(' ')
        .filter(Boolean)
        .slice(0, 2)
        .map(
            part =>
                part[0]
                    .toUpperCase()
        )
        .join('');


avatar.textContent =
    initials || '?';

/* -------------------------
   MEETING DISPLAY
------------------------- */

if (isMeeting) {

    /*
     * Use the meeting title as the
     * primary name in the existing
     * ServiceCall window.
     */
    personNameElement.textContent =
        meetingTitle ||
        personName ||
        'ServiceCall Meeting';


    /*
     * Show that this is a meeting
     * instead of a person's department.
     */
    departmentElement.textContent =
        'Meeting';


    /*
     * Show meeting number when available.
     */
    if (meetingNumber) {

        callNumberElement.textContent =
            meetingNumber;
    }


    /*
     * Meeting-specific action labels.
     */
    const addUserButton =
        document.getElementById(
            'addUserButton'
        );

    const recordButton =
        document.getElementById(
            'recordButton'
        );


    if (addUserButton) {

        addUserButton.textContent =
            'Add People';
    }


    if (recordButton) {

        recordButton.textContent =
            'Record Meeting';
    }
}

/* -------------------------
   RINGTONE
------------------------- */
 
function playRingTone(
    frequency = 440
) {
 
    try {
 
        if (!audioContext) {
 
            audioContext =
                new (
                    window.AudioContext ||
                    window.webkitAudioContext
                )();
        }
 
 
        const oscillator =
            audioContext.createOscillator();
 
        const gain =
            audioContext.createGain();
 
 
        oscillator.type =
            'sine';
 
        oscillator.frequency.value =
            frequency;
 
 
        gain.gain.value =
            0.08;
 
 
        oscillator.connect(
            gain
        );
 
        gain.connect(
            audioContext.destination
        );
 
 
        oscillator.start();
 
 
        setTimeout(
            () => {
 
                try {
 
                    oscillator.stop();
 
                } catch (error) {
                    // Already stopped.
                }
 
            },
            500
        );
 
    } catch (error) {
 
        console.error(
            'Unable to play ServiceCall ringtone:',
            error
        );
    }
}
 
 
function startRingtone(
    type
) {
 
    stopRingtone();
 
 
    /*
     * Incoming call:
     * slightly higher tone.
     *
     * Outgoing call:
     * lower ringback tone.
     */
    const frequency =
        type === 'incoming'
            ? 520
            : 420;
 
 
    playRingTone(
        frequency
    );
 
 
    ringtoneInterval =
        setInterval(
            () => {
 
                playRingTone(
                    frequency
                );
 
            },
            1800
        );
}
 
 
function stopRingtone() {
 
    if (ringtoneInterval) {
 
        clearInterval(
            ringtoneInterval
        );
 
        ringtoneInterval =
            null;
    }
}

/* -------------------------
   AGORA AUDIO
------------------------- */

let agoraJoining = false;


async function startAgoraAudio() {

    if (
        !window.ServiceCallAgora
    ) {

        console.error(
            'ServiceCall Agora media service is not available.'
        );

        statusText.textContent =
            'Audio service unavailable';

        return;
    }


    if (
        window.ServiceCallAgora.isJoined() ||
        agoraJoining
    ) {
        return;
    }


    if (!callSysId) {

        console.error(
            'ServiceCall call sys_id is missing.'
        );

        statusText.textContent =
            'Audio configuration unavailable';

        return;
    }


    agoraJoining =
        true;


    try {

        console.log(
            'ServiceCall requesting dynamic media credentials...'
        );


        /*
         * Request short-lived credentials
         * for THIS specific ServiceCall call.
         *
         * Electron -> ServiceNow
         * -> Cloudflare Worker -> Agora token
         */
        const credentials =
            await window.serviceCall
                .getMediaCredentials(
                    callSysId
                );


        if (
            !credentials ||
            !credentials.success
        ) {

            throw new Error(
                credentials &&
                credentials.message
                    ? credentials.message
                    : 'Media credentials could not be obtained.'
            );
        }


        const media =
            credentials.media;


        if (
            !media ||
            !media.app_id ||
            !media.channel ||
            !media.token ||
            !media.uid
        ) {

            throw new Error(
                'ServiceCall returned incomplete media credentials.'
            );
        }


        console.log(
            'ServiceCall media credentials received.',
            'Channel:',
            media.channel,
            'UID:',
            media.uid
        );


        /*
         * IMPORTANT:
         *
         * Never log media.token.
         */


        console.log(
            'ServiceCall joining Agora audio channel...'
        );


        await window.ServiceCallAgora.join({

            appId:
                media.app_id,

            token:
                media.token,

            channel:
                media.channel,

            uid:
                media.uid,


            /*
             * TOKEN RENEWAL CALLBACK
             *
             * agora-service.js calls this
             * when Agora warns that the
             * current token will expire.
             *
             * We request fresh credentials
             * for the SAME ServiceCall call.
             */
            renewToken:
                async () => {

                    console.log(
                        'ServiceCall requesting renewed media credentials...'
                    );


                    const renewedCredentials =
                        await window.serviceCall
                            .getMediaCredentials(
                                callSysId
                            );


                    if (
                        !renewedCredentials ||
                        !renewedCredentials.success
                    ) {

                        throw new Error(
                            renewedCredentials &&
                            renewedCredentials.message
                                ? renewedCredentials.message
                                : 'Unable to obtain renewed media credentials.'
                        );
                    }


                    const renewedMedia =
                        renewedCredentials.media;


                    if (
                        !renewedMedia ||
                        !renewedMedia.token
                    ) {

                        throw new Error(
                            'ServiceCall returned an incomplete renewed media token.'
                        );
                    }


                    /*
                     * Safety check:
                     *
                     * Renewal must remain on
                     * the SAME Agora channel
                     * and SAME participant UID.
                     */
                    if (
                        renewedMedia.channel !==
                        media.channel
                    ) {

                        throw new Error(
                            'Renewed media channel does not match the active call.'
                        );
                    }


                    if (
                        Number(
                            renewedMedia.uid
                        ) !==
                        Number(
                            media.uid
                        )
                    ) {

                        throw new Error(
                            'Renewed media UID does not match the active participant.'
                        );
                    }


                    console.log(
                        'ServiceCall renewed media credentials received.',
                        'Channel:',
                        renewedMedia.channel,
                        'UID:',
                        renewedMedia.uid
                    );


                    /*
                     * NEVER log renewedMedia.token.
                     *
                     * Return only the token to
                     * agora-service.js.
                     */
                    return renewedMedia.token;
                }
        });


        console.log(
            'ServiceCall audio connected successfully.'
        );


    } catch (error) {

        console.error(
            'ServiceCall audio connection failed:',
            error
        );


        statusText.textContent =
            'Connected - audio unavailable';


    } finally {

        agoraJoining =
            false;
    }
}


async function stopAgoraAudio() {

    if (
        !window.ServiceCallAgora
    ) {
        return;
    }


    try {

        await window.ServiceCallAgora
            .leave();

    } catch (error) {

        console.error(
            'Unable to leave ServiceCall audio:',
            error
        );
    }
}

/* -------------------------
   UI STATE
------------------------- */

function setMode(
    newMode
) {

    currentMode =
        newMode;


    incomingActions
        .classList
        .add('hidden');

    callingActions
        .classList
        .add('hidden');

    connectedActions
        .classList
        .add('hidden');

    timerElement
        .classList
        .add('hidden');


    if (
        newMode ===
        'incoming'
    ) {

        statusText.textContent =
            isConference
                ? 'Conference call'
                : 'is calling you...';

        incomingActions
            .classList
            .remove('hidden');


        startRingtone('incoming');
        return;
    }


    if (
        newMode ===
        'calling'
    ) {

        statusText.textContent =
            'Ringing';

        callingActions
            .classList
            .remove('hidden');

        startRingtone('outgoing');

        return;
    }


    if (
        newMode ===
        'connected'
    ) {

        stopRingtone();

        statusText.textContent =
            'Connected';

        connectedActions
            .classList
            .remove('hidden');

        timerElement
            .classList
            .remove('hidden');

        startDurationTimer();

        startAgoraAudio();

        return;
    }


    if (
    newMode ===
    'declined'
) {

    stopRingtone();
 
    statusText.textContent =
        'Call declined';
 
    stopAllTimers();
 
    closeCallWindowAfterDelay();
 
    return;
}


    if (
    newMode ===
    'cancelled'
) {

    stopRingtone();
 
    statusText.textContent =
        'Call cancelled';
 
    stopAllTimers();
 
    closeCallWindowAfterDelay();
 
    return;
}


    if (
    newMode ===
    'completed'
) {

    stopRingtone();

    statusText.textContent =
        'Call ended';

    stopAllTimers();

    stopAgoraAudio();

    closeCallWindowAfterDelay();

    return;
}
}


/* -------------------------
   CALL TIMER
------------------------- */

function startDurationTimer() {

    if (durationTimer) {
        return;
    }


    callStartedAt =
        Date.now();


    durationTimer =
        setInterval(
            () => {

                const seconds =
                    Math.floor(
                        (
                            Date.now() -
                            callStartedAt
                        ) / 1000
                    );


                const minutes =
                    Math.floor(
                        seconds / 60
                    );


                const remainingSeconds =
                    seconds % 60;


                timerElement.textContent =
                    String(minutes)
                        .padStart(
                            2,
                            '0'
                        ) +
                    ':' +
                    String(
                        remainingSeconds
                    )
                        .padStart(
                            2,
                            '0'
                        );

            },
            1000
        );
}


/* -------------------------
   STATUS POLLING
------------------------- */

async function checkCallStatus() {

    if (!callSysId) {
        return;
    }


    try {

        const result =
            await window.serviceCall
                .getCallStatus(
                    callSysId
                );


        if (
            !result ||
            !result.success
        ) {
            return;
        }


        const state =
            result.state;

        /*
 * Keep the recording indicator synchronized
 * for every participant in the call.
 *
 * /call-status is already polled every
 * 2 seconds, so no additional timer or
 * REST request is required.
 */
syncRecordingIndicator(
    result.recording_active === true
);

        /* -------------------------
   OWNER / PARTICIPANT STATE
------------------------- */

currentUserIsOwner =
    result.is_owner === true;


participantRole =
    result.participant_role ||
    'participant';


const endButton =
    document.getElementById(
        'endButton'
    );


if (currentUserIsOwner) {

    endButton.textContent =
        isMeeting
            ? 'End Meeting'
            : 'End Conference';

} else {

    endButton.textContent =
        isMeeting
            ? 'Leave Meeting'
            : 'Leave Call';
}


/* -------------------------
   PARTICIPANT LEFT
------------------------- */

const participantStatus =
    result.participant_status ||
    '';


if (
    participantStatus === 'left' ||
    participantStatus === 'disconnected'
) {

    /*
     * The overall conference may still be
     * connected, but THIS participant is
     * no longer part of it.
     */

    await stopAgoraAudio();

    stopRingtone();

    stopAllTimers();

    statusText.textContent =
        participantStatus === 'left'
            ? 'You left the call'
            : 'Call ended';

    closeCallWindowAfterDelay();

    return;
}

        if (
            state === 'connected' &&
            currentMode !==
                'connected'
        ) {

            setMode(
                'connected'
            );

            return;
        }


        if (
            state === 'declined'
        ) {

            setMode(
                'declined'
            );

            return;
        }


        if (
            state === 'cancelled'
        ) {

            setMode(
                'cancelled'
            );

            return;
        }


        if (
            state === 'completed'
        ) {

            setMode(
                'completed'
            );

            return;
        }

    } catch (error) {

        console.error(
            'Call status error:',
            error
        );
    }
}


function startCallStatusPolling() {

    if (!callSysId) {
        return;
    }


    checkCallStatus();


    callStatusTimer =
        setInterval(
            checkCallStatus,
            2000
        );
}


/* -------------------------
   ACCEPT
------------------------- */

document
    .getElementById(
        'acceptButton'
    )
    .addEventListener(
        'click',
        async () => {

            try {

                const result =
                    await window
                        .serviceCall
                        .acceptCall(
                            callSysId
                        );


                if (
                    result.success
                ) {

                    setMode(
                        'connected'
                    );
                }

            } catch (error) {

                console.error(
                    'Accept call failed:',
                    error
                );

                statusText.textContent =
                    error.message ||
                    'Unable to accept call.';
            }
        }
    );


/* -------------------------
   DECLINE
------------------------- */

document
    .getElementById(
        'declineButton'
    )
    .addEventListener(
        'click',
        async () => {

            try {

                const result =
                    await window
                        .serviceCall
                        .declineCall(
                            callSysId
                        );


                if (
                    result.success
                ) {

                    setMode(
                        'declined'
                    );
                }

            } catch (error) {

                console.error(
                    'Decline call failed:',
                    error
                );

                statusText.textContent =
                    error.message ||
                    'Unable to decline call.';
            }
        }
    );


/* -------------------------
   CANCEL
------------------------- */

document
    .getElementById(
        'cancelButton'
    )
    .addEventListener(
        'click',
        async () => {

            try {

                const result =
                    await window
                        .serviceCall
                        .cancelCall(
                            callSysId
                        );


                if (
                    result.success
                ) {

                    setMode(
                        'cancelled'
                    );
                }

            } catch (error) {

                console.error(
                    'Cancel call failed:',
                    error
                );

                statusText.textContent =
                    error.message ||
                    'Unable to cancel call.';
            }
        }
    );

/* -------------------------
   MUTE / UNMUTE
------------------------- */

document
    .getElementById(
        'muteButton'
    )
    .addEventListener(
        'click',
        async () => {

            try {

                if (
                    !window.ServiceCallAgora ||
                    !window.ServiceCallAgora.isJoined()
                ) {

                    statusText.textContent =
                        'Audio is not connected';

                    return;
                }


                const currentlyMuted =
                    window.ServiceCallAgora
                        .isMuted();


                const result =
                    await window.ServiceCallAgora
                        .setMuted(
                            !currentlyMuted
                        );


                if (result.success) {

                    document
                        .getElementById(
                            'muteButton'
                        )
                        .textContent =
                            result.muted
                                ? 'Unmute'
                                : 'Mute';


                    console.log(
                        result.muted
                            ? 'ServiceCall microphone muted.'
                            : 'ServiceCall microphone unmuted.'
                    );
                }


            } catch (error) {

                console.error(
                    'ServiceCall mute failed:',
                    error
                );


                statusText.textContent =
                    'Unable to change microphone state';
            }
        }
    );

/* -------------------------
   ADD PARTICIPANT
------------------------- */

const addUserButton =
    document.getElementById(
        'addUserButton'
    );

const addParticipantModal =
    document.getElementById(
        'addParticipantModal'
    );

const closeParticipantModalButton =
    document.getElementById(
        'closeParticipantModal'
    );

const participantSearch =
    document.getElementById(
        'participantSearch'
    );

const participantResults =
    document.getElementById(
        'participantResults'
    );

const participantMessage =
    document.getElementById(
        'participantMessage'
    );


let participantSearchTimer =
    null;


/* -------------------------
   OPEN MODAL
------------------------- */

function openParticipantModal() {

    participantMessage.textContent =
        '';

    participantMessage.className =
        'participant-message';


    participantSearch.value =
        '';


    participantResults.innerHTML =
        '<div class="participant-empty">' +
        'Start typing a user\'s name.' +
        '</div>';


    addParticipantModal
        .classList
        .remove(
            'hidden'
        );


    setTimeout(
        () => {

            participantSearch.focus();

        },
        100
    );
}


/* -------------------------
   CLOSE MODAL
------------------------- */

function closeParticipantModal() {

    addParticipantModal
        .classList
        .add(
            'hidden'
        );


    if (participantSearchTimer) {

        clearTimeout(
            participantSearchTimer
        );

        participantSearchTimer =
            null;
    }
}


/* -------------------------
   SEARCH USERS
------------------------- */

async function searchParticipantUsers() {

    const searchText =
        participantSearch
            .value
            .trim();


    participantMessage.textContent =
        '';

    participantMessage.className =
        'participant-message';


    if (
        searchText.length < 2
    ) {

        participantResults.innerHTML =
            '<div class="participant-empty">' +
            'Enter at least 2 characters.' +
            '</div>';

        return;
    }


    participantResults.innerHTML =
        '<div class="participant-empty">' +
        'Searching...' +
        '</div>';


    try {

        const result =
            await window.serviceCall
                .searchUsers(
                    searchText
                );


        /*
         * Ignore an old search result if the
         * user has already typed something else.
         */
        if (
            participantSearch
                .value
                .trim() !==
            searchText
        ) {
            return;
        }


        if (
            !result ||
            !result.success
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


        if (
            users.length === 0
        ) {

            participantResults.innerHTML =
                '<div class="participant-empty">' +
                'No matching users found.' +
                '</div>';

            return;
        }


        /*
         * Build results safely with DOM APIs.
         * Do not inject ServiceNow user values
         * directly into HTML.
         */
        participantResults.innerHTML =
            '';


        users.forEach(
            (user) => {

                const row =
                    document.createElement(
                        'div'
                    );

                row.className =
                    'participant-result';


                const details =
                    document.createElement(
                        'div'
                    );

                details.className =
                    'participant-details';


                const name =
                    document.createElement(
                        'div'
                    );

                name.className =
                    'participant-name';

                name.textContent =
                    user.name ||
                    'Unknown User';


                const subtitle =
                    document.createElement(
                        'div'
                    );

                subtitle.className =
                    'participant-subtitle';


                const subtitleParts =
                    [];


                if (user.department) {

                    subtitleParts.push(
                        user.department
                    );
                }


                if (user.email) {

                    subtitleParts.push(
                        user.email
                    );
                }


                subtitle.textContent =
                    subtitleParts.join(
                        ' • '
                    ) ||
                    user.user_name ||
                    'ServiceNow user';


                details.appendChild(
                    name
                );

                details.appendChild(
                    subtitle
                );


                const addButton =
                    document.createElement(
                        'button'
                    );

                addButton.className =
                    'participant-add';

                addButton.textContent =
                    'Add';


                addButton.addEventListener(
                    'click',
                    async () => {

                        await inviteServiceCallParticipant(
                            user,
                            addButton
                        );
                    }
                );


                row.appendChild(
                    details
                );

                row.appendChild(
                    addButton
                );


                participantResults.appendChild(
                    row
                );
            }
        );


    } catch (error) {

        console.error(
            'ServiceCall user search failed:',
            error
        );


        participantResults.innerHTML =
            '<div class="participant-empty">' +
            'Unable to search users.' +
            '</div>';


        participantMessage.textContent =
            error.message ||
            'Unable to search users.';

        participantMessage.className =
            'participant-message error';
    }
}


/* -------------------------
   INVITE USER
------------------------- */

async function inviteServiceCallParticipant(
    user,
    addButton
) {

    if (
        !user ||
        !user.sys_id
    ) {
        return;
    }


    addButton.disabled =
        true;

    addButton.textContent =
        'Adding...';


    participantMessage.textContent =
        'Inviting ' +
        (
            user.name ||
            'user'
        ) +
        '...';

    participantMessage.className =
        'participant-message';


    try {

        const result =
            await window.serviceCall
                .inviteParticipant(
                    callSysId,
                    user.sys_id
                );


        if (
            !result ||
            !result.success
        ) {

            throw new Error(
                result &&
                result.message
                    ? result.message
                    : 'Unable to invite participant.'
            );
        }


        addButton.textContent =
            'Invited';


        participantMessage.textContent =
            (
                user.name ||
                'Participant'
            ) +
            ' has been invited to the call.';

        participantMessage.className =
            'participant-message success';


        /*
         * Leave the success message visible
         * briefly, then close the modal.
         */
        setTimeout(
            () => {

                closeParticipantModal();

            },
            1000
        );


    } catch (error) {

        console.error(
            'ServiceCall participant invite failed:',
            error
        );


        addButton.disabled =
            false;

        addButton.textContent =
            'Add';


        participantMessage.textContent =
            error.message ||
            'Unable to invite participant.';

        participantMessage.className =
            'participant-message error';
    }
}


/* -------------------------
   ADD USER EVENTS
------------------------- */

addUserButton.addEventListener(
    'click',
    () => {

        if (
            currentMode !==
            'connected'
        ) {
            return;
        }


        openParticipantModal();
    }
);


closeParticipantModalButton
    .addEventListener(
        'click',
        closeParticipantModal
    );


/*
 * Clicking the dark area outside
 * the card closes the modal.
 */
addParticipantModal.addEventListener(
    'click',
    (event) => {

        if (
            event.target ===
            addParticipantModal
        ) {

            closeParticipantModal();
        }
    }
);


/*
 * Escape closes the modal.
 */
document.addEventListener(
    'keydown',
    (event) => {

        if (
            event.key ===
            'Escape' &&
            !addParticipantModal
                .classList
                .contains(
                    'hidden'
                )
        ) {

            closeParticipantModal();
        }
    }
);


/*
 * Small debounce so we don't hit
 * ServiceNow on every keystroke.
 */
participantSearch.addEventListener(
    'input',
    () => {

        if (participantSearchTimer) {

            clearTimeout(
                participantSearchTimer
            );
        }


        participantSearchTimer =
            setTimeout(
                searchParticipantUsers,
                350
            );
    }
);

/* =======================================================
   SERVICECALL SCREEN SHARING
======================================================= */

const shareScreenButton =
    document.getElementById(
        'shareScreenButton'
    );

const screenShareContainer =
    document.getElementById(
        'screenShareContainer'
    );

const screenShareVideo =
    document.getElementById(
        'screenShareVideo'
    );

const screenSharePlaceholder =
    document.getElementById(
        'screenSharePlaceholder'
    );

const screenShareTitle =
    document.getElementById(
        'screenShareTitle'
    );

const screenSourceModal =
    document.getElementById(
        'screenSourceModal'
    );

const screenSourceResults =
    document.getElementById(
        'screenSourceResults'
    );

const closeScreenSourceModalButton =
    document.getElementById(
        'closeScreenSourceModal'
    );


let localScreenStream =
    null;

let localScreenAgoraTrack =
    null;

let localScreenNativeTrack =
    null;

let screenShareActionInProgress =
    false;

let activeRemoteScreenUid =
    null;


/* -------------------------------------------------------
   CALL WINDOW LAYOUT
------------------------------------------------------- */

async function setScreenCallLayout() {

    if (
        !window.serviceCall ||
        typeof window.serviceCall
            .setCallWindowLayout !==
            'function'
    ) {
        return;
    }


    try {

        await window.serviceCall
            .setCallWindowLayout(
                'screen'
            );

    } catch (error) {

        console.error(
            'Unable to expand ServiceCall window:',
            error
        );
    }
}


async function setCompactCallLayout() {

    if (
        !window.serviceCall ||
        typeof window.serviceCall
            .setCallWindowLayout !==
            'function'
    ) {
        return;
    }


    try {

        await window.serviceCall
            .setCallWindowLayout(
                'compact'
            );

    } catch (error) {

        console.error(
            'Unable to restore ServiceCall window:',
            error
        );
    }
}


/* -------------------------------------------------------
   SCREEN VIEW
------------------------------------------------------- */

async function showScreenShareView(
    title
) {

    screenShareTitle.textContent =
        title ||
        'Screen sharing';


    screenShareContainer
        .classList
        .remove(
            'hidden'
        );


    screenSharePlaceholder
        .classList
        .remove(
            'hidden'
        );


    await setScreenCallLayout();
}


async function hideScreenShareView() {

    screenShareVideo.innerHTML =
        '';


    screenSharePlaceholder
        .classList
        .remove(
            'hidden'
        );


    screenShareContainer
        .classList
        .add(
            'hidden'
        );


    activeRemoteScreenUid =
        null;


    /*
     * Do not collapse if THIS participant
     * is still sharing.
     */
    if (
        !window.ServiceCallAgora ||
        !window.ServiceCallAgora
            .isScreenSharing()
    ) {

        await setCompactCallLayout();
    }
}


/* -------------------------------------------------------
   SOURCE MODAL
------------------------------------------------------- */

function closeScreenSourceModal() {

    screenSourceModal
        .classList
        .add(
            'hidden'
        );


    screenSourceResults.innerHTML =
        '';
}


async function openScreenSourceModal() {

    if (
        currentMode !==
        'connected'
    ) {

        statusText.textContent =
            'The call must be connected before sharing your screen.';

        return;
    }


    if (
        !window.ServiceCallAgora ||
        !window.ServiceCallAgora
            .isJoined()
    ) {

        statusText.textContent =
            'Call media is not connected yet.';

        return;
    }


    if (
        !window.serviceCall ||
        typeof window.serviceCall
            .getScreenSources !==
            'function'
    ) {

        statusText.textContent =
            'Screen sharing is unavailable.';

        return;
    }


    screenSourceModal
        .classList
        .remove(
            'hidden'
        );


    screenSourceResults.innerHTML =
        '<div class="screen-source-message">' +
        'Loading screens and windows...' +
        '</div>';


    try {

        const result =
            await window.serviceCall
                .getScreenSources();


        if (
            !result ||
            !result.success
        ) {

            throw new Error(
                result &&
                result.message
                    ? result.message
                    : 'Unable to retrieve screens and windows.'
            );
        }


        const sources =
            Array.isArray(
                result.sources
            )
                ? result.sources
                : [];


        screenSourceResults.innerHTML =
            '';


        if (
            sources.length === 0
        ) {

            screenSourceResults.innerHTML =
                '<div class="screen-source-message">' +
                'No screens or windows are available.' +
                '</div>';

            return;
        }


        sources.forEach(
            (source) => {

                const item =
                    document.createElement(
                        'button'
                    );


                item.type =
                    'button';

                item.className =
                    'screen-source-item';


                if (source.thumbnail) {

                    const thumbnail =
                        document.createElement(
                            'img'
                        );


                    thumbnail.className =
                        'screen-source-thumbnail';

                    thumbnail.src =
                        source.thumbnail;

                    thumbnail.alt =
                        '';


                    item.appendChild(
                        thumbnail
                    );
                }


                const sourceName =
                    document.createElement(
                        'div'
                    );


                sourceName.className =
                    'screen-source-name';

                sourceName.textContent =
                    source.name ||
                    'Screen';


                item.appendChild(
                    sourceName
                );


                item.addEventListener(
                    'click',

                    async () => {

                        await startServiceCallScreenShare(
                            source
                        );
                    }
                );


                screenSourceResults.appendChild(
                    item
                );
            }
        );


    } catch (error) {

        console.error(
            'Unable to open ServiceCall screen picker:',
            error
        );


        screenSourceResults.innerHTML =
            '';


        const message =
            document.createElement(
                'div'
            );


        message.className =
            'screen-source-message';

        message.textContent =
            error.message ||
            'Unable to load screens and windows.';


        screenSourceResults.appendChild(
            message
        );
    }
}


/* -------------------------------------------------------
   CAPTURE ELECTRON SOURCE
------------------------------------------------------- */

async function captureDesktopSource(
    sourceId
) {

    if (!sourceId) {

        throw new Error(
            'Screen source ID was not provided.'
        );
    }


    /*
     * Electron DesktopCapturerSource.id is passed
     * to Chromium as chromeMediaSourceId.
     *
     * We deliberately capture VIDEO ONLY.
     *
     * Conference audio already comes from Agora,
     * therefore desktop/system audio must not be
     * injected into the call.
     */
    const stream =
        await navigator.mediaDevices
            .getUserMedia({

                audio:
                    false,

                video: {

                    mandatory: {

                        chromeMediaSource:
                            'desktop',

                        chromeMediaSourceId:
                            sourceId,

                        maxWidth:
                            1920,

                        maxHeight:
                            1080,

                        maxFrameRate:
                            30
                    }
                }
            });


    const videoTracks =
        stream.getVideoTracks();


    if (
        videoTracks.length === 0
    ) {

        stream
            .getTracks()
            .forEach(
                track =>
                    track.stop()
            );


        throw new Error(
            'The selected screen did not provide a video track.'
        );
    }


    return stream;
}


/* -------------------------------------------------------
   START LOCAL SCREEN SHARE
------------------------------------------------------- */

async function startServiceCallScreenShare(
    source
) {

    if (screenShareActionInProgress) {
        return;
    }


    screenShareActionInProgress =
        true;

    shareScreenButton.disabled =
        true;


    try {

        if (
            !source ||
            !source.id
        ) {

            throw new Error(
                'Please select a valid screen or window.'
            );
        }


        if (
            !window.ServiceCallAgora ||
            !window.ServiceCallAgora
                .isJoined()
        ) {

            throw new Error(
                'Call media is not connected.'
            );
        }


        closeScreenSourceModal();


        statusText.textContent =
            'Starting screen share...';


        /*
         * -----------------------------------------
         * 1. CAPTURE WINDOWS SCREEN/WINDOW
         * -----------------------------------------
         */

        const stream =
            await captureDesktopSource(
                source.id
            );


        const nativeTrack =
            stream
                .getVideoTracks()[0];


        /*
         * -----------------------------------------
         * 2. CREATE AGORA VIDEO TRACK
         * -----------------------------------------
         */

        const agoraTrack =
            await window.ServiceCallAgora
                .createScreenVideoTrack(
                    nativeTrack
                );


        /*
         * Keep references BEFORE publication so
         * cleanup remains possible if publication
         * fails.
         */
        localScreenStream =
            stream;

        localScreenNativeTrack =
            nativeTrack;

        localScreenAgoraTrack =
            agoraTrack;


        /*
         * -----------------------------------------
         * 3. HANDLE WINDOWS/OS STOP SHARING
         * -----------------------------------------
         */

        nativeTrack.addEventListener(
            'ended',

            () => {

                console.log(
                    'ServiceCall screen capture ended by the operating system.'
                );


                stopServiceCallScreenShare()
                    .catch(
                        error => {

                            console.error(
                                'Unable to stop ServiceCall screen share:',
                                error
                            );
                        }
                    );
            },

            {
                once: true
            }
        );


        /*
         * -----------------------------------------
         * 4. PUBLISH THROUGH EXISTING AGORA CALL
         * -----------------------------------------
         */

        await window.ServiceCallAgora
            .startScreenShare(
                agoraTrack
            );

        /*
 * If recording is already running,
 * tell the recorder that screen video
 * has now entered the recording.
 */
if (
    window.ServiceCallRecorder &&
    window.ServiceCallRecorder
        .isRecording()
) {

    await window.ServiceCallRecorder
        .attachScreen(
            agoraTrack
        );
}


        /*
         * -----------------------------------------
         * 5. SHOW LOCAL PREVIEW
         * -----------------------------------------
         */

        await showScreenShareView(
            'You are sharing: ' +
            (
                source.name ||
                'Screen'
            )
        );


        screenShareVideo.innerHTML =
            '';


        screenSharePlaceholder
            .classList
            .add(
                'hidden'
            );


        agoraTrack.play(
            screenShareVideo
        );


        /*
         * -----------------------------------------
         * 6. BUTTON/UI
         * -----------------------------------------
         */

        shareScreenButton.textContent =
            'Stop Sharing';


        shareScreenButton
            .classList
            .add(
                'screen-sharing-active'
            );


        statusText.textContent =
            'Connected';


        console.log(
            'ServiceCall screen sharing started:',
            source.name ||
            source.id
        );


    } catch (error) {

        console.error(
            'Unable to start ServiceCall screen sharing:',
            error
        );


        /*
         * Clean up any partially-created capture.
         */
        if (localScreenAgoraTrack) {

            try {

                localScreenAgoraTrack.stop();

            } catch (stopError) {
                // Ignore.
            }


            try {

                localScreenAgoraTrack.close();

            } catch (closeError) {
                // Ignore.
            }
        }


        if (localScreenStream) {

            localScreenStream
                .getTracks()
                .forEach(
                    track => {

                        try {
                            track.stop();
                        } catch (stopError) {
                            // Ignore.
                        }
                    }
                );
        }


        localScreenAgoraTrack =
            null;

        localScreenNativeTrack =
            null;

        localScreenStream =
            null;


        shareScreenButton.textContent =
            'Share Screen';


        shareScreenButton
            .classList
            .remove(
                'screen-sharing-active'
            );


        statusText.textContent =
            error.message ||
            'Unable to share screen.';


    } finally {

        screenShareActionInProgress =
            false;

        shareScreenButton.disabled =
            false;
    }
}


/* -------------------------------------------------------
   STOP LOCAL SCREEN SHARE
------------------------------------------------------- */

async function stopServiceCallScreenShare() {

    if (screenShareActionInProgress) {
        return;
    }


    screenShareActionInProgress =
        true;

    shareScreenButton.disabled =
        true;


    try {

        /*
 * Recording continues even though
 * live screen sharing is stopping.
 *
 * The recorder returns to blank video,
 * but remembers that screen sharing
 * occurred, therefore final output
 * remains MP4.
 */
if (
    window.ServiceCallRecorder &&
    window.ServiceCallRecorder
        .isRecording()
) {

    window.ServiceCallRecorder
        .detachScreen();
}

        /*
         * Agora owns publication state.
         */
        if (
            window.ServiceCallAgora &&
            window.ServiceCallAgora
                .isScreenSharing()
        ) {

            await window.ServiceCallAgora
                .stopScreenShare();
        }


        /*
         * Stop the underlying Electron capture.
         */
        if (localScreenStream) {

            localScreenStream
                .getTracks()
                .forEach(
                    track => {

                        try {

                            track.stop();

                        } catch (error) {
                            // Ignore cleanup error.
                        }
                    }
                );
        }


        localScreenStream =
            null;

        localScreenNativeTrack =
            null;

        localScreenAgoraTrack =
            null;


        screenShareVideo.innerHTML =
            '';


        screenShareContainer
            .classList
            .add(
                'hidden'
            );


        shareScreenButton.textContent =
            'Share Screen';


        shareScreenButton
            .classList
            .remove(
                'screen-sharing-active'
            );


        /*
         * If nobody else's screen is currently
         * being displayed, return to compact mode.
         */
        if (!activeRemoteScreenUid) {

            await setCompactCallLayout();
        }


        if (
            currentMode ===
            'connected'
        ) {

            statusText.textContent =
                'Connected';
        }


        console.log(
            'ServiceCall local screen sharing stopped.'
        );


    } finally {

        screenShareActionInProgress =
            false;

        shareScreenButton.disabled =
            false;
    }
}


/* -------------------------------------------------------
   REMOTE SCREEN SHARE
------------------------------------------------------- */

function configureRemoteScreenSharing() {

    if (
        !window.ServiceCallAgora ||
        typeof window.ServiceCallAgora
            .setRemoteScreenHandler !==
            'function'
    ) {

        console.error(
            'ServiceCall remote screen handler is unavailable.'
        );

        return;
    }


    window.ServiceCallAgora
        .setRemoteScreenHandler(

            async (event) => {

                if (!event) {
                    return;
                }


                /*
                 * ---------------------------------
                 * REMOTE SCREEN STARTED
                 * ---------------------------------
                 */
                if (
                    event.action ===
                    'started' &&
                    event.track
                ) {

                    activeRemoteScreenUid =
                        String(
                            event.uid
                        );

/*
     * If I am recording and another
     * participant starts sharing,
     * include their screen in my recording.
     */
    if (
        window.ServiceCallRecorder &&
        window.ServiceCallRecorder
            .isRecording()
    ) {

        await window.ServiceCallRecorder
            .attachScreen(
                event.track
            );
    }


                    await showScreenShareView(
                        'Participant is sharing their screen'
                    );


                    screenShareVideo.innerHTML =
                        '';


                    screenSharePlaceholder
                        .classList
                        .add(
                            'hidden'
                        );


                    try {

                        event.track.play(
                            screenShareVideo
                        );


                        console.log(
                            'ServiceCall remote screen displayed:',
                            event.uid
                        );


                    } catch (error) {

                        console.error(
                            'Unable to display remote ServiceCall screen:',
                            error
                        );


                        screenSharePlaceholder.textContent =
                            'Unable to display shared screen.';

                        screenSharePlaceholder
                            .classList
                            .remove(
                                'hidden'
                            );
                    }


                    return;
                }


                /*
                 * ---------------------------------
                 * REMOTE SCREEN STOPPED
                 * ---------------------------------
                 */
                if (
                    event.action ===
                    'stopped'
                ) {

                    const stoppedUid =
                        String(
                            event.uid ||
                            ''
                        );


                    /*
                     * Ignore stop events for some
                     * other remote video track.
                     */
                    if (
                        activeRemoteScreenUid &&
                        stoppedUid !==
                            activeRemoteScreenUid
                    ) {
                        return;
                    }


                    activeRemoteScreenUid =
                        null;

                    /*
 * The remote participant stopped
 * sharing their screen.
 *
 * Recording itself continues.
 * It still remembers that a screen
 * was used, so final output remains MP4.
 */
if (
    window.ServiceCallRecorder &&
    window.ServiceCallRecorder
        .isRecording()
) {

    window.ServiceCallRecorder
        .detachScreen();
}


                    screenShareVideo.innerHTML =
                        '';


                    /*
                     * If THIS user is sharing,
                     * their local preview should
                     * remain visible.
                     */
                    if (
                        window.ServiceCallAgora &&
                        window.ServiceCallAgora
                            .isScreenSharing() &&
                        localScreenAgoraTrack
                    ) {

                        screenShareTitle.textContent =
                            'You are sharing your screen';


                        screenSharePlaceholder
                            .classList
                            .add(
                                'hidden'
                            );


                        try {

                            localScreenAgoraTrack.play(
                                screenShareVideo
                            );

                        } catch (error) {

                            console.error(
                                'Unable to restore local screen preview:',
                                error
                            );
                        }


                        return;
                    }


                    await hideScreenShareView();


                    console.log(
                        'ServiceCall remote screen sharing stopped.'
                    );
                }
            }
        );
}


/* -------------------------------------------------------
   SHARE SCREEN BUTTON
------------------------------------------------------- */

shareScreenButton.addEventListener(
    'click',

    async () => {

        if (
            currentMode !==
            'connected'
        ) {
            return;
        }


        try {

            if (
                window.ServiceCallAgora &&
                window.ServiceCallAgora
                    .isScreenSharing()
            ) {

                await stopServiceCallScreenShare();

            } else {

                await openScreenSourceModal();
            }


        } catch (error) {

            console.error(
                'ServiceCall screen sharing action failed:',
                error
            );


            statusText.textContent =
                error.message ||
                'Unable to change screen sharing.';
        }
    }
);


/* -------------------------------------------------------
   SCREEN MODAL EVENTS
------------------------------------------------------- */

closeScreenSourceModalButton
    .addEventListener(
        'click',
        closeScreenSourceModal
    );


screenSourceModal.addEventListener(
    'click',

    (event) => {

        if (
            event.target ===
            screenSourceModal
        ) {

            closeScreenSourceModal();
        }
    }
);


/*
 * We already have an Escape handler for the
 * participant modal. This separate listener is
 * safe and handles only the screen picker.
 */
document.addEventListener(
    'keydown',

    (event) => {

        if (
            event.key ===
            'Escape' &&
            !screenSourceModal
                .classList
                .contains(
                    'hidden'
                )
        ) {

            closeScreenSourceModal();
        }
    }
);


/* -------------------------------------------------------
   INITIALIZE REMOTE SCREEN HANDLER
------------------------------------------------------- */

configureRemoteScreenSharing();

/* -------------------------
   SERVICECALL RECORDING
------------------------- */

const recordButton =
    document.getElementById(
        'recordButton'
    );

const recordingText =
    document.getElementById(
        'recordingText'
    );


let recordingStartedAt =
    null;

let serviceNowRecordingSysId =
    null;

let recordingActionInProgress =
    false;

let remoteRecordingActive = false;


/* -------------------------
   SHOW RECORDING MESSAGE
------------------------- */

function showRecordingMessage(
    message,
    autoHide = false
) {

    recordingText.textContent =
        message;


    recordingText
        .classList
        .remove(
            'hidden'
        );


    if (autoHide) {

        setTimeout(
            () => {

                /*
                 * Do not hide the message if
                 * another recording has started
                 * during the timeout.
                 */
                if (
                    !window.ServiceCallRecorder ||
                    !window.ServiceCallRecorder
                        .isRecording()
                ) {

                    recordingText
                        .classList
                        .add(
                            'hidden'
                        );
                }

            },
            3000
        );
    }
}

/* -------------------------
   SYNC RECORDING INDICATOR
------------------------- */

function syncRecordingIndicator(
    recordingActive
) {

    remoteRecordingActive =
        recordingActive === true;


    /*
     * Never overwrite local recording
     * workflow messages such as:
     *
     * Starting recording...
     * Processing recording...
     * Saving recording...
     */
    if (recordingActionInProgress) {
        return;
    }


    /*
     * This desktop is itself actively
     * recording the call.
     *
     * Its local state is authoritative.
     */
    if (
        window.ServiceCallRecorder &&
        window.ServiceCallRecorder
            .isRecording()
    ) {

        showRecordingMessage(
            'Call is being recorded'
        );

        return;
    }


    /*
     * Another participant is recording,
     * or this participant joined after
     * recording had already started.
     */
    if (remoteRecordingActive) {

        showRecordingMessage(
            'Call is being recorded'
        );

        return;
    }


    /*
     * No active recording exists.
     */
    recordingText
        .classList
        .add(
            'hidden'
        );
}


/* -------------------------
   START RECORDING
------------------------- */

async function startServiceCallRecording() {

    if (recordingActionInProgress) {
        return;
    }


    if (
        currentMode !==
        'connected'
    ) {

        throw new Error(
            'The call must be connected before recording.'
        );
    }


    if (
        !window.ServiceCallRecorder
    ) {

        throw new Error(
            'ServiceCall recording service is unavailable.'
        );
    }


    if (
        !window.ServiceCallAgora ||
        !window.ServiceCallAgora
            .isJoined()
    ) {

        throw new Error(
            'Call audio is not connected yet.'
        );
    }


    if (
        !window.serviceCall ||
        typeof window.serviceCall
            .startRecording !==
            'function'
    ) {

        throw new Error(
            'ServiceCall recording API is unavailable.'
        );
    }


    recordingActionInProgress =
        true;


    recordButton.disabled =
        true;


    showRecordingMessage(
        'Starting recording...'
    );


    try {

        /*
         * -----------------------------------------
         * 1. CREATE SERVICENOW RECORDING SESSION
         * -----------------------------------------
         */

        const serviceNowResult =
            await window.serviceCall
                .startRecording(
                    callSysId
                );


        if (
            !serviceNowResult ||
            !serviceNowResult.success
        ) {

            throw new Error(
                serviceNowResult &&
                serviceNowResult.message
                    ? serviceNowResult.message
                    : 'Unable to create ServiceCall recording.'
            );
        }


        const recordingSysId =
            serviceNowResult
                .recording_sys_id;


        if (!recordingSysId) {

            throw new Error(
                'ServiceNow did not return a recording ID.'
            );
        }


        serviceNowRecordingSysId =
            recordingSysId;


        /*
         * -----------------------------------------
         * 2. START LOCAL MIXED AUDIO CAPTURE
         * -----------------------------------------
         */

        const localResult =
            await window.ServiceCallRecorder
                .start();


        if (
            !localResult ||
            !localResult.success
        ) {

            throw new Error(
                localResult &&
                localResult.message
                    ? localResult.message
                    : 'Unable to start local recording.'
            );
        }


        recordingStartedAt =
            Date.now();


        recordButton.textContent =
            'Stop Recording';


        showRecordingMessage(
            'Call is being recorded'
        );


        console.log(
            'ServiceCall recording started.',
            'Recording:',
            serviceNowRecordingSysId,
            'Capture format:',
            localResult.mimeType
        );


    } catch (error) {

        /*
         * If ServiceNow successfully created the
         * recording record but local capture could
         * not start, keep the sys_id for diagnostics.
         *
         * Later we can add a dedicated failed-state
         * endpoint. Do not falsely mark it Available.
         */

        console.error(
            'Unable to start ServiceCall recording:',
            error
        );


        recordButton.textContent =
            'Record Call';


        showRecordingMessage(
            error.message ||
            'Unable to start recording.',
            true
        );


        throw error;


    } finally {

        recordingActionInProgress =
            false;

        recordButton.disabled =
            false;
    }
}


/* -------------------------
   STOP + FINALIZE RECORDING
------------------------- */

async function stopServiceCallRecording() {

    if (recordingActionInProgress) {
        return;
    }


    if (
        !window.ServiceCallRecorder ||
        !window.ServiceCallRecorder
            .isRecording()
    ) {

        return;
    }


    if (!serviceNowRecordingSysId) {

        throw new Error(
            'ServiceNow recording ID is unavailable.'
        );
    }


    if (
        !window.serviceCall ||
        typeof window.serviceCall
            .finalizeVoiceRecording !==
            'function'
    ) {

        throw new Error(
            'ServiceCall recording finalization API is unavailable.'
        );
    }


    recordingActionInProgress =
        true;


    recordButton.disabled =
        true;


    showRecordingMessage(
        'Processing recording...'
    );


    try {

        /*
         * -----------------------------------------
         * 1. STOP LOCAL MEDIARECORDER
         * -----------------------------------------
         */

        const localResult =
            await window.ServiceCallRecorder
                .stop();


        if (
            !localResult ||
            !localResult.success ||
            !localResult.blob
        ) {

            throw new Error(
                localResult &&
                localResult.message
                    ? localResult.message
                    : 'Unable to stop local recording.'
            );
        }


        const durationSeconds =
            recordingStartedAt
                ? Math.max(
                    1,
                    Math.round(
                        (
                            Date.now() -
                            recordingStartedAt
                        ) / 1000
                    )
                )
                : 0;


        console.log(
            'ServiceCall local recording stopped.',
            'Duration:',
            durationSeconds,
            'seconds',
            'WebM size:',
            localResult.size
        );


        /*
         * -----------------------------------------
         * 2. BLOB -> UINT8ARRAY
         * -----------------------------------------
         *
         * We do not expose OAuth credentials here.
         *
         * Only recording bytes cross the preload
         * IPC bridge into Electron's main process.
         */

        const arrayBuffer =
            await localResult.blob
                .arrayBuffer();


        const webmData =
            new Uint8Array(
                arrayBuffer
            );


        if (
            webmData.byteLength <= 0
        ) {

            throw new Error(
                'The captured recording is empty.'
            );
        }


        /*
         * -----------------------------------------
         * 3. ELECTRON MAIN PROCESS
         * -----------------------------------------
         *
         * main.js performs:
         *
         * WebM
         *   -> FFmpeg
         *   -> real MP3
         *   -> /finish-recording
         *   -> Attachment API
         *   -> /complete-recording
         */

        showRecordingMessage(
            'Saving recording...'
        );


        /*
 * Decide the final format from what actually
 * happened during this recording.
 *
 * No screen share:
 *     WebM audio -> MP3
 *
 * Screen share occurred at any point:
 *     WebM audio/video -> MP4
 */
const hadScreenShare =
    localResult.hadScreenShare === true;


console.log(
    'ServiceCall final recording type:',
    hadScreenShare
        ? 'MP4 - screen sharing occurred'
        : 'MP3 - audio only'
);


let finalResult;


if (hadScreenShare) {

    /*
     * Screen sharing occurred at least once.
     *
     * Even if sharing stopped before the
     * recording stopped, the final recording
     * remains MP4.
     */
    if (
        !window.serviceCall ||
        typeof window.serviceCall
            .finalizeScreenRecording !==
            'function'
    ) {

        throw new Error(
            'ServiceCall screen recording finalization is unavailable.'
        );
    }


    finalResult =
        await window.serviceCall
            .finalizeScreenRecording(
                serviceNowRecordingSysId,
                webmData
            );

} else {

    /*
     * Voice-only recording.
     */
    finalResult =
        await window.serviceCall
            .finalizeVoiceRecording(
                serviceNowRecordingSysId,
                webmData
            );
}


        if (
            !finalResult ||
            !finalResult.success
        ) {

            throw new Error(
                finalResult &&
                finalResult.message
                    ? finalResult.message
                    : 'Unable to save ServiceCall recording.'
            );
        }


        console.log(
            'ServiceCall recording available.',
            'Recording:',
            finalResult.recording_sys_id,
            'Attachment:',
            finalResult.attachment_sys_id,
            'Format:',
            finalResult.format,
            'Size:',
            finalResult.file_size,
            'Expires:',
            finalResult.expires_at
        );


        /*
         * -----------------------------------------
         * 4. SUCCESS
         * -----------------------------------------
         */

        serviceNowRecordingSysId =
            null;

        recordingStartedAt =
            null;


        recordButton.textContent =
            'Record Call';


        showRecordingMessage(
            'Recording saved',
            true
        );


    } catch (error) {

        console.error(
            'ServiceCall recording finalization failed:',
            error
        );


        /*
         * Local recording has already stopped,
         * therefore reset the button.
         *
         * We intentionally do NOT pretend the
         * ServiceNow recording is Available.
         */

        recordButton.textContent =
            'Record Call';


        recordingStartedAt =
            null;


        showRecordingMessage(
            error.message ||
            'Unable to save recording.'
        );


        throw error;


    } finally {

        recordingActionInProgress =
            false;

        recordButton.disabled =
            false;
    }
}


/* -------------------------
   RECORD BUTTON
------------------------- */

recordButton.addEventListener(
    'click',

    async () => {

        if (recordingActionInProgress) {
            return;
        }


        try {

            if (
                window.ServiceCallRecorder &&
                window.ServiceCallRecorder
                    .isRecording()
            ) {

                await stopServiceCallRecording();

            } else {

                await startServiceCallRecording();
            }


        } catch (error) {

            console.error(
                'ServiceCall recording action failed:',
                error
            );
        }
    }
);

/* -------------------------
   END CONFERENCE / LEAVE CALL
------------------------- */

document
    .getElementById(
        'endButton'
    )
    .addEventListener(
        'click',
        async () => {

            const endButton =
                document.getElementById(
                    'endButton'
                );


            if (recordingActionInProgress) {

                statusText.textContent =
                    'Please wait for the recording to finish processing.';

                return;
            }


            endButton.disabled =
                true;


            try {

                /*
                 * -----------------------------------------
                 * 1. FINALIZE ACTIVE RECORDING FIRST
                 * -----------------------------------------
                 *
                 * Never terminate the call/media before
                 * the active recording has been stopped,
                 * converted, uploaded and completed.
                 */
                if (
                    window.ServiceCallRecorder &&
                    window.ServiceCallRecorder
                        .isRecording()
                ) {

                    statusText.textContent =
                        'Saving recording before ending call...';


                    console.log(
                        'ServiceCall call ending while recording is active. Finalizing recording first.'
                    );


                    await stopServiceCallRecording();


                    console.log(
                        'ServiceCall recording finalized before call termination.'
                    );
                }


                let result;


                /* -------------------------
                   2. OWNER
                ------------------------- */

                if (currentUserIsOwner) {

                    statusText.textContent =
                        'Ending conference...';


                    result =
                        await window
                            .serviceCall
                            .endCall(
                                callSysId
                            );


                    if (
                        !result ||
                        !result.success
                    ) {

                        throw new Error(
                            result &&
                            result.message
                                ? result.message
                                : 'Unable to end conference.'
                        );
                    }


                    /*
                     * ServiceNow has successfully
                     * ended the conference.
                     *
                     * We can now leave Agora.
                     */
                    await stopAgoraAudio();


                    setMode(
                        'completed'
                    );


                    return;
                }


                /* -------------------------
                   3. PARTICIPANT
                ------------------------- */

                statusText.textContent =
                    'Leaving call...';


                result =
                    await window
                        .serviceCall
                        .leaveCall(
                            callSysId
                        );


                if (
                    !result ||
                    !result.success
                ) {

                    throw new Error(
                        result &&
                        result.message
                            ? result.message
                            : 'Unable to leave call.'
                    );
                }


                /*
                 * Leave Agora only on THIS desktop.
                 *
                 * Other conference participants
                 * remain in the same Agora channel.
                 */
                await stopAgoraAudio();


                stopAllTimers();

                stopRingtone();


                statusText.textContent =
                    'You left the call';


                closeCallWindowAfterDelay();


            } catch (error) {

                console.error(
                    currentUserIsOwner
                        ? 'End conference failed:'
                        : 'Leave call failed:',
                    error
                );


                /*
                 * IMPORTANT:
                 *
                 * If recording finalization failed,
                 * we intentionally do NOT continue
                 * with End Conference / Leave Call.
                 *
                 * This gives the user a chance to
                 * retry rather than knowingly
                 * discarding the recording.
                 */
                statusText.textContent =
                    error.message ||
                    (
                        currentUserIsOwner
                            ? 'Unable to end conference.'
                            : 'Unable to leave call.'
                    );


                endButton.disabled =
                    false;
            }
        }
    );

/* -------------------------
   CLOSE CALL WINDOW
------------------------- */
 
function closeCallWindowAfterDelay() {
 
    if (closeTimer) {
        return;
    }
 
 
    closeTimer =
        setTimeout(
            () => {
 
                stopAllTimers();
 
                window.close();
 
            },
            1000
        );
}

/* -------------------------
   STOP TIMERS
------------------------- */

function stopAllTimers() {

    if (callStatusTimer) {

        clearInterval(
            callStatusTimer
        );

        callStatusTimer =
            null;
    }


    if (durationTimer) {

        clearInterval(
            durationTimer
        );

        durationTimer =
            null;
    }
}


/* -------------------------
   START
------------------------- */

setMode(
    currentMode
);

startCallStatusPolling();


window.addEventListener(
    'beforeunload',
    () => {

        stopAllTimers();

        stopRingtone();

        /*
         * Final media safety cleanup.
         *
         * Do not await here because the window
         * is already being destroyed.
         */
        stopAgoraAudio();
    }
);
