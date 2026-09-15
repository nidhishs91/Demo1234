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


let currentMode =
    mode;

let callStatusTimer =
    null;

let callStartedAt =
    null;

let durationTimer =
    null;


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
            'is calling you...';

        incomingActions
            .classList
            .remove('hidden');

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

        return;
    }


    if (
        newMode ===
        'connected'
    ) {

        statusText.textContent =
            'Connected';

        connectedActions
            .classList
            .remove('hidden');

        timerElement
            .classList
            .remove('hidden');

        startDurationTimer();

        return;
    }


    if (
        newMode ===
        'declined'
    ) {

        statusText.textContent =
            'Call declined';

        stopAllTimers();

        return;
    }


    if (
        newMode ===
        'cancelled'
    ) {

        statusText.textContent =
            'Call cancelled';

        stopAllTimers();

        return;
    }


    if (
        newMode ===
        'completed'
    ) {

        statusText.textContent =
            'Call ended';

        stopAllTimers();

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
   END
------------------------- */

document
    .getElementById(
        'endButton'
    )
    .addEventListener(
        'click',
        async () => {

            try {

                const result =
                    await window
                        .serviceCall
                        .endCall(
                            callSysId
                        );


                if (
                    result.success
                ) {

                    setMode(
                        'completed'
                    );
                }

            } catch (error) {

                console.error(
                    'End call failed:',
                    error
                );

                statusText.textContent =
                    error.message ||
                    'Unable to end call.';
            }
        }
    );


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
    stopAllTimers
);
