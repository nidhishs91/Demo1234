<!DOCTYPE html>
<html>

<head>
    <meta charset="UTF-8">

    <title>ServiceCall</title>

    <style>
        body {
            margin: 0;
            font-family: Arial, sans-serif;
            background: #f4f8f7;
            color: #1f2d2a;
        }

        .topbar {
            background: #0b4f46;
            color: white;
            padding: 16px 20px;
            font-size: 18px;
            font-weight: bold;
        }

        .container {
            padding: 36px 28px;
            text-align: center;
        }

        .avatar {
            width: 92px;
            height: 92px;
            border-radius: 50%;
            background: #dcefeb;
            margin: 0 auto 18px auto;

            display: flex;
            align-items: center;
            justify-content: center;

            font-size: 30px;
            font-weight: bold;
            color: #0b4f46;
        }

        .name {
            font-size: 24px;
            font-weight: bold;
            margin-bottom: 7px;
        }

        .department {
            font-size: 14px;
            color: #65736f;
            margin-bottom: 18px;
        }

        .status {
            font-size: 17px;
            margin-bottom: 8px;
        }

        .call-number {
            font-size: 13px;
            color: #7b8582;
            margin-bottom: 30px;
        }

        .timer {
            font-size: 14px;
            margin-bottom: 18px;
            color: #4e5d59;
        }

        .button-row {
            display: flex;
            justify-content: center;
            gap: 14px;
            flex-wrap: wrap;
        }

        button {
            border: none;
            border-radius: 8px;
            padding: 13px 22px;
            cursor: pointer;
            font-size: 14px;
        }

        .accept {
            background: #16856d;
            color: white;
        }

        .decline,
        .end,
        .cancel {
            background: #d64545;
            color: white;
        }

        .secondary {
            background: #e8efed;
            color: #1f2d2a;
        }

        .hidden {
            display: none;
        }

        .recording-text {
            margin-top: 18px;
            font-size: 13px;
            color: #b13b3b;
        }
    </style>
</head>

<body>

    <div class="topbar">
        ServiceCall
    </div>

    <div class="container">

        <div class="avatar" id="avatar">
        </div>

        <div class="name" id="personName">
        </div>

        <div class="department" id="department">
        </div>

        <div class="status" id="statusText">
        </div>

        <div class="timer hidden" id="timer">
            00:00
        </div>

        <div class="call-number" id="callNumber">
        </div>


        <!-- INCOMING -->

        <div class="button-row hidden" id="incomingActions">

            <button class="decline" id="declineButton">
                Decline
            </button>

            <button class="accept" id="acceptButton">
                Accept
            </button>

        </div>


        <!-- CALLING -->

        <div class="button-row hidden" id="callingActions">

            <button class="cancel" id="cancelButton">
                Cancel Call
            </button>

        </div>


        <!-- CONNECTED -->

        <div class="button-row hidden" id="connectedActions">

            <button class="secondary" id="muteButton">
                Mute
            </button>

            <button class="secondary" id="addUserButton">
                Add User
            </button>

            <button class="secondary" id="shareScreenButton">
                Share Screen
            </button>

            <button class="secondary" id="recordButton">
                Record Call
            </button>

            <button class="end" id="endButton">
                End Call
            </button>

        </div>


        <div class="recording-text hidden" id="recordingText">
            Call is being recorded
        </div>

    </div>


    <script src="call-window.js">
    </script>

</body>

</html>
