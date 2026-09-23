<!DOCTYPE html>

<html lang="en">



<head>



    <meta charset="UTF-8">



    <meta name="viewport" content="width=device-width, initial-scale=1.0">



    <title>ServiceCall Notification</title>



    <style>

        * {

            box-sizing: border-box;

        }



        html,

        body {

            width: 100%;

            height: 100%;



            margin: 0;

            padding: 0;



            overflow: hidden;



            background: transparent;



            font-family:

                -apple-system,

                BlinkMacSystemFont,

                "Segoe UI",

                Arial,

                sans-serif;

        }





        /* -----------------------------------------

           POPUP

        ----------------------------------------- */



        .notification-popup {



            position: absolute;



            left: 8px;

            right: 8px;

            bottom: 8px;



            min-height: 134px;



            padding:

                16px 18px;



            background:

                rgba(255,

                    255,

                    255,

                    0.98);



            border:

                1px solid rgba(15,

                    92,

                    78,

                    0.14);



            border-radius:

                14px;



            box-shadow:

                0 12px 34px rgba(0,

                    0,

                    0,

                    0.18);



            opacity: 0;



            transform:

                translateX(45px);



            animation:

                popupEnter 320ms cubic-bezier(0.22,

                    1,

                    0.36,

                    1) forwards;

        }





        @keyframes popupEnter {



            from {



                opacity: 0;



                transform:

                    translateX(45px);

            }



            to {



                opacity: 1;



                transform:

                    translateX(0);

            }

        }





        /* -----------------------------------------

           HEADER

        ----------------------------------------- */



        .notification-header {



            display: flex;



            align-items:

                flex-start;



            gap: 11px;

        }





        .notification-icon {



            width: 38px;

            height: 38px;



            flex:

                0 0 38px;



            display: flex;



            align-items: center;

            justify-content: center;



            border-radius:

                11px;



            background:

                rgba(11,

                    107,

                    88,

                    0.10);



            font-size:

                19px;

        }





        .notification-heading {



            flex: 1;



            min-width: 0;

        }





        .notification-app {



            margin-bottom:

                2px;



            color:

                #0b6b58;



            font-size:

                11px;



            font-weight:

                700;



            letter-spacing:

                0.5px;



            text-transform:

                uppercase;

        }





        .notification-title {



            color:

                #1d2926;



            font-size:

                15px;



            font-weight:

                700;



            line-height:

                1.25;



            white-space:

                nowrap;



            overflow:

                hidden;



            text-overflow:

                ellipsis;

        }





        /* -----------------------------------------

           CLOSE

        ----------------------------------------- */



        .notification-close {



            width: 28px;

            height: 28px;



            flex:

                0 0 28px;



            margin-top:

                -3px;



            border: 0;



            border-radius:

                50%;



            background:

                transparent;



            color:

                #67736f;



            font-size:

                20px;



            line-height:

                28px;



            cursor:

                pointer;



            transition:

                background 150ms ease,

                color 150ms ease;

        }





        .notification-close:hover {



            background:

                rgba(0,

                    0,

                    0,

                    0.06);



            color:

                #1d2926;

        }





        /* -----------------------------------------

           MESSAGE

        ----------------------------------------- */



        .notification-message {



            margin:

                10px 0 0 49px;



            color:

                #53605c;



            font-size:

                13px;



            line-height:

                1.4;



            display:

                -webkit-box;



            line-clamp:

                2;



            -webkit-line-clamp:

                2;



            -webkit-box-orient:

                vertical;



            overflow:

                hidden;

        }





        /* -----------------------------------------

           CLICKABLE

        ----------------------------------------- */



        .notification-popup {

            cursor: pointer;

        }





        .notification-popup.closing {



            animation:

                popupExit 220ms ease forwards;

        }





        @keyframes popupExit {



            from {



                opacity: 1;



                transform:

                    translateX(0);

            }



            to {



                opacity: 0;



                transform:

                    translateX(45px);

            }

        }

    </style>



</head>



<body>



    <div id="notificationPopup" class="notification-popup">



        <div class="notification-header">



            <div id="notificationIcon" class="notification-icon">

                🔔

            </div>





            <div class="notification-heading">



                <div class="notification-app">

                    ServiceCall

                </div>



                <div id="notificationTitle" class="notification-title">

                    Notification

                </div>



            </div>





            <button id="notificationClose" class="notification-close" type="button" aria-label="Dismiss notification">

                ×

            </button>



        </div>





        <div id="notificationMessage" class="notification-message">

        </div>



    </div>





    <script>



        const params =

            new URLSearchParams(

                window.location.search

            );





        const notificationSysId =

            params.get(

                'notificationSysId'

            ) || '';





        const type =

            (

                params.get(

                    'type'

                ) ||

                'Notification'

            )

                .toLowerCase();





        const title =

            params.get(

                'title'

            ) ||

            'ServiceCall';





        const message =

            params.get(

                'message'

            ) ||

            '';





        const popup =

            document.getElementById(

                'notificationPopup'

            );





        const icon =

            document.getElementById(

                'notificationIcon'

            );





        const titleElement =

            document.getElementById(

                'notificationTitle'

            );





        const messageElement =

            document.getElementById(

                'notificationMessage'

            );





        const closeButton =

            document.getElementById(

                'notificationClose'

            );





        /* -----------------------------------------

   NOTIFICATION SOUND

----------------------------------------- */



        const notificationSound =

            new Audio(

                './assets/sounds/notifications.mp3'

            );





        notificationSound.preload =

            'auto';





        notificationSound.volume =

            0.65;





        /*

         * Play the sound once when this

         * notification popup appears.

         */

        window.addEventListener(

            'DOMContentLoaded',

            async () => {



                try {



                    notificationSound.currentTime =

                        0;



                    await notificationSound.play();



                } catch (error) {



                    console.warn(

                        'ServiceCall notification sound could not be played:',

                        error

                    );

                }

            }

        );



        /* -----------------------------------------

           CONTENT

        ----------------------------------------- */



        titleElement.textContent =

            title;





        messageElement.textContent =

            message;





        /*

         * Basic icon selection.

         *

         * Later this same popup can support

         * calls, chats and system notifications.

         */

        if (

            type.includes(

                'meeting'

            )

        ) {



            icon.textContent =

                '📅';



        } else if (

            type.includes(

                'call'

            )

        ) {



            icon.textContent =

                '☎';



        } else if (

            type.includes(

                'chat'

            )

        ) {



            icon.textContent =

                '💬';



        } else {



            icon.textContent =

                '🔔';

        }





        /* -----------------------------------------

           DISMISS ANIMATION

        ----------------------------------------- */



        function dismissPopup() {



            if (

                popup.classList.contains(

                    'closing'

                )

            ) {

                return;

            }





            popup.classList.add(

                'closing'

            );





            /*

             * For this first visual test we

             * simply hide the popup content.

             *

             * Main-process window destruction

             * will be wired next.

             */

            setTimeout(

                () => {



                    if (

                        window.serviceCall &&

                        window.serviceCall.dismissNotificationPopup

                    ) {



                        window.serviceCall

                            .dismissNotificationPopup();

                    }



                },

                220

            );

        }



        /* -----------------------------------------

   AUTO DISMISS

----------------------------------------- */



        const autoDismissTimer =

            setTimeout(

                () => {



                    dismissPopup();



                },

                7000

            );



        /* -----------------------------------------

           X BUTTON

        ----------------------------------------- */



        closeButton.addEventListener(

            'click',

            event => {



                event.stopPropagation();



                clearTimeout(

                    autoDismissTimer

                );



                dismissPopup();

            }

        );





        /* -----------------------------------------

           POPUP CLICK

        ----------------------------------------- */



        popup.addEventListener(

            'click',

            () => {



                console.log(

                    'ServiceCall notification clicked:',

                    notificationSysId

                );



                /*

                 * Exact notification navigation

                 * comes after the visual test.

                 */

            }

        );



    </script>



</body>



</html>
