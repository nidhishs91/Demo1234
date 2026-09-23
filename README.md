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
     * For V1 we display one popup at a time.
     *
     * If another notification arrives while one
     * is visible, close the old popup first.
     *
     * We can add stacking later.
     */
    if (
        notificationPopupWindow &&
        !notificationPopupWindow.isDestroyed()
    ) {

        notificationPopupWindow.destroy();

        notificationPopupWindow =
            null;
    }


    const {
        screen
    } = require(
        'electron'
    );


    const display =
        screen.getPrimaryDisplay();


    const workArea =
        display.workArea;


    const popupWidth =
        380;

    const popupHeight =
        150;

    const margin =
        18;


    const popupX =
        Math.round(
            workArea.x +
            workArea.width -
            popupWidth -
            margin
        );


    const popupY =
        Math.round(
            workArea.y +
            workArea.height -
            popupHeight -
            margin
        );


    notificationPopupWindow =
        new BrowserWindow({

            width:
                popupWidth,

            height:
                popupHeight,

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


    notificationPopupWindow.loadFile(
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


    notificationPopupWindow.once(
        'ready-to-show',
        () => {

            if (
                !notificationPopupWindow ||
                notificationPopupWindow.isDestroyed()
            ) {
                return;
            }


            /*
             * Show without stealing keyboard focus
             * from whatever the user is doing.
             */
            notificationPopupWindow.showInactive();
        }
    );


    notificationPopupWindow.on(
        'closed',
        () => {

            notificationPopupWindow =
                null;
        }
    );
}
