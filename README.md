async function checkNotificationsOnce() {

    try {

        /*
         * Only fetch unread notifications.
         *
         * We are NOT showing popups yet.
         * First we prove background detection works.
         */

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


        /*
         * Support the notification array returned
         * by the ServiceCall notifications API.
         */

        const notifications =
            Array.isArray(result.notifications)
                ? result.notifications
                : [];


        if (
            notifications.length === 0
        ) {

            console.log(
                'ServiceCall notification check: no unread notifications.'
            );

            return;
        }


        console.log(
            'ServiceCall unread notifications detected:',
            notifications.length
        );


        /*
         * Detection only for now.
         *
         * DO NOT:
         * - show popup
         * - play sound
         * - mark as read
         *
         * We will add those after this test passes.
         */

        notifications.forEach(
    (notification) => {
 
        const notificationSysId =
            String(
                notification.sys_id || ''
            ).trim();
 
 
        /*
         * A notification without a sys_id cannot
         * be safely deduplicated.
         */
 
        if (!notificationSysId) {
 
            console.warn(
                'ServiceCall notification skipped because sys_id is missing:',
                notification
            );
 
            return;
        }
 
 
        /*
         * This notification has already been
         * surfaced during the current app session.
         */
 
        if (
            surfacedNotificationIds.has(
                notificationSysId
            )
        ) {
 
            return;
        }
 
 
        /*
         * Remember it immediately.
         *
         * This prevents the next polling cycle
         * from processing the same unread
         * notification again.
         */
 
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
                    notification.meeting_sys_id || ''
            }
        );

        showNotificationPopup(notification);
    }
);
 


    } catch (error) {

        console.error(
            'ServiceCall notification check failed:',
            error
        );


        /*
         * Authentication expiry is handled
         * consistently with our other monitors.
         */

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
