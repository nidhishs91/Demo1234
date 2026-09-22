<!DOCTYPE html>
<html>

<head>

    <meta charset="UTF-8">

    <meta http-equiv="Content-Security-Policy" content="
        default-src 'self' data: blob:;
        script-src 'self';
        style-src 'self' 'unsafe-inline';
        img-src 'self' data: blob:;
        media-src 'self' blob:;
        connect-src 'self' https: wss:;
    ">

    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>
        ServiceCall
    </title>


    <style>
        * {
            box-sizing: border-box;
        }


        body {
            margin: 0;

            font-family:
                Arial,
                Helvetica,
                sans-serif;

            background:
                #f4f7f6;

            color:
                #1f2d2a;

            min-height:
                100vh;
        }


        button {
            font-family:
                inherit;
        }


        .hidden {
            display:
                none !important;
        }


        /* --------------------------------
           TOP BAR
        -------------------------------- */

        .topbar {
            height:
                54px;

            background:
                #173a33;

            color:
                white;

            display:
                flex;

            align-items:
                center;

            padding:
                0 18px;

            box-shadow:
                0 2px 8px rgba(0, 0, 0, 0.12);
        }


        .brand {
            display:
                flex;

            align-items:
                center;

            gap:
                10px;

            font-size:
                16px;

            font-weight:
                bold;
        }


        .brand-logo {
            width:
                26px;

            height:
                26px;

            border-radius:
                8px;

            display:
                flex;

            align-items:
                center;

            justify-content:
                center;

            background:
                #79d9c3;

            color:
                #173a33;

            font-size:
                13px;

            font-weight:
                bold;
        }


        /* --------------------------------
           MAIN CONTAINER
        -------------------------------- */

        .container {
            width:
                100%;

            max-width:
                980px;

            margin:
                0 auto;

            padding:
                28px 22px 32px 22px;

            text-align:
                center;
        }


        /* --------------------------------
           USER / CALL DETAILS
        -------------------------------- */

        .avatar {
            width:
                92px;

            height:
                92px;

            margin:
                4px auto 18px auto;

            border-radius:
                50%;

            display:
                flex;

            align-items:
                center;

            justify-content:
                center;

            background:
                #dcefeb;

            color:
                #0b4f46;

            font-size:
                31px;

            font-weight:
                bold;

            box-shadow:
                0 5px 18px rgba(0, 0, 0, 0.08);
        }


        .person-name {
            font-size:
                24px;

            font-weight:
                bold;

            margin-bottom:
                6px;

            word-break:
                break-word;
        }


        .department {
            min-height:
                18px;

            font-size:
                14px;

            color:
                #687772;

            margin-bottom:
                14px;
        }


        .status {
            min-height:
                22px;

            margin-bottom:
                6px;

            font-size:
                15px;

            color:
                #40514d;
        }


        .timer {
            min-height:
                24px;

            font-size:
                18px;

            font-weight:
                bold;

            color:
                #0b6b58;

            margin-bottom:
                8px;
        }


        .call-number {
            min-height:
                18px;

            margin-bottom:
                22px;

            font-size:
                12px;

            color:
                #8a9793;
        }


        /* --------------------------------
           SCREEN SHARING VIEW
        -------------------------------- */

        .screen-share-container {
            width:
                100%;

            max-width:
                900px;

            margin:
                0 auto 24px auto;

            background:
                #101817;

            border-radius:
                12px;

            overflow:
                hidden;

            box-shadow:
                0 8px 28px rgba(0, 0, 0, 0.18);
        }


        .screen-share-header {
            display:
                flex;

            align-items:
                center;

            justify-content:
                space-between;

            gap:
                12px;

            padding:
                10px 14px;

            background:
                #173a33;

            color:
                white;

            font-size:
                13px;
        }


        .screen-share-title {
            font-weight:
                bold;

            overflow:
                hidden;

            text-overflow:
                ellipsis;

            white-space:
                nowrap;
        }


        .screen-share-view {
            position:
                relative;

            width:
                100%;

            aspect-ratio:
                16 / 9;

            background:
                #101817;
        }


        .screen-share-video {
            position:
                absolute;

            inset:
                0;

            width:
                100%;

            height:
                100%;

            overflow:
                hidden;
        }


        .screen-share-video video {
            width:
                100% !important;

            height:
                100% !important;

            object-fit:
                contain !important;
        }


        .screen-share-placeholder {
            position:
                absolute;

            inset:
                0;

            display:
                flex;

            align-items:
                center;

            justify-content:
                center;

            padding:
                20px;

            color:
                #d5dfdc;

            font-size:
                14px;

            text-align:
                center;

            pointer-events:
                none;
        }


        .screen-sharing-active {
            background:
                #dcefeb !important;

            color:
                #0b4f46 !important;

            font-weight:
                bold;
        }


        /* --------------------------------
           ACTIONS
        -------------------------------- */

        .actions {
            display:
                flex;

            align-items:
                center;

            justify-content:
                center;

            flex-wrap:
                wrap;

            gap:
                12px;

            margin-top:
                18px;
        }


        .actions button {
            min-width:
                112px;

            min-height:
                42px;

            padding:
                10px 16px;

            border:
                none;

            border-radius:
                8px;

            cursor:
                pointer;

            font-size:
                14px;

            transition:
                transform 0.12s ease,
                box-shadow 0.12s ease,
                opacity 0.12s ease;
        }


        .actions button:hover {
            transform:
                translateY(-1px);

            box-shadow:
                0 4px 12px rgba(0, 0, 0, 0.10);
        }


        .actions button:disabled {
            cursor:
                not-allowed;

            opacity:
                0.55;

            transform:
                none;

            box-shadow:
                none;
        }


        .primary {
            background:
                #0b6b58;

            color:
                white;
        }


        .secondary {
            background:
                #e6eeec;

            color:
                #243631;
        }


        .danger {
            background:
                #c94343;

            color:
                white;
        }


        .accept {
            background:
                #16856d;

            color:
                white;
        }


        /* --------------------------------
           RECORDING
        -------------------------------- */

        .recording-text {
            margin-top:
                18px;

            min-height:
                18px;

            color:
                #b33838;

            font-size:
                13px;

            font-weight:
                bold;
        }


        /* --------------------------------
           LIVE PARTICIPANTS PANEL
        -------------------------------- */

        .participants-panel {
            position:
                fixed;

            top:
                54px;

            right:
                0;

            bottom:
                0;

            width:
                340px;

            max-width:
                92vw;

            z-index:
                900;

            display:
                flex;

            flex-direction:
                column;

            background:
                #ffffff;

            border-left:
                1px solid #dfe8e5;

            box-shadow:
                -8px 0 28px rgba(0, 0, 0, 0.12);

            text-align:
                left;
        }


        .participants-panel-header {
            display:
                flex;

            align-items:
                center;

            justify-content:
                space-between;

            gap:
                12px;

            padding:
                18px;

            border-bottom:
                1px solid #e5ecea;
        }


        .participants-panel-heading {
            display:
                flex;

            align-items:
                center;

            gap:
                7px;
        }


        .participants-panel-title {
            font-size:
                17px;

            font-weight:
                bold;

            color:
                #1f2d2a;
        }


        .participants-panel-count {
            display:
                inline-flex;

            align-items:
                center;

            justify-content:
                center;

            min-width:
                26px;

            height:
                26px;

            padding:
                0 8px;

            border-radius:
                13px;

            background:
                #e4f3ef;

            color:
                #0b6b58;

            font-size:
                12px;

            font-weight:
                bold;
        }


        .participants-panel-close {
            width:
                34px;

            height:
                34px;

            border:
                none;

            border-radius:
                8px;

            background:
                #eef3f2;

            color:
                #324540;

            cursor:
                pointer;

            font-size:
                22px;

            line-height:
                1;
        }


        .participants-panel-close:hover {
            background:
                #e1e9e7;
        }


        .participants-panel-body {
            flex:
                1;

            overflow-y:
                auto;

            padding:
                14px;
        }


        .participants-section-label {
            margin:
                3px 4px 10px 4px;

            color:
                #687772;

            font-size:
                11px;

            font-weight:
                bold;

            text-transform:
                uppercase;

            letter-spacing:
                0.5px;
        }


        .live-participant {
            display:
                flex;

            align-items:
                center;

            gap:
                11px;

            padding:
                11px 10px;

            margin-bottom:
                5px;

            border-radius:
                10px;

            transition:
                background 0.12s ease;
        }


        .live-participant:hover {
            background:
                #f4f8f7;
        }


        .live-participant-avatar {
            width:
                40px;

            height:
                40px;

            flex-shrink:
                0;

            display:
                flex;

            align-items:
                center;

            justify-content:
                center;

            border-radius:
                50%;

            background:
                #dcefeb;

            color:
                #0b4f46;

            font-size:
                14px;

            font-weight:
                bold;
        }


        .live-participant-info {
            flex:
                1;

            min-width:
                0;
        }


        .live-participant-name {
            overflow:
                hidden;

            color:
                #1f2d2a;

            font-size:
                14px;

            font-weight:
                600;

            text-overflow:
                ellipsis;

            white-space:
                nowrap;
        }


        .live-participant-meta {
            margin-top:
                3px;

            color:
                #71807c;

            font-size:
                12px;
        }


        .live-participant-status {
            display:
                flex;

            align-items:
                center;

            gap:
                6px;

            color:
                #53635f;

            font-size:
                11px;

            white-space:
                nowrap;
        }


        .live-participant-status-dot {
            width:
                8px;

            height:
                8px;

            flex-shrink:
                0;

            border-radius:
                50%;

            background:
                #16856d;
        }


        .participants-empty {
            padding:
                40px 18px;

            text-align:
                center;

            color:
                #71807c;

            font-size:
                13px;

            line-height:
                1.5;
        }


        /* --------------------------------
           MODAL BASE
        -------------------------------- */

        .modal-overlay {
            position:
                fixed;

            inset:
                0;

            z-index:
                1000;

            display:
                flex;

            align-items:
                center;

            justify-content:
                center;

            padding:
                22px;

            background:
                rgba(9, 22, 19, 0.58);
        }


        .modal-card {
            width:
                100%;

            max-width:
                520px;

            max-height:
                82vh;

            overflow:
                hidden;

            background:
                white;

            border-radius:
                14px;

            box-shadow:
                0 18px 50px rgba(0, 0, 0, 0.22);

            text-align:
                left;
        }


        .modal-header {
            display:
                flex;

            align-items:
                center;

            justify-content:
                space-between;

            gap:
                14px;

            padding:
                16px 18px;

            border-bottom:
                1px solid #e5ecea;
        }


        .modal-title {
            font-size:
                17px;

            font-weight:
                bold;

            color:
                #1f2d2a;
        }


        .modal-close {
            width:
                34px;

            height:
                34px;

            border:
                none;

            border-radius:
                8px;

            background:
                #eef3f2;

            color:
                #324540;

            cursor:
                pointer;

            font-size:
                23px;

            line-height:
                1;
        }


        .modal-body {
            padding:
                18px;

            overflow-y:
                auto;

            max-height:
                calc(82vh - 68px);
        }


        /* --------------------------------
           ADD PARTICIPANT
        -------------------------------- */

        .participant-help {
            margin-bottom:
                12px;

            color:
                #687772;

            font-size:
                13px;

            line-height:
                1.45;
        }


        .participant-search {
            width:
                100%;

            height:
                42px;

            padding:
                0 12px;

            border:
                1px solid #ccd8d5;

            border-radius:
                8px;

            outline:
                none;

            font-size:
                14px;

            margin-bottom:
                12px;
        }


        .participant-search:focus {
            border-color:
                #16856d;
        }


        .participant-results {
            display:
                flex;

            flex-direction:
                column;

            gap:
                8px;

            max-height:
                330px;

            overflow-y:
                auto;
        }


        .participant-result {
            display:
                flex;

            align-items:
                center;

            justify-content:
                space-between;

            gap:
                12px;

            padding:
                11px 12px;

            border:
                1px solid #e0e8e6;

            border-radius:
                8px;

            background:
                #fafcfc;
        }


        .participant-result button {
            border:
                none;

            border-radius:
                7px;

            background:
                #0b6b58;

            color:
                white;

            padding:
                8px 12px;

            cursor:
                pointer;
        }


        .participant-message {
            padding:
                18px 10px;

            text-align:
                center;

            color:
                #71807c;

            font-size:
                13px;
        }


        /* --------------------------------
           SCREEN SOURCE MODAL
        -------------------------------- */

        .screen-source-modal-card {
            width:
                100%;

            max-width:
                760px;

            max-height:
                82vh;

            background:
                white;

            border-radius:
                14px;

            box-shadow:
                0 18px 50px rgba(0, 0, 0, 0.22);

            overflow:
                hidden;

            text-align:
                left;
        }


        .screen-source-results {
            display:
                grid;

            grid-template-columns:
                repeat(auto-fit,
                    minmax(190px, 1fr));

            gap:
                14px;

            max-height:
                480px;

            overflow-y:
                auto;

            padding-top:
                4px;
        }


        .screen-source-item {
            border:
                1px solid #dbe5e2;

            border-radius:
                10px;

            background:
                white;

            overflow:
                hidden;

            cursor:
                pointer;

            transition:
                transform 0.15s ease,
                border-color 0.15s ease,
                box-shadow 0.15s ease;
        }


        .screen-source-item:hover {
            transform:
                translateY(-2px);

            border-color:
                #16856d;

            box-shadow:
                0 6px 18px rgba(0, 0, 0, 0.10);
        }


        .screen-source-thumbnail {
            width:
                100%;

            aspect-ratio:
                16 / 9;

            object-fit:
                cover;

            display:
                block;

            background:
                #101817;
        }


        .screen-source-name {
            padding:
                10px 11px;

            font-size:
                13px;

            color:
                #1f2d2a;

            white-space:
                nowrap;

            overflow:
                hidden;

            text-overflow:
                ellipsis;
        }


        .screen-source-message {
            grid-column:
                1 / -1;

            padding:
                25px;

            text-align:
                center;

            color:
                #65736f;

            font-size:
                13px;
        }


        /* --------------------------------
           SMALL WINDOW BEHAVIOUR
        -------------------------------- */

        @media (max-width: 520px) {

            .container {
                padding:
                    24px 14px 28px 14px;
            }


            .avatar {
                width:
                    82px;

                height:
                    82px;

                font-size:
                    28px;
            }


            .person-name {
                font-size:
                    21px;
            }


            .actions {
                gap:
                    9px;
            }


            .actions button {
                min-width:
                    105px;
            }


            .modal-overlay {
                padding:
                    12px;
            }


            .screen-source-results {
                grid-template-columns:
                    1fr;
            }


            .participants-panel {
                width:
                    100%;

                max-width:
                    100%;
            }
        }
    </style>

</head>


<body>


    <!-- --------------------------------
         TOP BAR
    --------------------------------- -->

    <div class="topbar">

        <div class="brand">

            <div class="brand-logo">
                now
            </div>

            <div>
                ServiceCall
            </div>

        </div>

    </div>



    <!-- --------------------------------
         MAIN CALL UI
    --------------------------------- -->

    <div class="container">


        <div class="avatar" id="avatar">
            ?
        </div>


        <div class="person-name" id="personName">
            Unknown User
        </div>


        <div class="department" id="department">
        </div>


        <div class="status" id="statusText">
            Connecting...
        </div>


        <div class="timer hidden" id="timer">
            00:00
        </div>


        <div class="call-number" id="callNumber">
        </div>



        <!-- --------------------------------
             SCREEN SHARE VIEW
        --------------------------------- -->

        <div class="screen-share-container hidden" id="screenShareContainer">

            <div class="screen-share-header">

                <div class="screen-share-title" id="screenShareTitle">
                    Screen sharing
                </div>

            </div>


            <div class="screen-share-view">

                <div class="screen-share-video" id="screenShareVideo">
                </div>


                <div class="screen-share-placeholder" id="screenSharePlaceholder">
                    Connecting to shared screen...
                </div>

            </div>

        </div>



        <!-- --------------------------------
             INCOMING CALL ACTIONS
        --------------------------------- -->

        <div class="actions hidden" id="incomingActions">

            <button class="danger" id="declineButton" type="button">
                Decline
            </button>


            <button class="accept" id="acceptButton" type="button">
                Accept
            </button>

        </div>



        <!-- --------------------------------
             OUTGOING / RINGING ACTIONS
        --------------------------------- -->

        <div class="actions hidden" id="callingActions">

            <button class="danger" id="cancelButton" type="button">
                Cancel
            </button>

        </div>



        <!-- --------------------------------
             CONNECTED CALL ACTIONS
        --------------------------------- -->

        <div class="actions hidden" id="connectedActions">

            <button class="secondary" id="muteButton" type="button">
                Mute
            </button>


            <button class="secondary" id="participantsButton" type="button">
                Participants
            </button>


            <button class="secondary" id="addUserButton" type="button">
                Add User
            </button>


            <button class="secondary" id="shareScreenButton" type="button">
                Share Screen
            </button>


            <button class="secondary" id="recordButton" type="button">
                Record Call
            </button>


            <button class="danger" id="endButton" type="button">
                End Call
            </button>

        </div>


        <div class="recording-text hidden" id="recordingText">
            Call is being recorded
        </div>


    </div>



    <!-- --------------------------------
         LIVE PARTICIPANTS PANEL
    --------------------------------- -->

    <div class="participants-panel hidden" id="participantsPanel">

        <div class="participants-panel-header">

            <div class="participants-panel-heading">

                <div class="participants-panel-title">
                    Participants
                </div>


                <span class="participants-panel-count" id="participantsPanelCount">
                    0
                </span>

            </div>


            <button class="participants-panel-close" id="closeParticipantsPanel" type="button"
                title="Close participants">
                &times;
            </button>

        </div>


        <div class="participants-panel-body" id="participantsPanelBody">

            <div class="participants-section-label">
                In this call
            </div>


            <div class="participants-empty">
                Loading participants...
            </div>

        </div>

    </div>



    <!-- --------------------------------
         ADD PARTICIPANT MODAL
    --------------------------------- -->

    <div class="modal-overlay hidden" id="addParticipantModal">

        <div class="modal-card">

            <div class="modal-header">

                <div class="modal-title">
                    Add participant
                </div>


                <button class="modal-close" id="closeParticipantModal" type="button">
                    &times;
                </button>

            </div>


            <div class="modal-body">

                <div class="participant-help">
                    Search for a ServiceCall user to invite to this call.
                </div>


                <input class="participant-search" id="participantSearch" type="text" autocomplete="off"
                    placeholder="Search users...">


                <div class="participant-results" id="participantResults">

                    <div class="participant-empty">
                        Start typing a user's name.
                    </div>

                </div>


                <div class="participant-message" id="participantMessage">
                </div>

            </div>

        </div>

    </div>



    <!-- --------------------------------
         SCREEN SOURCE MODAL
    --------------------------------- -->

    <div class="modal-overlay hidden" id="screenSourceModal">

        <div class="screen-source-modal-card">

            <div class="modal-header">

                <div class="modal-title">
                    Share your screen
                </div>


                <button class="modal-close" id="closeScreenSourceModal" type="button">
                    &times;
                </button>

            </div>


            <div class="modal-body">

                <div class="participant-help">
                    Choose a screen or window to share.
                </div>


                <div class="screen-source-results" id="screenSourceResults">

                    <div class="screen-source-message">
                        Loading screens and windows...
                    </div>

                </div>

            </div>

        </div>

    </div>



    <!-- --------------------------------
         SERVICECALL SCRIPTS
    --------------------------------- -->

    <script src="agora-service.bundle.js"></script>

    <script src="recording-service.bundle.js"></script>

    <script src="call-window.js"></script>


</body>

</html>
